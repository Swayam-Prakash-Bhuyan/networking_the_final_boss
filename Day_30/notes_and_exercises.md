# Day 30: Mega Project — Cloud Networking Challenge

## Overview
Congratulations on reaching Day 30! This capstone project consolidates every concept from the curriculum into a single, production-grade cloud networking challenge. You will design, build, document, and troubleshoot a complete multi-tier, multi-AZ cloud network — the kind of infrastructure that runs real production systems.

**Estimated Time:** 6-8 hours (can be split across days)

---

## Project Brief

### Scenario
You are the Lead Cloud Network Engineer at **FinFlow**, a fintech startup that processes payments. The CTO has tasked you with designing and deploying a production-ready AWS network infrastructure that meets the following requirements:

### Requirements

**Functional:**
- Three-tier architecture: Web, Application, Database
- High availability across 3 Availability Zones
- Secure outbound internet access for app/DB tiers (software updates, API calls)
- Inbound traffic via HTTPS only
- Internal service communication using private IPs
- On-premises connectivity via Site-to-Site VPN (simulated)

**Security (PCI-DSS inspired):**
- No direct internet access to application or database tiers
- Web tier accessible only through a load balancer
- Database tier accessible only from application tier
- All traffic encrypted in transit
- VPC Flow Logs enabled
- Bastion host or SSM for admin access (no direct SSH from internet)

**Reliability:**
- No single point of failure
- NAT Gateway per AZ
- Multi-AZ load balancing
- Health checks on all services

**Observability:**
- VPC Flow Logs → CloudWatch Logs
- CloudWatch alarms for network anomalies
- Dashboard for key network metrics

**Automation:**
- All infrastructure defined as code (Terraform)
- Networking config documented in runbooks

---

## Architecture Design

### IP Address Plan

```
VPC CIDR: 10.0.0.0/16 (65,534 usable IPs)

┌──────────────────────────────────────────────────────────────────┐
│                    VPC: 10.0.0.0/16                             │
│                                                                  │
│  AZ-a (us-east-1a)   AZ-b (us-east-1b)   AZ-c (us-east-1c)    │
│  ─────────────────   ─────────────────   ─────────────────      │
│  Public (Web/ALB):   Public (Web/ALB):   Public (Web/ALB):      │
│  10.0.1.0/24         10.0.2.0/24         10.0.3.0/24           │
│                                                                  │
│  Private (App):      Private (App):      Private (App):         │
│  10.0.11.0/24        10.0.12.0/24        10.0.13.0/24          │
│                                                                  │
│  Private (DB):       Private (DB):       Private (DB):          │
│  10.0.21.0/24        10.0.22.0/24        10.0.23.0/24          │
│                                                                  │
│  Management:         (shared)            (shared)               │
│  10.0.100.0/24                                                  │
│                                                                  │
│  VPN/Transit:                                                    │
│  10.0.200.0/28  (for VPN simulation)                            │
└──────────────────────────────────────────────────────────────────┘

On-premises (simulated): 192.168.0.0/16
```

### Traffic Flow Diagram

```
Internet
    │ HTTPS 443
    ▼
[AWS WAF]
    │
    ▼
[Application Load Balancer] ← Public Subnets (10.0.1-3.0/24)
    │ HTTP 8080
    ▼
[Auto Scaling: Web/App Servers] ← Private App Subnets (10.0.11-13.0/24)
    │ PostgreSQL 5432
    ▼
[Aurora PostgreSQL Multi-AZ] ← Private DB Subnets (10.0.21-23.0/24)

Outbound Internet (Private Subnets):
App/DB → NAT Gateway (Public Subnet) → Internet Gateway → Internet

Admin Access:
Admin → AWS SSM Session Manager → EC2 (no SSH port needed)

On-premises (simulated):
192.168.0.0/16 ← VPN Tunnel → Virtual Private Gateway → App Subnet
```

---

## Part 1: Terraform Infrastructure Code

### Project Structure
```
finflow-network/
├── main.tf
├── variables.tf
├── outputs.tf
├── vpc.tf
├── subnets.tf
├── gateways.tf
├── routing.tf
├── security_groups.tf
├── flow_logs.tf
├── monitoring.tf
├── vpn.tf
└── terraform.tfvars
```

### vpc.tf
```hcl
resource "aws_vpc" "finflow" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.project}-${var.environment}-vpc"
  })
}

# IPv6 support (optional but recommended)
resource "aws_egress_only_internet_gateway" "finflow" {
  vpc_id = aws_vpc.finflow.id
  tags   = merge(var.common_tags, { Name = "${var.project}-eigw" })
}
```

### subnets.tf
```hcl
locals {
  azs = var.availability_zones
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = length(local.azs)
  vpc_id                  = aws_vpc.finflow.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = false  # Use ALB public IP, not instance

  tags = merge(var.common_tags, {
    Name = "${var.project}-public-${local.azs[count.index]}"
    Tier = "public"
    "kubernetes.io/role/elb" = "1"  # If EKS used later
  })
}

# Private App Subnets
resource "aws_subnet" "private_app" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.finflow.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 11)
  availability_zone = local.azs[count.index]

  tags = merge(var.common_tags, {
    Name = "${var.project}-private-app-${local.azs[count.index]}"
    Tier = "private-app"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

# Private DB Subnets
resource "aws_subnet" "private_db" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.finflow.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 21)
  availability_zone = local.azs[count.index]

  tags = merge(var.common_tags, {
    Name = "${var.project}-private-db-${local.azs[count.index]}"
    Tier = "private-db"
  })
}

# Management Subnet
resource "aws_subnet" "management" {
  vpc_id            = aws_vpc.finflow.id
  cidr_block        = "10.0.100.0/24"
  availability_zone = local.azs[0]

  tags = merge(var.common_tags, {
    Name = "${var.project}-management"
    Tier = "management"
  })
}
```

### gateways.tf
```hcl
# Internet Gateway
resource "aws_internet_gateway" "finflow" {
  vpc_id = aws_vpc.finflow.id
  tags   = merge(var.common_tags, { Name = "${var.project}-igw" })
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = length(local.azs)
  domain = "vpc"

  tags = merge(var.common_tags, {
    Name = "${var.project}-nat-eip-${local.azs[count.index]}"
  })

  depends_on = [aws_internet_gateway.finflow]
}

# NAT Gateways — one per AZ for HA
resource "aws_nat_gateway" "finflow" {
  count         = length(local.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(var.common_tags, {
    Name = "${var.project}-nat-${local.azs[count.index]}"
  })

  depends_on = [aws_internet_gateway.finflow]
}
```

### routing.tf
```hcl
# Public Route Table (shared across all public subnets)
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.finflow.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.finflow.id
  }

  tags = merge(var.common_tags, { Name = "${var.project}-public-rt" })
}

resource "aws_route_table_association" "public" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private App Route Tables — one per AZ (each routes to its own NAT GW)
resource "aws_route_table" "private_app" {
  count  = length(local.azs)
  vpc_id = aws_vpc.finflow.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.finflow[count.index].id
  }

  # Route to on-premises via VPN
  route {
    cidr_block = "192.168.0.0/16"
    gateway_id = aws_vpn_gateway.finflow.id
  }

  tags = merge(var.common_tags, {
    Name = "${var.project}-private-app-rt-${local.azs[count.index]}"
  })
}

resource "aws_route_table_association" "private_app" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app[count.index].id
}

# Private DB Route Tables — NO internet access (NAT-less)
resource "aws_route_table" "private_db" {
  count  = length(local.azs)
  vpc_id = aws_vpc.finflow.id
  # Only local route — no 0.0.0.0/0

  tags = merge(var.common_tags, {
    Name = "${var.project}-private-db-rt-${local.azs[count.index]}"
  })
}

resource "aws_route_table_association" "private_db" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.private_db[count.index].id
  route_table_id = aws_route_table.private_db[count.index].id
}

# S3 Gateway Endpoint — free, keeps S3 traffic off NAT GW
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.finflow.id
  service_name = "com.amazonaws.${var.region}.s3"
  route_table_ids = concat(
    aws_route_table.private_app[*].id,
    aws_route_table.private_db[*].id
  )
  tags = merge(var.common_tags, { Name = "${var.project}-s3-endpoint" })
}

# Secrets Manager Interface Endpoint — DB credentials, no internet needed
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = aws_vpc.finflow.id
  service_name        = "com.amazonaws.${var.region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.endpoint.id]
  private_dns_enabled = true

  tags = merge(var.common_tags, { Name = "${var.project}-secretsmanager-endpoint" })
}
```

### security_groups.tf
```hcl
# ALB Security Group — internet-facing
resource "aws_security_group" "alb" {
  name        = "${var.project}-alb-sg"
  description = "ALB - allows HTTPS from internet"
  vpc_id      = aws_vpc.finflow.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet"
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP redirect"
  }

  egress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    security_groups = [aws_security_group.app.id]
    description = "To app servers"
  }

  tags = merge(var.common_tags, { Name = "${var.project}-alb-sg" })
}

# App Server Security Group
resource "aws_security_group" "app" {
  name        = "${var.project}-app-sg"
  description = "App servers - accepts from ALB only"
  vpc_id      = aws_vpc.finflow.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "From ALB"
  }

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
    description     = "SSH from bastion (backup)"
  }

  egress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    security_groups = [aws_security_group.db.id]
    description = "To database"
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS to internet (via NAT) for APIs, updates"
  }

  egress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP to internet via NAT"
  }

  tags = merge(var.common_tags, { Name = "${var.project}-app-sg" })
}

# Database Security Group
resource "aws_security_group" "db" {
  name        = "${var.project}-db-sg"
  description = "Database - accepts from app only"
  vpc_id      = aws_vpc.finflow.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "PostgreSQL from app servers only"
  }

  # No egress rules = implicit deny all (DB should never initiate outbound)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.0.0/16"]
    description = "Internal VPC only"
  }

  tags = merge(var.common_tags, { Name = "${var.project}-db-sg" })
}

# Bastion Host Security Group
resource "aws_security_group" "bastion" {
  name        = "${var.project}-bastion-sg"
  description = "Bastion - SSH from corporate IP only"
  vpc_id      = aws_vpc.finflow.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.corporate_ip_ranges
    description = "SSH from corporate IPs only"
  }

  egress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "SSH to internal hosts"
  }

  tags = merge(var.common_tags, { Name = "${var.project}-bastion-sg" })
}

# VPC Endpoint Security Group
resource "aws_security_group" "endpoint" {
  name        = "${var.project}-endpoint-sg"
  description = "VPC endpoints"
  vpc_id      = aws_vpc.finflow.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "HTTPS from within VPC"
  }

  tags = merge(var.common_tags, { Name = "${var.project}-endpoint-sg" })
}
```

### flow_logs.tf
```hcl
# CloudWatch Log Group for Flow Logs
resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/${var.project}-${var.environment}/flowlogs"
  retention_in_days = 90
  kms_key_id        = aws_kms_key.logs.arn

  tags = var.common_tags
}

# IAM Role for Flow Logs
resource "aws_iam_role" "flow_logs" {
  name = "${var.project}-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "flow_logs" {
  role = aws_iam_role.flow_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup", "logs:CreateLogStream",
        "logs:PutLogEvents", "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Resource = "*"
    }]
  })
}

# Enable VPC Flow Logs
resource "aws_flow_log" "finflow" {
  vpc_id          = aws_vpc.finflow.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn

  tags = merge(var.common_tags, { Name = "${var.project}-flow-logs" })
}
```

### monitoring.tf
```hcl
# SNS Topic for network alerts
resource "aws_sns_topic" "network_alerts" {
  name = "${var.project}-network-alerts"
  tags = var.common_tags
}

# Alarm: NAT Gateway errors
resource "aws_cloudwatch_metric_alarm" "nat_error_port_allocation" {
  count               = length(local.azs)
  alarm_name          = "${var.project}-nat-port-alloc-error-${local.azs[count.index]}"
  alarm_description   = "NAT Gateway port allocation errors — may indicate SNAT exhaustion"
  namespace           = "AWS/NATGateway"
  metric_name         = "ErrorPortAllocation"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 3
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  alarm_actions       = [aws_sns_topic.network_alerts.arn]

  dimensions = {
    NatGatewayId = aws_nat_gateway.finflow[count.index].id
  }
}

# Alarm: ALB 5xx errors
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "${var.project}-alb-5xx-errors"
  alarm_description   = "ALB 5xx errors above threshold"
  namespace           = "AWS/ApplicationELB"
  metric_name         = "HTTPCode_ELB_5XX_Count"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 5
  threshold           = 10
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.network_alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.finflow.arn_suffix
  }
}

# Alarm: VPN Tunnel down
resource "aws_cloudwatch_metric_alarm" "vpn_tunnel_down" {
  alarm_name          = "${var.project}-vpn-tunnel-down"
  alarm_description   = "VPN tunnel state is DOWN"
  namespace           = "AWS/VPN"
  metric_name         = "TunnelState"
  statistic           = "Minimum"
  period              = 60
  evaluation_periods  = 2
  threshold           = 1
  comparison_operator = "LessThanThreshold"
  alarm_actions       = [aws_sns_topic.network_alerts.arn]

  dimensions = {
    VpnId = aws_vpn_connection.finflow.id
  }
}
```

### variables.tf
```hcl
variable "project"            { default = "finflow" }
variable "environment"        { default = "production" }
variable "region"             { default = "us-east-1" }
variable "vpc_cidr"           { default = "10.0.0.0/16" }
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
variable "corporate_ip_ranges" {
  default     = ["203.0.113.0/24"]
  description = "IP ranges allowed to SSH to bastion"
}
variable "on_premises_cidr"   { default = "192.168.0.0/16" }
variable "common_tags" {
  default = {
    Project     = "finflow"
    Environment = "production"
    ManagedBy   = "terraform"
    Owner       = "network-team"
  }
}
```

### outputs.tf
```hcl
output "vpc_id"                { value = aws_vpc.finflow.id }
output "vpc_cidr"              { value = aws_vpc.finflow.cidr_block }
output "public_subnet_ids"     { value = aws_subnet.public[*].id }
output "private_app_subnet_ids"{ value = aws_subnet.private_app[*].id }
output "private_db_subnet_ids" { value = aws_subnet.private_db[*].id }
output "nat_gateway_ips"       { value = aws_eip.nat[*].public_ip }
output "alb_sg_id"             { value = aws_security_group.alb.id }
output "app_sg_id"             { value = aws_security_group.app.id }
output "db_sg_id"              { value = aws_security_group.db.id }
output "flow_log_group"        { value = aws_cloudwatch_log_group.flow_logs.name }
output "vpn_connection_id"     { value = aws_vpn_connection.finflow.id }
```

---

## Part 2: Validation Checklist

After deploying, validate each requirement:

### Connectivity Validation Script
```bash
#!/bin/bash
# Run from a test EC2 instance in each subnet tier

SUBNET_TIER=${1:-"app"}  # web, app, or db

echo "=== FinFlow Network Validation ==="
echo "Tier: $SUBNET_TIER"
echo "Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)"
echo "AZ: $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)"
echo "IP: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)"
echo ""

# Test internet access
echo "[Internet Access]"
if curl -sf --max-time 5 https://checkip.amazonaws.com > /dev/null; then
    PUBLIC_IP=$(curl -s https://checkip.amazonaws.com)
    echo "  ✓ Internet reachable via NAT: $PUBLIC_IP"
else
    echo "  ✗ No internet access (expected for DB tier)"
fi

# Test DNS
echo -e "\n[DNS Resolution]"
for host in "google.com" "s3.amazonaws.com" "secretsmanager.us-east-1.amazonaws.com"; do
    if nslookup $host > /dev/null 2>&1; then
        IP=$(dig +short $host | head -1)
        echo "  ✓ $host → $IP"
    else
        echo "  ✗ $host — FAILED"
    fi
done

# Test S3 endpoint
echo -e "\n[S3 VPC Endpoint]"
if aws s3 ls > /dev/null 2>&1; then
    echo "  ✓ S3 accessible via VPC endpoint"
else
    echo "  ✗ S3 not accessible"
fi

# Test Secrets Manager endpoint
echo -e "\n[Secrets Manager VPC Endpoint]"
if aws secretsmanager list-secrets --region us-east-1 > /dev/null 2>&1; then
    echo "  ✓ Secrets Manager accessible via endpoint"
else
    echo "  ✗ Secrets Manager not accessible"
fi

# Test blocked ports
echo -e "\n[Blocked Access (security validation)]"
if [ "$SUBNET_TIER" = "db" ]; then
    if curl -sf --max-time 3 https://google.com > /dev/null; then
        echo "  ✗ SECURITY ISSUE: DB tier can reach internet!"
    else
        echo "  ✓ DB tier correctly blocked from internet"
    fi
fi

echo -e "\n=== Validation Complete ==="
```

### Security Validation
```bash
#!/bin/bash
# Validate security group rules are correct

echo "Security Group Audit"
echo "===================="

# Check ALB SG allows only 80 and 443 from internet
echo "ALB Security Group:"
aws ec2 describe-security-groups --group-ids $ALB_SG_ID \
  --query 'SecurityGroups[0].IpPermissions[*].[FromPort,ToPort,IpRanges[0].CidrIp]' \
  --output table 2>/dev/null

# Check DB SG has NO internet egress
echo -e "\nDB Security Group Egress:"
aws ec2 describe-security-groups --group-ids $DB_SG_ID \
  --query 'SecurityGroups[0].IpPermissionsEgress[*].[FromPort,ToPort,IpRanges[*].CidrIp]' \
  --output table 2>/dev/null

# Check VPC Flow Logs enabled
echo -e "\nVPC Flow Logs:"
aws ec2 describe-flow-logs \
  --filter Name=resource-id,Values=$VPC_ID \
  --query 'FlowLogs[*].[FlowLogId,TrafficType,LogDestination]' \
  --output table 2>/dev/null

# Check NAT Gateways are AVAILABLE
echo -e "\nNAT Gateways:"
aws ec2 describe-nat-gateways \
  --filter Name=vpc-id,Values=$VPC_ID \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId]' \
  --output table 2>/dev/null
```

---

## Part 3: Runbook — Common Operations

### Runbook 1: Adding a New Service
```
# When deploying a new microservice (e.g., "fraud-detection-service")

1. Determine the tier: Private-App
2. Identify required connections:
   - Inbound: port 9000 from App SG (other services)
   - Outbound: port 5432 to DB SG, port 443 to internet

3. Update security_groups.tf:
   Add ingress rule to fraud-sg from app-sg on port 9000
   Add egress rule from app-sg to fraud-sg on port 9000

4. PR → review → apply to staging → test → apply to production

5. Register in Cloud Map:
   aws servicediscovery register-instance \
     --service-id srv-xxx \
     --instance-id fraud-1 \
     --attributes AWS_INSTANCE_IPV4=10.0.11.50,AWS_INSTANCE_PORT=9000

6. Verify traffic flows:
   nc -zv fraud-detection-service.production.local 9000
```

### Runbook 2: Incident Response — Network Issue
```
Severity: HIGH — Service degradation

Step 1: Identify scope
  - Which service? Which AZ? Which tier?
  - Single user or all users?

Step 2: Check ALB health
  aws elbv2 describe-target-health --target-group-arn $TG_ARN

Step 3: Check NAT Gateway
  aws ec2 describe-nat-gateways --filter Name=state,Values=available
  aws cloudwatch get-metric-statistics \
    --namespace AWS/NATGateway \
    --metric-name ErrorPortAllocation ...

Step 4: Check VPC Flow Logs
  aws logs start-query \
    --log-group-name /aws/vpc/finflow-production/flowlogs \
    --query-string 'fields srcAddr, dstAddr, action | filter action="REJECT" | limit 20'

Step 5: SSH/SSM to affected instance
  aws ssm start-session --target i-xxxxxxxx

Step 6: Run validation script
  ./validate-network.sh app

Step 7: Escalate if unresolved within 15 minutes
```

---

## Part 4: Project Extensions (Bonus Challenges)

### Challenge 1: Transit Gateway Multi-VPC
```
Add a second VPC (shared-services: 10.1.0.0/16) for shared tools:
- Jenkins CI/CD
- Internal package registry
- Monitoring (Prometheus, Grafana)

Connect both VPCs via Transit Gateway.
Ensure finflow-vpc can reach shared-services-vpc but NOT the internet through it.

Deliverable: Updated Terraform with TGW attachment and route tables
```

### Challenge 2: CloudFront + WAF
```
Add CloudFront in front of the ALB:
- Origin: finflow ALB
- Cache behaviors: /static/* cached 30 days, /api/* not cached
- WAF rules: OWASP Core Rule Set, rate limiting (1000 req/5min per IP)
- Custom error pages for 403, 404, 500

Deliverable: Terraform cloudfront.tf and waf.tf
```

### Challenge 3: Multi-Region Active-Passive
```
Deploy a second stack in us-west-2 (passive).
Configure:
- Aurora Global Database with primary in us-east-1
- Route 53 failover routing: us-east-1 primary, us-west-2 secondary
- Health check on primary ALB
- Automated failover runbook

Test: Simulate us-east-1 failure, verify traffic shifts
```

### Challenge 4: Kubernetes Network Policies
```
Deploy EKS in the private-app subnets.
Implement NetworkPolicy for:
- frontend pods can only reach backend pods on port 8080
- backend pods can only reach postgres pods on port 5432
- monitoring pods (label: role=monitoring) can reach all pods on port 9090
- deny all other ingress/egress by default

Deliverable: network-policies.yaml with all required policies
```

### Challenge 5: Zero Trust Overlay
```
Implement Consul service mesh across the app tier:
- Install Consul with 3 server nodes
- Register all services with health checks
- Enable Consul Connect (mTLS sidecar)
- Create intentions:
  - ALLOW: web → api
  - ALLOW: api → database
  - DENY: * → database (catch-all)
  - ALLOW: monitoring → * (for Prometheus scraping)

Deliverable: Consul configuration files + intentions
```

---

## Part 5: Documentation Deliverables

Complete all of the following:

```
□ Architecture diagram (draw.io, Lucidchart, or ASCII)
□ IP address plan spreadsheet
□ Security Group matrix (who can talk to whom)
□ Terraform README with:
  - Prerequisites
  - terraform.tfvars example
  - Deployment steps
  - Destruction steps
□ Runbooks:
  - Add new service
  - Incident response
  - Rotate certificates
  - Expand to new AZ
□ Cost estimate (AWS Pricing Calculator)
□ Threat model (what attacks does this architecture prevent/not prevent)
```

---

## Final Self-Assessment

Rate yourself (1=beginner, 5=expert) on each area after completing the project:

| Area | Score (1-5) | Notes |
|------|------------|-------|
| IP addressing and subnetting | | |
| VPC design and routing | | |
| Security Groups and NACLs | | |
| Load balancing (ALB/NLB) | | |
| VPN and hybrid connectivity | | |
| DNS and service discovery | | |
| Network automation (Terraform) | | |
| Cloud monitoring and observability | | |
| Zero Trust and microsegmentation | | |
| Troubleshooting methodology | | |
| **Overall** | | |

---

## Completion Checklist

### Core (Required)
- [ ] IP address plan documented for all subnets across 3 AZs
- [ ] Complete Terraform code for VPC, subnets, IGW, NAT GW, route tables
- [ ] Security Groups for all tiers with least-privilege rules
- [ ] VPC Flow Logs enabled and CloudWatch alarms configured
- [ ] Validation script confirms all connectivity requirements
- [ ] Architecture diagram drawn
- [ ] Runbooks written for common operations

### Extended (Recommended)
- [ ] Challenge 1: Transit Gateway multi-VPC completed
- [ ] Challenge 2: CloudFront + WAF deployed
- [ ] Challenge 3: Multi-region failover tested
- [ ] Challenge 4: Kubernetes NetworkPolicies implemented
- [ ] Challenge 5: Consul service mesh with intentions

### Documentation
- [ ] README with deployment instructions
- [ ] Cost estimate calculated
- [ ] Threat model documented
- [ ] Self-assessment completed

---

## Congratulations!

You have completed the **30-Day Networking Curriculum** — from fundamentals to production-grade cloud networking. Here's what you've mastered:

```
Days 00-08  → Networking fundamentals and all 7 OSI layers
Days 09-11  → TCP/IP model, IP addressing, routing
Days 12-14  → Linux tools, DNS, DHCP, NAT
Days 15-16  → Firewalls, VPNs, tunneling
Days 17-18  → Load balancing, HTTPS, API gateways
Days 19     → Monitoring and troubleshooting
Days 20-23  → Cloud networking (AWS, GCP, Azure, hybrid)
Days 24-25  → Service discovery, automation (Terraform, Ansible)
Days 26-27  → Zero Trust, CDN, Anycast
Days 28     → Performance, QoS, cloud monitoring
Days 29     → Real-world scenarios and interview prep
Day 30      → Capstone project — production cloud network
```

### What Next?

**Certifications to pursue:**
- AWS Certified Advanced Networking – Specialty
- Cisco CCNA / CCNP
- GCP Professional Cloud Network Engineer
- Azure Network Engineer Associate (AZ-700)
- HashiCorp Terraform Associate

**Practice environments:**
- AWS Free Tier (most services in this curriculum qualify)
- GCP Free Tier
- Terraform Cloud (free tier)
- Cisco DevNet Sandbox (free network device labs)

**Stay current:**
- AWS re:Invent networking sessions (YouTube)
- The Cloudcast, Software Engineering Daily (podcasts)
- AWS, GCP, Azure networking blogs
- NANOG conference talks

**Keep building** — every production incident you troubleshoot teaches you more than any lab. Document everything, ask questions, and share your knowledge.

Good luck! 🚀
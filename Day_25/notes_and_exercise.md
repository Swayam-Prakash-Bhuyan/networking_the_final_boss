# Day 25: Network Automation (Ansible, Terraform, IaC, NetDevOps)

## Learning Objectives
By the end of Day 25, you will:
- Automate network configuration with Ansible playbooks
- Provision cloud network infrastructure with Terraform
- Apply GitOps and CI/CD practices to network changes (NetDevOps)
- Write idempotent, version-controlled network-as-code

**Estimated Time:** 3-4 hours

---

## Notes

### Why Network Automation?

Manual network configuration:
- Error-prone (typos, missed steps)
- Not reproducible
- Not version-controlled
- Slow to scale
- No audit trail

Network automation solves all of these with:
- **Consistency:** Same config applied the same way every time
- **Speed:** Deploy to hundreds of devices in minutes
- **Auditability:** All changes tracked in git
- **Idempotency:** Run multiple times — same result
- **Testability:** Test changes in staging before production

### Automation Tools Overview

| Tool | Use Case | Language | Agentless? |
|------|---------|----------|-----------|
| Ansible | Configuration mgmt, device automation | YAML | Yes |
| Terraform | Infrastructure provisioning (IaC) | HCL | Yes (API-based) |
| Nornir | Python-native network automation | Python | Yes |
| NAPALM | Multi-vendor network device abstraction | Python | Yes |
| Netmiko | SSH to network devices | Python | Yes |
| Salt | Config mgmt, event-driven automation | YAML/Python | Optional |

---

### Ansible for Network Automation

Ansible uses **playbooks** (YAML) to describe desired configuration state and **modules** to interact with devices.

#### Ansible Network Modules
```
ansible.netcommon.*    → General network utilities
cisco.ios.*            → Cisco IOS devices
cisco.nxos.*           → Cisco NX-OS
junipernetworks.junos.* → Juniper JunOS
arista.eos.*           → Arista EOS
community.network.*    → Generic network modules
amazon.aws.*           → AWS networking resources
azure.azcollection.*   → Azure networking
google.cloud.*         → GCP networking
```

#### Example: Configure VLANs on Cisco Switch
```yaml
# site.yml
---
- name: Configure Network VLANs
  hosts: switches
  gather_facts: false

  vars:
    vlans:
      - id: 10
        name: Sales
      - id: 20
        name: Engineering
      - id: 30
        name: Management

  tasks:
    - name: Create VLANs
      cisco.ios.ios_vlans:
        config:
          - vlan_id: "{{ item.id }}"
            name: "{{ item.name }}"
            state: active
        state: merged
      loop: "{{ vlans }}"

    - name: Configure Access Ports
      cisco.ios.ios_l2_interfaces:
        config:
          - name: GigabitEthernet0/1
            access:
              vlan: 10
            mode: access
        state: merged

    - name: Save Running Config
      cisco.ios.ios_command:
        commands: write memory
```

#### Inventory File
```ini
# inventory/hosts.ini
[switches]
sw1 ansible_host=192.168.1.10
sw2 ansible_host=192.168.1.11

[routers]
rtr1 ansible_host=192.168.1.1

[network:children]
switches
routers

[network:vars]
ansible_connection=network_cli
ansible_network_os=cisco.ios.ios
ansible_user=admin
ansible_password={{ vault_password }}
ansible_become=yes
ansible_become_method=enable
```

#### Example: AWS VPC with Ansible
```yaml
---
- name: Create AWS Network Infrastructure
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Create VPC
      amazon.aws.ec2_vpc_net:
        name: production-vpc
        cidr_block: 10.0.0.0/16
        region: us-east-1
        tags:
          Environment: production
      register: vpc

    - name: Create Public Subnet
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: 10.0.1.0/24
        az: us-east-1a
        tags:
          Name: public-1a
          Tier: public

    - name: Create Internet Gateway
      amazon.aws.ec2_vpc_igw:
        vpc_id: "{{ vpc.vpc.id }}"
        tags:
          Name: production-igw

    - name: Set up public route table
      amazon.aws.ec2_vpc_route_table:
        vpc_id: "{{ vpc.vpc.id }}"
        subnets:
          - 10.0.1.0/24
        routes:
          - dest: 0.0.0.0/0
            gateway_id: igw
```

---

### Terraform for Network Infrastructure

Terraform is the industry standard for **infrastructure as code (IaC)**. It provisions infrastructure declaratively using HCL (HashiCorp Configuration Language).

#### Terraform Workflow
```
terraform init     → Download providers and modules
terraform plan     → Show what will change (dry run)
terraform apply    → Apply changes
terraform destroy  → Tear down infrastructure
terraform state    → Inspect/manipulate state
```

#### Example: AWS VPC with Terraform
```hcl
# main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "${var.environment}-igw" }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 11)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.environment}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  }
}

# NAT Gateways (one per AZ for HA)
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags = { Name = "${var.environment}-nat-${count.index}" }
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.environment}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables (one per AZ)
resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = { Name = "${var.environment}-private-rt-${count.index}" }
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

```hcl
# variables.tf
variable "region"             { default = "us-east-1" }
variable "environment"        { default = "production" }
variable "vpc_cidr"           { default = "10.0.0.0/16" }
variable "availability_zones" { default = ["us-east-1a", "us-east-1b", "us-east-1c"] }
```

```hcl
# outputs.tf
output "vpc_id"          { value = aws_vpc.main.id }
output "public_subnets"  { value = aws_subnet.public[*].id }
output "private_subnets" { value = aws_subnet.private[*].id }
```

#### Terraform Security Groups
```hcl
resource "aws_security_group" "web" {
  name        = "${var.environment}-web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-web-sg" }
}
```

---

### NetDevOps: CI/CD for Network Changes

Apply software development best practices to network changes:

```
Developer → Git commit → Pull Request
                             ↓
                     CI Pipeline:
                     - Lint (ansible-lint, terraform validate)
                     - Unit tests (Molecule, terratest)
                     - Plan (terraform plan)
                     - Security scan (tfsec, checkov)
                             ↓
                     Code Review + Approval
                             ↓
                     CD Pipeline:
                     - Apply to staging
                     - Integration tests
                     - Apply to production
                             ↓
                     Monitor + Alert
```

#### GitHub Actions for Terraform
```yaml
# .github/workflows/network.yml
name: Network Infrastructure

on:
  pull_request:
    paths: ['terraform/network/**']
  push:
    branches: [main]
    paths: ['terraform/network/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init
        working-directory: terraform/network

      - name: Terraform Validate
        run: terraform validate
        working-directory: terraform/network

      - name: Terraform Format Check
        run: terraform fmt -check
        working-directory: terraform/network

      - name: Security Scan (tfsec)
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: terraform/network

      - name: Terraform Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color
        working-directory: terraform/network
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  apply:
    needs: validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: terraform/network
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Hands-On Labs

### Lab 1: Terraform Local Network Simulation
```bash
#!/bin/bash
# Install Terraform and validate basic config

if ! command -v terraform &>/dev/null; then
    echo "Installing Terraform..."
    wget -q https://releases.hashicorp.com/terraform/1.6.6/terraform_1.6.6_linux_amd64.zip
    unzip -q terraform_1.6.6_linux_amd64.zip
    sudo mv terraform /usr/local/bin/
    rm terraform_1.6.6_linux_amd64.zip
fi

# Create test Terraform config
mkdir -p /tmp/tf-network-test
cat > /tmp/tf-network-test/main.tf <<'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}

variable "subnets" {
  default = [
    { name = "public-1a",  cidr = "10.0.1.0/24", tier = "public"  },
    { name = "private-1a", cidr = "10.0.11.0/24", tier = "private" },
    { name = "db-1a",      cidr = "10.0.21.0/24", tier = "database"},
  ]
}

resource "local_file" "network_config" {
  for_each = { for s in var.subnets : s.name => s }
  filename = "/tmp/tf-network-test/subnet-${each.key}.txt"
  content  = "Subnet: ${each.value.name}\nCIDR: ${each.value.cidr}\nTier: ${each.value.tier}\n"
}

output "subnet_names" {
  value = [for s in var.subnets : s.name]
}
EOF

cd /tmp/tf-network-test
terraform init -input=false 2>/dev/null
terraform validate
terraform plan -no-color
terraform apply -auto-approve -no-color | tail -10
echo -e "\nGenerated subnet files:"
ls /tmp/tf-network-test/*.txt 2>/dev/null && cat /tmp/tf-network-test/subnet-public-1a.txt
```

### Lab 2: Ansible Lint and Test
```bash
#!/bin/bash
# Write and lint an Ansible network playbook

pip install ansible ansible-lint --quiet 2>/dev/null

mkdir -p /tmp/ansible-net-test

cat > /tmp/ansible-net-test/playbook.yml <<'EOF'
---
- name: Network Audit Playbook
  hosts: localhost
  gather_facts: true

  tasks:
    - name: Gather network interfaces
      ansible.builtin.setup:
        filter: ansible_interfaces

    - name: Display interfaces
      ansible.builtin.debug:
        msg: "Interface: {{ item }}"
      loop: "{{ ansible_interfaces }}"

    - name: Check default route exists
      ansible.builtin.command: ip route show default
      register: default_route
      changed_when: false

    - name: Report default route
      ansible.builtin.debug:
        msg: "Default route: {{ default_route.stdout }}"

    - name: Test DNS resolution
      ansible.builtin.command: nslookup google.com
      register: dns_result
      changed_when: false
      ignore_errors: true

    - name: DNS status
      ansible.builtin.debug:
        msg: "DNS {{ 'OK' if dns_result.rc == 0 else 'FAILED' }}"
EOF

echo "Running Ansible playbook..."
ansible-playbook /tmp/ansible-net-test/playbook.yml 2>/dev/null | grep -E "TASK|ok:|changed:|failed:|PLAY RECAP"
```

### Lab 3: Network Config Generator
```python
#!/usr/bin/env python3
# Generate Terraform and Ansible config from YAML spec

import yaml, json

NETWORK_SPEC = """
environment: production
vpc_cidr: 10.0.0.0/16
regions:
  - name: us-east-1
    azs: [us-east-1a, us-east-1b, us-east-1c]
tiers:
  - name: public
    cidr_offset: 0
    has_internet: true
  - name: private
    cidr_offset: 10
    has_internet: false
  - name: database
    cidr_offset: 20
    has_internet: false
security_groups:
  - name: web-sg
    rules:
      - port: 443
        source: 0.0.0.0/0
      - port: 80
        source: 0.0.0.0/0
  - name: app-sg
    rules:
      - port: 8080
        source: web-sg
  - name: db-sg
    rules:
      - port: 5432
        source: app-sg
"""

spec = yaml.safe_load(NETWORK_SPEC)
env = spec['environment']
vpc_cidr = spec['vpc_cidr']

print(f"# Generated Network Config for: {env}")
print(f"# VPC CIDR: {vpc_cidr}")
print()

import ipaddress
vpc = ipaddress.IPv4Network(vpc_cidr)
subnets = list(vpc.subnets(prefixlen_diff=8))

print("## Subnet Plan")
for region in spec['regions']:
    for tier in spec['tiers']:
        for i, az in enumerate(region['azs']):
            idx = tier['cidr_offset'] + i
            if idx < len(subnets):
                s = subnets[idx]
                inet = "IGW" if tier['has_internet'] else "NAT/None"
                print(f"  {env}-{tier['name']}-{az}: {s}  ({inet})")

print("\n## Security Group Rules")
for sg in spec['security_groups']:
    print(f"\n  [{sg['name']}]")
    for rule in sg['rules']:
        print(f"    TCP {rule['port']} from {rule['source']}")

print("\n## Terraform resource count estimate")
n_azs = len(spec['regions'][0]['azs'])
n_tiers = len(spec['tiers'])
n_sgs = len(spec['security_groups'])
print(f"  Subnets: {n_azs * n_tiers}")
print(f"  Route tables: {n_azs + 1}")
print(f"  NAT Gateways: {n_azs}")
print(f"  Security Groups: {n_sgs}")
```

---

## Sample Exercises

1. **Idempotency:** Explain what makes a Terraform or Ansible task idempotent. Why does `command: apt-get install nginx` violate idempotency in Ansible?
2. **State management:** Where should Terraform state be stored for a team of 5 engineers? Why not locally?
3. **Ansible vs Terraform:** You need to configure an ACL on a Cisco router AND create an AWS VPC. Which tool for each, and why?
4. **GitOps flow:** Design a branch strategy and CI/CD pipeline for a team making network changes to 3 environments (dev, staging, prod).
5. **Security:** How do you avoid committing AWS credentials or device passwords to git in Ansible/Terraform configs?

## Solutions

1. **Idempotency:** A task is idempotent if running it multiple times produces the same result. `apt-get install nginx` is not idempotent because it runs every time, even if nginx is installed. Use `ansible.builtin.apt: name=nginx state=present` — Ansible checks if nginx is already installed and skips if so. Terraform is inherently idempotent — it compares desired state to actual state and only makes necessary changes.

2. **Terraform state for a team:** Store in a **remote backend** — AWS S3 + DynamoDB (for locking), Azure Blob Storage, GCP GCS, or Terraform Cloud. Remote state: prevents simultaneous applies (locking), allows team access, provides history. Never store locally — a second engineer can't see your local state file and will overwrite/conflict.

3. **Ansible vs Terraform:**
   - **Cisco ACL → Ansible** (`cisco.ios.ios_acls` module): Ansible excels at configuring existing devices via SSH. Terraform doesn't have Cisco router providers.
   - **AWS VPC → Terraform**: Terraform is the standard for AWS infrastructure provisioning. Ansible AWS modules work but Terraform's state management makes it far better for infrastructure.

4. **GitOps branch strategy:**
   ```
   main → deploys to prod (protected, requires PR + 2 approvals)
   staging → deploys to staging (requires PR + 1 approval)
   dev → deploys to dev (can push directly)
   feature/* → no deployment (plan only in CI)
   ```

5. **Secrets management:**
   - Terraform: Use environment variables (`AWS_ACCESS_KEY_ID`), IAM roles (EC2/ECS), or HCP Vault/AWS Secrets Manager. Never hardcode in `.tf` files. Add `*.tfvars` to `.gitignore`.
   - Ansible: Use `ansible-vault encrypt_string` to encrypt passwords in YAML. Store vault password in CI/CD secrets (GitHub Actions Secrets, GitLab CI Variables). Use `ansible-vault encrypt vars/secrets.yml` for entire files.

## Completion Checklist
- [ ] Write an Ansible playbook to configure network resources
- [ ] Write Terraform code to provision a VPC with subnets, IGW, and NAT
- [ ] Understand Terraform state and remote backends
- [ ] Set up a CI/CD pipeline for network infrastructure changes
- [ ] Manage secrets securely in Ansible and Terraform
- [ ] Generate infrastructure configs from YAML specifications

## Key Takeaways
- Network automation eliminates human error and enables consistent, repeatable deployments
- Terraform manages cloud infrastructure lifecycle (create/update/destroy) with state
- Ansible is ideal for configuring existing devices and orchestrating multi-step workflows
- Store Terraform state remotely with locking for team environments
- Treat network config like application code: version control, review, test, deploy via CI/CD
- Never commit credentials — use vaults, environment variables, and IAM roles

## Next Steps
Proceed to [Day 26: Zero Trust, Security Groups, Microsegmentation, SASE](../Day_26/notes_and_exercises.md).
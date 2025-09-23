# Day 06: Layer 5 - Session Layer (Sessions, Authentication, Real-World Use)

## Learning Objectives
- Understand session establishment, management, and termination
- Learn authentication mechanisms and protocols
- Explore session security and encryption
- Apply session concepts to real-world applications

## Theory

### Session Layer Functions
- **Session Establishment**: Create communication sessions between applications
- **Session Management**: Maintain session state and control dialog
- **Session Termination**: Properly close sessions and clean up resources
- **Checkpointing**: Save session state for recovery
- **Dialog Control**: Manage full-duplex or half-duplex communication

### Session Management Concepts

#### Session States
- **Inactive**: No active session
- **Establishing**: Session setup in progress
- **Active**: Session established and data flowing
- **Suspending**: Temporary session pause
- **Terminating**: Session cleanup in progress

#### Session Identifiers
- **Session ID**: Unique identifier for each session
- **Tokens**: Temporary credentials for session access
- **Cookies**: Client-side session state storage
- **Tickets**: Time-limited access credentials

### Authentication Mechanisms

#### Authentication Types
- **Something you know**: Passwords, PINs
- **Something you have**: Tokens, smart cards, certificates
- **Something you are**: Biometrics, fingerprints
- **Multi-Factor Authentication (MFA)**: Combination of above

#### Authentication Protocols
- **Kerberos**: Ticket-based authentication system
- **LDAP**: Lightweight Directory Access Protocol
- **RADIUS**: Remote Authentication Dial-In User Service
- **SAML**: Security Assertion Markup Language
- **OAuth**: Open Authorization framework
- **OpenID Connect**: Identity layer on top of OAuth 2.0

### Session Security

#### Session Hijacking Prevention
- **Session Token Randomization**: Unpredictable session IDs
- **Secure Transmission**: HTTPS/TLS encryption
- **Session Timeout**: Automatic session expiration
- **IP Binding**: Tie sessions to client IP addresses
- **User Agent Validation**: Verify client browser consistency

#### Session Management Best Practices
- **Secure Session Storage**: Server-side session data
- **Session Regeneration**: New session ID after authentication
- **Proper Logout**: Complete session cleanup
- **Session Monitoring**: Track active sessions

## Hands-On Labs

### Lab 1: SSH Session Management
```bash
# Establish SSH session with key-based authentication
ssh-keygen -t rsa -b 4096 -f ~/.ssh/lab_key
ssh-copy-id -i ~/.ssh/lab_key.pub user@remote-server

# SSH with session multiplexing
ssh -M -S ~/.ssh/control-%r@%h:%p user@remote-server

# Reuse existing SSH connection
ssh -S ~/.ssh/control-%r@%h:%p user@remote-server

# Monitor SSH sessions
who
w
last

# SSH session configuration
cat >> ~/.ssh/config << EOF
Host lab-server
    HostName remote-server.example.com
    User labuser
    IdentityFile ~/.ssh/lab_key
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 10m
EOF
```

### Lab 2: Web Session Analysis
```bash
# Monitor HTTP sessions with cookies
curl -c cookies.txt -b cookies.txt -L http://httpbin.org/cookies/set/session/abc123
curl -b cookies.txt http://httpbin.org/cookies

# Analyze session cookies
echo "Session cookie analysis:"
cat cookies.txt

# Test session persistence
curl -b cookies.txt http://httpbin.org/headers | grep Cookie
```

### Lab 3: Database Session Management
```bash
# MySQL session management
mysql -u root -p << EOF
-- Show current sessions
SHOW PROCESSLIST;

-- Show session variables
SHOW SESSION VARIABLES LIKE 'autocommit';

-- Set session timeout
SET SESSION wait_timeout = 300;

-- Create session-specific table
CREATE TEMPORARY TABLE session_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(255)
);

-- Session will end when connection closes
EOF
```

## Practical Exercises

### Exercise 1: Session Token Generator
```python
#!/usr/bin/env python3
import secrets
import hashlib
import time
import json

class SessionManager:
    def __init__(self):
        self.sessions = {}
        self.session_timeout = 3600  # 1 hour
    
    def create_session(self, user_id):
        """Create a new session for user"""
        session_id = secrets.token_urlsafe(32)
        session_data = {
            'user_id': user_id,
            'created_at': time.time(),
            'last_accessed': time.time(),
            'ip_address': None  # Would be set from request
        }
        self.sessions[session_id] = session_data
        return session_id
    
    def validate_session(self, session_id):
        """Validate session and update last accessed time"""
        if session_id not in self.sessions:
            return False
        
        session = self.sessions[session_id]
        current_time = time.time()
        
        # Check if session expired
        if current_time - session['last_accessed'] > self.session_timeout:
            del self.sessions[session_id]
            return False
        
        # Update last accessed time
        session['last_accessed'] = current_time
        return True
    
    def destroy_session(self, session_id):
        """Destroy session"""
        if session_id in self.sessions:
            del self.sessions[session_id]
            return True
        return False
    
    def cleanup_expired_sessions(self):
        """Remove expired sessions"""
        current_time = time.time()
        expired_sessions = []
        
        for session_id, session_data in self.sessions.items():
            if current_time - session_data['last_accessed'] > self.session_timeout:
                expired_sessions.append(session_id)
        
        for session_id in expired_sessions:
            del self.sessions[session_id]
        
        return len(expired_sessions)

# Example usage
if __name__ == "__main__":
    sm = SessionManager()
    
    # Create session
    session_id = sm.create_session("user123")
    print(f"Created session: {session_id}")
    
    # Validate session
    if sm.validate_session(session_id):
        print("Session is valid")
    
    # Destroy session
    sm.destroy_session(session_id)
    print("Session destroyed")
```

### Exercise 2: Kerberos Authentication Simulation
```bash
#!/bin/bash
# Kerberos authentication workflow simulation

# Install Kerberos client tools
sudo apt install krb5-user

# Configure Kerberos realm
sudo tee /etc/krb5.conf << EOF
[libdefaults]
    default_realm = EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = false

[realms]
    EXAMPLE.COM = {
        kdc = kdc.example.com
        admin_server = kdc.example.com
    }

[domain_realm]
    .example.com = EXAMPLE.COM
    example.com = EXAMPLE.COM
EOF

# Obtain Kerberos ticket
echo "Obtaining Kerberos ticket..."
kinit username@EXAMPLE.COM

# List current tickets
klist

# Use ticket for authentication
# (This would be used by applications automatically)

# Destroy tickets when done
kdestroy
```

### Exercise 3: OAuth 2.0 Flow Simulation
```python
#!/usr/bin/env python3
import requests
import base64
import json
from urllib.parse import urlencode, parse_qs

class OAuth2Client:
    def __init__(self, client_id, client_secret, auth_url, token_url):
        self.client_id = client_id
        self.client_secret = client_secret
        self.auth_url = auth_url
        self.token_url = token_url
    
    def get_authorization_url(self, redirect_uri, scope="read"):
        """Generate authorization URL"""
        params = {
            'response_type': 'code',
            'client_id': self.client_id,
            'redirect_uri': redirect_uri,
            'scope': scope,
            'state': 'random_state_string'
        }
        return f"{self.auth_url}?{urlencode(params)}"
    
    def exchange_code_for_token(self, code, redirect_uri):
        """Exchange authorization code for access token"""
        auth_header = base64.b64encode(
            f"{self.client_id}:{self.client_secret}".encode()
        ).decode()
        
        headers = {
            'Authorization': f'Basic {auth_header}',
            'Content-Type': 'application/x-www-form-urlencoded'
        }
        
        data = {
            'grant_type': 'authorization_code',
            'code': code,
            'redirect_uri': redirect_uri
        }
        
        response = requests.post(self.token_url, headers=headers, data=data)
        return response.json()

# Example usage (would need real OAuth provider)
if __name__ == "__main__":
    client = OAuth2Client(
        client_id="your_client_id",
        client_secret="your_client_secret",
        auth_url="https://provider.com/oauth/authorize",
        token_url="https://provider.com/oauth/token"
    )
    
    # Step 1: Get authorization URL
    auth_url = client.get_authorization_url("http://localhost:8080/callback")
    print(f"Visit this URL: {auth_url}")
    
    # Step 2: User authorizes and gets redirected with code
    # Step 3: Exchange code for token (in real app)
    # token_response = client.exchange_code_for_token(code, redirect_uri)
```

## Real-World Applications

### Web Application Session Management
```python
# Flask session example
from flask import Flask, session, request, jsonify
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(16)

@app.route('/login', methods=['POST'])
def login():
    username = request.json.get('username')
    password = request.json.get('password')
    
    # Validate credentials (simplified)
    if username == "admin" and password == "secret":
        session['user_id'] = username
        session['authenticated'] = True
        return jsonify({'status': 'success'})
    
    return jsonify({'status': 'failed'}), 401

@app.route('/protected')
def protected():
    if not session.get('authenticated'):
        return jsonify({'error': 'Not authenticated'}), 401
    
    return jsonify({'message': 'Access granted', 'user': session['user_id']})

@app.route('/logout')
def logout():
    session.clear()
    return jsonify({'status': 'logged out'})
```

### API Session Management
```bash
# JWT token-based session
#!/bin/bash

# Login and get JWT token
TOKEN=$(curl -s -X POST http://api.example.com/login \
    -H "Content-Type: application/json" \
    -d '{"username":"user","password":"pass"}' | \
    jq -r '.token')

# Use token for API calls
curl -H "Authorization: Bearer $TOKEN" \
    http://api.example.com/protected-resource

# Refresh token before expiration
NEW_TOKEN=$(curl -s -X POST http://api.example.com/refresh \
    -H "Authorization: Bearer $TOKEN" | \
    jq -r '.token')
```

### Database Connection Pooling
```python
#!/usr/bin/env python3
import mysql.connector.pooling

# Create connection pool
config = {
    'user': 'dbuser',
    'password': 'dbpass',
    'host': 'localhost',
    'database': 'testdb',
    'pool_name': 'mypool',
    'pool_size': 10,
    'pool_reset_session': True
}

pool = mysql.connector.pooling.MySQLConnectionPool(**config)

def execute_query(query):
    """Execute query using connection from pool"""
    connection = pool.get_connection()
    try:
        cursor = connection.cursor()
        cursor.execute(query)
        result = cursor.fetchall()
        connection.commit()
        return result
    finally:
        cursor.close()
        connection.close()  # Returns to pool
```

## Security Considerations

### Session Security Checklist
```bash
# 1. Use HTTPS for all session-related communications
# 2. Generate cryptographically secure session IDs
# 3. Implement proper session timeout
# 4. Regenerate session ID after authentication
# 5. Validate session on each request
# 6. Implement proper logout functionality
# 7. Monitor for session anomalies

# Session monitoring script
#!/bin/bash
LOG_FILE="/var/log/session_monitor.log"

while true; do
    # Monitor active sessions
    ACTIVE_SESSIONS=$(who | wc -l)
    
    # Check for suspicious login patterns
    FAILED_LOGINS=$(grep "Failed password" /var/log/auth.log | tail -10 | wc -l)
    
    # Log session statistics
    echo "$(date): Active sessions: $ACTIVE_SESSIONS, Failed logins: $FAILED_LOGINS" >> $LOG_FILE
    
    sleep 60
done
```

## Key Takeaways
- Session layer manages communication sessions between applications
- Authentication is crucial for secure session establishment
- Proper session management prevents security vulnerabilities
- Different applications require different session strategies
- Session security involves multiple layers of protection

## Next Steps
Tomorrow we'll explore Layer 6 (Presentation Layer), covering data encoding, encryption, TLS/SSL, and data transformation.

## Resources
- [RFC 4120 - Kerberos Network Authentication Service](https://tools.ietf.org/html/rfc4120)
- [OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [JWT Introduction](https://jwt.io/introduction/)
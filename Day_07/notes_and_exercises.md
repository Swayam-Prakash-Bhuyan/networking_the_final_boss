# Day 07: Layer 6 - Presentation Layer (Encoding, TLS/SSL, Encryption)

## Learning Objectives
- Understand data representation and encoding formats
- Master TLS/SSL encryption and certificate management
- Learn compression and data transformation techniques
- Explore cryptographic protocols and implementations

## Theory

### Presentation Layer Functions
- **Data Translation**: Convert between different data formats
- **Encryption/Decryption**: Secure data transmission
- **Compression/Decompression**: Reduce data size for transmission
- **Character Encoding**: Handle different character sets
- **Data Formatting**: Structure data for applications

### Data Encoding and Formats

#### Character Encoding
- **ASCII**: 7-bit character encoding (128 characters)
- **UTF-8**: Variable-length Unicode encoding
- **UTF-16**: 16-bit Unicode encoding
- **ISO-8859-1**: Extended ASCII (Latin-1)
- **Base64**: Binary-to-text encoding scheme

#### Data Serialization Formats
- **JSON**: JavaScript Object Notation
- **XML**: eXtensible Markup Language
- **YAML**: YAML Ain't Markup Language
- **Protocol Buffers**: Google's serialization format
- **MessagePack**: Efficient binary serialization

#### Image and Media Formats
- **JPEG**: Lossy image compression
- **PNG**: Lossless image compression
- **GIF**: Graphics Interchange Format
- **MP3**: Audio compression
- **H.264**: Video compression

### Encryption and Cryptography

#### Symmetric Encryption
- **AES**: Advanced Encryption Standard
- **DES/3DES**: Data Encryption Standard
- **ChaCha20**: Stream cipher
- **Blowfish**: Block cipher

#### Asymmetric Encryption
- **RSA**: Rivest-Shamir-Adleman
- **ECC**: Elliptic Curve Cryptography
- **Diffie-Hellman**: Key exchange protocol
- **DSA**: Digital Signature Algorithm

#### Hash Functions
- **SHA-256**: Secure Hash Algorithm
- **MD5**: Message Digest (deprecated)
- **BLAKE2**: Cryptographic hash function
- **HMAC**: Hash-based Message Authentication Code

### TLS/SSL Protocol

#### TLS Handshake Process
```
Client                                Server
  |                                     |
  |-------ClientHello------------------>|
  |                                     |
  |<------ServerHello-------------------|
  |<------Certificate-------------------|
  |<------ServerKeyExchange-------------|
  |<------ServerHelloDone---------------|
  |                                     |
  |-------ClientKeyExchange------------>|
  |-------ChangeCipherSpec------------->|
  |-------Finished-------------------->|
  |                                     |
  |<------ChangeCipherSpec--------------|
  |<------Finished---------------------|
  |                                     |
  |       Application Data              |
  |<----------------------------------->|
```

#### TLS Versions
- **TLS 1.0**: Deprecated (RFC 2246)
- **TLS 1.1**: Deprecated (RFC 4346)
- **TLS 1.2**: Current standard (RFC 5246)
- **TLS 1.3**: Latest version (RFC 8446)

#### Certificate Management
- **X.509**: Certificate format standard
- **CA**: Certificate Authority
- **CSR**: Certificate Signing Request
- **CRL**: Certificate Revocation List
- **OCSP**: Online Certificate Status Protocol

## Hands-On Labs

### Lab 1: Data Encoding and Conversion
```bash
# Base64 encoding/decoding
echo "Hello, World!" | base64
echo "SGVsbG8sIFdvcmxkIQo=" | base64 -d

# URL encoding
python3 -c "import urllib.parse; print(urllib.parse.quote('Hello World!'))"

# Hex encoding
echo "Hello" | xxd
echo "48656c6c6f0a" | xxd -r -p

# Character encoding conversion
echo "Hello" | iconv -f UTF-8 -t UTF-16
file -bi /etc/passwd  # Check file encoding
```

### Lab 2: TLS/SSL Certificate Management
```bash
# Generate private key
openssl genrsa -out server.key 2048

# Create certificate signing request
openssl req -new -key server.key -out server.csr -subj "/C=US/ST=CA/L=SF/O=Test/CN=example.com"

# Generate self-signed certificate
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

# View certificate details
openssl x509 -in server.crt -text -noout

# Test SSL connection
openssl s_client -connect google.com:443 -servername google.com

# Check certificate expiration
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -dates
```

### Lab 3: Encryption and Decryption
```bash
# Symmetric encryption with OpenSSL
echo "Secret message" | openssl enc -aes-256-cbc -a -salt -pass pass:mypassword

# Decrypt the message
echo "U2FsdGVkX1..." | openssl enc -aes-256-cbc -d -a -pass pass:mypassword

# Generate RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# RSA encryption/decryption
echo "Secret" | openssl rsautl -encrypt -pubin -inkey public.pem | base64
echo "encrypted_data" | base64 -d | openssl rsautl -decrypt -inkey private.pem

# Hash generation
echo "Hello World" | sha256sum
echo "Hello World" | md5sum
```

## Practical Exercises

### Exercise 1: Data Format Converter
```python
#!/usr/bin/env python3
import json
import yaml
import xml.etree.ElementTree as ET
import base64

class DataConverter:
    def __init__(self):
        self.data = {}
    
    def json_to_yaml(self, json_str):
        """Convert JSON to YAML"""
        data = json.loads(json_str)
        return yaml.dump(data, default_flow_style=False)
    
    def yaml_to_json(self, yaml_str):
        """Convert YAML to JSON"""
        data = yaml.safe_load(yaml_str)
        return json.dumps(data, indent=2)
    
    def dict_to_xml(self, data, root_name="root"):
        """Convert dictionary to XML"""
        root = ET.Element(root_name)
        
        def build_xml(element, data):
            if isinstance(data, dict):
                for key, value in data.items():
                    child = ET.SubElement(element, key)
                    build_xml(child, value)
            elif isinstance(data, list):
                for item in data:
                    child = ET.SubElement(element, "item")
                    build_xml(child, item)
            else:
                element.text = str(data)
        
        build_xml(root, data)
        return ET.tostring(root, encoding='unicode')
    
    def encode_base64(self, data):
        """Encode data to Base64"""
        return base64.b64encode(data.encode()).decode()
    
    def decode_base64(self, encoded_data):
        """Decode Base64 data"""
        return base64.b64decode(encoded_data).decode()

# Example usage
if __name__ == "__main__":
    converter = DataConverter()
    
    # Sample data
    sample_json = '{"name": "John", "age": 30, "city": "New York"}'
    
    # Convert JSON to YAML
    yaml_output = converter.json_to_yaml(sample_json)
    print("YAML output:")
    print(yaml_output)
    
    # Convert back to JSON
    json_output = converter.yaml_to_json(yaml_output)
    print("JSON output:")
    print(json_output)
    
    # Base64 encoding
    encoded = converter.encode_base64("Hello, World!")
    print(f"Base64 encoded: {encoded}")
    print(f"Base64 decoded: {converter.decode_base64(encoded)}")
```

### Exercise 2: TLS Certificate Validator
```python
#!/usr/bin/env python3
import ssl
import socket
import datetime
from urllib.parse import urlparse

class CertificateValidator:
    def __init__(self):
        pass
    
    def get_certificate_info(self, hostname, port=443):
        """Get certificate information for a host"""
        try:
            context = ssl.create_default_context()
            with socket.create_connection((hostname, port), timeout=10) as sock:
                with context.wrap_socket(sock, server_hostname=hostname) as ssock:
                    cert = ssock.getpeercert()
                    return cert
        except Exception as e:
            return {"error": str(e)}
    
    def check_certificate_expiry(self, hostname, port=443):
        """Check if certificate is expired or expiring soon"""
        cert = self.get_certificate_info(hostname, port)
        
        if "error" in cert:
            return cert
        
        # Parse expiry date
        expiry_date = datetime.datetime.strptime(
            cert['notAfter'], '%b %d %H:%M:%S %Y %Z'
        )
        
        current_date = datetime.datetime.now()
        days_until_expiry = (expiry_date - current_date).days
        
        return {
            'hostname': hostname,
            'expiry_date': cert['notAfter'],
            'days_until_expiry': days_until_expiry,
            'is_expired': days_until_expiry < 0,
            'expires_soon': days_until_expiry < 30
        }
    
    def validate_certificate_chain(self, hostname, port=443):
        """Validate certificate chain"""
        try:
            context = ssl.create_default_context()
            with socket.create_connection((hostname, port), timeout=10) as sock:
                with context.wrap_socket(sock, server_hostname=hostname) as ssock:
                    return {"valid": True, "protocol": ssock.version()}
        except ssl.SSLError as e:
            return {"valid": False, "error": str(e)}

# Example usage
if __name__ == "__main__":
    validator = CertificateValidator()
    
    # Check certificate expiry
    result = validator.check_certificate_expiry("google.com")
    print(f"Certificate info for google.com:")
    print(f"Expires: {result['expiry_date']}")
    print(f"Days until expiry: {result['days_until_expiry']}")
    
    # Validate certificate chain
    chain_result = validator.validate_certificate_chain("google.com")
    print(f"Certificate chain valid: {chain_result['valid']}")
```

### Exercise 3: Encryption Utility
```python
#!/usr/bin/env python3
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import base64
import os

class EncryptionUtil:
    def __init__(self):
        pass
    
    def generate_key_from_password(self, password, salt=None):
        """Generate encryption key from password"""
        if salt is None:
            salt = os.urandom(16)
        
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(password.encode()))
        return key, salt
    
    def encrypt_data(self, data, password):
        """Encrypt data with password"""
        key, salt = self.generate_key_from_password(password)
        f = Fernet(key)
        encrypted_data = f.encrypt(data.encode())
        
        # Combine salt and encrypted data
        return base64.b64encode(salt + encrypted_data).decode()
    
    def decrypt_data(self, encrypted_data, password):
        """Decrypt data with password"""
        # Decode and separate salt from encrypted data
        combined_data = base64.b64decode(encrypted_data)
        salt = combined_data[:16]
        encrypted_data = combined_data[16:]
        
        # Generate key from password and salt
        key, _ = self.generate_key_from_password(password, salt)
        f = Fernet(key)
        
        decrypted_data = f.decrypt(encrypted_data)
        return decrypted_data.decode()

# Example usage
if __name__ == "__main__":
    encryptor = EncryptionUtil()
    
    # Encrypt data
    original_data = "This is a secret message!"
    password = "my_secure_password"
    
    encrypted = encryptor.encrypt_data(original_data, password)
    print(f"Encrypted: {encrypted}")
    
    # Decrypt data
    decrypted = encryptor.decrypt_data(encrypted, password)
    print(f"Decrypted: {decrypted}")
```

## Real-World Applications

### Web Server TLS Configuration
```bash
# Nginx TLS configuration
sudo tee /etc/nginx/sites-available/secure-site << EOF
server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    
    # Modern TLS configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
```

### API Data Encryption
```python
# Flask API with encryption
from flask import Flask, request, jsonify
from cryptography.fernet import Fernet
import base64

app = Flask(__name__)
encryption_key = Fernet.generate_key()
cipher_suite = Fernet(encryption_key)

@app.route('/encrypt', methods=['POST'])
def encrypt_data():
    data = request.json.get('data')
    encrypted_data = cipher_suite.encrypt(data.encode())
    return jsonify({
        'encrypted_data': base64.b64encode(encrypted_data).decode()
    })

@app.route('/decrypt', methods=['POST'])
def decrypt_data():
    encrypted_data = request.json.get('encrypted_data')
    decoded_data = base64.b64decode(encrypted_data)
    decrypted_data = cipher_suite.decrypt(decoded_data)
    return jsonify({
        'decrypted_data': decrypted_data.decode()
    })
```

### Compression Implementation
```python
#!/usr/bin/env python3
import gzip
import zlib
import bz2
import lzma

class CompressionUtil:
    def __init__(self):
        pass
    
    def compress_gzip(self, data):
        """Compress data using gzip"""
        return gzip.compress(data.encode())
    
    def decompress_gzip(self, compressed_data):
        """Decompress gzip data"""
        return gzip.decompress(compressed_data).decode()
    
    def compress_zlib(self, data):
        """Compress data using zlib"""
        return zlib.compress(data.encode())
    
    def decompress_zlib(self, compressed_data):
        """Decompress zlib data"""
        return zlib.decompress(compressed_data).decode()
    
    def compare_compression_ratios(self, data):
        """Compare compression ratios of different algorithms"""
        original_size = len(data.encode())
        
        gzip_compressed = gzip.compress(data.encode())
        zlib_compressed = zlib.compress(data.encode())
        bz2_compressed = bz2.compress(data.encode())
        lzma_compressed = lzma.compress(data.encode())
        
        return {
            'original_size': original_size,
            'gzip': {'size': len(gzip_compressed), 'ratio': len(gzip_compressed) / original_size},
            'zlib': {'size': len(zlib_compressed), 'ratio': len(zlib_compressed) / original_size},
            'bz2': {'size': len(bz2_compressed), 'ratio': len(bz2_compressed) / original_size},
            'lzma': {'size': len(lzma_compressed), 'ratio': len(lzma_compressed) / original_size}
        }

# Example usage
if __name__ == "__main__":
    compressor = CompressionUtil()
    
    # Test data
    test_data = "This is a test string that will be compressed using various algorithms. " * 100
    
    # Compare compression ratios
    results = compressor.compare_compression_ratios(test_data)
    print("Compression comparison:")
    for algorithm, stats in results.items():
        if algorithm != 'original_size':
            print(f"{algorithm}: {stats['size']} bytes ({stats['ratio']:.2%} of original)")
```

## Key Takeaways
- Presentation layer handles data formatting, encryption, and compression
- TLS/SSL provides secure communication channels
- Proper certificate management is crucial for security
- Data encoding ensures compatibility across different systems
- Compression reduces bandwidth usage and improves performance

## Next Steps
Tomorrow we'll explore Layer 7 (Application Layer), covering HTTP, DNS, SMTP, REST APIs, and cloud applications.

## Resources
- [RFC 8446 - TLS 1.3](https://tools.ietf.org/html/rfc8446)
- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Cryptography Best Practices](https://cryptography.io/)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
# Day 07: Layer 6 - Presentation Layer (Encoding, TLS/SSL, Encryption)

## Learning Objectives
By the end of Day 7, you will:
- Understand data representation and encoding formats
- Master TLS/SSL encryption and certificate management
- Learn compression and data transformation techniques
- Explore cryptographic protocols and implementations

**Estimated Time:** 2-3 hours

## Notes

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

### TLS/SSL Protocol

#### TLS Handshake Process
```
Client                                Server
  |                                     |
  | 1. ClientHello                      |
  |------------------------------------>|
  |   - TLS version                     |
  |   - Cipher suites                   |
  |   - Random number                   |
  |                                     |
  | 2. ServerHello                      |
  |<------------------------------------|
  |   - Selected cipher suite           |
  |   - Server random                   |
  |                                     |
  | 3. Certificate                      |
  |<------------------------------------|
  |   - Server's public key             |
  |                                     |
  | 4. ServerHelloDone                  |
  |<------------------------------------|
  |                                     |
  | 5. ClientKeyExchange                |
  |------------------------------------>|
  |   - Pre-master secret (encrypted)   |
  |                                     |
  | 6. ChangeCipherSpec                 |
  |------------------------------------>|
  |                                     |
  | 7. Finished                         |
  |------------------------------------>|
  |                                     |
  | 8. ChangeCipherSpec                 |
  |<------------------------------------|
  |                                     |
  | 9. Finished                         |
  |<------------------------------------|
  |                                     |
  |       Encrypted Application Data    |
  |<----------------------------------->|
```

#### TLS Versions
- **TLS 1.0**: Deprecated (RFC 2246)
- **TLS 1.1**: Deprecated (RFC 4346)
- **TLS 1.2**: Current standard (RFC 5246)
- **TLS 1.3**: Latest version (RFC 8446)

### Encryption Fundamentals

#### Symmetric Encryption
- **AES**: Advanced Encryption Standard
- **DES/3DES**: Data Encryption Standard (deprecated)
- **ChaCha20**: Stream cipher
- **Key Sharing**: Same key for encryption and decryption

#### Asymmetric Encryption
- **RSA**: Rivest-Shamir-Adleman
- **ECC**: Elliptic Curve Cryptography
- **Key Pairs**: Public key (encrypt) + Private key (decrypt)
- **Use Cases**: Key exchange, digital signatures

#### Hash Functions
- **SHA-256**: Secure Hash Algorithm
- **MD5**: Message Digest (deprecated)
- **HMAC**: Hash-based Message Authentication Code
- **Purpose**: Data integrity verification

### Certificate Management

#### X.509 Certificate Structure
```
Certificate:
├── Version
├── Serial Number
├── Signature Algorithm
├── Issuer (CA)
├── Validity Period
│   ├── Not Before
│   └── Not After
├── Subject (Server)
├── Public Key Info
│   ├── Algorithm
│   └── Public Key
└── Extensions
    ├── Subject Alternative Names
    ├── Key Usage
    └── Extended Key Usage
```

#### Certificate Chain
```
Root CA Certificate
    ↓ (signs)
Intermediate CA Certificate
    ↓ (signs)
Server Certificate (End Entity)
```

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
```

### Lab 2: TLS/SSL Certificate Analysis
```bash
# Generate self-signed certificate
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt -days 365 -nodes \
  -subj "/C=US/ST=CA/L=SF/O=Test/CN=localhost"

# View certificate details
openssl x509 -in server.crt -text -noout

# Check certificate expiration
openssl x509 -in server.crt -noout -dates

# Test SSL connection to website
openssl s_client -connect google.com:443 -servername google.com

# Check certificate chain
openssl s_client -connect google.com:443 -showcerts
```

### Lab 3: Encryption and Hashing
```bash
# Symmetric encryption with OpenSSL
echo "Secret message" | openssl enc -aes-256-cbc -a -salt -k "password"

# Generate hash
echo "Hello World" | sha256sum
echo "Hello World" | md5sum

# Generate RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# View key details
openssl rsa -in private.pem -text -noout
```

## Practical Exercises

### Exercise 1: Simple Data Converter
```bash
#!/bin/bash
# Data format converter

convert_to_base64() {
    echo "$1" | base64
}

convert_from_base64() {
    echo "$1" | base64 -d
}

convert_to_hex() {
    echo "$1" | xxd -p | tr -d '\n'
}

# Usage examples
echo "Original: Hello World"
ENCODED=$(convert_to_base64 "Hello World")
echo "Base64: $ENCODED"
echo "Decoded: $(convert_from_base64 "$ENCODED")"
```

### Exercise 2: Certificate Validator
```bash
#!/bin/bash
# Simple certificate validator

check_certificate() {
    local hostname=$1
    local port=${2:-443}
    
    echo "Checking certificate for $hostname:$port"
    
    # Get certificate info
    CERT_INFO=$(echo | openssl s_client -connect $hostname:$port -servername $hostname 2>/dev/null | openssl x509 -noout -dates 2>/dev/null)
    
    if [ $? -eq 0 ]; then
        echo "Certificate found:"
        echo "$CERT_INFO"
        
        # Check if certificate is expired
        NOT_AFTER=$(echo "$CERT_INFO" | grep "notAfter" | cut -d= -f2)
        EXPIRY_DATE=$(date -d "$NOT_AFTER" +%s 2>/dev/null)
        CURRENT_DATE=$(date +%s)
        
        if [ $EXPIRY_DATE -gt $CURRENT_DATE ]; then
            DAYS_LEFT=$(( (EXPIRY_DATE - CURRENT_DATE) / 86400 ))
            echo "✓ Certificate valid for $DAYS_LEFT more days"
        else
            echo "✗ Certificate expired"
        fi
    else
        echo "✗ Could not retrieve certificate"
    fi
}

# Usage
check_certificate google.com
```

### Exercise 3: Encryption Demo
```bash
#!/bin/bash
# Simple encryption demonstration

# Generate random key
KEY=$(openssl rand -hex 16)
echo "Generated key: $KEY"

# Encrypt message
MESSAGE="This is a secret message"
echo "Original message: $MESSAGE"

ENCRYPTED=$(echo "$MESSAGE" | openssl enc -aes-128-cbc -a -K $KEY -iv $(openssl rand -hex 16))
echo "Encrypted: $ENCRYPTED"

# Note: In practice, you'd need to store the IV to decrypt
echo "Note: IV needed for decryption (not shown for simplicity)"
```

## Sample Exercises

1. **Data Format Analysis**: Compare JSON, XML, and YAML for representing the same data structure.

2. **TLS Configuration**: Configure a web server with proper TLS settings and test security.

3. **Certificate Management**: Create a certificate chain with root CA, intermediate CA, and server certificate.

4. **Encryption Comparison**: Compare symmetric vs asymmetric encryption performance and use cases.

5. **Compression Analysis**: Test different compression algorithms on various data types.

## Solutions

1. **Data Format Comparison:**
   ```
   JSON: Lightweight, widely supported, human-readable
   XML: Verbose, schema validation, namespace support
   YAML: Human-readable, configuration files, indentation-based
   
   Use Cases:
   - JSON: APIs, web applications, data exchange
   - XML: Enterprise systems, SOAP, document markup
   - YAML: Configuration files, DevOps tools, documentation
   ```

2. **TLS Configuration Best Practices:**
   - Use TLS 1.2 or higher
   - Disable weak cipher suites
   - Enable HSTS headers
   - Use strong key lengths (2048+ bit RSA, 256+ bit ECC)
   - Regular certificate renewal

3. **Certificate Chain Creation:**
   ```bash
   # Create root CA
   openssl genrsa -out rootCA.key 4096
   openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
   
   # Create intermediate CA
   openssl genrsa -out intermediateCA.key 4096
   openssl req -new -key intermediateCA.key -out intermediateCA.csr
   openssl x509 -req -in intermediateCA.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out intermediateCA.crt -days 500 -sha256
   
   # Create server certificate
   openssl genrsa -out server.key 2048
   openssl req -new -key server.key -out server.csr
   openssl x509 -req -in server.csr -CA intermediateCA.crt -CAkey intermediateCA.key -CAcreateserial -out server.crt -days 365 -sha256
   ```

4. **Encryption Comparison:**
   - **Symmetric**: Fast, efficient for large data, requires secure key sharing
   - **Asymmetric**: Slower, secure key exchange, digital signatures
   - **Hybrid**: Use asymmetric for key exchange, symmetric for data encryption

5. **Compression Analysis:**
   - **gzip**: Good general-purpose compression
   - **brotli**: Better compression ratios for web content
   - **lz4**: Fast compression/decompression
   - **zstd**: Good balance of speed and compression ratio

## Completion Checklist
- [ ] Understand data encoding and format conversion
- [ ] Know TLS/SSL handshake process and certificate management
- [ ] Can distinguish between symmetric and asymmetric encryption
- [ ] Familiar with hash functions and their applications
- [ ] Can analyze and validate certificates
- [ ] Understand compression techniques and trade-offs

## Key Takeaways
- Presentation layer handles data formatting, encryption, and compression
- TLS/SSL provides secure communication channels through encryption
- Certificate management is crucial for maintaining security
- Different encoding formats serve different purposes and use cases
- Proper encryption implementation requires understanding of both symmetric and asymmetric methods

## Next Steps
Proceed to [Day 8: Layer 7 - Application Layer](../Day_08/notes_and_exercises.md) to learn about HTTP, DNS, SMTP, REST APIs, and cloud applications.
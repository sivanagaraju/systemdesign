# TLS and Encryption

> Securing data in transit between distributed systems.

---

## 🔐 **Secret Decoder Ring Analogy**

Remember childhood secret decoder rings?

| Concept | Decoder Ring | TLS |
|---------|--------------|-----|
| **Encryption** | Scramble message | Encrypt data |
| **Key** | Ring setting | Shared secret |
| **Without key** | Gibberish | Secure transmission |

---

## 🎯 Why TLS?

```mermaid
graph TB
    subgraph "Without TLS"
        C1[Client] -->|Password: abc123| A[Attacker]
        A -->|Password: abc123| S1[Server]
    end
    
    subgraph "With TLS"
        C2[Client] -->|🔒 Encrypted| S2[Server]
    end
    
    style A fill:#f44336,color:#fff
```

---

## 📋 TLS Handshake

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: 1. ClientHello<br/>(supported ciphers)
    S->>C: 2. ServerHello<br/>(chosen cipher + certificate)
    C->>C: 3. Verify certificate
    C->>S: 4. Key exchange
    S->>C: 5. Finished
    
    Note over C,S: 🔒 Encrypted communication
```

---

## 📜 Certificates

### The Trust Chain

```mermaid
graph TB
    Root[Root CA<br/>Trusted by browsers]
    Inter[Intermediate CA]
    Leaf[Your Certificate<br/>*.example.com]
    
    Root -->|Signs| Inter
    Inter -->|Signs| Leaf
```

### 🏦 **Bank ID Analogy**

| Real World | TLS |
|------------|-----|
| Government issues ID | Root CA signs leaf |
| Bank checks your ID | Client verifies certificate |
| ID proves identity | Certificate proves server |

---

## 🔧 Types of Encryption

### Symmetric (Fast)

```mermaid
graph LR
    Plain[Hello] -->|Key: 🔑| Cipher[X#@!]
    Cipher -->|Key: 🔑| Plain2[Hello]
```

**Same key** for encrypt and decrypt. Fast but key distribution is hard.

### Asymmetric (Secure Key Exchange)

```mermaid
graph TB
    subgraph "Server"
        Private[Private Key 🔐]
        Public[Public Key 🔓]
    end
    
    Client -->|Encrypt with| Public
    Public -->|Only| Private
    Private -->|Can decrypt| Client
```

**Public key** encrypts, **private key** decrypts.

### TLS Uses Both!

```mermaid
graph LR
    A[Asymmetric] -->|Exchange| Key[Session Key]
    Key -->|Symmetric| Data[Fast data transfer]
```

---

## 🔒 mTLS (Mutual TLS)

Regular TLS: Server proves identity  
mTLS: **Both** prove identity

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    S->>C: Server certificate
    C->>C: Verify server ✅
    C->>S: Client certificate
    S->>S: Verify client ✅
    
    Note over C,S: Both authenticated!
```

**Used in**: Microservices, zero-trust networks

---

## 🔥 Real-World: Let's Encrypt

```mermaid
graph TB
    LE[Let's Encrypt<br/>Free CA]
    Auto[Automatic renewal]
    Wildcard[Wildcard certs]
    
    LE --> Auto
    LE --> Wildcard
    
    Note[80%+ of HTTPS sites<br/>use Let's Encrypt]
```

---

## 📊 TLS Versions

| Version | Status | Notes |
|---------|--------|-------|
| TLS 1.0 | ❌ Deprecated | Vulnerabilities |
| TLS 1.1 | ❌ Deprecated | Vulnerabilities |
| TLS 1.2 | ✅ Acceptable | Still widely used |
| TLS 1.3 | ✅ Recommended | Faster, more secure |

---

## ✅ Key Takeaways

1. **TLS** encrypts data in transit
2. **Certificates** prove server identity
3. **Asymmetric crypto** for key exchange, **symmetric** for data
4. **mTLS** = Both sides authenticate (microservices)
5. **Use TLS 1.2+**, prefer TLS 1.3

---

[← Previous: Network Layers](./01-network-layers.md) | [Next: Authentication →](./03-authentication-oauth.md)

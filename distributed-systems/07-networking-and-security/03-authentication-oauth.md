# Authentication and OAuth

> Proving identity in distributed systems.

---

## 🎫 **Concert Ticket Analogy**

| Concept | Concert | Distributed Systems |
|---------|---------|---------------------|
| **Authentication** | Show ID at box office | Prove you are who you claim |
| **Authorization** | Ticket shows your seat | What you're allowed to access |
| **Token** | Wristband | JWT, session token |

---

## 🎯 Authentication vs Authorization

```mermaid
graph LR
    AuthN[Authentication<br/>WHO are you?] --> AuthZ[Authorization<br/>WHAT can you do?]
    
    AuthN --> ID[Identity verified]
    AuthZ --> Perm[Permissions checked]
```

---

## 🔐 Common Authentication Methods

### 1. Basic Auth

```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Pros**: Simple  
**Cons**: Sends credentials every request (use HTTPS!)

### 2. Session-Based

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant DB as Session Store
    
    C->>S: Login (username/password)
    S->>DB: Store session
    S->>C: Set-Cookie: session_id=abc
    
    C->>S: Request + Cookie
    S->>DB: Lookup session
    DB->>S: User info
    S->>C: Response
```

### 3. Token-Based (JWT)

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Server
    participant R as Resource Server
    
    C->>A: Login
    A->>C: JWT token
    
    C->>R: Request + JWT
    R->>R: Verify JWT signature
    R->>C: Response
```

---

## 🎟️ JWT (JSON Web Tokens)

```
Header.Payload.Signature
```

```mermaid
graph TB
    subgraph "JWT Structure"
        H[Header<br/>Algorithm, type]
        P[Payload<br/>Claims: user, exp, etc.]
        S[Signature<br/>Verify integrity]
    end
    
    H --> P --> S
```

**Example Payload:**
```json
{
  "sub": "user123",
  "name": "Alice",
  "exp": 1609459200,
  "roles": ["admin", "user"]
}
```

---

## 🔄 OAuth 2.0 Flow

```mermaid
sequenceDiagram
    participant U as User
    participant A as Your App
    participant G as Google
    
    U->>A: Click "Login with Google"
    A->>G: Redirect to Google login
    G->>U: Login page
    U->>G: Enter credentials
    G->>A: Authorization code
    A->>G: Exchange code for token
    G->>A: Access token
    A->>G: Get user info (with token)
    G->>A: User profile
    A->>U: Logged in!
```

---

## 🏠 **Hotel Key Card Analogy**

| OAuth | Hotel |
|-------|-------|
| Authorization Server | Front desk |
| Access Token | Key card |
| Refresh Token | Can get new card if lost |
| Scope | Which rooms you can access |

---

## 🔧 Service-to-Service Auth

### API Keys

```
X-API-Key: abc123xyz
```

Simple but static. Rotate regularly.

### mTLS (Mutual TLS)

Both services verify each other's certificates.

### Service Mesh (Istio/Linkerd)

```mermaid
graph TB
    subgraph "Service Mesh"
        A[Service A] -->|mTLS| Proxy1[Sidecar]
        Proxy1 -->|mTLS| Proxy2[Sidecar]
        Proxy2 --> B[Service B]
    end
    
    Note[Proxies handle auth<br/>Services don't need to know]
```

---

## 📊 Comparison

| Method | Stateless? | Best For |
|--------|------------|----------|
| Session | ❌ No | Traditional web apps |
| JWT | ✅ Yes | APIs, microservices |
| OAuth | - | Third-party login |
| API Key | ✅ Yes | Server-to-server |
| mTLS | ✅ Yes | High-security services |

---

## ✅ Key Takeaways

1. **Authentication** = WHO, **Authorization** = WHAT
2. **JWT** = Stateless tokens with claims
3. **OAuth** = Delegated authorization
4. **mTLS** = Mutual authentication for services
5. **Service mesh** handles auth transparently

| Remember | Analogy |
|----------|---------|
| JWT | Concert wristband |
| OAuth | Hotel key card |
| mTLS | Two-way ID check |

---

[← Previous: TLS](./02-tls-and-encryption.md) | [Back to Module →](./README.md)

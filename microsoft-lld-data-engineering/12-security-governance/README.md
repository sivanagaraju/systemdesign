# 12 - Security & Governance

> **Data protection and access control**

---

## 🔐 Encryption

| Level | Implementation |
|-------|----------------|
| **At-rest** | Azure Storage encryption (default) |
| **In-transit** | TLS/HTTPS |
| **Column-level** | Encrypt sensitive columns (Always Encrypted) |

---

## 🎭 Data Masking

```sql
-- Dynamic masking (shows real data to privileged users)
CREATE TABLE employees (
    emp_id INT,
    ssn VARCHAR(11) MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)')
);
```

| Type | Use Case |
|------|----------|
| **Static** | Create masked copy for dev/test |
| **Dynamic** | Mask at query time based on user |
| **Tokenization** | Replace with reversible token |

---

## 👥 Access Control

### RBAC (Role-Based)
```
Storage Blob Data Reader → Read access to container
Storage Blob Data Contributor → Read/Write access
```

### ACL (Fine-Grained)
```
Directory: /pii-data/
├── owner: data-team (rwx)
├── group: analysts (r--)
└── other: (---)
```

### Row-Level Security
```sql
CREATE FUNCTION security_filter(@region VARCHAR(10))
RETURNS TABLE
AS RETURN SELECT 1 WHERE @region = USER_REGION();
```

---

## 📋 Governance

- **Catalog:** Azure Purview for data discovery
- **Lineage:** Track data flow from source to report
- **Classification:** Auto-tag PII, financial data

# Sensitive-Data "Lock Box" Architecture (Design Doc)

Design reference for handling sensitive information in the Elemetris RAG bot. The
governing idea: the **vector store indexes references and metadata, while the
actual sensitive information stays encrypted behind authorization checks**, and
the LLM (Claude) only ever receives the minimum information needed to answer.

This is a target architecture. See "Current state vs. this design" at the end for
what the bot does today and what would need to be added.

---

## High-level flow

```
User
  |
Application
  |
Access Control Layer
  |
+--------------------+
| Sensitive Vault    |
| (encrypted data)   |
+--------------------+
          |
          v
+--------------------+
| Retrieval Service  |
| Metadata/Embeddings|
+--------------------+
          |
          v
       LLM (Claude)
```

The stronger enterprise pattern is two-stage, where the LLM never touches the
vault directly:

```
User Query
    |
Policy Engine
    |
Document Search
    |
Sensitive Data Filter
    |
Context Builder
    |
LLM (Claude)
```

The application retrieves data, enforces policies, redacts sensitive fields, and
only then sends approved context to the model.

---

## Core principles

### 1. Never store secrets in the vector database

Instead of embedding raw secrets:

```
John Smith
SSN: 123-45-6789
Bank Account: 987654321
```

store searchable metadata and a reference only:

```json
{
  "customer_id": "cust_123",
  "document_type": "identity_record",
  "vault_reference": "vault://records/abc123"
}
```

The vector store (pgvector, in this app's `chunks` table) holds references and
metadata — not the actual secret.

### 2. Encrypt sensitive data in a vault

Use a dedicated secret/document vault rather than the application database:

- HashiCorp Vault
- Cloud KMS (AWS KMS, GCP KMS, Azure Key Vault)
- Encrypted database tables
- Hardware security modules (HSMs)

Stored shape:

```json
{
  "record_id": "abc123",
  "encrypted_payload": "..."
}
```

### 3. Separate retrieval from disclosure

The RAG pipeline becomes: search embeddings → find references → check
authorization → retrieve sensitive content only if permitted → pass only the
minimum to the LLM.

```python
results = retriever.search(query)

if user.has_permission(results.record):
    data = vault.get(results.record)
else:
    data = "ACCESS DENIED"
```

### 4. Field-level controls (least privilege)

Don't expose whole records:

```json
{ "name": "John Smith", "ssn": "123-45-6789", "salary": 250000 }
```

Return only what the task needs:

```json
{ "name": "John Smith" }
```

### 5. Retrieval policies

Run a guard before any content reaches the context builder:

```python
def retrieval_guard(user, document):
    if document.classification == "restricted":
        return user.role == "security_admin"
    return True
```

### 6. Auditing

Log who requested data, what was retrieved, why, and what was sent to the model:

```json
{
  "user": "alice",
  "query": "show customer record",
  "retrieved_docs": ["abc123"],
  "timestamp": "2026-06-09T10:00:00Z"
}
```

---

## Worked example

Vector DB (safe to index):

```json
{
  "employee_id": "e123",
  "department": "Finance",
  "skills": ["tax", "audit"],
  "vault_ref": "vault://employees/e123"
}
```

Vault (encrypted, behind authorization):

```json
{ "salary": 250000, "ssn": "...", "home_address": "..." }
```

Query: *"Who in Finance has audit experience?"* → the system retrieves metadata
and returns names and skills. Salary, SSN, and addresses never enter the LLM
context.

---

## Recommended stack

| Concern | Choice |
|---|---|
| Embeddings / vector store | pgvector (in use), Weaviate, or Pinecone |
| Secrets management | HashiCorp Vault or cloud KMS |
| Authorization | RBAC or ABAC |
| LLM orchestration | LangChain or LlamaIndex |
| Audit logging | Dedicated logging / SIEM platform |

---

## Current state vs. this design

What the Elemetris bot does **today**:

- Uses pgvector for retrieval (`chunks` table) — matches the recommended store.
- Stores uploaded document text and embeddings directly in Postgres; there is
  **no separation** between "metadata/reference" and "sensitive payload."
- Has **no user accounts, roles, or per-user authorization** — `/admin` is a
  single shared password; the chat endpoint is anonymous.
- Has lightweight **output scrubbing** (`RESTRICTED_NAMES`) and basic request
  logging, but no structured audit trail of what context was sent to the model.

What this design would require **adding**:

1. **Identity + authorization layer** (RBAC/ABAC) — the biggest prerequisite.
   The vault pattern's permission checks assume known users with roles, which
   the current single-persona, anonymous app does not have.
2. **A separate vault** for encrypted payloads (HashiCorp Vault or a cloud KMS),
   with only references/metadata stored in pgvector.
3. **A retrieval guard + context builder** that enforces policy and redacts
   fields before assembling the prompt sent to Claude.
4. **Structured audit logging** of retrievals and model inputs.

### Incrementally adoptable pieces (no auth layer required)

These can land in the current app without building accounts first:

- **Keep secrets out of the vector store**: a pre-ingest redaction/classifier
  step so PII/secrets are never embedded.
- **Field/PII redaction before context assembly**: strip detected secrets from
  retrieved chunks before they reach Claude.
- **Audit log**: record query, retrieved document IDs, and the assembled context
  for each request.

Sequence the full build only after an identity layer exists; until then, the
three incremental pieces above give most of the data-protection benefit.

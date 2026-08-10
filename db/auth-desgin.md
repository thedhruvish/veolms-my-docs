# authentication  DB

*Counterproposal to [Authentication DB by nilesh_yadav and bhupeshb7](https://chat.veolms.org/veolms/pl/xw66spxdct8qprsyuxd18ikpar)*

| Status | Version | Date |
|---|---:|---|
| Proposed | 1.0 | 10 August 2026 |
## 1. Soft Delete Support

Add a `deleted_at` field to tables where records may need to be retained after deletion.

```sql
deleted_at TIMESTAMP
```

The purpose of `deleted_at` is to support **soft deletion** instead of permanently removing records.

This is particularly useful for users because the `users` table may have relationships with many other tables. Permanently deleting a user could either:

* Break relationships with related records.
* Require cascading deletes.
* Cause permanent loss of historical/audit data.

With `deleted_at`, the user can be considered deleted while their historical data and relationships remain intact.

### Recommended convention

```sql
created_at TIMESTAMP NOT NULL DEFAULT NOW(),
updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
deleted_at TIMESTAMP
```

A `NULL` value means the record is active.

A non-NULL value means the record has been soft deleted.

---

# 2. Authentication Methods

The authentication system should support both:

1. **TOTP**
2. **Passkeys / WebAuthn**

A user can use either authentication method.

The credentials should be stored in separate tables because Passkey and TOTP have completely different credential structures.

```text
users
  │
  ├── totp_credentials
  │
  └── passkey_credentials
```

---

# 3. TOTP Credentials

```sql
CREATE TABLE totp_credentials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    user_id UUID NOT NULL UNIQUE
        REFERENCES users(id)
        ON DELETE CASCADE,

    -- Encrypted TOTP secret.
    secret BYTEA NOT NULL,

    enabled BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    last_used_at TIMESTAMP,
    disabled_at TIMESTAMP,
    deleted_at TIMESTAMP
);
```

### Important fields

| Field          | Type      | Purpose                             |
| -------------- | --------- | ----------------------------------- |
| `id`           | UUID      | Internal credential ID              |
| `user_id`      | UUID      | User who owns the TOTP credential   |
| `secret`       | BYTEA     | Encrypted TOTP secret               |
| `enabled`      | BOOLEAN   | Whether TOTP is currently enabled   |
| `created_at`   | TIMESTAMP | Credential creation time            |
| `updated_at`   | TIMESTAMP | Last credential update              |
| `last_used_at` | TIMESTAMP | Last successful TOTP authentication |
| `disabled_at`  | TIMESTAMP | When TOTP was disabled              |
| `deleted_at`   | TIMESTAMP | Soft deletion                       |

`user_id UNIQUE` means one user can have one TOTP credential.

The TOTP secret must be **encrypted**, not hashed, because the server needs to retrieve the secret to verify TOTP codes.

---

# 4. Passkey Credentials

```sql
CREATE TABLE passkey_credentials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    user_id UUID NOT NULL
        REFERENCES users(id)
        ON DELETE CASCADE,

    -- WebAuthn credential ID
    credential_id BYTEA NOT NULL UNIQUE,

    -- WebAuthn public key
    public_key BYTEA NOT NULL,

    -- Signature counter
    sign_count BIGINT NOT NULL DEFAULT 0,

    -- Usually "public-key"
    credential_type VARCHAR(50) NOT NULL DEFAULT 'public-key',

    -- Authenticator AAGUID
    aaguid UUID,

    -- Example:
    -- ["internal"]
    -- ["usb"]
    -- ["nfc"]
    -- ["ble"]
    transports JSONB,

    -- Whether the credential is backed up/synchronized
    backed_up BOOLEAN NOT NULL DEFAULT FALSE,

    -- User-friendly device name
    device_name VARCHAR(255),

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    last_used_at TIMESTAMP,

    -- When this credential was revoked
    revoked_at TIMESTAMP,

    -- Soft deletion
    deleted_at TIMESTAMP,

    UNIQUE(user_id, credential_id)
);
```

---

# 5. Soft Delete vs Revocation

For Passkeys, there are two different concepts:

### `revoked_at`

Means:

> This credential should no longer be allowed to authenticate.

### `deleted_at`

Means:

> This database record has been logically deleted.

For example:

```text
Passkey
   │
   ├── revoked_at = 2026-08-10
   │
   └── deleted_at = NULL
```

The record still exists for audit/history, but authentication must reject it.

For a normal user-facing "remove passkey" operation, you could set:

```sql
revoked_at = NOW()
```

and optionally later:

```sql
deleted_at = NOW()
```

---

## 6. Now user table like this

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    email VARCHAR(255) NOT NULL UNIQUE,

    name VARCHAR(255),

    password_hash: string | null; -- null for pure OAuth users,

    email_verified_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP
);
```

---

This gives you a clean separation between **TOTP credentials** and **Passkey credentials**, while allowing the same user to authenticate through either method.

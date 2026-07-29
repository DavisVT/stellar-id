# Architecture

## Overview

StellarID is a single Soroban smart contract providing shared identity infrastructure on Stellar. All state lives on-chain. There is no backend server or database — consumers query the contract directly.

## Core Concepts

### Issuers
Entities approved by the contract admin to issue credentials. Each issuer has a trust level (1–100) that influences the reputation score of subjects they credential. Issuers can authorize sub-issuers to issue on their behalf.

### Schemas
Credential types defined by issuers. A schema has a name and description — for example "KYC Verified", "Accredited Investor", or "Merchant". Any approved issuer can register schemas. Schemas are immutable once registered.

### Credentials
Individual attestations issued to a subject wallet address. Each credential references a schema, has an issuer, and optionally has an expiry timestamp. Credentials can be revoked but never deleted — revocation is recorded permanently.

### Identities
Automatically created the first time a credential is issued to a wallet. Tracks credential count and a reputation score derived from how many credentials exist and the trust levels of the issuers.

## Data Model

```
Admin (instance storage)
  └── Controls issuer registry

Issuer (persistent storage, keyed by Address)
  ├── name
  ├── trust_level (1-100)
  ├── active
  └── credential_count

Schema (persistent storage, keyed by schema_id u32)
  ├── name
  ├── description
  ├── issuer
  └── active

Credential (persistent storage, keyed by credential_id u64)
  ├── subject
  ├── issuer
  ├── schema_id
  ├── issued_at
  ├── expires_at (0 = no expiry)
  └── revoked

Identity (persistent storage, keyed by subject Address)
  ├── credential_count
  ├── reputation_score
  └── created_at

SubjectCredentials (persistent storage, keyed by subject Address)
  └── Vec<u64> of credential IDs

SubIssuer (persistent storage, keyed by (parent, sub) tuple)
  └── bool — authorized or not
```

## Storage Strategy

- `instance()` — used for global counters (CredentialCount, SchemaCount) and Admin
- `persistent()` — used for all user data (Issuers, Schemas, Credentials, Identities, Accumulators)

Persistent storage entries have their own TTL and survive contract upgrades. Instance storage is tied to the contract instance.

## Access Control

| Function | Caller |
|---|---|
| `initialize` | Anyone (once) |
| `register_issuer` | Admin only |
| `deactivate_issuer` | Admin only |
| `authorize_sub_issuer` | Parent issuer only |
| `revoke_sub_issuer` | Parent issuer only |
| `register_schema` | Active issuer only |
| `issue_credential` | Active registered issuer only |
| `revoke_credential` | Original issuer only |
| All `get_*` / `has_*` / `verify_*` | Anyone — no auth required |

---

## Anonymous XOR Accumulator & Membership Proofs

StellarID maintains a privacy-preserving cryptographic accumulator for each credential schema.

### Accumulator Formula
For a set of credential holders `{addr1, addr2, ..., addrN}`:
```
acc_0 = 0x00...00 (32 bytes of 0s)
acc_k = SHA-256( acc_{k-1} XOR SHA-256(subject_bytes) )
```

### Verification & Witness
- `generate_membership_witness(env, subject, schema_id)` returns `(subject_hash, accumulator)`.
- `verify_membership_witness(env, schema_id, subject_hash, snapshot)` verifies whether `subject_hash` matches an active credential holder at `snapshot`.
- `get_schema_holder_count(env, schema_id)` exposes total active holder count without revealing wallet addresses.

### Limitations
This scheme relies on snapshot consistency. Off-chain verifiers should request recent accumulator snapshots to verify membership.

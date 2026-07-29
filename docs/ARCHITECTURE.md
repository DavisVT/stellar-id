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
- `persistent()` — used for all user data (Issuers, Schemas, Credentials, Identities, DelegationLinks)

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
| `issue_credential` | Active registered/delegated issuer only |
| `revoke_credential` | Original issuer only |
| `create_delegation` | Parent issuer only |
| `revoke_delegation` | Parent issuer only |
| All `get_*` / `has_*` / `verify_*` | Anyone — no auth required |

---

## Transitive Credential Delegation Chain

StellarID supports multi-hop hierarchical delegation chains with depth limits and trust decay.

### Delegation Link
A `DelegationLink` defines `(parent, delegate, max_depth, trust_fraction)`.

### Trust Decay Calculation
Trust decays across delegate hops using fixed-point integer math:
```
delegated_trust = (root_trust × fraction_1 / 10000 × fraction_2 / 10000 × ... × fraction_k / 10000)
```
where `trust_fraction` is expressed in basis points (max 10,000 = 100%).

### Safety Mechanisms
- **Circular Delegation**: `create_delegation` walks the parent chain and rejects circular loops.
- **Depth Limit**: Rejects delegation chains exceeding `max_depth`.
- **Cascading Revocation**: Calling `revoke_delegation(parent, delegate)` recursively revokes the link and all sub-delegate descendant links.

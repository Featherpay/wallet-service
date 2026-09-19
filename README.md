# Featherpay Wallet Service

**Embedded, non-user-facing key custody and transaction signing for Featherpay.**

This service generates and safeguards Stellar keypairs on behalf of every Featherpay user — creators and tippers alike — so that nobody ever installs a wallet extension, sees a public key, or manages a seed phrase to send or receive a payment as small as $0.50. It is the single most security-critical component in the Featherpay org and is architected, reviewed, and operated with that in mind.

> 🔒 **Internal service. No public ingress.** This service is only ever reachable from [`featherpay/api`](https://github.com/featherpay/api) over an authenticated, internal channel. It is never deployed with a public endpoint, in any environment, under any circumstances.

---

## Table of Contents

- [Overview](#overview)
- [Design Philosophy](#design-philosophy)
- [Architecture](#architecture)
- [Data Model](#data-model)
- [Repo Structure](#repo-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Internal API Reference](#internal-api-reference)
- [Key Lifecycle](#key-lifecycle)
- [Recovery Flow](#recovery-flow)
- [Security Model](#security-model)
- [Observability](#observability)
- [Testing](#testing)
- [Deployment](#deployment)
- [Incident Response](#incident-response)
- [Contributing](#contributing)
- [FAQ](#faq)
- [License](#license)

---

## Overview

When a user signs up for Featherpay — whether as a creator receiving tips or a tipper sending them — [`featherpay/api`](https://github.com/featherpay/api) calls this service to provision a Stellar keypair for them. From that point forward:

- The **public key** (Stellar account address) is known to `api` and stored in its own database, associated with the user
- The **private key** never leaves this service. It is encrypted at rest, decrypted only transiently in memory at signing time, and is never returned in any API response, log line, or error message
- Every transaction Featherpay ever submits on a user's behalf is built by `api`, sent here as an unsigned XDR payload, signed here, and returned as a signed envelope — `api` never sees the key, only the result of using it

This service does not decide *what* to sign — it has no concept of tips, balances, or business logic. Its only job is: generate keys safely, store them safely, sign with them safely, and help a legitimate user recover access if they lose it.

## Design Philosophy

A few principles shape every decision in this repo:

1. **Least knowledge.** This service knows as little as possible about the rest of the system. It doesn't know what a "tip" is. It signs XDR blobs for a `user_id`. This minimizes what an attacker gains even in a partial compromise.
2. **No homegrown cryptography.** Encryption at rest uses cloud KMS envelope encryption — an established, audited primitive — rather than any custom encryption scheme written for this project.
3. **Defense in depth over a single strong wall.** Network isolation (no public ingress) + service-to-service auth + encryption at rest + audit logging + anomaly detection are all present simultaneously, so that no single control failing is catastrophic on its own.
4. **Boring is good.** This is not the place for architectural cleverness. Every decision here should be the well-trodden, conservative option.

## Architecture

```
                    (no public ingress — this box is unreachable
                     from the internet in every environment)

┌──────────────┐  mTLS / signed service token   ┌───────────────────┐
│ featherpay/api │ ──────────────────────────────► │   wallet-service    │
│               │                                  │                     │
│               │ ◄──────────────────────────────  │  ┌───────────────┐  │
└──────────────┘         signed XDR envelope       │  │ signing engine │  │
                                                     │  └───────┬───────┘  │
                                                     │          │          │
                                                     │  ┌───────▼───────┐  │
                                                     │  │  key manager    │  │
                                                     │  └───────┬───────┘  │
                                                     └──────────┼──────────┘
                                                                │
                                        ┌───────────────────────┼───────────────────────┐
                                        ▼                       ▼                        ▼
                                ┌──────────────┐      ┌──────────────┐       ┌──────────────┐
                                │  Cloud KMS      │      │  Encrypted     │       │  Audit log     │
                                │  (envelope       │      │  key storage    │       │  store          │
                                │  encryption)      │      │  (Postgres)     │       │  (append-only)  │
                                └──────────────┘      └──────────────┘       └──────────────┘
```

## Data Model

| Table | Key Columns | Notes |
|---|---|---|
| `wallets` | `user_id`, `public_key`, `encrypted_private_key`, `kms_key_id`, `created_at` | One row per user. `encrypted_private_key` is ciphertext only — the plaintext key never touches this table or any log |
| `signing_requests` (audit log) | `id`, `user_id`, `requested_by_service`, `transaction_hash`, `signed_at`, `outcome` | Append-only. Every signing attempt is recorded, success or failure |
| `recovery_attempts` | `id`, `user_id`, `initiated_at`, `verification_method`, `outcome`, `ip_address` | Rate-limited and monitored — see [Recovery Flow](#recovery-flow) |

No table in this service ever stores a plaintext private key, a seed phrase, or a recoverable representation of key material outside of KMS-protected ciphertext.

## Repo Structure

```
wallet-service/
├── src/
│   ├── keys/
│   │   ├── generate.ts        # Keypair generation
│   │   ├── encrypt.ts          # Envelope encryption via KMS
│   │   └── decrypt.ts           # In-memory-only decryption for signing
│   ├── signing/
│   │   ├── sign.ts              # Core signing logic
│   │   └── validate.ts           # Validates incoming XDR before signing (sanity checks, not business logic)
│   ├── recovery/
│   │   ├── initiate.ts
│   │   └── verify.ts
│   ├── auth/
│   │   └── service-auth.ts       # Verifies requests genuinely come from featherpay/api
│   └── audit/
│       └── log.ts                 # Append-only audit logging
├── docs/
│   ├── threat-model.md            # Required reading before contributing — see below
│   └── incident-response.md
├── tests/
│   ├── unit/
│   └── security/                   # Secret-leakage and encryption round-trip tests
├── PLAN.md
└── README.md
```

## Prerequisites

- Node.js 20+
- Access to a cloud KMS provider (AWS KMS or GCP KMS) for staging/production — **local development uses a mock KMS module and must never be pointed at real KMS credentials**
- PostgreSQL 15+

## Getting Started

```bash
git clone https://github.com/featherpay/wallet-service.git
cd wallet-service
npm install

cp .env.example .env
# KMS_PROVIDER=mock is the required setting for local dev — do not change this locally

npm run migrate
npm run dev
```

Before your first contribution, read `docs/threat-model.md` in full. It is not optional reading for this repo.

## Environment Variables

| Variable | Description | Required |
|---|---|---|
| `KMS_PROVIDER` | `mock` for local dev, `aws` or `gcp` for staging/production | Yes |
| `KMS_KEY_ID` | The KMS key used for envelope encryption of user keys | Yes (non-mock) |
| `DATABASE_URL` | Postgres connection string for wallet and audit storage | Yes |
| `API_SERVICE_TOKEN_PUBLIC_KEY` | Public key used to verify that incoming requests are genuinely signed by `featherpay/api` | Yes |
| `RECOVERY_RATE_LIMIT_PER_HOUR` | Max recovery attempts per user per hour before lockout | Yes |
| `AUDIT_LOG_RETENTION_DAYS` | How long audit logs are retained before archival | Yes |
| `LOG_LEVEL` | Logging verbosity — never set above `info` in production; debug logs must never include request bodies | Yes |

See `.env.example` for a complete annotated list.

## Internal API Reference

This is an internal-only API, not exposed publicly. All endpoints require a valid service token signed by `featherpay/api`.

| Endpoint | Method | Description |
|---|---|---|
| `/wallets` | `POST` | Provision a new wallet for a `user_id`. Returns the public key only. |
| `/wallets/:user_id/sign` | `POST` | Accepts an unsigned transaction XDR, returns the signed envelope. Never returns key material. |
| `/wallets/:user_id/public-key` | `GET` | Returns the public key for a user (used for display/verification purposes by `api`) |
| `/recovery/initiate` | `POST` | Begins the email-based recovery flow for a user who has lost access |
| `/recovery/verify` | `POST` | Completes recovery after identity verification |

Every request and response is logged to the append-only audit store, including failures — a failed signing attempt is as important to know about as a successful one.

## Key Lifecycle

1. **Generation** — a keypair is generated using the Stellar SDK's key generation utilities inside this service's process memory
2. **Encryption** — the private key is immediately encrypted using envelope encryption: a KMS-managed key encrypts a locally-generated data key, which encrypts the private key; only the encrypted data key and encrypted private key are persisted
3. **Storage** — the encrypted private key and encrypted data key are written to Postgres; the plaintext key and plaintext data key are discarded from memory immediately after encryption completes
4. **Signing** — on a sign request, the data key is decrypted via KMS, used to decrypt the private key in memory, used to sign the transaction, and then all plaintext material is zeroed out of memory before the function returns
5. **Rotation** — KMS keys used for envelope encryption are rotated on a schedule (see `docs/threat-model.md` for the current policy); user keypairs themselves are not rotated (a Stellar account's signing key can be rotated on-chain if ever needed, but this is a deliberate, rare, audited operation, not a routine one)

## Recovery Flow

Because users never see or manage their own private key, losing access to their email is the primary recovery scenario this service needs to handle gracefully and safely:

1. User initiates recovery via `api`, which forwards the request here with a verified email address
2. This service sends a time-limited verification link to that email
3. On verification, this service re-associates signing rights with the existing keypair — **it never generates a new keypair during recovery**, since that would abandon any funds already on the original account
4. All recovery attempts are rate-limited per user and monitored for anomalous patterns (e.g. many recovery attempts across different users from the same IP in a short window)
5. Accounts above a configurable balance threshold may require a secondary verification factor before recovery completes (see `PLAN.md` M3)

Recovery is one of the most attractive attack surfaces for a custodial service — it is treated with the same rigor as the signing path itself, not as an afterthought.

## Security Model

- **No public ingress**, in any environment. Verified as part of every deployment's infrastructure review.
- **No plaintext key material** in logs, error messages, stack traces, or version control — enforced by automated secret-scanning in CI, on every commit, not just at merge time.
- **Service-to-service authentication only** — `api` authenticates via mTLS or signed, short-lived service tokens; there is no username/password or static API key for this service.
- **Append-only audit logging** of every signing and recovery attempt, success or failure, including the identity of the calling service and timestamp.
- **Two-reviewer rule** — any pull request touching `src/keys/` or `src/signing/` requires sign-off from two reviewers, not the standard single approval.
- **No homegrown crypto** — all encryption relies on cloud KMS primitives.

Found a vulnerability? **Do not open a public GitHub issue.** Email `security@featherpay.io` immediately with details; we operate a responsible disclosure process and will respond promptly.

## Observability

- **Audit log** — every key access and signing attempt, queryable by `user_id`, time range, and outcome
- **Anomaly alerting** — sudden spikes in signing volume for a single account, unusual recovery attempt patterns, or KMS errors all trigger alerts
- **Latency and error-rate dashboards** on the signing endpoint, since it sits on the critical path of every payment in the product

## Testing

```bash
npm run test                # Unit tests, including encryption/decryption round-trip correctness
npm run test:security        # Automated checks: no secrets appear in logs or error output under any test scenario
npm run test:recovery         # Recovery flow edge cases, including rate-limit enforcement
```

Test fixtures use clearly-fake, deterministic test keypairs only — real key material must never appear in a test file, fixture, or commit history, even for a "throwaway" test account.

## Deployment

This service deploys to a private network segment with no public-facing load balancer or ingress rule. Deployment pipelines include an automated check that fails the deploy if any public ingress configuration is detected.

```bash
docker build -t featherpay-wallet-service .
# Deployed via internal infra pipeline — see docs/deployment.md (internal)
```

## Incident Response

See `docs/incident-response.md` for the full runbook. At a high level: any suspected compromise of this service triggers an immediate freeze on signing for affected accounts, a KMS key rotation, and a full audit-log review before service is restored. This runbook is rehearsed, not just written.

## Contributing

1. Read `docs/threat-model.md` before opening any PR that touches this repo
2. Branch from `main`
3. Any change to `src/keys/` or `src/signing/` requires two reviewer approvals
4. Never include real key material in tests, fixtures, commit messages, or PR descriptions
5. If you're unsure whether a change is "security-sensitive," treat it as if it is and ask for the second review anyway

## FAQ

**Why is this a separate service instead of a module inside `api`?**
Isolation. Keeping custody logic in its own service with its own network boundary, its own access controls, and its own review bar limits the blast radius of a bug or breach anywhere else in the system.

**Why custodial instead of non-custodial?**
The target user for Featherpay's tipping flow — a casual reader or viewer tipping a creator — will not install a wallet or manage a seed phrase. Custodial embedded wallets are the tradeoff that makes the product usable by that audience. See the [org-level README](https://github.com/featherpay/.github) for the fuller reasoning and the regulatory implications this carries.

**What happens if KMS is unavailable?**
Signing fails closed — no fallback to unencrypted local key material under any circumstance. This is a deliberate availability-vs-security tradeoff.

## License
MIT

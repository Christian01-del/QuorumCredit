# Credential Flow — Sequence, Swimlane and API Call Sequences

This guide walks the end-to-end life of a credential in QuorumCredit: how one is
issued, how it is verified (identity, documents, challenges), how a holder proves
it to a third party offline, and which HTTP calls each step performs.

It complements:

- [`docs/credential-policy-guide.md`](credential-policy-guide.md) — capability tiers and thresholds
- [`docs/CREDENTIAL_METADATA_ENCRYPTION.md`](CREDENTIAL_METADATA_ENCRYPTION.md) — metadata encryption
- [`docs/credential-privacy-policy.md`](credential-privacy-policy.md) — privacy helpers

> All diagrams below are [Mermaid](https://mermaid.js.org/) and render inline in
> the GitHub UI (pan/zoom is available from the viewer). They are generated from
> the same source of truth as the code, so they stay editable as text alongside
> the implementation rather than as exported images.

## Actors

| Actor | Description | Code |
|-------|-------------|------|
| **Holder** | The subject a credential is issued to; submits documents and challenges | `Credential.holderId` |
| **Issuer** | Issues credentials for a holder | `credentialStore.issueCredential(...)` |
| **Broadcast API** | The QuorumCredit Node server that exposes the credential HTTP surface | `server/src/http/routes.ts` |
| **Credential store** | In-memory credential + verification records | `server/src/credentials/credentialStore.ts` |
| **Identity service** | Document verification, challenges, health scoring | `server/src/credentials/identityVerificationService.ts` |
| **Proof generator** | Signs and validates exportable credential proofs | `server/src/credentials/proofGenerator.ts` |
| **Third party** | Consumes an exported proof without API access | `POST /credentials/:credentialId/proof/validate` |

## Credential Lifecycle

```mermaid
stateDiagram-v2
    [*] --> active: issueCredential(holderId, type, issuer, expiresAt)
    active --> pending: verify-identity / document-verification / challenges
    pending --> verified: verification passes
    pending --> rejected: verification fails
    verified --> re_verification_required: now > nextVerificationRequired
    re_verification_required --> pending: holder re-submits
    active --> expired: now > expiresAt
    active --> revoked: revokeCredential(credentialId)
    verified --> expired: now > expiresAt
    verified --> revoked: revokeCredential(credentialId)
    expired --> [*]
    revoked --> [*]
```

`nextVerificationRequired` is set **365 days** after each recorded verification
(`credentialStore.recordVerification`), independent of the credential's own
`expiresAt`.

## Sequence 1 — Credential Issuance

Credential issuance is an **in-process call**, not an HTTP endpoint: the store
exposes `issueCredential` / `revokeCredential`, and there is currently no REST
route that wraps them. Verification and proof endpoints (below) are the HTTP
surface.

```mermaid
sequenceDiagram
    autonumber
    participant I as Issuer (in-process)
    participant S as credentialStore
    participant H as Holder

    I->>S: issueCredential(holderId, type, issuer, expiresAt, metadata)
    S->>S: id = cred_<n>_<timestamp>
    S-->>I: Credential { status: "active", ... }
    I-->>H: deliver credential id
    Note over S: stored in the in-memory Map until revoked or expired
```

## Sequence 2 — Identity Verification

```mermaid
sequenceDiagram
    autonumber
    participant H as Holder
    participant API as Broadcast API
    participant S as credentialStore
    participant M as metrics

    H->>API: POST /credentials/verify-identity { credentialId, holderId, documentType }
    API->>S: getCredential(credentialId)
    alt credential missing
        API-->>H: 404 { error: "credential not found" }
    else holder mismatch
        API-->>H: 403 { error: "holder ID does not match credential" }
    else accepted
        API->>S: recordVerification(credentialId, holderId, documentType, expiresAt)
        S->>S: nextVerificationRequired = now + 365d
        API->>M: qc_identity_verifications_initiated_total
        API-->>H: 201 { status: "pending", verificationId, message }
    end

    H->>API: GET /credentials/:credentialId/verification-status
    API->>S: getVerification(credentialId)
    API-->>H: 200 VerificationRecord | 404 { error: "verification not found" }
```

## Sequence 3 — Document Verification

```mermaid
sequenceDiagram
    autonumber
    participant H as Holder
    participant API as Broadcast API
    participant S as credentialStore
    participant IV as identityVerificationService

    H->>API: POST /credentials/:credentialId/document-verification { documentType, documentHash, expiresAt?, metadata? }
    API->>API: validate documentType in [passport, driver_license, national_id, utility_bill, bank_statement]
    API->>S: getCredential(credentialId)
    alt unknown documentType or missing documentHash
        API-->>H: 400 { error: "documentType must be one of ..." }
    else credential missing
        API-->>H: 404 { error: "credential not found" }
    else accepted
        API->>IV: verifyDocument(credentialId, documentType, documentHash, expiresAt, metadata)
        IV-->>API: DocumentVerification { verifiedAt, ... }
        API-->>H: 201 DocumentVerification
    end

    H->>API: GET /credentials/:credentialId/documents
    API->>IV: getDocumentVerifications(credentialId) + isDocumentValid(credentialId)
    API-->>H: 200 { credentialId, documents, isValid }
```

## Sequence 4 — Challenge-Based Verification

```mermaid
sequenceDiagram
    autonumber
    participant H as Holder
    participant API as Broadcast API
    participant S as credentialStore
    participant IV as identityVerificationService

    H->>API: POST /credentials/:credentialId/challenges { holderId, challengeType, metadata? }
    API->>S: getCredential(credentialId)
    alt credential missing
        API-->>H: 404 { error: "credential not found" }
    else accepted
        API->>IV: createChallenge(credentialId, holderId, challengeType)
        Note right of IV: challengeType in<br/>face_match | document_liveness | manual_review
        IV-->>API: VerificationChallenge { status: "pending" }
        API-->>H: 201 VerificationChallenge
    end

    H->>API: POST /credentials/:credentialId/challenges/:challengeId/complete { passed, metadata? }
    API->>IV: completeChallenge(challengeId, passed, metadata)
    alt challenge missing
        API-->>H: 404 { error: "challenge not found" }
    else completed
        IV->>IV: status = passed ? "passed" : "failed"
        API-->>H: 200 VerificationChallenge
    end
```

## Sequence 5 — Proof Export and Third-Party Validation

```mermaid
sequenceDiagram
    autonumber
    participant H as Holder
    participant API as Broadcast API
    participant PG as proofGenerator
    participant T as Third party

    H->>API: GET /credentials/:credentialId/proof
    API->>PG: generateProof(authSecret, id, holderId, type, issuer, expiresAt, metadata)
    PG->>PG: HMAC-SHA256 over payload, proofId = randomBytes(16)
    API-->>H: 200 CredentialProof { credentialId, signature, proofId, ... }

    H->>API: POST /credentials/:credentialId/proof/export
    API->>PG: exportProofJson(proof)
    API-->>H: 200 attachment proof-<credentialId>.json

    H-->>T: hand over the exported proof (offline)
    T->>API: POST /credentials/:credentialId/proof/validate { proofJson }
    API->>PG: importProofJson(proofJson, authSecret)
    PG-->>API: ProofValidationResult { valid, reason?, payload? }
    API-->>T: 200 { valid: true, proof } | 400 { valid: false, reason }
```

## Swimlane — Who Does What

```mermaid
flowchart TB
    subgraph HOLDER["Holder"]
        H1["Receive credential id"]
        H2["Submit document + hash"]
        H3["Complete challenge"]
        H4["Export signed proof"]
    end

    subgraph API["Broadcast API (server/src/http/routes.ts)"]
        A1["Validate request body"]
        A2["Resolve credential"]
        A3["Emit metrics counter"]
        A4["Return status + JSON"]
    end

    subgraph STORE["credentialStore"]
        S1["issueCredential"]
        S2["recordVerification"]
        S3["getVerificationStats"]
    end

    subgraph IDENTITY["identityVerificationService"]
        I1["verifyDocument"]
        I2["createChallenge / completeChallenge"]
        I3["getVerificationReport (health score)"]
    end

    subgraph THIRD["Third party"]
        T1["Validate exported proof offline"]
    end

    H1 --> S1
    H2 --> A1
    H3 --> A1
    H4 --> A1
    A1 --> A2 --> S2
    A2 --> I1
    A2 --> I2
    A2 --> I3
    A3 --> A4
    H4 --> T1
    T1 --> A2
```

## API Call Sequences

All paths below are relative to the server root (the router has **no** `/api/v1`
prefix). `:credentialId` / `:holderId` segments are URL-decoded before lookup.

### Proof endpoints

| Step | Method | Path | Request | Success | Errors |
|------|--------|------|---------|---------|--------|
| 1 | `GET` | `/credentials/:credentialId/proof` | — | `200 CredentialProof` | `404 credential not found` |
| 2 | `POST` | `/credentials/:credentialId/proof/export` | — | `200` JSON attachment `proof-<credentialId>.json` | `404 credential not found` |
| 3 | `POST` | `/credentials/:credentialId/proof/validate` | `{ proofJson: string }` | `200 { valid: true, reason, proof }` | `400 { error: "proofJson required" }`, `400 { valid: false, reason }` |

### Verification endpoints

| Step | Method | Path | Request | Success | Errors |
|------|--------|------|---------|---------|--------|
| 4 | `POST` | `/credentials/verify-identity` | `{ credentialId, holderId, documentType }` | `201 { status: "pending", verificationId }` | `400 missing fields`, `403 holder mismatch`, `404 credential not found` |
| 5 | `GET` | `/credentials/:credentialId/verification-status` | — | `200 VerificationRecord` | `404 verification not found` |
| 6 | `GET` | `/credentials/:holderId/verify-all` | — | `200 { stats, credentialsNeedingReVerification }` | — |
| 7 | `POST` | `/credentials/:credentialId/document-verification` | `{ documentType, documentHash, expiresAt?, metadata? }` | `201 DocumentVerification` | `400 invalid documentType / documentHash`, `404 credential not found` |
| 8 | `GET` | `/credentials/:credentialId/documents` | — | `200 { credentialId, documents, isValid }` | — |
| 9 | `POST` | `/credentials/:credentialId/challenges` | `{ holderId, challengeType, metadata? }` | `201 VerificationChallenge` | `404 credential not found` |
| 10 | `POST` | `/credentials/:credentialId/challenges/:challengeId/complete` | `{ passed, metadata? }` | `200 VerificationChallenge` | `404 challenge not found` |
| 11 | `GET` | `/credentials/:credentialId/verification-report` | — | `200 report { verificationScore, status }` | — |

`GET /credentials/:credentialId/verification-report` returns
`{ credentialId, documents, schedule, challenges, verificationScore, isReVerificationDue, status, lastUpdated }`.
`status` is derived from `verificationScore`
(`identityVerificationService.getVerificationScore`): `score >= 80 → verified`,
`score >= 50 → at_risk`, otherwise `failed`.

### Holder analytics endpoints

| Step | Method | Path | Request | Success |
|------|--------|------|---------|---------|
| 12 | `GET` | `/analytics/credentials/:holderId` | — | `200 dashboard data` |
| 13 | `GET` | `/analytics/credentials/:holderId/stats` | — | `200 { credentialStats, verificationStats, topTypes, healthScore }` |
| 14 | `GET` | `/analytics/credentials/:holderId/trends?days=<n>` | — | `200 { holderId, days, trends }` |
| 15 | `POST` | `/analytics/credentials/:holderId/record-trend` | `{ credentials, verified, pending, avgVerificationScore? }` | `201 { message }` |
| 16 | `GET` | `/analytics/metrics` | — | `200 usage metrics` |

### Verification status values

| Field | Source | Values |
|-------|--------|--------|
| `VerificationRecord.verificationStatus` | `credentialStore` | `pending`, `verified`, `rejected`, `re_verification_required` |
| `VerificationChallenge.status` | `identityVerificationService` | `pending`, `passed`, `failed` |
| `Credential.status` | `credentialStore` | `active`, `revoked`, `expired`, `suspended` |

## Metrics Emitted

Every step bumps a counter through `server/src/http/metricsRegistry.ts`, which is
what the dashboards in `observability/` consume:

| Counter | Emitted by |
|---------|------------|
| `qc_proofs_generated_total` | `GET .../proof` |
| `qc_proofs_exported_total` | `POST .../proof/export` |
| `qc_proofs_validated_total` | `POST .../proof/validate` |
| `qc_identity_verifications_initiated_total` | `POST /credentials/verify-identity` |
| `qc_verification_status_checks_total` | `GET .../verification-status` |
| `qc_verification_stats_total` | `GET .../verify-all` |
| `qc_documents_verified_total` | `POST .../document-verification` |
| `qc_documents_queried_total` | `GET .../documents` |
| `qc_verification_challenges_created_total` | `POST .../challenges` |
| `qc_verification_challenges_passed_total` / `..._failed_total` | `POST .../challenges/:id/complete` |
| `qc_verification_reports_generated_total` | `GET .../verification-report` |
| `qc_analytics_dashboard_generated_total` | `GET /analytics/credentials/:holderId` |

## Known Boundaries

- `credentialStore` is **in-memory**; the module header states production would
  back it with a database. Verification records therefore do not survive a
  restart.
- Issuance and revocation exist on the store (`issueCredential`,
  `revokeCredential`) but are **not** exposed over HTTP yet, so they are shown as
  in-process steps rather than endpoints.
- `recordVerification` keys records by a generated `verificationId` while
  `getVerification` looks up by `credentialId`; the flow above therefore assumes
  one active verification record per credential.

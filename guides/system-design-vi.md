# Thiết kế hệ thống — FAPEX

> **English summary:** Three-tier Web3 dApp — React client, Express indexer/API on Railway, six Solidity contracts on Sepolia. MongoDB caches events; IPFS stores files; Socket.io pushes updates.

**Cập nhật:** 2026-06-30

---

## 1. Kiến trúc tổng quan

```mermaid
flowchart TB
  subgraph client [Client Browser]
    FE[Vite React 19 + wagmi]
    MM[MetaMask Sepolia]
  end

  subgraph vercel [Vercel CDN]
    Static[SPA build]
  end

  subgraph railway [Railway Backend]
    API[Express REST]
    AUTH[SIWE + JWT]
    IDX[Event Indexer cron]
    RT[Realtime WSS optional]
    SOCK[Socket.io]
    DB[(MongoDB)]
    PIN[Pinata client]
  end

  subgraph sepolia [Sepolia L1]
    JR[JobRegistry]
    EV[EscrowVault]
    AP[ArbitratorPanel]
    PT[PlatformTreasury]
    RS[ReputationStore]
    USDC[MockUSDC]
  end

  subgraph ipfs [IPFS]
    GATE[Pinata Gateway]
  end

  FE --> Static
  FE --> MM
  MM --> sepolia
  FE -->|HTTPS JWT| API
  FE --> SOCK
  AUTH --> DB
  API --> DB
  API --> PIN
  PIN --> GATE
  IDX -->|eth_getLogs| sepolia
  IDX --> DB
  IDX --> SOCK
  RT -.->|SEPOLIA_WSS_URL| sepolia
  EV --> AP
  EV --> JR
  EV --> USDC
  AP --> PT
  AP --> RS
```

---

## 2. Phân lớp trách nhiệm

| Lớp | Thư mục | Trách nhiệm |
|-----|---------|-------------|
| **Presentation** | `frontend/` | UI, wallet connect, gửi tx, đọc contract view, Socket.io client |
| **Application API** | `backend/src/routes/` | Auth, CRUD jobs/bids/disputes, IPFS upload proxy |
| **Indexing** | `backend/src/services/blockchain/` | Poll events, cập nhật MongoDB, emit notifications |
| **Domain (on-chain)** | `contracts/` | Escrow, dispute, reputation — **source of truth** |
| **Storage** | Pinata | File deliverable, metadata JSON, evidence |

---

## 3. Smart contract interactions

```mermaid
sequenceDiagram
  participant C as Client
  participant F as Freelancer
  participant JR as JobRegistry
  participant EV as EscrowVault
  participant AP as ArbitratorPanel

  C->>JR: createJob(metadataCID, value, duration)
  F->>JR: submitProposal (optional on-chain)
  Note over C,F: Bid accept = off-chain DB
  C->>EV: depositEscrow(jobId, freelancer)
  EV->>JR: assignFreelancer + status ASSIGNED
  F->>EV: startWork
  F->>EV: submitWork(deliverableCID)
  alt Happy path
    C->>EV: approveAndRelease
  else Dispute
    C->>EV: raiseDispute
    EV->>AP: setupDisputePanel (sortition 5)
    C->>AP: submitEvidence
    F->>AP: submitEvidence
    AP->>AP: commitVote / revealVote
    C->>EV: finalizeDisputeVoting
    C->>EV: executeArbitrationResult
  end
```

### Authorization graph

```
EscrowVault ──authorized──► JobRegistry, ArbitratorPanel, PlatformTreasury, ReputationStore
ArbitratorPanel ──authorized──► PlatformTreasury, ReputationStore
Admin ──► setAuthorizedContract, joinPool (arbitrator), setPaused, adminForceResolve
```

---

## 4. Backend components

### 4.1 Entry points

| File | Vai trò |
|------|---------|
| `backend/src/server.js` | HTTP server + Socket.io attach |
| `backend/src/app.js` | Express middleware, CORS, routes |
| `backend/src/services/blockchain/eventIndexer.js` | Cron poll `eth_getLogs` |
| `backend/src/services/blockchain/realtimeListener.js` | Optional WSS subscription |
| `backend/src/services/notifications/socketService.js` | JWT rooms `wallet:*`, `job:*` |

### 4.2 MongoDB models

| Model | Mục đích |
|-------|----------|
| `User` | wallet, username, role, nonce, reputation cache |
| `Job` | compound key `(onchainJobId, jobRegistryAddress)` |
| `Bid` | off-chain proposals |
| `Dispute` | evidence metadata, arbitrator list mirror |
| `IndexerState` | `lastBlock` checkpoint |

### 4.3 jobScope (sau redeploy)

Mọi query job scoped theo `JOB_REGISTRY_ADDRESS` env. Job cũ từ registry trước redeploy dùng `LEGACY_JOB_REGISTRY_ADDRESS`:

```
LEGACY_JOB_REGISTRY_ADDRESS=0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9
```

Migration: `backend/scripts/migrate-job-registry-index.js`

---

## 5. Frontend architecture

```mermaid
flowchart LR
  App[App.tsx] --> Router[router.tsx]
  Router --> Landing[LandingPage]
  Router --> Browse[BrowsePage]
  Router --> Client[ClientDashboard]
  Router --> FL[FreelancerDashboard]
  Router --> Arb[ArbitratorDashboard]
  Router --> Job[JobDetailPage]
  Hooks[useAuth useContractTx useSocket] --> API[api/client.ts]
  Hooks --> Wagmi[wagmi contracts]
```

**Guards:** `RoleGuard` — client / freelancer / arbitrator (stake ≥50 USDC).

**Mismatch UX:** `WalletMismatchBanner` khi MetaMask ≠ on-chain party.

---

## 6. CORS & production networking

```mermaid
flowchart LR
  Vercel["*.vercel.app"] -->|HTTPS + JWT| Railway[fapex-backend-production.up.railway.app]
  Railway -->|ALLOWED_ORIGINS wildcard| CORS[corsOrigins.js]
  CORS -->|match https://*.vercel.app| Allow[Allow preview + prod]
```

`backend/src/utils/corsOrigins.js` hỗ trợ:
- Exact origin: `https://fapex.vercel.app`
- Wildcard: `https://*.vercel.app`, `*.vercel.app`

Railway env example:
```
ALLOWED_ORIGINS=http://localhost:3000,https://*.vercel.app
```

---

## 7. Dispute state machine

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: submitWork
  SUBMITTED --> DISPUTED: raiseDispute
  DISPUTED --> Evidence: 0-10 min
  Evidence --> Commit: 10-13 min
  Commit --> Reveal: 13-16 min
  Reveal --> Finalized: finalizeDisputeVoting
  Finalized --> Executed: executeArbitrationResult
  Finalized --> Appeal: fileAppeal within 30 min
  Appeal --> Round2: startAppealRound
```

---

## 8. Nguồn sự thật (source of truth)

| Dữ liệu | Authority |
|---------|-----------|
| Escrow balance, job status enum | **On-chain** |
| Dispute votes, evidence hash | **On-chain** |
| Reputation score | **On-chain** `ReputationStore` |
| Job title, description, skills | MongoDB + IPFS JSON |
| Bids | MongoDB (accept = DB; chain optional) |
| Evidence CID plaintext | MongoDB + IPFS (hash on-chain) |
| User display name | MongoDB |

---

## 9. v2 roadmap

| Thành phần | Thay thế |
|------------|----------|
| Custom indexer poll | The Graph subgraph + gateway |
| `prevrandao` sortition | Chainlink VRF callback |
| Manual `executeArbitrationResult` | Keeper / automation |

Xem: [chainlink-integration-vi.md](chainlink-integration-vi.md) · [demo-qa-defense-vi.md](demo-qa-defense-vi.md)

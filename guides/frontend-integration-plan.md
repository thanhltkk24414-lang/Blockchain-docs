# Fapex Frontend Integration Plan

Phase 1 audit and foundation (June 2026). Canonical app: **Vite + React** in `frontend/src/`.

## 1. Inventory summary

### Three trees in `frontend/`

| Tree | Files (approx.) | Framework | Wired to BE/contracts |
|------|-----------------|-----------|------------------------|
| **`src/`** (active) | ~25 TS/TSX | Vite 7, React 19, wagmi 2, RainbowKit, SIWE | Partial — jobs, auth, profile, arbitrator stake |
| **Root `app/`, `components/`** | ~20 TSX | Next.js (v0 export) | No — marketing only |
| **`fapex-frontend/`** | ~70 TS/TSX | Next.js 14, Tailwind 4, axios | Partial — many mock fallbacks, wrong auth path (`/api/users/siwe` vs `/api/auth/verify`) |

### Scaffold vs contributors

- **Scaffold (`391b352`)**: RainbowKit, SIWE → JWT, `GET /api/jobs`, Socket.io live feed.
- **Contributor 1** (`fapex-frontend/src/app/client/*`, `login`, `register`, profiles): client create job, escrow UX, dispute pages — UI only, mock-heavy.
- **Contributor 2** (`fapex-frontend/src/app/freelancer/*`, `JobCard`, `ProposalForm`, `IPFSUploader`, arbitrator admin): browse, bid, deliverable, dispute components + tests.
- **v0 reference** ([v0-fapexfreelance.vercel.app](https://v0-fapexfreelance.vercel.app/)): landing sections mirrored in root `components/marketing/` and simplified in `src/pages/LandingPage.tsx`.

### Conflicts / duplicates

| Item | Locations | Resolution |
|------|-----------|------------|
| JobCard | `fapex-frontend/.../JobCard.tsx`, `src/components/shared/JobCard.tsx` | Use Vite version; port styling from Next |
| StatusBadge | `components/common/status-badge.tsx`, `fapex-frontend/.../StatusBadge.tsx`, `src/.../JobCard.tsx` | Consolidate in Phase 2 |
| API client | `src/lib/api.ts`, `fapex-frontend/src/services/api.ts` | Use `src/lib/api/client.ts` (correct paths) |
| Wallet providers | `src/main.tsx`, `fapex-frontend/src/lib/providers.tsx` | Vite `main.tsx` only |
| ABIs | `backend/src/abi/`, `fapex-frontend/src/lib/abis/` | `npm run sync-abis` → `src/lib/contracts/abis/` |

---

## 2. Backend API matrix

Base: `https://fapex-backend-production.up.railway.app` or `http://127.0.0.1:5000`

### Implemented (use now)

| Method | Route | FE status |
|--------|-------|-----------|
| GET | `/health` | Available in API client |
| POST | `/api/auth/nonce` | Wired (SIWE) |
| POST | `/api/auth/verify` | Wired (SIWE) |
| GET | `/api/auth/me` | Wired |
| GET | `/api/jobs` | Browse + dashboards |
| GET | `/api/jobs/search` | Browse search |
| GET | `/api/jobs/:id` | Client ready |
| GET | `/api/jobs/client/:address` | Client dashboard |
| GET | `/api/jobs/freelancer/:address` | Freelancer dashboard |
| POST | `/api/jobs` | Phase 2 (create job) |
| PATCH | `/api/jobs/:id/status` | Phase 2 |
| GET | `/api/users/profile/:address` | Profile page |
| PUT | `/api/users/profile` | Profile page |
| POST | `/api/users/register` | Profile page |
| GET | `/api/users/check/:address` | Profile page |
| GET | `/api/users/reputation/:address` | Phase 2 |
| GET | `/api/users/stats/:address` | Freelancer dashboard |
| GET | `/api/arbitrator/:address/status` | Arbitrator dashboard |
| POST | `/api/ipfs/upload/file` | Phase 2 (auth required) |
| POST | `/api/ipfs/upload/metadata` | Phase 2 |
| WebSocket | `/socket.io` (JWT) | Client dashboard LiveFeed |

### Placeholder (backend returns "Coming soon")

| Route group | Needed for |
|-------------|------------|
| `/api/bids/*` | Proposals, accept/reject |
| `/api/disputes/*` | Dispute flows |
| `/api/reviews/*` | Post-job reviews |

### Path mismatches in `fapex-frontend`

- Auth: uses `POST /api/users/siwe` — **wrong**; use `/api/auth/verify`.
- Profile: uses `GET /api/users/:address` — **wrong**; use `/api/users/profile/:address`.
- Bids: uses `GET /api/bids/freelancer/:address` — route is `/api/bids/my/:address`.

---

## 3. Contract UI surface (Sepolia)

Addresses in `deployments/sepolia.json` / `VITE_*_ADDRESS` env vars.

### JobRegistry (writes)

- `createJob(metadataCID, contractValue, duration)`
- `submitProposal(jobId, bidAmount, proposalCID)`
- `assignFreelancer(jobId, freelancer)`
- `cancelOpenJob(jobId)`
- `submitDeliverable(jobId, deliverableCID)` (verify exact name in ABI)

### JobRegistry (reads)

- `getJob`, `getProposals`, `jobCounter`

### EscrowVault (writes)

- `depositEscrow` (or equivalent deposit function)
- `startWork`, `submitWork`
- `approveMilestone` / fund release
- `raiseDispute`

### MockUSDC

- `approve(spender, amount)`, `balanceOf`, `allowance`

### ArbitratorPanel

- `stake`, vote/finalize flows

### ReputationStore

- `getReputation(address)` — sync via `GET /api/users/sync/:address`

Foundation hooks: `src/hooks/contracts/useContracts.ts` (`useJobCounter`, `useUsdcBalance`).

---

## 4. v0 site → routes map

| v0 section / CTA | Phase 1 route | Notes |
|------------------|---------------|-------|
| Landing | `/` | `LandingPage.tsx` |
| Browse open jobs | `/browse` | Live API |
| Launch App / Login | `/profile` | SIWE + register |
| Client dashboard | `/client` | Jobs by client address |
| Freelancer dashboard | `/freelancer` | Jobs + stats |
| Admin / Arbitrator | `/arbitrator` | Stake status from chain |

---

## 5. Target folder structure (Vite)

```
frontend/src/
  components/
    layout/AppShell.tsx      # nav + wallet
    shared/JobCard.tsx       # shared JobCard, StatusBadge
    LiveFeed.tsx             # socket feed
  config/env.ts, wagmi.ts
  context/AuthContext.tsx
  hooks/
    useAuth.ts, useJobs.ts, useSocket.ts
    contracts/useContracts.ts
  lib/
    api/client.ts, types.ts
    auth.ts, socket.ts
    contracts/abis/, addresses.ts, config.ts
  pages/
    LandingPage.tsx, BrowsePage.tsx
    ClientDashboardPage.tsx, FreelancerDashboardPage.tsx
    ArbitratorDashboardPage.tsx, ProfilePage.tsx
  router.tsx
  App.tsx, main.tsx
```

Reference only: `contrib/README.md`, `fapex-frontend/`, root Next marketing files.

---

## 6. Phase roadmap

### Phase 1 — Done (this task)

- [x] Full inventory and API/contract audit
- [x] React Router: `/`, `/browse`, `/client`, `/freelancer`, `/arbitrator`, `/profile`
- [x] Expanded API client aligned with backend
- [x] Contract ABIs + addresses + read hooks
- [x] Landing page (v0-inspired)
- [x] Wire jobs, profile, arbitrator stake, socket feed
- [x] `.env.example` with contract addresses

### Phase 2 — Contributor 1 (done)

- [x] Create job form → `POST /api/jobs` (backend IPFS + on-chain `createJob`)
- [x] Client job detail `/client/jobs/:id` and public `/jobs/:id`
- [x] USDC approve + `depositEscrow` with `TxStatusModal`
- [x] Shared `JobCard`, `StatusBadge`, `MilestoneProgress` placeholder
- [x] `useJobs` refetch after create
- [x] Demo jobs guide (`frontend/docs/demo-jobs.md`)
- [ ] Login/register UX polish (port from `fapex-frontend/src/app/login`)
- [ ] Proposals list (when bids API live)
- [ ] Milestone approve / dispute raise (Phase 3+)
- [ ] Add `role` to backend `User.profile` if not present

### Phase 3 — Contributor 2

- [ ] Browse filters (port `JobFilters.tsx`)
- [ ] Submit bid form → bids API + `submitProposal`
- [ ] Deliverable upload → IPFS + on-chain hash
- [ ] Arbitrator dispute queue, evidence viewer, vote tx
- [ ] Shared: `MilestoneProgress`, full `TxStatus` modal

### Phase 4 — Polish

- [ ] Tailwind or design tokens from v0
- [ ] E2E on Sepolia testnet
- [ ] Remove `fapex-frontend` mock mode
- [ ] CI: `npm run build` on frontend submodule

---

## 7. How to run

```bash
cd frontend
cp .env.example .env   # set VITE_WALLETCONNECT_PROJECT_ID
npm install
npm run dev            # http://localhost:3000
```

Sync ABIs after contract changes:

```bash
npm run sync-abis
```

Backend CORS must include `http://localhost:3000` (`ALLOWED_ORIGINS` on Railway).

---

## 8. Open gaps

1. **Bids & disputes APIs** — placeholders; FE must wait or use chain-only flows.
2. **Dual framework clutter** — Next trees not deleted; migrate components incrementally.
3. **Profile `role` field** — UI sends `role`; confirm backend `User` schema persists it.
4. **fapex-frontend wrong API paths** — do not copy `services/api.ts` verbatim.
5. **CORS / SIWE domain** — production needs `SIWE_DOMAIN` and `APP_URL` matching deployed FE origin.
6. **No job detail routes yet** — `/jobs/:id` pages exist only in Next reference tree.

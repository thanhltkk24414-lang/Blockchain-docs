# Ma trận audit issue — FAPEX

**Cập nhật:** 2026-06-30

> **English summary:** Issue tracking matrix from security/architecture audit — done items and v2 deferrals.

| ID | Mô tả | Trạng thái | Hành động / ghi chú |
|----|--------|------------|---------------------|
| SC-1 | Dispute windows baked in bytecode | **Done (partial)** | `DisputeTimings.demo.sol` / `.prod.sol` + `prepare-dispute-timings.js`. Sepolia redeployed demo 2026-06-27. v2: governance `setDisputeWindow` |
| SC-2 | Reentrancy escrow release | **Done** | CEI trong `_splitAndPayout`; USDC không callback |
| SC-3 | Centralized arbitrator pool | **Done (documented)** | Stake ≥50 USDC, slash no-reveal, reputation. v2: VRF |
| SC-5 | Reputation off-chain only | **Done** | `ReputationStore` on-chain + `ReputationBadge` FE |
| SC-6 | Manual export ABIs | **Done** | Hardhat postcompile → `export-abis.js` → backend + frontend |
| BE-1 | Backend source of truth | **Done (arch)** | Indexer = cache; chain = truth. v2: The Graph |
| BE-2 | SIWE not Web3-native | **Done** | EIP-4361 `/api/auth/nonce` + `/verify`, `siwe-sign.html` |
| BE-3 | Centralized deliverables | **Done** | Pinata → CID on-chain |
| BE-4 | Missed blocks on restart | **Done** | `IndexerState.lastBlock` persist + replay |
| BE-5 | CORS Vercel previews | **Done** | `corsOrigins.js` wildcard `https://*.vercel.app` |
| BE-6 | JobRegistry redeploy scope | **Done** | `jobScope` + `LEGACY_JOB_REGISTRY_ADDRESS` + migration script |
| FE-1 | Network mismatch | **Done** | `NetworkBanner` + `switchChain` |
| FE-2 | Tx pending UX | **Done** | `TxStatusModal`, Etherscan link |
| FE-3 | RPC fallback | **Done** | wagmi `fallback([primary, secondary])` |
| FE-4 | Gas estimate display | **Done (partial)** | `useContractTx` + gas line; `raiseDispute` 700k–900k |
| FE-5 | Hardcoded ABI | **Done** | `deployments-sepolia.json` + `GET /api/config` |
| FE-6 | Wallet mismatch UX | **Done** | `WalletMismatchBanner` |
| FE-7 | English UI + theme | **Done** | Light/dark, no VI locale on prod |
| CL-1 | ETH/USD price feed | **Done (code) / UI off** | Feed Sepolia documented; live price removed from UI Jun 2026 |
| CL-2 | VRF sortition | **Deferred v2** | `prevrandao` MVP + `VRFSortitionStub.sol` |
| Brand | Landing + logos | **Done** | `/`, `fapex-icon.svg`, `fapex-wordmark.svg` |
| DOC | Comprehensive docs | **Done** | guides/*, platform-mechanisms, admin-roles, Q&A Jun 2026 |

---

## Deferred v2

| ID | Hạng mục | Lý do defer |
|----|----------|-------------|
| CL-2 | Chainlink VRF arbitrator sortition | Subscription + async complexity |
| BE-1b | The Graph subgraph | Custom indexer đủ thesis |
| SC-1b | On-chain mutable dispute windows | Cần proxy hoặc governance |
| — | CCIP / Chainlink Functions | Out of scope |
| — | OpenZeppelin ReentrancyGuard | Optional hardening (CEI sufficient) |
| — | Reviews API | Placeholder routes |

---

## Production checklist (2026-06)

- [x] Backend Railway: https://fapex-backend-production.up.railway.app
- [x] Frontend Vercel + CORS wildcard
- [x] Sepolia contracts synced `deployments/sepolia.json`
- [x] Demo dispute timings (5/10/13/16 min + appeal 30 min)
- [x] Pool appeal ≥10 documented (`check-dispute-demo.js`)
- [x] Legacy registry `0xE5425…` documented

---

## Tài liệu liên quan

- [demo-qa-defense-vi.md](demo-qa-defense-vi.md)
- [chainlink-integration-vi.md](chainlink-integration-vi.md)
- [on-chain-off-chain-map-vi.md](on-chain-off-chain-map-vi.md)

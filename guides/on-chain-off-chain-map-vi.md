# FAPEX — Bản đồ On-chain / Off-chain (Audit)

> Cập nhật: 2026-06-26 · JobRegistry Sepolia: `0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9`

## Mô hình hai ví (INDEXER relay)

| Vai trò | Ví | Hành động |
|--------|-----|-----------|
| **API client** | Ví SIWE (người đăng job) | `POST /api/jobs`, accept bid, UI dashboard |
| **On-chain client** | `INDEXER_PRIVATE_KEY` (Railway) | `JobRegistry.createJob`, `depositEscrow` phải ký từ ví này |
| **Freelancer** | MetaMask freelancer | `submitProposal`, `startWork`, `submitWork` |
| **Indexer backend** | Cùng INDEXER + cron | Đọc events → MongoDB, không thay thế escrow ký user |

Frontend hiển thị `WalletMismatchBanner` khi SIWE client ≠ on-chain client — **đúng thiết kế demo**.

---

## Bảng mapping dữ liệu

| Trường / Khái niệm | Chỉ on-chain | Chỉ off-chain (MongoDB) | Kết hợp (sync) |
|---------------------|--------------|-------------------------|----------------|
| `onchainJobId` | `JobRegistry.jobCounter` | — | Khóa compound với `jobRegistryAddress` |
| `jobRegistryAddress` / `chainId` | Địa chỉ deploy hiện tại | Lưu lúc tạo job | Phân biệt job cũ sau redeploy |
| `clientAddress` | — | Ví SIWE (chủ job API) | Khác `onchainClientAddress` khi relay INDEXER |
| `onchainClientAddress` | `Job.client` on registry | Lưu sau create | Dùng cho preflight `depositEscrow` |
| `freelancerAddress` | `Job.freelancer` sau deposit | Bid accepted (tạm) | Indexer + `EscrowDeposited` ghi đè |
| `status` | `JobRegistry.JobStatus` enum 0–7 | `Job.status` string | Indexer events + `GET /jobs/:id` reconcile |
| `metadataCID` | `jobMetadataCID` | IPFS upload trước create | API ghi đầy đủ; indexer chỉ CID chain |
| `deliverableCID` | `deliverableCID` | — | `WorkSubmitted` event |
| `contractValue`, `deadline` | Job struct | Form tạo job | Phải khớp đơn vị USDC 6 decimals |
| `title`, `description`, `skills` | — | MongoDB + IPFS JSON | Không có on-chain |
| **Bids / Proposals** | `submitProposal` (optional) | `Bid` collection | Bid API = off-chain; chain optional |
| **User profile, role** | `ReputationStore` score/tier | `User` model | Tier map 0–3 → Restricted…Trusted |
| **Dispute chi tiết** | `ArbitratorPanel` votes, evidence hash | `Dispute` model | Indexer `DisputeSetup` / `DisputeFinalized` |
| **Escrow balance** | `EscrowVault` + USDC | — | UI đọc `balanceOf(EscrowVault)` |
| **Platform fee 3%** | `depositEscrow` tính on-chain | `totalDeposit` ≈ value×1.03 | Off-chain chỉ hiển thị |

---

## Trạng thái job — đồng bộ contract ↔ BE ↔ FE

| Mã | On-chain | Backend `Job.status` | Ghi chú |
|----|----------|----------------------|---------|
| 0 | OPEN | OPEN | Sau accept bid DB có thể ASSIGNED **trước** deposit — chain vẫn OPEN |
| 1 | ASSIGNED | ASSIGNED | Sau `depositEscrow` (gán FL + nạp tiền) |
| 2 | IN_PROGRESS | IN_PROGRESS | `startWork` |
| 3 | SUBMITTED | SUBMITTED | `submitWork` |
| 4 | DISPUTED | DISPUTED | `raiseDispute` |
| 5 | COMPLETED | COMPLETED | `approveAndRelease` / timeout / dispute win FL |
| 6 | REFUNDED | REFUNDED | `cancelContract` / dispute client win |
| 7 | CANCELLED | CANCELLED | `cancelOpenJob` |

**FE:** `effectiveJobStatus()` và `useOnChainJob` ưu tiên trạng thái đọc từ registry.

**BE:** `eventIndexer` + `getJobById` gọi `isChainStatusAhead` để kéo DB lên khi chain tiến hơn.

---

## Luồng E2E đúng (Sepolia demo)

1. Client SIWE → `POST /api/jobs` → backend INDEXER gọi `createJob` → lưu MongoDB (`jobReconcile`).
2. Freelancer → `POST /api/bids` (off-chain).
3. Client → `PATCH /api/bids/:id/accept` → DB gán FL; **không** gọi `assignFreelancer` (chỉ EscrowVault được gọi).
4. Client chuyển MetaMask sang **ví INDEXER** → `approve` USDC + `depositEscrow(jobId, freelancer)`.
5. Freelancer → `startWork` → `submitWork`.
6. Client INDEXER → `approveAndRelease` hoặc dispute flow.

### Dispute demo timings (`disputeTimings: demo` trong `deployments/sepolia.json`)

| Phase | Demo (on-chain) | Production |
|-------|-----------------|------------|
| Evidence initial | 15 phút | 72 giờ |
| Rebuttal | 30 phút | 120 giờ |
| Commit vote | 30–45 phút | 120–144 giờ |
| Reveal vote | 45–60 phút | 144–168 giờ |
| Appeal window | 2 giờ | 72 giờ |

FE: `frontend/src/lib/contracts/disputeTimings.ts` → `DISPUTE_PHASES_DEMO`.

---

## Lỗi đã sửa (job creation)

**Triệu chứng:** `409` — *"This on-chain job id already exists for the current JobRegistry in MongoDB."*

**Nguyên nhân gốc:**

1. `createJob` on-chain **thành công** (job id mới từ `jobCounter`).
2. **Race:** `eventIndexer` chèn bản ghi stub (`clientAddress` = ví INDEXER) giữa `findOne` và `save` → duplicate key `(onchainJobId, jobRegistryAddress)`.
3. Hoặc stub indexer tồn tại, `clientAddress` ≠ ví SIWE → trước đây trả 409 thay vì merge metadata API.

**Sửa:** `src/utils/jobReconcile.js` — adopt stub / retry sau duplicate; indexer set `onchainClientAddress`; `createJob` trả 200 `reconciled` khi link thành công.

**Test:** `node scripts/test-job-reconcile.js`

---

## Mismatch đã biết (không chặn demo)

| Vấn đề | Mức | Hướng xử lý |
|--------|-----|-------------|
| DB `ASSIGNED` sau accept, chain `OPEN` | Thấp | UI dùng on-chain cho deposit; indexer sync khi deposit |
| `fapex-frontend_v2` ABI tên hàm cũ (`depositAndAssign`, `submitDeliverable`) | Tham chiếu | Không merge — canonical `frontend/src` đúng ABI |
| `POST /api/jobs/:id/assign-freelancer` | Tránh dùng | Chỉ EscrowVault được authorize `assignFreelancer` |

---

## File tham chiếu

| Layer | Path |
|-------|------|
| Contracts | `contracts/FreelanceSystem.sol`, `contracts/config/DisputeTimings.demo.sol` |
| Backend create | `backend/src/controllers/jobController.js`, `backend/src/utils/jobReconcile.js` |
| Indexer | `backend/src/services/blockchain/eventIndexer.js` |
| Contract service | `backend/src/services/blockchain/contractService.js` |
| FE contracts | `frontend/src/lib/contracts/`, `deployments/sepolia.json` |
| FE escrow | `frontend/src/hooks/useEscrowDeposit.ts` |

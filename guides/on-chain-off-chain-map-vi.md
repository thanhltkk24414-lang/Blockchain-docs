# FAPEX — Bản đồ On-chain / Off-chain (Audit)

> Cập nhật: 2026-06-26 · JobRegistry Sepolia: `0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9`

## Mô hình ví (client-signed createJob)

| Vai trò | Ví | Hành động |
|--------|-----|-----------|
| **Client** | Cùng ví SIWE + MetaMask | `createJob`, `depositEscrow`, `approveAndRelease` |
| **Freelancer** | MetaMask freelancer | `submitProposal`, `startWork`, `submitWork` |
| **Indexer backend** | `INDEXER_PRIVATE_KEY` | Đọc events → MongoDB; **không** relay `createJob` |

`clientAddress` (MongoDB) và `onchainClientAddress` (JobRegistry) **phải trùng** sau khi client ký `createJob`.

Frontend: `WalletMismatchBanner` cảnh báo khi MetaMask ≠ client on-chain (thường do đổi ví sau SIWE).

---

## Bảng mapping dữ liệu

| Trường / Khái niệm | Chỉ on-chain | Chỉ off-chain (MongoDB) | Kết hợp (sync) |
|---------------------|--------------|-------------------------|----------------|
| `onchainJobId` | `JobRegistry.jobCounter` | — | Khóa compound với `jobRegistryAddress` |
| `jobRegistryAddress` / `chainId` | Địa chỉ deploy hiện tại | Lưu lúc tạo job | Phân biệt job cũ sau redeploy |
| `clientAddress` | — | Ví SIWE (chủ job API) | Trùng `onchainClientAddress` sau fix |
| `onchainClientAddress` | `Job.client` on registry | Lưu sau register API | Dùng cho preflight `depositEscrow` |
| `freelancerAddress` | `Job.freelancer` sau deposit | Bid accepted (tạm) | Indexer + `EscrowDeposited` ghi đè |
| `status` | `JobRegistry.JobStatus` enum 0–7 | `Job.status` string | Indexer events + `GET /jobs/:id` reconcile |
| `metadataCID` | `jobMetadataCID` | IPFS upload trước create | FE upload → on-chain → API register |
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

1. Client SIWE → upload IPFS → MetaMask ký `createJob` → `POST /api/jobs` (kèm `onchainJobId`).
2. Freelancer → `POST /api/bids` (off-chain).
3. Client → `PATCH /api/bids/:id/accept` → DB gán FL; **không** gọi `assignFreelancer`.
4. Client **cùng ví** → mint MockUSDC (nếu cần) → `approve` + `depositEscrow(jobId, freelancer)`.
5. Freelancer → `startWork` → `submitWork`.
6. Client → `approveAndRelease` hoặc dispute flow.

Chi tiết: `frontend/docs/job-create-flow-vi.md`.

### Dispute demo timings (`disputeTimings: demo` trong `deployments/sepolia.json`)

| Phase | Demo (on-chain) | Production |
|-------|-----------------|------------|
| Evidence initial | 15 phút | 72 giờ |
| Rebuttal | 30 phút | 120 giờ |
| Commit vote | 30–45 phút | 120–144 giờ |
| Reveal vote | 45–60 phút | 144–168 giờ |
| Appeal window | 2 giờ | 72 giờ |

FE: `frontend/src/lib/contracts/disputeTimings.ts` → `DISPUTE_PHASES_DEMO`.

### UI tranh chấp (2026-06-27)

| Panel | Ai thấy | Nội dung |
|-------|---------|----------|
| `DisputeEvidencePanel` | Mọi người (DISPUTED) | Danh sách bằng chứng on-chain (`getEvidences`) + off-chain (`GET /api/disputes/:id/evidences`); form nộp chỉ client/FL trong 30 phút đầu |
| `ArbitratorDisputePanel` | Arbitrator được chọn | Commit/reveal; **tổng vote đã reveal** (`getVote` × 5); disable reveal nếu `AlreadyRevealed`; Finalize / Execute |
| `DisputeResultPanel` | Client / freelancer | **Kết quả** (`getPendingResult`) sau finalize; countdown kháng cáo 2h; nút `fileAppeal` |

Đọc contract: `ArbitratorPanel.getVote`, `getPendingResult`, `getEvidences` · `EscrowVault.fileAppeal`, `appealFiled`.

---

## Lỗi đã sửa (2026-06-26)

### Freelancer stats (Completed / Total earned)

**Triệu chứng:** Job COMPLETED on-chain nhưng dashboard hiển thị Completed 0, Total earned 0 USDC.

**Nguyên nhân:** `GET /api/users/stats/:address` chỉ trả `user.stats` cache — không bao giờ được cập nhật khi `FundsReleased`.

**Sửa:**
- `freelancerStatsService.js` — tính `jobsCompleted` / `totalEarned` từ MongoDB `Job` (status `COMPLETED`, `freelancerAddress` khớp) + reconcile on-chain khi indexer chậm.
- `eventIndexer` — `$inc` stats khi sync `FundsReleased` (số tiền chính xác từ event).
- FE `FreelancerDashboardPage` không đổi — vẫn gọi `/api/users/stats/:address`.

### raiseDispute revert không decode

**Triệu nhân:** Gas cap `raiseDispute` = 400k trong khi Sepolia cần ~641k (sortition + 5 arbitrator) → OOG hiện dạng "execution reverted".

**Sửa:** `contractGas.ts` min 700k / cap 900k; `useJobActions` preflight tier, pool ≥5, số dư USDC phí 2%.

### Accept bid không đóng bid khác

**Triệu chứng:** Job đã accept 1 bid + nạp escrow nhưng bid khác vẫn hiện "Accept bid".

**Nguyên nhân:** `acceptBid` chỉ set `accepted` cho bid được chọn; job DB vẫn `OPEN` đến khi `depositEscrow` nên FE chỉ check `jobStatus !== 'OPEN'` không đủ.

**Sửa:**
- `PATCH /api/bids/:id/accept` → `updateMany` reject các bid `pending` còn lại.
- `AcceptBidButton` + `JobDetailPage` — prop `hasAcceptedBid` ẩn nút accept trên bid khác.

---

## Lỗi đã sửa (job creation)

**Triệu chứng cũ:** Client SIWE ≠ on-chain client (INDEXER) → mint USDC vô dụng, phải đổi ví để deposit.

**Sửa:** Client ký `createJob` on-chain; API chỉ register + verify. Không redeploy contract.

**Race indexer:** `jobReconcile.js` vẫn adopt stub khi indexer chèn trước API.

---

## Mismatch đã biết (không chặn demo)

| Vấn đề | Mức | Hướng xử lý |
|--------|-----|-------------|
| Job cũ (INDEXER là client on-chain) | Thấp | Tạo job mới hoặc deposit bằng ví INDEXER |
| DB `ASSIGNED` sau accept, chain `OPEN` | Thấp | UI dùng on-chain cho deposit; indexer sync khi deposit |
| Stats freelancer = 0 dù job COMPLETED | **Đã sửa** | `freelancerStatsService` + indexer `FundsReleased` |
| `raiseDispute` OOG / revert mơ hồ | **Đã sửa** | Gas 700k–900k + preflight FE |
| Nhiều bid vẫn "Accept" sau accept | **Đã sửa** | BE reject others + `hasAcceptedBid` FE |
| `fapex-frontend_v2` ABI tên hàm cũ | Tham chiếu | Canonical `frontend/src` |
| `POST /api/jobs/:id/assign-freelancer` | Legacy | Chỉ job cũ; dùng `depositEscrow` cho job mới |

---

## File tham chiếu

| Layer | Path |
|-------|------|
| Contracts | `contracts/FreelanceSystem.sol`, `contracts/config/DisputeTimings.demo.sol` |
| Backend create | `backend/src/controllers/jobController.js`, `backend/src/utils/jobReconcile.js` |
| Indexer | `backend/src/services/blockchain/eventIndexer.js` |
| FE create | `frontend/src/hooks/useCreateJob.ts`, `CreateJobForm.tsx` |
| FE escrow | `frontend/src/hooks/useEscrowDeposit.ts` |

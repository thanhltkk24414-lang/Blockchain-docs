# Phân công công việc Backend (đã điều chỉnh)

Tài liệu này căn chỉnh task split với **contract thực tế** (`FreelanceSystem.sol` — 6 contract deploy riêng trên Sepolia).

## Sửa tên / sự kiện quan trọng

| Tài liệu cũ | Thực tế on-chain |
|-------------|------------------|
| `claimAfterTimeout()` | **`claimTimeoutRelease(jobId)`** trên `EscrowVault` |
| `FundsDeposited` | **`EscrowDeposited`** |
| `CONTRACT_ADDRESS` (monolith) | 6 địa chỉ: `JobRegistry`, `EscrowVault`, `ArbitratorPanel`, `PlatformTreasury`, `ReputationStore`, `MockUSDC` |
| `arbitratorStakes` trên contract tổng | **`PlatformTreasury.arbitratorStakes(address)`** |
| `acceptProposal(index)` on-chain | **Không có** — client chọn freelancer qua địa chỉ khi `depositEscrow(jobId, freelancer)` |

## Ma trận phân công

| Hạng mục | Contributor 1 | Repo owner | Contributor 2 (bạn) |
|----------|---------------|------------|---------------------|
| Upload metadata job → Pinata CID | ✅ `POST /api/ipfs/upload/metadata` | — | QA Postman |
| Upload file deliverable / evidence | ✅ `POST /api/ipfs/upload/file` | Lưu CID vào Dispute model | — |
| Đồng bộ escrow events → MongoDB | ✅ `EscrowDeposited`, `FundsReleased`, `DisputeRaised` (indexer + optional WSS) | Job/Bid/Dispute schemas đầy đủ | — |
| Kiểm tra cọc trọng tài | ✅ `GET /api/arbitrator/:address/status` | — | Gắn vào flow vote UI |
| CRUD jobs / bids / reviews API | Stub routes owner | ✅ Controllers + models | Wire frontend |
| Event indexer JobRegistry | Partial (owner indexer) | ✅ `JobCreated`, `JobStatusUpdated`, `FreelancerAssigned` | — |
| Reputation on-chain | — | ✅ `ReputationStore` + review API | — |
| Cron `claimTimeoutRelease` | — | Scaffold | ✅ **Owner tích hợp** — `src/cron/claimTimeout.js` (cần `INDEXER_PRIVATE_KEY`) |
| SIWE + JWT middleware | — | Stub `auth.js` | ✅ Implement SIWE login route |
| WebSocket thông báo | Realtime listener escrow (nội bộ) | — | ✅ Socket.io cho client UI |
| Swagger / Postman | — | — | ✅ |
| Deploy Railway + CORS Vercel | — | CORS trong `app.js` | ✅ `ALLOWED_ORIGINS` |

## Luồng tạo job (MVP)

1. Frontend gọi `POST /api/ipfs/upload/metadata` → nhận `metadataCID`.
2. Frontend gọi `JobRegistry.createJob(metadataCID, value, duration)` từ ví client.
3. Indexer bắt `JobCreated` → ghi MongoDB.
4. Client `depositEscrow(jobId, freelancerAddress)` → `EscrowDeposited` → status `ASSIGNED`.

## Luồng nghiệm thu quá hạn

- Sau **7 ngày** kể từ `submittedAt` (status `SUBMITTED`), cron gọi `EscrowVault.claimTimeoutRelease(jobId)`.
- Emit `FundsReleased` → DB status `COMPLETED`.

## Ghi chú MVP

- MongoDB là **cache**, chain là source of truth.
- Proposal/bid: off-chain trong DB; client chọn freelancer bằng địa chỉ khi deposit.
- Không cần sửa contract cho scope backend hiện tại.

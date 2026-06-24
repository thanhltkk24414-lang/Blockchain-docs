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

---

## Contributor 2 — tiến độ task

| # | Hạng mục | Trạng thái | Ngày hoàn thành | Ghi chú |
|---|----------|------------|-----------------|---------|
| 1 | SIWE + JWT middleware (`/api/auth/nonce`, `/verify`, `/me`) | ✅ Hoàn thành | 2026-06-23 | Guide: [auth-api.md](./auth-api.md); smoke test `npm run test:auth` |
| 2 | Postman / REST Client / hướng dẫn test local | ✅ Hoàn thành | 2026-06-23 | [postman-testing.md](./postman-testing.md), [postman-walkthrough-vi.md](./postman-walkthrough-vi.md); `backend/api-tests.http`; collection `backend/postman/` |
| 3 | Deploy Railway + CORS production | ✅ Hoàn thành | 2026-06-24 | URL: `https://fapex-backend-production.up.railway.app`; guide: [deploy-backend.md](./deploy-backend.md); SIWE domain/CORS đã cấu hình |
| 4 | WebSocket Socket.io thông báo realtime | ✅ Hoàn thành | 2026-06-24 | JWT auth trên `/socket.io`; events `job:updated`, `escrow:*`; mục 7.5 trong [deploy-backend.md](./deploy-backend.md) |

**Còn lại (Contributor 2 / team):** wire frontend React + RainbowKit; tùy chọn bật event indexer (`ENABLE_EVENT_INDEXER`, `SEPOLIA_WSS_URL`) khi demo đồng bộ chain → DB → socket.

*Báo cáo tiến độ chi tiết:* [contributor2-progress.md](../report/contributor2-progress.md)

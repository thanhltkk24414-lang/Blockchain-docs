# FAPEX — Cơ chế nền tảng (on-chain)

> **English summary:** Authoritative description of dispute outcomes (including SPLIT 50-50), missed-deliverable paths, appeal, pool sizing, and reputation — derived from `FreelanceSystem.sol` / `EscrowVault` / `ArbitratorPanel`.

**Cập nhật:** 2026-06-30 · **Mạng:** Sepolia · `disputeTimings: demo`

---

## 1. Hội đồng arbitrator & quorum

| Tham số | Giá trị on-chain |
|---------|------------------|
| `PANEL_SIZE` | **5** arbitrator / vòng |
| `MIN_QUORUM` | **≥3** reveal hợp lệ để `finalizeDispute` |
| Sortition | `block.prevrandao` + `block.timestamp` + `jobId` + `round` |
| Stake tối thiểu | **50 USDC** (`PlatformTreasury.stakeAsArbitrator`) |
| Reputation tối thiểu | **≥80** (tier Normal+) |

**Pool size vận hành:**

| Luồng | `poolSize` tối thiểu | Lý do |
|-------|---------------------|--------|
| `raiseDispute` (vòng 1) | **≥5** | Contract revert `NotEnoughArbitrators` nếu `<5` |
| `fileAppeal` → vòng 2 | **≥10** (khuyến nghị) | Vòng 2 **loại trừ** 5 arbitrator vòng 1; cần đủ 5 người mới trong pool còn lại |

`npm run check:dispute` cảnh báo khi `poolSize < 10`.

---

## 2. Luồng tranh chấp — timeline demo

Tính từ `disputes[jobId].createdAt` (lúc `raiseDispute`):

| Phase | Demo (Sepolia) | Production |
|-------|----------------|------------|
| Evidence initial | **0 → 5 phút** | 0 → 72 h |
| Evidence rebuttal | **5 → 10 phút** | 72 → 120 h |
| Commit vote | **10 → 13 phút** | 120 → 144 h |
| Reveal vote | **13 → 16 phút** | 144 → 168 h |
| Finalize | sau **16 phút** (cửa reveal đóng) | sau 168 h |
| Appeal window | **30 phút** sau `resultAt` | 72 h |
| Execute | sau appeal window (hoặc sau finalize vòng 2) | tương tự |

**Chuỗi tx chuẩn:**

```
raiseDispute → submitEvidence (2 party) → commitVote (≥3 arb)
→ revealVote (≥3 arb) → finalizeDisputeVoting → [fileAppeal?]
→ executeArbitrationResult
```

---

## 3. Kết quả vote & SPLIT 50-50

### 3.1 Cách tính majority (`ArbitratorPanel.finalizeDispute`)

Đếm reveal hợp lệ: `FREELANCER_WIN` (1), `CLIENT_WIN` (2), `SPLIT_50_50` (3).

- Nếu `votesFree > votesClient` **và** `votesFree > votesSplit` → **FREELANCER_WIN**
- Nếu `votesClient > votesFree` **và** `votesClient > votesSplit` → **CLIENT_WIN**
- **Mọi trường hợp còn lại** (hòa FL/Client, đa số SPLIT, ba chiều bằng nhau…) → **SPLIT_50_50**

Ví dụ hòa 2–2–1 hoặc 2–2–0 (chỉ 4 reveal, quorum fail) — cần ≥3 reveal trước; với 3 reveal 1–1–1 → SPLIT.

### 3.2 Escrow khi `executeArbitrationResult` — SPLIT

Gọi `_splitAndPayout(job, jobId, 50)`:

| Bên | Nhận gì |
|-----|---------|
| **Freelancer** | **50%** `contractValue` (gross), trừ **service fee 2%** trên phần đó |
| **Client** | Phần còn lại từ tổng đã nạp (`contractValue × 1.03` − phần FL gross − platform fee tương ứng) |
| **Platform** | Service fee + platform fee trên phần freelancer nhận → `PlatformTreasury` |
| **Trạng thái job** | `COMPLETED` (không phải `REFUNDED`) |
| **Reputation** | FL **+10**, Client **+5** (giống release bình thường) |

### 3.3 Dispute fee khi SPLIT — `_handleSplitDisputeFee`

| Phần | Điểm đến |
|------|----------|
| **50%** dispute fee | Hoàn cho **bên khởi tạo** tranh chấp (`disputeInitiator`) |
| **50%** còn lại | `PlatformTreasury` (quỹ vận hành) |
| **Arbitrator** | **Không** nhận reward — không có “bên thắng” rõ ràng |

So với FL_WIN / CLIENT_WIN: 50% fee vào reward pool chia cho arbitrator vote đúng; bên thua (nếu khác initiator) chịu phần còn lại.

---

## 4. Kháng cáo (appeal) — vòng 2

1. Sau `finalizeDisputeVoting` vòng 1, cửa sổ **30 phút** (demo).
2. Client hoặc Freelancer gọi `EscrowVault.fileAppeal(jobId)`:
   - Phí = **1.3×** dispute fee gốc (`APPEAL_FEE_NUM/DEN = 130/100`).
   - Cần `approve` MockUSDC cho `EscrowVault` trước.
3. `ArbitratorPanel.startAppealRound` — chọn **5 arbitrator mới**, loại toàn bộ arbitrator vòng 1.
4. Vòng 2 chạy lại evidence → commit → reveal → finalize.
5. **Không có vòng 3** — `AppealNotAllowed` nếu `round > 1`.
6. `executeArbitrationResult`: nếu đã appeal, phải chờ finalize vòng 2 (`round == 2`).

**Điều kiện pool:** seed **≥10** arbitrator trước demo appeal (`npm run seed:arbitrators` mở rộng hoặc chạy nhiều lần).

---

## 5. Freelancer không nộp deliverable đúng hạn

### 5.1 Trường `deadline` on-chain

- `createJob(..., duration)` ghi `deadline = block.timestamp + duration`.
- **`submitWork` không kiểm tra `deadline`** — chỉ yêu cầu status `IN_PROGRESS`.
- **Không có** tx permissionless tự refund client khi quá `deadline` mà chưa `submitWork`.

### 5.2 Các nhánh theo trạng thái

| Trạng thái | Freelancer chưa làm gì | Client có thể |
|------------|------------------------|---------------|
| **ASSIGNED** (chưa `startWork`) | Sau **72h** từ `assignedAt` | `cancelContract` — hoàn **100%** đã nạp (`contractValue + 3%` platform fee). Permissionless sau 72h. |
| **ASSIGNED** | Trong 72h đầu | Chờ hoặc không có tx hủy sớm on-chain |
| **IN_PROGRESS** (đã `startWork`, chưa `submitWork`) | Không có auto-settle | `raiseDispute` — status **IN_PROGRESS** được phép. Không `approveAndRelease` (cần SUBMITTED). Không `claimTimeoutRelease`. |
| **SUBMITTED** (`deliverableCID` có) | — | `approveAndRelease`, `raiseDispute`, hoặc chờ 7 ngày → `claimTimeoutRelease` (ai cũng gọi được) |

### 5.3 Khi không có `deliverableCID`

- On-chain `deliverableCID` rỗng → chưa `submitWork`.
- Client **không** phê duyệt release on-chain.
- UI/backend có thể có file tạm off-chain (IPFS upload) nhưng **không có hiệu lực thanh toán** cho đến khi freelancer ký `submitWork(cid)`.

### 5.4 Vai trò backend / IPFS

| Bước | Off-chain | On-chain |
|------|---------|----------|
| Upload file deliverable | `POST /api/ipfs/upload/*` → Pinata → CID | — |
| Cam kết deliverable | — | `submitWork(jobId, deliverableCID)` |
| Indexer | `WorkSubmitted` → MongoDB | `deliverableCID` + status SUBMITTED |
| Hiển thị | Gateway Pinata + DB | `JobRegistry.getJob` |

---

## 6. Reputation (`ReputationStore`)

**Mặc định:** ví chưa init → **100 điểm**.

| Sự kiện | Thay đổi |
|---------|----------|
| Release / FL thắng dispute / SPLIT payout | FL **+10**, Client **+5** |
| Client thắng dispute (refund) | FL **−15** |
| Arbitrator commit không reveal | **−10** |

| Tier | Điểm | Hệ quả |
|------|------|--------|
| Restricted | &lt; 50 | Không `createJob` |
| Warning | 50–79 | Không bid, không `raiseDispute` |
| Normal | 80–119 | Bid, dispute, join pool (≥80) |
| Trusted | ≥ 120 | Tier cao nhất |

---

## Tài liệu liên quan

- [admin-roles-vi.md](admin-roles-vi.md) — governance roles
- [workflow-e2e-vi.md](workflow-e2e-vi.md) — luồng đầy đủ
- [demo-qa-defense-vi.md](demo-qa-defense-vi.md) — Q&A phòng vấn
- [on-chain-off-chain-map-vi.md](on-chain-off-chain-map-vi.md) — mapping dữ liệu

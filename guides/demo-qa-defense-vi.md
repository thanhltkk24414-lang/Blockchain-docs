# FAQ & phòng vấn — Demo FAPEX

> **English summary:** Anticipated Q&A for thesis defense and live demo — architecture decisions, trade-offs, and honest limitations.

**Dùng khi:** Hội đồng chấm đồ án, demo live Q&A, review GitHub.

**Cập nhật:** 2026-06-28

---

## A. Tổng quan dự án

### Q1: FAPEX khác Upwork/Fiverr thế nào?

**Trả lời:** Tiền khóa trong `EscrowVault` on-chain, không qua tài khoản platform. Client phê duyệt hoặc tranh chấp theo smart contract. Identity qua ví (SIWE), không email/password. Phí 3% deposit ghi rõ trong bytecode.

### Q2: Tại sao chọn Sepolia, không mainnet?

**Trả lời:** Chi phí gas và rủi ro tài chính thấp cho đồ án. Dùng **MockUSDC** (mint miễn phí), không phải USDC Circle thật. Logic contract giống production; chỉ khác token và dispute timings.

### Q3: Demo dispute mất bao lâu?

**Trả lời (on-chain sau redeploy 2026-06-27):**

| Phase | Demo |
|-------|------|
| Evidence (initial + rebuttal) | **0 → 10 phút** |
| Commit vote | **10 → 13 phút** |
| Reveal vote | **13 → 16 phút** |
| Appeal window | **30 phút** sau finalize |

Production: 72h / 120h / 144h / 168h / appeal 72h (`DisputeTimings.prod.sol`).

---

## B. Web3 & backend

### Q4: "Web3" mà sao vẫn có backend? Có phi tập trung không?

**Trả lời:** Escrow, status job, vote tranh chấp = **on-chain** (phi tập trung về tài sản). Backend là **lớp tiện ích**:
- Indexer: đọc events → MongoDB (cache, không quyết định payout)
- IPFS upload proxy (Pinata API key không để trên browser)
- SIWE session JWT cho API rate limit & profile

**Nguyên tắc:** Nếu backend tắt, user vẫn gọi contract trực tiếp qua Etherscan/wagmi; tiền an toàn on-chain.

### Q5: Tại sao SIWE thay vì password?

**Trả lời:**
- Wallet = identity Web3-native (EIP-4361)
- Không lưu password hash → giảm surface tấn công
- Tx on-chain bắt buộc ký MetaMask anyway — SIWE thống nhất một ví
- JWT chỉ cho REST API (bids, metadata), không thay thế chữ ký tx

### Q6: MongoDB có bị sửa để ăn cắp tiền không?

**Trả lời:** Không. MongoDB chỉ mirror metadata. `depositEscrow`, `approveAndRelease`, dispute payout chỉ contract thực thi. Indexer **đọc** chain, không ký payout (trừ `createJob` relay tùy chọn).

### Q7: IPFS đóng vai trò gì? Sao không lưu file on-chain?

**Trả lời:** On-chain lưu **CID/hash** (rẻ gas). File (PDF, ảnh) trên IPFS qua Pinata. `submitWork(jobId, deliverableCID)` và `submitEvidence(jobId, keccak256(CID))` — minh bạch + chi phí hợp lý.

### Q8: CORS `https://*.vercel.app` có an toàn không?

**Trả lời:** Chỉ cho phép subdomain `*.vercel.app` gọi API với credentials. Không mở `*`. Wildcard implement trong `corsOrigins.js` — match hostname suffix. Production Railway set `ALLOWED_ORIGINS` gồm localhost dev + Vercel.

---

## C. Smart contracts & dispute

### Q9: Tại sao chưa dùng Chainlink VRF cho sortition?

**Trả lời:**
- VRF cần subscription LINK, callback contract, latency async — phức tạp cho deadline đồ án
- MVP dùng `block.prevrandao` + **stake 50 USDC** + slash no-reveal → economic security
- Stub `VRFSortitionStub.sol` và doc v2 đã sẵn — roadmap rõ

**Nếu bị hỏi sâu:** Prevrandao trên PoS Sepolia khó manipulate single block cho demo; production mainnet nên VRF.

### Q10: Tại sao prevrandao mà không random thuần?

**Trả lời:** `block.prevrandao` (EIP-4399) là beacon randomness post-Merge, không cho phép miner predict trước trong cùng block context. Tốt hơn `block.timestamp` alone. Vẫn không bằng VRF verifiable off-chain.

### Q11: Stake và slash arbitrator hoạt động thế nào?

**Trả lời:**
- Join pool: stake **≥50 USDC** trên `PlatformTreasury`, reputation **≥80**
- Commit nhưng không reveal: slash **5 USDC** + reputation **−10**
- Vote thiểu số (sai kết quả): **không** bị phạt (khác Kleros coherent penalty)
- Vote đúng: reward từ pool dispute fee

### Q12: Sao cần 5 arbitrator? Pool < 5 thì sao?

**Trả lời:** `PANEL_SIZE = 5`, quorum reveal **≥3**. `raiseDispute` revert nếu `poolSize < 5`. Demo: `npm run seed:arbitrators` mint/stake/join 5 ví.

### Q13: Demo timing vs production — redeploy có ảnh hưởng gì?

**Trả lời:** Dispute windows **bake trong bytecode** (`DisputeTimings.sol` tại compile). Đổi timing = redeploy EscrowVault + ArbitratorPanel (+ link lại). Job cũ trên **Legacy JobRegistry** `0xE5425…` không migrate tự động — tạo job mới trên registry mới `0x3026…`.

### Q14: So với Kleros thì thiếu gì?

**Trả lời:** Xem [dispute-kleros-comparison-vi.md](dispute-kleros-comparison-vi.md). Tóm tắt: không có coherent vote penalty, appeal 1 vòng, manual execute, admin force resolve khi quorum fail.

### Q14b: `finalizeDisputeVoting` revert khi &lt;3 reveal — có đúng không?

**Trả lời:** **Đúng — đây là hành vi on-chain mong đợi.** Sau cửa reveal, contract đếm vote hợp lệ (FREELANCER_WIN / CLIENT_WIN / SPLIT). Nếu `validVotes < MIN_QUORUM` (3) → `revert InsufficientQuorum()` (selector `0x50884582`). UI arbitrator **tắt nút Finalize** khi chưa đủ 3 reveal; hiển thị link [`/admin#quorum-failed`](/admin#quorum-failed).

**Cơ chế demo khi quorum fail:**
1. Chỉ 1–2 arbitrator reveal (hoặc commit nhưng không reveal → bị slash 5 USDC + reputation −10).
2. Sau phút 16 (demo timing), bất kỳ ai gọi finalize → revert `InsufficientQuorum`.
3. Admin ví có `ROLE_FORCE_RESOLVER` mở **`/admin` → Quorum failed** → `adminForceResolve` + event `AdminForceResolved` trên Etherscan.

**One-liner:** *Quorum fail không phải bug — đó là lúc governance khẩn cấp thay thế panel.*

### Q14c: Pauser / Force resolver có nhận USDC như arbitrator không?

**Trả lời:** **Không.** Pauser (`ROLE_PAUSER`) và Force resolver (`ROLE_FORCE_RESOLVER`) là **vai trò vận hành / governance** — không có payout USDC trực tiếp trong bytecode.

| Vai trò | Thu nhập on-chain |
|---------|-------------------|
| **Arbitrator** (vote đúng đa số) | Nhận phần **50% dispute fee** chia đều cho arbitrator vote trùng kết quả (`_rewardCorrectArbitrators` → `PlatformTreasury.rewardArbitrator`) |
| **Arbitrator** (commit, không reveal) | **Slash 5 USDC** stake + reputation **−10** |
| **Pauser** | Không — chỉ `setPaused` khẩn cấp |
| **Force resolver** | Không — chỉ `adminForceResolve` khi quorum fail |
| **Platform** | 3% deposit fee + phần dispute fee còn lại vào `PlatformTreasury` |

Arbitrator còn cần **stake ≥50 USDC** để vào pool; unstake bị chặn khi `activeDisputeCount > 0`.

### Q14d: Điểm reputation hoạt động thế nào? (`ReputationStore.sol`)

**Trả lời:**

**Điểm mặc định:** Ví chưa khởi tạo → **100 điểm** (`getScore`).

**Tăng điểm (authorized contracts gọi `updateScore(user, true, amount)`):**
| Sự kiện | Điểm |
|---------|------|
| Freelancer thắng dispute / release bình thường | **+10** (freelancer) |
| Client thắng hoặc approve release | **+5** (client) |

**Giảm điểm (`updateScore(user, false, amount)`):**
| Sự kiện | Điểm |
|---------|------|
| Freelancer thua dispute (client refund) | **−15** |
| Arbitrator commit nhưng không reveal | **−10** |

**4 tầng (`getTier`):**

| Tier | Điểm | Hệ quả demo |
|------|------|-------------|
| **Restricted** | &lt; 50 | Không `createJob` (client) |
| **Warning** | 50–79 | Không bid / không `raiseDispute` |
| **Normal** | 80–119 | Đủ điều kiện bid, dispute, **join arbitrator pool** (`MIN_SCORE = 80`) |
| **Trusted** | ≥ 120 | Tier cao nhất |

**One-liner:** *Reputation là cổng on-chain — không phải token; arbitrator cần ≥80 + stake 50 USDC.*

---

## D. Chainlink & oracle

### Q15: Landing ghi "Oracle: Chainlink" nhưng không thấy giá ETH?

**Trả lời:** Chainlink ETH/USD feed Sepolia (`0x694AA…`) đã tích hợp code nhưng **gỡ live UI** (giảm RPC, scope đồ án). Feed vẫn documented; stats landing giữ label Oracle cho roadmap. VRF = v2.

### Q16: The Graph thay indexer được không?

**Trả lời:** v2 plan. Hiện custom indexer đủ cho thesis: checkpoint `lastBlock`, Socket.io push, không phụ thuộc hosted subgraph. The Graph tốt hơn cho public query và historical analytics.

---

## E. Demo thực tế

### Q17: Cần mấy ví MetaMask?

**Trả lời tối thiểu:**
1. **Client** — createJob, deposit, approve/dispute
2. **Freelancer** — bid, startWork, submitWork
3. **Arbitrator** (1 trong 5 seed) — commit/reveal vote

Có thể dùng 2 ví nếu bỏ qua arbitrator live (chỉ show UI).

### Q18: Lệnh seed trước demo?

```bash
# Từ monorepo root
npm run seed:arbitrators    # pool ≥ 5
npm run check:dispute       # verify poolSize
```

Cần Sepolia ETH trên deployer wallet (`contracts/.env` `PRIVATE_KEY`).

### Q19: JobRegistry mismatch — banner đỏ trên UI?

**Trả lời:** Frontend `VITE_JOB_REGISTRY_ADDRESS` ≠ Railway `JOB_REGISTRY_ADDRESS`. Đồng bộ với `deployments/sepolia.json` `0x302629f82d51b0972ffc3A99cbE355F4acEf908d`.

### Q20: raiseDispute revert?

**Checklist:**
- Job status SUBMITTED hoặc IN_PROGRESS
- `poolSize ≥ 5`
- Client đủ MockUSDC cho dispute fee (2%, cap 50)
- Gas ≥700k (sortition tốn gas)
- Đúng ví client on-chain

---

## F. Bảo mật & hạn chế thành thật

### Force resolve vs thao túng

**Câu hỏi:** Admin có thể “chọn người thắng” tranh chấp không? Force resolve có phải backdoor?

**Trả lời (demo script):**

1. **Luồng chuẩn (99% case):** Client `raiseDispute` → 5 arbitrator sortition → evidence → commit–reveal → **≥3 vote** → bất kỳ ai gọi `finalizeDisputeVoting` → `executeArbitrationResult`. Admin **không** set outcome.
2. **Force resolve = fallback quorum fail:** Chỉ khi sau cửa reveal vẫn **&lt;3** vote hợp lệ (`InsufficientQuorum` nếu ai đó thử finalize). Hợp đồng emit `AdminForceResolved` — công khai trên Etherscan.
3. **UI minh bạch (`/admin`):** Panel force resolve **tắt nút** nếu job chưa DISPUTED, reveal chưa hết, hoặc đã đủ quorum; hiển thị phase, reveal count, danh sách arbitrator; bắt checkbox + gõ `FORCE` trước submit.
4. **Thành thật MVP:** Bytecode vẫn cho admin gọi `adminForceResolve` khi job DISPUTED (chưa thêm guard quorum on-chain — tránh redeploy). Lớp tin cậy = UI + docs + audit log + **roadmap multisig/timelock**.
5. **Production:** `transferAdmin` → Safe multisig; `ROLE_FORCE_RESOLVER` chỉ grant cho multisig; rationale khẩn cấp công bố trước tx.

**One-liner:** *“Arbitrators quyết định tranh chấp; admin chỉ dọn đống quorum fail — và mọi lần dọn đều có event on-chain.”*

| Câu hỏi | Trả lời ngắn |
|---------|--------------|
| Admin có quyền gì? | On-chain: `setPaused`, `grantRole`/`revokeRole`, `joinPool`, `adminForceResolve`. UI tại **`/admin`** (hook `useAdminAccess` đọc `admin()` + delegated roles) |
| Reentrancy? | CEI pattern; USDC không callback |
| Front-running vote? | Commit–reveal che choice đến reveal phase |
| Sybil arbitrator? | Stake 50 USDC + reputation gate |
| Indexer lag? | Poll ~2 phút; UI đọc on-chain khi cần |
| Reviews? | API placeholder — chưa production |

---

## H. Admin dashboard (`/admin`)

**Câu hỏi:** Có giao diện quản trị không? Ai được vào?

**Trả lời:**

1. Route **`/admin`** — chỉ hiện nav **Admin** khi ví có quyền on-chain.
2. **Access guard:** `EscrowVault.admin()`, `ArbitratorPanel.admin()`, deployer trong `deployments/sepolia.json`, hoặc delegated roles (`ROLE_PAUSER`, `ROLE_FORCE_RESOLVER`, `ROLE_ARBITRATOR_MANAGER`).
3. **Không có off-chain admin RBAC** — Mongo enum `admin` không dùng; demo trung thực: quyền tài chính/khẩn cấp nằm trên contract.
4. **Panels:** contract addresses (copy), **How governance works** (normal vs emergency), pause/unpause (modal confirm), grant/revoke roles (cảnh báo force_resolver + role holders), arbitrator pool + `joinPool`, force resolve (đọc on-chain + gõ `FORCE`), stats từ `GET /api/admin/stats`.
5. **Tx pattern:** `simulateContract` + `sendTransaction` (tránh lỗi MetaMask RPC).
6. **Demo wallet:** deployer `0x523eBd853a1638065f148A05c0Ca423E490D92f7` trên Sepolia.

**One-liner:** *"Governance is wallet-gated on-chain; the dashboard is a thin UI over existing admin functions, not a custodial backdoor."*

---

## G. One-liner cho slide kết

> **FAPEX khóa thanh toán freelance trong escrow Sepolia, giải quyết tranh chấp bằng hội đồng arbitrator có stake và vote commit–reveal; backend và IPFS phục vụ UX, chain giữ quyền quyết định tài chính.**

---

## Tài liệu liên quan

- [demo-script-vi.md](demo-script-vi.md)
- [demo-seed-data-vi.md](demo-seed-data-vi.md)
- [admin-cheatsheet-vi.md](admin-cheatsheet-vi.md)
- [on-chain-off-chain-map-vi.md](on-chain-off-chain-map-vi.md)
- [issue-audit-status-vi.md](issue-audit-status-vi.md)

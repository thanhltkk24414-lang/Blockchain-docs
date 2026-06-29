# FAPEX — Vai trò quản trị & governance

> **English summary:** On-chain admin and delegated roles — responsibilities, when to use each, role application flow, and earnings (arbitrators earn USDC dispute fees; pauser/force resolver do not).

**Cập nhật:** 2026-06-30 · **Dashboard:** `/admin` · **Deployer Sepolia:** `0x523eBd853a1638065f148A05c0Ca423E490D92f7`

---

## 1. Tổng quan quyền

| Vai trò | Contract | Cách nhận | Thu nhập USDC on-chain |
|---------|----------|-----------|------------------------|
| **Contract admin** | `EscrowVault`, `ArbitratorPanel`, `ReputationStore`, `PlatformTreasury`, `JobRegistry` | `constructor` → deployer; `transferAdmin` | Không trực tiếp từ dispute |
| **ROLE_PAUSER** | `EscrowVault` (`ROLE_PAUSER = 1`) | `grantRole(addr, 1)` bởi admin | **Không** |
| **ROLE_FORCE_RESOLVER** | `EscrowVault` (`ROLE_FORCE_RESOLVER = 2`) | `grantRole(addr, 2)` bởi admin | **Không** |
| **ROLE_ARBITRATOR_MANAGER** | `ArbitratorPanel` (`ROLE_ARBITRATOR_MANAGER = 1`) | `grantRole(addr, 1)` bởi admin | **Không** |
| **Arbitrator (pool)** | `ArbitratorPanel` + stake | Stake 50 USDC + `joinPool` | **Có** — 50% dispute fee chia cho arb vote đúng kết quả (trừ SPLIT) |

**Nguyên tắc:** Pauser và Force resolver là **vận hành / khẩn cấp** — không tham gia reward pool tranh chấp.

---

## 2. Contract admin (deployer)

**Địa chỉ hiện tại:** xem `deployments/sepolia.json` → `deployer`.

### Quyền đầy đủ (mọi contract)

- `transferAdmin(newAdmin)` — chuyển quyền admin
- `setAuthorizedContract` — authorize EscrowVault ↔ Registry/Panel/Treasury/Reputation
- `grantRole` / `revokeRole` — delegated roles
- `setPaused` — emergency pause (admin không cần ROLE_PAUSER)
- `adminForceResolve` — khi quorum fail (admin không cần ROLE_FORCE_RESOLVER)
- `ArbitratorPanel.joinPool(addr)` — thêm arbitrator vào pool
- `npm run seed:arbitrators` — script dùng deployer key

### Khi nào dùng

| Tình huống | Hành động |
|------------|-----------|
| Redeploy / cấu hình môi trường | Update env Railway + Vercel |
| Seed pool demo | `seed:arbitrators`, `join:arbitrators` |
| Production roadmap | `transferAdmin` → Gnosis Safe multisig |

---

## 3. ROLE_PAUSER

**Hàm:** `EscrowVault.setPaused(bool)`

### Khi bùng phát sự cố

- Lỗi nghiêm trọng phát hiện trên contract đang deploy
- Tấn công / exploit đang diễn ra — **dừng** tx mới: `depositEscrow`, `approveAndRelease`, `raiseDispute`, `fileAppeal`
- Tiền trong escrow **không** bị rút khi pause — chỉ chặn thao tác mới

### Khi KHÔNG dùng

- Tranh chấp bình thường giữa client/freelancer
- Quorum fail (dùng Force resolver)
- Thay thế quyết định arbitrator

**Lưu ý:** `startWork` vẫn có thể được phép khi paused (theo thiết kế terms) — kiểm tra test `Emergency pause`.

---

## 4. ROLE_FORCE_RESOLVER

**Hàm:** `EscrowVault.adminForceResolve(jobId, decision)`

### Chỉ khi quorum fail

1. Sau cửa reveal, **&lt;3** vote reveal hợp lệ.
2. `finalizeDisputeVoting` revert `InsufficientQuorum`.
3. Force resolver (hoặc admin) chọn outcome: `FREELANCER_WIN`, `CLIENT_WIN`, hoặc `SPLIT_50_50`.
4. Emit `AdminForceResolved` — công khai Etherscan.

### Khi KHÔNG dùng

- Đã có ≥3 reveal và majority rõ ràng
- Muốn “đảo” kết quả arbitrator đã vote — vi phạm tin cậy MVP

**UI:** `/admin` → Quorum failed — disable nếu chưa đủ điều kiện; yêu cầu gõ `FORCE`.

**Production:** chỉ grant cho multisig; rationale công bố off-chain trước tx.

---

## 5. ROLE_ARBITRATOR_MANAGER

**Hàm:** `ArbitratorPanel.joinPool(address)` **thay người khác**

### Khi dùng

- Arbitrator đã stake 50 USDC nhưng chưa tự `joinPool`
- Demo / onboarding — admin join hộ sau khi verify stake + reputation ≥80
- `npm run seed:arbitrators` dùng admin key

### Khi KHÔNG dùng

- Thay thế sortition hoặc chọn arbitrator cho dispute cụ thể
- Join người chưa stake đủ 50 USDC (revert `InsufficientStake`)

---

## 6. Luồng đăng ký vai trò (Profile + Admin)

Governance **on-chain**; đăng ký **off-chain** qua API để admin review trước khi `grantRole` / `joinPool`.

### 6.1 Delegated roles (Pauser, Force resolver, Arbitrator manager)

| Bước | Ai | Hành động |
|------|-----|-----------|
| 1 | User | `/profile` → Apply for role → chọn `pauser` / `force_resolver` / `arbitrator_manager` + lý do ≥20 ký tự |
| 2 | Backend | `POST /api/admin/role-applications` (SIWE JWT) |
| 3 | Admin | `/admin` → xem pending → approve/reject (`PATCH /api/admin/role-applications/:id`) |
| 4 | Admin ví on-chain | `EscrowVault.grantRole(wallet, ROLE)` hoặc `ArbitratorPanel.grantRole` — **thủ công qua UI `/admin` hoặc** `scripts/grant-platform-roles.js` |

**Lưu ý:** Approve API **không** tự grant on-chain — admin phải gửi tx `grantRole`.

### 6.2 Arbitrator pool

| Bước | Ai | Hành động |
|------|-----|-----------|
| 1 | User | Stake ≥50 USDC trên `PlatformTreasury` |
| 2 | User | `/profile` hoặc arbitrator dashboard → Apply → `POST /api/arbitrator/applications` |
| 3 | Backend | Verify stake on-chain trước khi lưu application |
| 4 | Admin | Approve application → `joinPool(wallet)` on-chain (admin hoặc ROLE_ARBITRATOR_MANAGER) |

### 6.3 CLI thay thế

```bash
node scripts/grant-platform-roles.js --pauser 0x...
node scripts/grant-platform-roles.js --force-resolver 0x...
npm run seed:arbitrators
```

---

## 7. Thu nhập arbitrator vs governance

| Nguồn | Arbitrator | Pauser / Force resolver / Manager |
|-------|------------|-----------------------------------|
| Vote đúng đa số (FL/Client win) | 50% dispute fee / số arb đúng | — |
| SPLIT 50-50 | **Không** reward | — |
| No-reveal | Slash **5 USDC** + rep −10 | — |
| Platform fee 3% + 2% service | — | Vào Treasury (admin không rút trực tiếp trong MVP) |

---

## 8. Truy cập `/admin`

Hook `useAdminAccess` kiểm tra:

- `EscrowVault.admin()` hoặc `ArbitratorPanel.admin()`
- Deployer trong `deployments/sepolia.json`
- Delegated roles trên chain

**Không** dùng MongoDB `User.role === admin` cho quyền tài chính.

---

## Tài liệu liên quan

- [admin-cheatsheet-vi.md](admin-cheatsheet-vi.md) — 1 trang demo
- [platform-mechanisms-vi.md](platform-mechanisms-vi.md) — SPLIT, appeal, pool ≥10
- [demo-qa-defense-vi.md](demo-qa-defense-vi.md) — Q&A force resolve
- [manual-vi.md](manual-vi.md) — Railway, Vercel, SIWE_DOMAIN

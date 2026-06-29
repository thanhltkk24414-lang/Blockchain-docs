# Demo script thuyết trình — FAPEX

> **English summary:** ~15-minute live demo script on Sepolia — wallets, timing, seed commands, and talking points.

**Thời lượng:** ~15 phút · **Mạng:** Sepolia · **Cập nhật:** 2026-06-30

---

## 0. Chuẩn bị trước khi vào phòng

### 0.1 Ví MetaMask (tối thiểu 3)

| Ví | Vai trò | Cần có |
|----|---------|--------|
| **Wallet A** | Client | Sepolia ETH + MockUSDC (~150 USDC cho job 100 + fee) |
| **Wallet B** | Freelancer | Sepolia ETH |
| **Wallet C** | Arbitrator (1 trong pool seed) | Sepolia ETH + đã stake 50 USDC |

### 0.2 Seed arbitrator pool (bắt buộc nếu pool < 5)

```bash
# Từ monorepo root — cần PRIVATE_KEY + SEPOLIA_RPC trong contracts/.env
npm run seed:arbitrators
npm run check:dispute          # poolSize ≥ 5 (dispute); ≥ 10 nếu demo appeal
```

Nếu deployer hết ETH: `npm run join:arbitrators`

Private keys seed: `deployments/sepolia-arbitrators.json` (gitignored)

### 0.3 Kiểm tra production

| Check | URL / lệnh |
|-------|------------|
| Backend health | https://fapex-backend-production.up.railway.app/health |
| Frontend | Vercel production URL |
| Contract sync | So khớp `VITE_JOB_REGISTRY_ADDRESS` = `0x302629f82d51b0972ffc3A99cbE355F4acEf908d` |

### 0.4 Địa chỉ contract (nhắc nhanh)

| Contract | Address |
|----------|---------|
| MockUSDC | `0x2293193Eaa5CE5253d5e081046a06dB077f26f8e` |
| JobRegistry | `0x302629f82d51b0972ffc3A99cbE355F4acEf908d` |
| EscrowVault | `0x5f8C4c552F49103cA84dF455571155C8268C2aF5` |
| ArbitratorPanel | `0x490Afc952af85aB0dEb375Bd36A65db5E1F47418` |

---

## 1. Mở đầu — Landing (2 phút)

**URL:** `/`

**Nói:**
> FAPEX là nền tảng freelance Web3 trên Sepolia. Escrow USDC on-chain, tranh chấp qua hội đồng arbitrator có stake. Không password — đăng nhập bằng ví SIWE.

**Show:**
- Hero search, tags (Solidity, React, …)
- Stats: Network Sepolia, Platform fee 3%, Settlement USDC
- Light/dark theme toggle
- **Không** show live ETH price (đã gỡ — mention Chainlink trong Q&A)

**Chuyển:** `/browse` hoặc Connect wallet

---

## 2. Đăng nhập & đăng ký (1 phút)

1. Connect MetaMask → **Sign in (SIWE)**
2. `/profile` → username + role **Client**

**Điểm nhấn:** EIP-4361, JWT chỉ cho API — tx vẫn ký MetaMask.

---

## 3. Client flow (4–5 phút)

**Ví:** Wallet A (Client)

| # | Hành động | Nói khi demo |
|---|-----------|--------------|
| 1 | `/client` → Create job | Metadata upload IPFS qua backend Pinata |
| 2 | Budget 100 USDC, duration | Client ký `createJob` on-chain |
| 3 | Chờ job list / refresh | Indexer sync MongoDB (~2 phút max) |
| 4 | Mở job detail | Show on-chain job ID + Etherscan link |

**Tip:** Nếu indexer chậm, show tx hash trên Sepolia Etherscan.

---

## 4. Freelancer flow (3–4 phút)

**Đổi ví:** Wallet B (Freelancer)

| # | Hành động |
|---|-----------|
| 1 | SIWE + role Freelancer |
| 2 | `/jobs/:id` → Submit bid |
| 3 | Đổi lại Wallet A → Accept bid |
| 4 | Mint MockUSDC nếu cần |
| 5 | **Approve & Deposit escrow** — show gas modal |
| 6 | Wallet B → Start work → upload file → Submit |

**Điểm nhấn:** Freelancer gán on-chain lúc deposit, không lúc accept bid.

---

## 5A. Nhánh nhanh — Happy path (2 phút)

**Ví:** Wallet A

1. **Approve & Release**
2. Show freelancer nhận USDC (trừ service fee 2%)
3. Job status COMPLETED on-chain

**Kết:** Escrow minh bạch, không cần platform giữ tiền.

---

## 5B. Nhánh dispute — Demo đầy đủ (timing)

> Dùng khi có thời gian chờ hoặc đã pre-record clip. Toàn bộ on-chain ~16 phút + appeal 30 phút.

### Timeline sau `raiseDispute`

| Giai đoạn | Thời gian | Hành động |
|-----------|-----------|-----------|
| Evidence | **0 → 10 phút** | Client + Freelancer nộp bằng chứng |
| Commit vote | **10 → 13 phút** | Arbitrator Wallet C → commit |
| Reveal vote | **13 → 16 phút** | Arbitrator → reveal |
| Finalize | sau 16 phút | Bất kỳ ai → `finalizeDisputeVoting` |
| Appeal window | **30 phút** | Optional `fileAppeal` |
| Execute | sau appeal | `executeArbitrationResult` |

### Các bước live (rút gọn cho hội đồng)

1. Wallet A → **Raise dispute** (job SUBMITTED)
2. Cả hai party → submit evidence (show IPFS panel)
3. Wallet C → `/arbitrator` → commit + reveal (giải thích commit–reveal chống copy)
4. **Nếu không đủ thời gian live:** show Etherscan tx đã record + UI phase stepper
5. Finalize → show kết quả FREELANCER_WIN / CLIENT_WIN / **SPLIT 50-50**
6. **SPLIT talking point:** escrow chia đôi — FL 50% (trừ fee), client nhận phần còn lại; dispute fee 50% hoàn initiator, **không** thưởng arbitrator

### 5C. Nhánh appeal (nếu còn thời gian + pool ≥10)

| Bước | Hành động | Nói |
|------|-----------|-----|
| 1 | Sau finalize vòng 1 | Cửa sổ **30 phút** |
| 2 | Party thua → `fileAppeal` | Phí **1.3×** dispute fee — approve USDC trước |
| 3 | Panel mới 5 arbitrator | Loại toàn bộ arb vòng 1 — cần pool **≥10** |
| 4 | Lặp evidence → vote → finalize vòng 2 | Không vòng 3 |
| 5 | `executeArbitrationResult` | Kết quả vòng 2 là final |

Pre-record clip nếu không đủ 30+ phút live.

---

## 6. Arbitrator highlight (1–2 phút)

**Nói:**
- Stake 50 USDC economic security
- Sortition 5 từ pool (`prevrandao` — VRF v2 roadmap)
- Slash 5 USDC nếu commit không reveal

**Show:** `/arbitrator` dashboard hoặc `npm run check:dispute`

---

## 7. Backend & realtime (1 phút)

**Nói:**
- Railway API index events → MongoDB cache
- Socket.io push khi job/dispute update
- Chain vẫn source of truth

**Optional:** `GET /health` trên browser

---

## 8. Kết & Q&A

**Slide kết:** 3 bullet
1. Escrow on-chain Sepolia
2. Dispute commit–reveal + stake
3. Full stack: contracts + indexer + Vercel UI

**Chuẩn bị Q&A:** [demo-qa-defense-vi.md](demo-qa-defense-vi.md)

---

## Troubleshooting nhanh

| Vấn đề | Xử lý |
|--------|--------|
| Wrong network | Banner → Switch Sepolia |
| Wallet mismatch | Đổi MetaMask đúng ví on-chain |
| JobRegistry mismatch | Sync env với `deployments/sepolia.json` |
| raiseDispute revert | `npm run check:dispute` — pool ≥5, đủ USDC |
| CORS | Railway `ALLOWED_ORIGINS` có `https://*.vercel.app` |
| Job cũ không hoạt động | Tạo job mới sau redeploy (registry `0x3026…`) |

---

## Script một dòng (copy cho MC)

```
Landing → SIWE Client → createJob → Freelancer bid → accept → deposit → submitWork
→ [approve HOẶC dispute evidence→vote→finalize] → Q&A
```

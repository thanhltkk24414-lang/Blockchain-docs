# Hướng dẫn seed data & demo hai ví — FAPEX

> **English summary:** Sepolia demo setup — two-wallet flow, arbitrator seeding, dispute timings, and current contract addresses.

**Cập nhật:** 2026-06-28 · Redeploy: 2026-06-27 (`disputeTimings: demo`)

---

## 1. Địa chỉ contract Sepolia

| Contract | Address |
|----------|---------|
| MockUSDC | `0x2293193Eaa5CE5253d5e081046a06dB077f26f8e` |
| JobRegistry | `0x302629f82d51b0972ffc3A99cbE355F4acEf908d` |
| EscrowVault | `0x5f8C4c552F49103cA84dF455571155C8268C2aF5` |
| ArbitratorPanel | `0x490Afc952af85aB0dEb375Bd36A65db5E1F47418` |
| PlatformTreasury | `0x666aF0Ec040377026E0D40870Bce7c165f741530` |
| ReputationStore | `0x5e457db6a8A44C143180043c5Bb7223C7222898E` |
| **Legacy JobRegistry** | `0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9` |

Nguồn: `deployments/sepolia.json`

---

## 2. Seed arbitrator pool

### Lệnh

```bash
# Monorepo root — cần contracts/.env (PRIVATE_KEY, SEPOLIA_RPC_URL)
npm run seed:arbitrators

# Kiểm tra
npm run check:dispute

# Nếu deployer hết ETH, arbitrator tự join:
npm run join:arbitrators
```

### Script làm gì?

Mỗi arbitrator (target **5**):
1. `MockUSDC.mint(50 USDC)`
2. `approve` + `PlatformTreasury.stakeAsArbitrator(50e6)`
3. Admin `ArbitratorPanel.joinPool(address)`

Keys lưu: `deployments/sepolia-arbitrators.json` (**gitignored**)

### Arbitrator addresses (sau seed — ví dụ)

Import private key từ `sepolia-arbitrators.json` vào MetaMask cho demo vote:

| # | Address (example from last seed) |
|---|----------------------------------|
| 1 | Check `deployments/sepolia-arbitrators.json` |
| 2 | … |
| 3 | … |
| 4 | … |
| 5 | … |

> Địa chỉ thay đổi mỗi lần seed ví mới. Luôn đọc file JSON sau `npm run seed:arbitrators`.

---

## 3. Demo hai ví (Client + Freelancer)

### Ví mẫu thường dùng

| Vai trò | Ghi chú |
|---------|---------|
| Client on-chain | Ví ký `createJob` — **phải** dùng khi `depositEscrow` |
| Freelancer | Ví gửi bid + `submitWork` |

**Lưu ý:** SIWE có thể dùng ví bất kỳ cho UI. Chỉ **transaction on-chain** yêu cầu đúng ví.

### Luồng từng bước

| Bước | MetaMask | Hành động |
|------|----------|-----------|
| 1 | Client | `createJob` |
| 2 | Freelancer | `POST /api/bids` |
| 3 | Tùy ý (SIWE) | Accept bid (DB) |
| 4 | **Client on-chain** | `depositEscrow` — freelancer zero trước deposit là **bình thường** |
| 5 | **Freelancer** | `startWork` → `submitWork` |

### Freelancer `0x000…000` khi OPEN

Trước `depositEscrow`, `Job.freelancer` on-chain = zero address. UI không coi là lỗi — freelancer gán lúc deposit.

---

## 4. Dispute demo timings

Sau redeploy 2026-06-27 (`DisputeTimings.demo`):

| Giai đoạn | Thời gian (từ `raiseDispute`) |
|-----------|-------------------------------|
| Evidence initial | 0 → **5 phút** |
| Evidence rebuttal | 5 → **10 phút** |
| Commit vote | **10 → 13 phút** |
| Reveal vote | **13 → 16 phút** |
| Finalize | sau **16 phút** |
| Appeal window | **30 phút** sau kết quả |

**Tổng evidence:** 0 → 10 phút.

Production (`npm run deploy:sepolia:prod`): 72h / 120h / 144h / 168h / appeal 72h.

```bash
node scripts/prepare-dispute-timings.js demo   # trước compile
node scripts/prepare-dispute-timings.js prod
npm run deploy:sepolia                         # auto demo
```

---

## 5. Điều kiện `raiseDispute`

| Yêu cầu | Giá trị |
|---------|---------|
| `ArbitratorPanel.poolSize` | ≥ **5** |
| Stake mỗi arbitrator | ≥ 50 USDC |
| Reputation | ≥ 80 (default 100) |
| Job status | **SUBMITTED** hoặc **IN_PROGRESS** |
| Dispute fee | 2% contract value, cap 50 USDC |

---

## 6. Demo dispute — checklist

1. `npm run seed:arbitrators` (một lần sau redeploy)
2. **Tạo job mới** trên registry `0x3026…` (job cũ legacy không dùng)
3. Freelancer `submitWork`
4. Client `raiseDispute`
5. 0–10 min: evidence
6. 10–13 min: arbitrator commit
7. 13–16 min: reveal
8. Finalize → execute (sau appeal window nếu không appeal)

### Kiểm tra nhanh

```bash
npm run check:dispute
curl https://fapex-backend-production.up.railway.app/health
```

---

## 7. Sau redeploy — việc cần làm

| # | Task |
|---|------|
| 1 | Cập nhật `deployments/sepolia.json` |
| 2 | Railway: contract env vars |
| 3 | Vercel: `VITE_*_ADDRESS` |
| 4 | `LEGACY_JOB_REGISTRY_ADDRESS=0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9` |
| 5 | `node backend/scripts/migrate-job-registry-index.js` |
| 6 | `npm run seed:arbitrators` |
| 7 | Tạo job demo mới |

MockUSDC thường **không** đổi địa chỉ khi redeploy (reuse `USDC_ADDRESS`).

---

## Tài liệu liên quan

- [demo-script-vi.md](demo-script-vi.md)
- [demo-qa-defense-vi.md](demo-qa-defense-vi.md)
- [on-chain-off-chain-map-vi.md](on-chain-off-chain-map-vi.md)

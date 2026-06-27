# Luồng E2E — FAPEX (VI)

## 1. Client đăng job

1. Kết nối MetaMask (Sepolia) → **Sign in (SIWE)** → `/profile` chọn role **Client**
2. `/client` → Create job: mô tả → upload metadata IPFS (`POST /api/ipfs/upload/metadata`)
3. On-chain `createJob(metadataCID, contractValue, duration)` — ví client ký
4. MongoDB: job từ API + indexer đồng bộ `JobCreated`

## 2. Freelancer bid & được chọn

1. Freelancer SIWE + role **Freelancer** → `/browse` tìm job
2. `POST /api/bids` → client `PATCH /api/bids/:id/accept`
3. Client nạp escrow: approve MockUSDC + `depositEscrow(jobId, freelancer)` — gán freelancer on-chain

## 3. Giao hàng

1. Freelancer `startWork` → upload file IPFS → `submitWork(jobId, deliverableCID)`
2. CID lưu on-chain trong JobRegistry

## 4. Phê duyệt hoặc tranh chấp

- **Happy path:** Client `approveAndRelease` → USDC cho freelancer
- **Tranh chấp:** `raiseDispute` → evidence IPFS `submitEvidence` → arbitrator commit/reveal → `finalizeDisputeVoting` → `executeArbitrationResult`

## 5. Arbitrator

- Stake ≥50 USDC trên `PlatformTreasury` + `joinPool`
- Console `/arbitrator` khi stake hợp lệ

## Nguồn sự thật

| Dữ liệu | Nguồn |
|---------|--------|
| Escrow, status job, dispute | **On-chain** |
| Mô tả job, bids, user profile | MongoDB (cache / off-chain) |
| File deliverable | IPFS (CID on-chain) |
| Reputation score | `ReputationStore` on-chain |

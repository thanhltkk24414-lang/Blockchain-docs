# Demo script thuyết trình — FAPEX (VI)

**Thời lượng:** ~15 phút · **Mạng:** Sepolia

## Chuẩn bị

- 3 ví MetaMask: Client, Freelancer, Arbitrator (đã stake)
- MockUSDC mint cho client
- `npm run seed:arbitrators` nếu pool < 5

## Kịch bản

### A. Landing (2 phút)

1. Mở `/` — giới thiệu hero search, tags, ETH/USD Chainlink
2. Tìm "Solidity" → chuyển `/browse`

### B. Client flow (5 phút)

1. SIWE + role Client
2. Tạo job 100 USDC, metadata IPFS
3. Chờ indexer / refresh job list

### C. Freelancer (4 phút)

1. Ví Freelancer bid → client accept
2. Client deposit escrow (show gas estimate modal)
3. Freelancer startWork + submit deliverable CID

### D. Release hoặc dispute (4 phút)

- **Nhanh:** Client approveAndRelease
- **Demo dispute:** raiseDispute → evidence → arbitrator vote (demo windows 15–60 phút)

## Điểm nhấn kỹ thuật

- SIWE không password
- CID on-chain, file trên IPFS
- ReputationStore badge trên header
- Network banner nếu sai chain

## Troubleshooting

| Vấn đề | Cách xử lý |
|--------|------------|
| Sai mạng | Banner vàng → Chuyển Sepolia |
| JobRegistry mismatch | Đồng bộ env Railway với `deployments/sepolia.json` |
| raiseDispute revert | Cần ≥5 arbitrator trong pool |

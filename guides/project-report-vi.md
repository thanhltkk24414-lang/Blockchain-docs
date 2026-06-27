# Báo cáo dự án — FAPEX (outline VI)

## 1. Giới thiệu

Nền tảng freelance Web3: escrow USDC, milestone, tranh chấp on-chain, reputation soulbound.

## 2. Công nghệ

| Lớp | Stack |
|-----|-------|
| Contracts | Solidity 0.8.20, Hardhat, Sepolia |
| Backend | Node, Express, MongoDB, Socket.io, Pinata |
| Frontend | React, Vite, wagmi, RainbowKit, SIWE |
| Oracle | Chainlink ETH/USD (Sepolia) |

## 3. Smart contracts

- JobRegistry, EscrowVault, ArbitratorPanel, PlatformTreasury, ReputationStore
- Dispute: commit-reveal, demo timings 15–60 phút trên Sepolia

## 4. Backend

- SIWE → JWT
- Event indexer với checkpoint `IndexerState`
- IPFS upload, không lưu file nhạy cảm chỉ trên server

## 5. Frontend

- Landing HYVE-style, browse, role dashboards
- Tx UX: modal, gas estimate, Etherscan

## 6. Bảo mật & hạn chế

- CEI payout, stake/slash arbitrator
- MongoDB là cache; chain là nguồn sự thật cho escrow
- VRF / The Graph: roadmap v2

## 7. Kết quả & demo

- Deploy Sepolia live
- Script demo: `docs/guides/demo-script-vi.md`

## 8. Hướng phát triển

- Chainlink VRF sortition
- The Graph indexing
- Mainnet + USDC thật

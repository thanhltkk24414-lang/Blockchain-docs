# Fapex Docs

> Tài liệu chính thức FAPEX — Web3 freelance escrow trên Sepolia.  
> **English:** See [overview-vi.md](guides/overview-vi.md) for project summary.

**Cập nhật:** 2026-06-30

---

## Bắt đầu nhanh

| Tài liệu | Mô tả |
|----------|--------|
| [**Tổng quan**](guides/overview-vi.md) | FAPEX là gì, monorepo, production URLs |
| [**Tech stack**](guides/tech-stack-vi.md) | Công nghệ và mục đích từng thành phần |
| [**Manual**](guides/manual-vi.md) | Hướng dẫn user + dev local |
| [**Demo script**](guides/demo-script-vi.md) | Kịch bản thuyết trình ~15 phút |
| [**Q&A phòng vấn**](guides/demo-qa-defense-vi.md) | FAQ hội đồng / demo |
| [**Cơ chế nền tảng**](guides/platform-mechanisms-vi.md) | SPLIT, appeal, deadline, reputation |
| [**Governance roles**](guides/admin-roles-vi.md) | Admin, pauser, force resolver, applications |

---

## Báo cáo & thiết kế

| Tài liệu | Mô tả |
|----------|--------|
| [Báo cáo dự án (VI)](guides/project-report-vi.md) | Outline khóa luận / đồ án |
| [Thiết kế hệ thống (VI)](guides/system-design-vi.md) | Kiến trúc, mermaid diagrams |
| [Luồng E2E (VI)](guides/workflow-e2e-vi.md) | Workflows Client / Freelancer / Arbitrator |
| [On-chain / Off-chain map](guides/on-chain-off-chain-map-vi.md) | Nguồn sự thật dữ liệu |
| [So sánh Kleros](guides/dispute-kleros-comparison-vi.md) | Dispute model |

---

## Triển khai & kỹ thuật

| Tài liệu | Mô tả |
|----------|--------|
| [Chainlink integration](guides/chainlink-integration-vi.md) | Price feed, VRF roadmap |
| [Contract interaction](guides/contract-interaction.md) | Địa chỉ Sepolia, luồng on-chain |
| [Deploy backend (Railway)](guides/deploy-backend.md) | Docker, env, Socket.io |
| [SIWE + JWT auth](guides/auth-api.md) | Wallet login API |
| [Admin cheatsheet](guides/admin-cheatsheet-vi.md) | 1 trang demo governance |
| [Admin roles](guides/admin-roles-vi.md) | Job descriptions chi tiết |
| [Demo seed data](guides/demo-seed-data-vi.md) | `seed:arbitrators`, pool ≥10 appeal, timings |
| [Ma trận audit](guides/issue-audit-status-vi.md) | Issue status SC/BE/FE/CL |

---

## Kiểm thử API

| Tài liệu | Mô tả |
|----------|--------|
| [Postman (tóm tắt)](guides/postman-testing.md) | health → auth → IPFS → jobs |
| [Postman chi tiết (VI)](guides/postman-walkthrough-vi.md) | Troubleshooting |
| [Production API test (VI)](guides/production-api-test-vi.md) | Railway smoke test |

---

## Báo cáo tiến độ (archive)

| Tài liệu | Mô tả |
|----------|--------|
| [Contributor 2 — backend progress](report/contributor2-progress.md) | SIWE, Railway, WebSocket |
| [Outline báo cáo gợi ý](report/report-outline-suggestions.md) | Điều chỉnh PHẦN 4 & 5 |

---

## Legal

- [Điều khoản hợp đồng — Fapex](legal/freelance-terms.md)

---

## Production (Jun 2026)

| Dịch vụ | URL |
|---------|-----|
| **Backend API** | https://fapex-backend-production.up.railway.app |
| Health | `/health` |
| Config | `/api/config` |
| Admin stats | `/api/admin/stats` |
| SIWE demo | `/siwe-sign.html` |
| WebSocket | `/socket.io` (JWT) |
| **Frontend** | Vercel (`*.vercel.app`) |

### Sepolia contracts

| Contract | Address |
|----------|---------|
| MockUSDC | `0x2293193Eaa5CE5253d5e081046a06dB077f26f8e` |
| JobRegistry | `0x302629f82d51b0972ffc3A99cbE355F4acEf908d` |
| EscrowVault | `0x5f8C4c552F49103cA84dF455571155C8268C2aF5` |
| ArbitratorPanel | `0x490Afc952af85aB0dEb375Bd36A65db5E1F47418` |
| PlatformTreasury | `0x666aF0Ec040377026E0D40870Bce7c165f741530` |
| ReputationStore | `0x5e457db6a8A44C143180043c5Bb7223C7222898E` |
| Legacy JobRegistry | `0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9` |

Canonical: monorepo `deployments/sepolia.json` · `disputeTimings: demo`

---

## Monorepo links

- [Root README](../README.md)
- [contracts/README](../contracts/README.md)
- [backend/README](../backend/README.md)
- [frontend/README](../frontend/README.md)

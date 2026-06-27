# Hướng dẫn sử dụng FAPEX (VI)

## Yêu cầu

- MetaMask + Sepolia ETH (gas)
- MockUSDC trên Sepolia (mint permissionless)

## Bước 1 — Ví & mạng

1. Cài MetaMask, thêm Sepolia
2. Nếu banner **Sai mạng** xuất hiện → bấm **Chuyển sang Sepolia**

## Bước 2 — Đăng nhập

1. **Connect** MetaMask
2. **Sign in (SIWE)** — ký message, không cần mật khẩu
3. `/profile` — username + chọn **Client** hoặc **Freelancer**

## Bước 3 — Client

- `/client` tạo job, upload mô tả
- Khi có bid: accept → mint USDC nếu thiếu → **Deposit escrow**
- Job detail: phê duyệt hoặc khiếu nại

## Bước 4 — Freelancer

- `/browse` lọc job → bid trên `/jobs/:id`
- Khi assigned: **Start work** → upload file → **Submit**

## Bước 5 — Arbitrator (tùy chọn)

- Stake 50 USDC + join pool (script admin)
- `/arbitrator` — vote tranh chấp

## API công khai

- Health: `GET /health`
- Config: `GET /api/config`

## Tài liệu thêm

- [Luồng E2E](workflow-e2e-vi.md)
- [Demo script](demo-script-vi.md)
- [Audit status](issue-audit-status-vi.md)

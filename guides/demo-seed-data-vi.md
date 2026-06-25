# Hướng dẫn demo hai ví (Sepolia)

Tài liệu này mô tả luồng demo Fapex với **hai ví MetaMask khác nhau**: một ví client (nạp escrow) và một ví freelancer (gửi proposal + nộp bàn giao).

## Địa chỉ mẫu (seed / INDEXER)

| Vai trò | Địa chỉ | Ghi chú |
|---------|---------|---------|
| Client on-chain | `0x523eBd853a1638065f148A05c0Ca423E490D92f7` | Ví backend / INDEXER tạo job trên JobRegistry |
| Freelancer (bid) | `0xA7aC8154fa3019f5e95Ba3720240C782C0e3ED70` | Ví dùng khi gửi proposal |

**Lưu ý:** SIWE đăng nhập có thể dùng ví bất kỳ. Chỉ các giao dịch on-chain mới yêu cầu đúng ví.

## Trạng thái OPEN và freelancer `0x000…000`

Khi job on-chain còn **OPEN**, trường `freelancer` trên JobRegistry thường là **`0x0000000000000000000000000000000000000000`**.

Đây là **bình thường** — freelancer chưa được gán. Client gán freelancer khi gọi `depositEscrow(jobId, freelancerAddress, …)` trong bước nạp escrow.

UI **không** coi đây là lỗi mismatch với bid đã accept. Deposit vẫn tiếp tục với địa chỉ từ bid đã chấp nhận.

## Luồng demo từng bước

### Bước 1 — Client tạo job (backend / INDEXER)

- Job được đăng ký on-chain với client = `0x523e…D92f7`.
- Trạng thái: **OPEN**, freelancer = `0x000…000`.

### Bước 2 — Freelancer gửi proposal

1. Chuyển MetaMask sang ví bidder, ví dụ `0xA7aC…ED70`.
2. Đăng nhập SIWE (có thể cùng ví bidder).
3. Gửi proposal cho job.

### Bước 3 — Client accept bid (off-chain / DB)

- Client đăng nhập SIWE (ví tùy ý để quản lý UI).
- Accept proposal → DB lưu `freelancerAddress` từ bid.
- On-chain vẫn OPEN, freelancer vẫn `0x000…000` cho đến bước 4.

### Bước 4 — Client nạp escrow (`depositEscrow`)

1. **Chuyển MetaMask sang `0x523e…D92f7`** (client on-chain) — không phải ví SIWE nếu khác.
2. Mint MockUSDC testnet nếu cần.
3. Bấm **Approve & deposit escrow**.
4. UI khóa địa chỉ freelancer = bid đã accept (`0xA7aC…ED70`).
5. Sau khi tx thành công: job **ASSIGNED**, freelancer on-chain = địa chỉ bid.

### Bước 5 — Freelancer nộp bàn giao

1. **Chuyển MetaMask sang `0xA7aC…ED70`** (cùng ví đã bid).
2. Gọi `startWork` (nếu ASSIGNED) rồi `submitWork` qua UI **Nộp bàn giao**.

Nếu MetaMask dùng ví khác byte (không chỉ khác chữ hoa), contract revert `OnlyFreelancer`.

## Job #6 (OPEN — deposit bị chặn nhầm)

| Trường | Giá trị |
|--------|---------|
| On-chain status | OPEN |
| Freelancer on-chain | `0x000…000` (chưa gán) |
| Bid đã accept | `0xA7aC8154fa3019f5e95Ba3720240C782C0e3ED70` |

**Cách xử lý:** Client chuyển MetaMask sang `0x523e…D92f7`, vào trang job, bấm nạp escrow. Không cần sửa gì on-chain — freelancer zero là đúng trước deposit.

## Job #5 (IN_PROGRESS — ví freelancer sai)

| Trường | Giá trị |
|--------|---------|
| Freelancer on-chain | `0xA7aC8154fa3019f5e95Ba3720240C782C0e3ED70` |
| Status | IN_PROGRESS |

Freelancer thấy banner mismatch nếu MetaMask không trùng địa chỉ on-chain.

**Cách xử lý:**

1. Đổi MetaMask sang đúng `0xA7aC…ED70` (nếu có private key), **hoặc**
2. Tạo job mới và deposit với đúng địa chỉ bid.

Không thể đổi freelancer on-chain sau khi đã `depositEscrow`.

## Tóm tắt nhanh

| Bước | MetaMask phải là |
|------|------------------|
| Đăng nhập / accept bid | Tùy ý (SIWE) |
| `depositEscrow` | `0x523e…D92f7` (client on-chain) |
| `submitWork` | Ví đã gửi proposal (bid wallet) |
| OPEN + freelancer `0x0` | Bình thường — chưa deposit |

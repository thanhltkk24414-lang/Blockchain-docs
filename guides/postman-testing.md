# Hướng dẫn test API bằng Postman

Kiểm thử luồng backend Fapex: **health → đăng nhập SIWE → upload IPFS → tạo job**.

> **Hướng dẫn chi tiết từng bước (tiếng Việt):** xem [postman-walkthrough-vi.md](./postman-walkthrough-vi.md) — gồm cách ký SIWE MetaMask, checklist, và troubleshooting đầy đủ.

## Yêu cầu

| Thành phần | Biến `.env` | Ghi chú |
|------------|-------------|---------|
| MongoDB | `MONGODB_URI` | Bắt buộc cho nonce, jobs, auth/me |
| JWT | `JWT_SECRET` | Bắt buộc cho verify và route có Bearer |
| Pinata | `PINATA_JWT` hoặc API key | Bắt buộc cho upload IPFS |
| Sepolia RPC | `RPC_URL` | Cho POST job (on-chain) và arbitrator status |
| SIWE | `SIWE_DOMAIN`, `APP_URL`, `CHAIN_ID` | Khớp với ví khi ký |

Xem thêm: [auth-api.md](./auth-api.md).

## Khởi động MongoDB (Windows)

API `/health` và `/api/arbitrator/*` chạy **không cần** MongoDB. Các route auth, jobs và event indexer cần `MONGODB_URI` hợp lệ.

**Docker (khuyến nghị):**

```bash
cd backend
npm run docker:mongo
# Script chạy container detached và thoát ngay — không treo terminal.
# Nếu container "mongo" đã tồn tại, script chỉ start lại và in connection string.
```

Trong `backend/.env`:

```env
MONGODB_URI=mongodb://127.0.0.1:27017/freelance-platform
```

Dùng `127.0.0.1` thay vì `localhost` — tránh lỗi `ECONNREFUSED ::1:27017` trên Windows.

**MongoDB Atlas (free tier):** tạo cluster tại [mongodb.com/atlas](https://www.mongodb.com/atlas), lấy connection string và đặt vào `MONGODB_URI` (xem `backend/.env.example`).

Nếu MongoDB chưa chạy, server vẫn khởi động và ghi cảnh báo; background services (indexer, listener, cron) được bỏ qua — không còn lỗi `buffering timed out`.

**Tắt event indexer khi test Postman** (tránh lỗi Infura `Too Many Requests` / `-32005`):

```env
ENABLE_EVENT_INDEXER=false
```

Khởi động lại backend sau khi sửa `.env`.

## Khởi động server

```bash
cd backend
npm install
cp .env.example .env   # điền MONGODB_URI, JWT_SECRET, Pinata, RPC_URL
npm run dev            # hoặc: npm start
```

Server mặc định chạy cổng **5000** (`PORT` trong `.env`).

**Windows:** dùng `http://127.0.0.1:5000` thay vì `localhost` — tránh treo do IPv6.

Kiểm tra nhanh không cần MongoDB:

```bash
npm run test:api
```

Với MongoDB đang chạy:

```bash
npm run test:api -- --with-nonce
```

## Import Postman

1. Mở Postman → **Import**
2. Chọn hai file trong repo backend:
   - `backend/postman/Fapex.postman_collection.json`
   - `backend/postman/Fapex-Local.postman_environment.json`
3. Chọn environment **Fapex — Local**
4. Sửa biến `walletAddress` thành địa chỉ ví Sepolia của bạn
5. Giữ `baseUrl` = `http://127.0.0.1:5000` (hoặc cổng bạn cấu hình)

## Luồng test từng bước

### Bước 1 — Health (không cần DB)

- Request: **01 — Health → GET /health**
- Kỳ vọng: `status: "ok"`, trường `mongodb`:
  - `"connected"` → có thể test nonce/jobs
  - `"disconnected"` → cần khởi động MongoDB hoặc cập nhật `MONGODB_URI`

### Bước 2 — Lấy nonce (cần MongoDB)

- Request: **02 — Auth → POST /api/auth/nonce**
- Body: `{ "walletAddress": "0x..." }`
- Lưu `nonce`, `domain`, `chainId` từ response (Postman tự set biến môi trường)

### Bước 3 — Xác thực SIWE (cần ví — thủ công)

Postman **không** ký SIWE thay ví. Làm một trong hai cách:

**Cách A — Console trình duyệt (MetaMask):**

```javascript
import { SiweMessage } from 'https://esm.sh/siwe@3';

const walletAddress = '0xYourAddress';
const nonce = '...'; // từ bước 2
const domain = 'localhost';
const chainId = 11155111;
const uri = 'http://localhost:3000'; // khớp APP_URL trong backend/.env

const message = new SiweMessage({
  domain,
  address: walletAddress,
  statement: 'Sign in to Fapex',
  uri,
  version: '1',
  chainId,
  nonce,
});
const prepared = message.prepareMessage();
const signature = await ethereum.request({
  method: 'personal_sign',
  params: [prepared, walletAddress],
});
console.log({ message: prepared, signature });
```

Dán `message` vào biến Postman `siweMessage`, `signature` vào `siweSignature`.

**Cách B — Frontend dev:** dùng flow trong [auth-api.md](./auth-api.md), copy token vào biến `authToken`.

- Request: **POST /api/auth/verify**
- Postman lưu `authToken` tự động khi thành công

### Bước 4 — Kiểm tra session

- Request: **GET /api/auth/me** (header `Authorization: Bearer {{authToken}}`)

### Bước 5 — Upload metadata IPFS

- Request: **03 — IPFS → POST /api/ipfs/upload/metadata**
- Cần Bearer token + Pinata đã cấu hình
- Response: `metadataCID` — dùng cho on-chain hoặc tham chiếu job

### Bước 6 — Tạo job

- Request: **04 — Jobs → POST /api/jobs**
- Cần: auth, MongoDB, Pinata, RPC Sepolia
- Backend tự upload metadata lên IPFS và gọi contract `createJob`
- User phải đã đăng nhập (có bản ghi trong MongoDB) và không ở tier `Restricted`

Ví dụ body tối thiểu:

```json
{
  "title": "Smart Contract Audit",
  "description": "Audit Solidity escrow contracts on Sepolia testnet for security.",
  "category": "development",
  "contractValue": 100,
  "duration": 604800,
  "skills": ["Solidity", "Security"],
  "deliverables": "PDF audit report.",
  "acceptanceCriteria": "All critical issues documented."
}
```

### Bước 7 — Danh sách jobs

- Request: **GET /api/jobs** (public, cần MongoDB)

### Bước 8 — Trạng thái trọng tài

- Request: **GET /api/arbitrator/:address/status**
- Chỉ cần RPC — không cần MongoDB
- Kiểm tra `stakedAmount` và `isValid` (tối thiểu 50 USDC)

## Endpoint trong collection

| Method | Path | Auth | MongoDB |
|--------|------|------|---------|
| GET | `/health` | Không | Không |
| POST | `/api/auth/nonce` | Không | Có |
| POST | `/api/auth/verify` | Không | Có |
| GET | `/api/auth/me` | Bearer | Có |
| POST | `/api/ipfs/upload/metadata` | Bearer | Không* |
| GET | `/api/jobs` | Không | Có |
| POST | `/api/jobs` | Bearer | Có |
| GET | `/api/arbitrator/:address/status` | Không | Không |

\* IPFS chỉ cần Pinata; user lookup cho Bearer vẫn cần MongoDB.

## Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|-----|-------------|------------|
| `buffering timed out` | MongoDB chưa chạy | Start MongoDB hoặc dùng Atlas URI |
| `JWT_SECRET is not defined` | Thiếu biến env | Thêm vào `backend/.env` |
| `SIWE verification failed` | domain/chainId/nonce sai | Khớp `SIWE_DOMAIN`, `CHAIN_ID`, gọi nonce mới |
| `Không thể upload metadata` | Pinata chưa cấu hình | Thêm `PINATA_JWT` trong `.env` |
| Request treo với `localhost` | IPv6 trên Windows | Dùng `127.0.0.1` |
| `Too Many Requests` / `-32005` | Infura rate limit từ event indexer | `ENABLE_EVENT_INDEXER=false` trong `.env` |
| `Index JobStatusUpdated error` | Indexer gọi `eth_getLogs` quá nhiều | Tắt indexer hoặc dùng RPC provider khác |

## Script kiểm thử trong repo

```bash
cd backend
npm run test:auth    # load module JWT/SIWE, không cần server
npm run test:api     # GET /health (server phải đang chạy)
```

# Hướng dẫn sử dụng & phát triển — FAPEX

> **English summary:** User guide for Client/Freelancer/Arbitrator roles on Sepolia, plus developer setup for local monorepo.

**Cập nhật:** 2026-06-28

---

## Phần A — Hướng dẫn người dùng

### A.1 Yêu cầu

| Yêu cầu | Chi tiết |
|---------|----------|
| Ví | MetaMask (hoặc RainbowKit-supported) |
| Mạng | **Sepolia** (chainId `11155111`) |
| Gas | Sepolia ETH (faucet) |
| Token | **MockUSDC** — mint miễn phí trên UI (không phải USDC thật) |

**Production frontend:** Vercel URL của team · **API:** https://fapex-backend-production.up.railway.app

### A.2 Kết nối ví & đăng nhập

1. Mở app → **Connect Wallet**
2. Nếu banner **Wrong network** → **Switch to Sepolia**
3. **Sign in (SIWE)** — ký message EIP-4361 (không password)
4. `/profile` — đặt username, chọn role **Client** hoặc **Freelancer**

**MetaMask domain warning:** SIWE message `domain` phải trùng hostname frontend (ví dụ `frontend-smoky-eight-51.vercel.app`), **không** dùng domain Railway API. Frontend ký với `window.location.host`; Railway backend set `SIWE_DOMAIN` và `APP_URL` trùng Vercel (xem B.5).

### A.3 Client

| Bước | Hành động | Route |
|------|-----------|-------|
| 1 | Tạo job (mô tả + budget USDC) | `/client` |
| 2 | Chờ bid → Accept bid | Job detail |
| 3 | Mint MockUSDC nếu thiếu | Job detail |
| 4 | Approve + **Deposit escrow** | Job detail |
| 5 | Phê duyệt hoặc **Raise dispute** | Job detail |

**Lưu ý:** Ví MetaMask khi `depositEscrow` phải trùng **on-chain client** (cùng ví đã ký `createJob`).

### A.4 Freelancer

| Bước | Hành động | Route |
|------|-----------|-------|
| 1 | Duyệt job | `/browse` |
| 2 | Submit bid | `/jobs/:id` |
| 3 | Sau khi client deposit | Job detail |
| 4 | **Start work** → upload file | Job detail |
| 5 | **Submit deliverable** | Job detail |

### A.5 Arbitrator

| Bước | Hành động |
|------|-----------|
| 1 | Mint ≥50 MockUSDC |
| 2 | **Stake** trên PlatformTreasury |
| 3 | Admin/script **joinPool** (hoặc self-join nếu đủ điều kiện) |
| 4 | Khi được sortition chọn → `/arbitrator` hoặc job detail |
| 5 | **Commit** → **Reveal** vote trong cửa sổ thời gian |

Pool cần **≥5** member để `raiseDispute` hoạt động.

### A.6 Platform admin (governance dashboard)

| Bước | Hành động |
|------|-----------|
| 1 | Connect ví **deployer** Sepolia (`0x523e…92f7`) hoặc ví đã được `grantRole` |
| 2 | Mở **`/admin`** (link **Admin** trên nav khi có quyền, hoặc **Governance** ở footer landing) |
| 3 | Xem contract addresses, pool size, indexer stats |
| 4 | **Pause/unpause** escrow (admin hoặc ROLE_PAUSER) |
| 5 | **Grant/revoke** delegated roles (chỉ contract admin) |
| 6 | **joinPool** hộ arbitrator (admin hoặc ROLE_ARBITRATOR_MANAGER) |
| 7 | **adminForceResolve** khi quorum fail (admin hoặc ROLE_FORCE_RESOLVER) |

Quyền admin **on-chain** — MongoDB enum `admin` không dùng. CLI thay thế: `scripts/grant-platform-roles.js`.

### A.7 Theme & ngôn ngữ

- UI **tiếng Anh**
- Light / dark theme (toggle header)

---

## Phần B — Hướng dẫn developer

### B.1 Clone monorepo

```bash
git clone --recurse-submodules https://github.com/thanhltkk24414-lang/Blockchain.git
cd Blockchain
git checkout dev
git submodule update --init --recursive
```

### B.2 Contracts (root)

```bash
npm install
cp contracts/.env.example contracts/.env   # PRIVATE_KEY, SEPOLIA_RPC_URL
npm run compile
npm run deploy:sepolia      # demo timings
npm run seed:arbitrators
```

Địa chỉ: `deployments/sepolia.json`

### B.3 Backend

```bash
cd backend
npm install
cp .env.example .env
# Điền: MONGODB_URI, RPC_URL, JWT_SECRET, contract addresses, PINATA_JWT
npm run dev                 # http://127.0.0.1:5000
```

| Endpoint | Mô tả |
|----------|--------|
| `GET /health` | MongoDB + contract env |
| `GET /api/config` | Addresses cho frontend |
| `GET /api/admin/stats` | Job counts, indexer lastBlock (read-only, demo) |
| `POST /api/auth/nonce` | SIWE bước 1 |
| `POST /api/auth/verify` | SIWE → JWT |

Local MongoDB: `npm run docker:mongo` (trong `backend/`)

### B.4 Frontend

```bash
cd frontend
npm install
cp .env.example .env
# VITE_API_URL=http://127.0.0.1:5000
npm run dev                 # http://localhost:3000
```

### B.5 Biến môi trường quan trọng

**Backend (`backend/.env`):**

```
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
MONGODB_URI=mongodb://127.0.0.1:27017/freelance-platform
RPC_URL=https://sepolia.infura.io/v3/...
JOB_REGISTRY_ADDRESS=0x302629f82d51b0972ffc3A99cbE355F4acEf908d
LEGACY_JOB_REGISTRY_ADDRESS=0xE5425cFE21BAe73d54138Bb290B671bF4c55FBC9
SIWE_DOMAIN=localhost
APP_URL=http://localhost:3000
```

**Frontend (`frontend/.env`):** sync `VITE_*_ADDRESS` với `deployments/sepolia.json`

**Railway production:**
```
ALLOWED_ORIGINS=http://localhost:3000,https://*.vercel.app
SIWE_DOMAIN=frontend-smoky-eight-51.vercel.app
APP_URL=https://frontend-smoky-eight-51.vercel.app
```

`SIWE_DOMAIN` = hostname frontend (không `https://`). MetaMask so sánh với URL trình duyệt — nếu backend trả `railway.app` mà user mở `vercel.app` sẽ cảnh báo domain mismatch.

### B.6 Sau redeploy JobRegistry

```bash
cd backend
node scripts/migrate-job-registry-index.js
```

Cập nhật `JOB_REGISTRY_ADDRESS` trên Railway + Vercel env.

### B.7 Kiểm thử API

- [postman-walkthrough-vi.md](postman-walkthrough-vi.md)
- [auth-api.md](auth-api.md)

---

## Phần C — Troubleshooting

| Triệu chứng | Cách xử lý |
|-------------|------------|
| Wrong network | Switch Sepolia trong MetaMask |
| Wallet mismatch banner | Đổi MetaMask sang đúng ví on-chain |
| JobRegistry mismatch | Đồng bộ env với `deployments/sepolia.json` |
| CORS error trên Vercel | Railway `ALLOWED_ORIGINS` có `https://*.vercel.app` |
| raiseDispute revert | `npm run check:dispute` — pool ≥5, đủ USDC fee |
| Indexer chậm | Đợi ~2 phút hoặc refresh; UI đọc chain trực tiếp |

---

## Tài liệu liên quan

- [workflow-e2e-vi.md](workflow-e2e-vi.md)
- [demo-script-vi.md](demo-script-vi.md)
- [demo-qa-defense-vi.md](demo-qa-defense-vi.md)
- [deploy-backend.md](deploy-backend.md)

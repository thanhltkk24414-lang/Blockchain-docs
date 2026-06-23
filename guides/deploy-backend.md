# Hướng dẫn deploy Backend Fapex lên Railway hoặc Render

Tài liệu **từng bước** để deploy API **blockchain-backend** (Fapex) lên [Railway](https://railway.app) **hoặc** [Render](https://render.com). Bạn **chọn một** nền tảng — không cần deploy cả hai.

> **Lưu ý bảo mật:** Không commit file `.env`, không dán secret vào GitHub Issues/PR. Mọi giá trị nhạy cảm chỉ nhập trong dashboard Railway/Render.

**Repo deploy:** [thanhltkk24414-lang/blockchain-backend](https://github.com/thanhltkk24414-lang/blockchain-backend)  
**Entry point:** `src/server.js` (`npm start`)  
**Health check:** `GET /health` (trả 200 ngay cả khi MongoDB chưa kết nối)

---

## 0. Kiểm tra đồng bộ repo (sau khi merge dev → main)

Sau khi merge PR monorepo `dev → main`, submodule `backend` trên `main` phải trỏ cùng commit với `blockchain-backend` nhánh `main`.

```powershell
cd d:\projects\Blockchain
git fetch --all
git log -1 --oneline origin/main          # monorepo root
git ls-tree origin/main backend           # commit hash submodule backend

cd backend
git fetch --all
git log -1 --oneline origin/main          # blockchain-backend repo
```

**Kết quả mong đợi (tháng 06/2026):**

| Repo | Nhánh | Commit gần nhất |
|------|-------|-----------------|
| Monorepo root | `main` | `fc960e7` — Merge PR #7 dev→main |
| Submodule `backend` | `main` | `1b159a7` — Merge PR #6 dev→main |
| `git ls-tree origin/main backend` | — | `1b159a7f46c49f814dd0fc9ea771d2f5ca0c6fe4` |

Hai hash **phải khớp**. Nếu lệch: `git submodule update --init --remote backend` rồi commit pointer submodule trên monorepo.

**Deploy từ repo nào?** Railway/Render kết nối repo **`blockchain-backend`** (không phải monorepo root). Chọn nhánh **`main`** sau khi PR deploy đã merge.

---

## 1. Điều kiện tiên quyết

Chuẩn bị **trước** khi mở dashboard deploy:

| Dịch vụ | Mục đích | Free tier |
|---------|----------|-----------|
| **MongoDB Atlas** | User, job, auth, trạng thái event indexer | Cluster M0 |
| **Infura** hoặc **Alchemy** | `RPC_URL` — đọc/ghi Sepolia | Có |
| **Pinata** | Upload metadata job lên IPFS | Có |
| **GitHub** | Kết nối repo với Railway/Render | — |
| **MetaMask + Sepolia** | Test SIWE sau deploy | Faucet Sepolia |

Tạo sẵn:

- `JWT_SECRET` — chuỗi ngẫu nhiên ≥ 32 ký tự (xem mục 3.2)
- 6 địa chỉ contract từ `deployments/sepolia.json` (monorepo root)

### Địa chỉ contract Sepolia hiện tại

Lấy từ `deployments/sepolia.json` — **cập nhật lại** nếu team deploy contract mới:

| Key JSON | Biến env | Địa chỉ |
|----------|----------|---------|
| `addresses.MockUSDC` | `MOCK_USDC_ADDRESS` | `0x2293193Eaa5CE5253d5e081046a06dB077f26f8e` |
| `addresses.ReputationStore` | `REPUTATION_STORE_ADDRESS` | `0x7A96219812e9363dBdbD43BE14384820E5f9b0DC` |
| `addresses.PlatformTreasury` | `PLATFORM_TREASURY_ADDRESS` | `0x0110BfF85E484b82205833D3950fC7C61714c0e7` |
| `addresses.JobRegistry` | `JOB_REGISTRY_ADDRESS` | `0xeF5cc7a22D7Ff9e7FA0c5Fe714F088c98758A549` |
| `addresses.ArbitratorPanel` | `ARBITRATOR_PANEL_ADDRESS` | `0x324e7d8Cfe5aBdb62caa236Bb23626E23BC7EC4F` |
| `addresses.EscrowVault` | `ESCROW_VAULT_ADDRESS` | `0xf2143d1EA4D5a8716344c2cef862f9ed41244ED5` |

`CHAIN_ID` = `11155111` (Sepolia).

---

## 2. MongoDB Atlas — hướng dẫn chi tiết (screenshot-level)

Backend **bắt buộc** MongoDB cho auth (`/api/auth/nonce`, `/api/auth/verify`), jobs, và event indexer.

### Bước 2.1 — Tạo tài khoản và project

1. Mở [https://cloud.mongodb.com](https://cloud.mongodb.com) → **Sign Up** hoặc **Sign In**.
2. Trang **Projects** → nút **Create** (hoặc **New Project**).
3. Đặt tên project, ví dụ `fapex-production` → **Next** → **Create Project**.

### Bước 2.2 — Tạo cluster M0 (miễn phí)

1. Trong project → **Build a Database** (hoặc **+ Create**).
2. Chọn **M0 FREE** (Shared).
3. **Provider / Region:** chọn region gần user (ví dụ `AWS / Singapore (ap-southeast-1)` hoặc `Frankfurt`).
4. **Cluster Name:** giữ mặc định `Cluster0` hoặc đổi `fapex-cluster`.
5. Bỏ tick **Preload sample dataset** (không cần).
6. **Create Deployment** — đợi 1–3 phút (trạng thái chuyển sang **Active**).

### Bước 2.3 — Tạo database user

1. Popup **Security Quickstart** (hoặc menu trái **Database Access**).
2. **Add New Database User**:
   - **Authentication:** Password
   - **Username:** ví dụ `fapex_api`
   - **Password:** bấm **Autogenerate Secure Password** → **Copy** và lưu vào password manager (không commit).
3. **Database User Privileges:** `Read and write to any database` (hoặc `Atlas admin` cho dev).
4. **Add User**.

> Mật khẩu có ký tự đặc biệt (`@`, `#`, `%`, …) phải **URL-encode** khi dán vào connection string (ví dụ `@` → `%40`).

### Bước 2.4 — Network Access (quan trọng cho Railway/Render)

Cloud host **không có IP cố định** trên free tier → cần cho phép mọi IP (chấp nhận rủi ro dev/staging) hoặc dùng VPC peering trên plan trả phí.

1. Menu trái **Network Access** → **Add IP Address**.
2. Chọn **Allow Access from Anywhere** → IP hiển thị `0.0.0.0/0`.
3. **Confirm** → đợi vài giây trạng thái **Active**.

> Production nghiêm ngặt: thu hẹp IP khi Railway/Render cung cấp egress IP tĩnh (thường chỉ trên plan trả phí).

### Bước 2.5 — Lấy connection string

1. Menu trái **Database** → cluster → nút **Connect**.
2. Chọn **Drivers** (hoặc **Connect your application**).
3. **Driver:** Node.js, **Version:** 5.5 or later (hoặc 4.x — đều được).
4. Copy chuỗi dạng:
   ```
   mongodb+srv://fapex_api:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority
   ```
5. Thêm tên database vào path (trước `?`):
   ```
   mongodb+srv://fapex_api:<password>@cluster0.xxxxx.mongodb.net/freelance-platform?retryWrites=true&w=majority
   ```
6. Thay `<password>` bằng mật khẩu thật (đã URL-encode nếu cần).

Đây là giá trị **`MONGODB_URI`** — dán vào Railway/Render Variables (mục 3).

### Bước 2.6 — Kiểm tra Atlas (tùy chọn)

Trong **Database** → **Browse Collections** — sau lần deploy đầu và gọi `POST /api/auth/nonce`, sẽ thấy collection `users` xuất hiện.

---

## 3. Bảng đầy đủ biến môi trường (production)

Nhập trong tab **Variables** / **Environment** của Railway hoặc Render. Tham chiếu: `backend/.env.example`, `backend/ENV_SETUP.md`.

### 3.1 — Bắt buộc

| Biến | Giá trị / ví dụ | Lấy từ đâu | Ghi chú |
|------|-----------------|------------|---------|
| `NODE_ENV` | `production` | Tự đặt | Bật chế độ production |
| `PORT` | *(để platform inject)* | Railway/Render tự set | App đọc `process.env.PORT \|\| 5000`. **Không** hardcode `5000` trên Render nếu platform gán port khác — app đã xử lý đúng |
| `MONGODB_URI` | `mongodb+srv://user:pass@cluster.../freelance-platform?retryWrites=true&w=majority` | MongoDB Atlas (mục 2) | Bắt buộc cho auth/jobs |
| `JWT_SECRET` | Chuỗi hex 64 ký tự | Tự generate (mục 3.2) | ≥ 32 ký tự ngẫu nhiên |
| `RPC_URL` | `https://sepolia.infura.io/v3/YOUR_PROJECT_ID` | Infura / Alchemy dashboard | Cùng URL với `SEPOLIA_RPC_URL` trong `contracts/.env` |
| `CHAIN_ID` | `11155111` | `deployments/sepolia.json` | Sepolia |
| `MOCK_USDC_ADDRESS` | `0x2293193Eaa5CE5253d5e081046a06dB077f26f8e` | `deployments/sepolia.json` | EIP-55 checksum |
| `REPUTATION_STORE_ADDRESS` | `0x7A96219812e9363dBdbD43BE14384820E5f9b0DC` | deployments | |
| `PLATFORM_TREASURY_ADDRESS` | `0x0110BfF85E484b82205833D3950fC7C61714c0e7` | deployments | |
| `JOB_REGISTRY_ADDRESS` | `0xeF5cc7a22D7Ff9e7FA0c5Fe714F088c98758A549` | deployments | |
| `ARBITRATOR_PANEL_ADDRESS` | `0x324e7d8Cfe5aBdb62caa236Bb23626E23BC7EC4F` | deployments | |
| `ESCROW_VAULT_ADDRESS` | `0xf2143d1EA4D5a8716344c2cef862f9ed41244ED5` | deployments | |
| `PINATA_JWT` | `eyJ...` | [Pinata → API Keys](https://app.pinata.cloud/developers/api-keys) | **Hoặc** dùng cặp key bên dưới |
| `SIWE_DOMAIN` | `your-app.up.railway.app` | **Sau deploy** — hostname public | **Không** có `https://` |
| `APP_URL` | `https://your-app.up.railway.app` | **Sau deploy** — URL HTTPS đầy đủ | Dùng làm SIWE `uri` |
| `ALLOWED_ORIGINS` | `https://your-frontend.vercel.app` | URL frontend production | Phân tách bằng dấu phẩy, không slash cuối |

### 3.2 — Tạo `JWT_SECRET` (PowerShell)

```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Copy output → dán vào `JWT_SECRET`. **Không** dùng `your_super_secret_key_change_me`.

### 3.3 — Infura `RPC_URL`

1. [https://infura.io](https://infura.io) → đăng ký → **Create API Key**.
2. Chọn network **Ethereum** → bật **Sepolia**.
3. Copy endpoint:
   ```
   https://sepolia.infura.io/v3/<PROJECT_ID>
   ```
4. Dán vào `RPC_URL`.

**Alchemy thay thế:** `https://eth-sepolia.g.alchemy.com/v2/<API_KEY>`.

### 3.4 — Pinata (IPFS)

**Cách A — JWT (khuyến nghị):**

```env
IPFS_GATEWAY_URL=https://gateway.pinata.cloud
PINATA_JWT=<jwt_từ_pinata>
```

**Cách B — API key pair:**

```env
IPFS_GATEWAY_URL=https://gateway.pinata.cloud
PINATA_API_KEY=<key>
PINATA_SECRET_API_KEY=<secret>
```

Tạo key: [app.pinata.cloud](https://app.pinata.cloud) → **Developers → API Keys → New Key** → bật `pinJSONToIPFS`, `pinFileToIPFS`.

### 3.5 — SIWE (cập nhật **sau** khi có URL public)

| Biến | Đúng | Sai |
|------|------|-----|
| `SIWE_DOMAIN` | `fapex-backend-production-xxxx.up.railway.app` | `https://fapex-backend...` |
| `APP_URL` | `https://fapex-backend-production-xxxx.up.railway.app` | Thiếu `https://` |
| `CHAIN_ID` | `11155111` | `sepolia` (sai kiểu) |

**Lần deploy đầu:** có thể tạm để `SIWE_DOMAIN=localhost` và `APP_URL=http://localhost:3000` để build chạy được → sau khi có domain → **sửa 3 biến** (`SIWE_DOMAIN`, `APP_URL`, `ALLOWED_ORIGINS`) → **redeploy**.

### 3.6 — Tùy chọn (khuyến nghị cho free tier)

| Biến | Mặc định | Khi nào dùng |
|------|----------|--------------|
| `JWT_EXPIRES_IN` | `7d` | Đổi thời hạn JWT |
| `ENABLE_EVENT_INDEXER` | `true` | Đặt `false` nếu Infura báo rate limit (mục 8) |
| `INDEXER_BATCH_SIZE` | `100` | Tune indexer |
| `INDEXER_POLL_CRON` | `0 */2 * * * *` | Cron poll mỗi 2 phút |
| `INDEXER_RPC_DELAY_MS` | `500` | Delay giữa RPC calls |
| `INDEXER_PRIVATE_KEY` | *(trống)* | Ví Sepolia để indexer/cron gửi tx on-chain |
| `SEPOLIA_WSS_URL` | *(trống)* | WebSocket RPC cho realtime listener |
| `SEPOLIA_RPC_URL` | — | Alias của `RPC_URL` (backend chấp nhận cả hai) |
| `IPFS_API_KEY` / `IPFS_API_SECRET` | — | Alias legacy của Pinata key pair |

---

## 4. Luồng deploy tổng quan

```mermaid
flowchart TD
  A[Chuẩn bị Atlas + Infura + Pinata] --> B{Chọn platform}
  B -->|Railway| C[Deploy từ GitHub main]
  B -->|Render| D[Blueprint hoặc Web Service Docker]
  C --> E[Thêm env vars lần 1]
  D --> E
  E --> F[Deploy lần 1]
  F --> G[Lấy URL public HTTPS]
  G --> H[Cập nhật SIWE_DOMAIN APP_URL ALLOWED_ORIGINS]
  H --> I[Redeploy]
  I --> J[Verify: /health, siwe-sign, Postman]
```

---

## 5. Đường A — Deploy lên Railway

Railway đọc `railway.toml` + `Dockerfile` tại **root repo blockchain-backend** (không phải monorepo).

### 5.1 — Tạo project

1. Đăng nhập [railway.app](https://railway.app).
2. **New Project** → **Deploy from GitHub repo**.
3. Cấp quyền GitHub nếu được hỏi.
4. Chọn repo **`thanhltkk24414-lang/blockchain-backend`**.
5. **Branch:** `main` (hoặc `dev` để staging).

### 5.2 — Cấu hình service

Railway tự detect từ `railway.toml`:

- **Builder:** Dockerfile
- **Health check path:** `/health` (timeout 120s)
- **Root directory:** `/` (root của repo backend — **không** set `backend/` vì đây là repo độc lập)

Kiểm tra **Settings → Deploy**:

- Dockerfile path: `Dockerfile`
- Start command: `npm start` (từ `CMD` trong Dockerfile)

### 5.3 — Biến môi trường (lần 1)

Vào service → tab **Variables** → **Raw Editor** hoặc thêm từng biến.

**Lần 1 — điền tất cả trừ SIWE/CORS chính xác:**

- `NODE_ENV=production`
- `MONGODB_URI`, `JWT_SECRET`, `RPC_URL`, `CHAIN_ID`, 6 contract addresses, `PINATA_JWT`, `IPFS_GATEWAY_URL`
- Tạm: `SIWE_DOMAIN=localhost`, `APP_URL=http://localhost:3000`, `ALLOWED_ORIGINS=http://localhost:3000`
- Khuyến nghị free tier: `ENABLE_EVENT_INDEXER=false`

**Không** set `PORT` thủ công — Railway inject tự động.

### 5.4 — Deploy lần 1

1. **Deploy** (hoặc push lên `main` trigger auto-deploy).
2. Mở tab **Deployments** → xem log build:
   - `npm ci --omit=dev` trong Docker
   - Container start → `Server running on port ...`
3. Nếu **Failed**: xem mục 8 (Troubleshooting).

### 5.5 — Tạo domain public

1. **Settings** → **Networking** → **Generate Domain**.
2. Copy URL, ví dụ:
   ```
   https://blockchain-backend-production-a1b2.up.railway.app
   ```

### 5.6 — Cập nhật SIWE + CORS + Redeploy

Sửa Variables:

```env
SIWE_DOMAIN=blockchain-backend-production-a1b2.up.railway.app
APP_URL=https://blockchain-backend-production-a1b2.up.railway.app
ALLOWED_ORIGINS=https://your-frontend.vercel.app
```

- Nhiều frontend: `https://app1.com,https://app2.com`
- Chưa có frontend: có thể tạm `ALLOWED_ORIGINS=https://blockchain-backend-production-a1b2.up.railway.app` để test `siwe-sign.html` cùng origin

Railway thường **tự restart** khi đổi env. Nếu không: **Deployments → Redeploy**.

---

## 6. Đường B — Deploy lên Render

### 6.1a — Blueprint (khuyến nghị — có `render.yaml`)

1. [dashboard.render.com](https://dashboard.render.com) → **New** → **Blueprint**.
2. Connect GitHub → chọn **`thanhltkk24414-lang/blockchain-backend`**.
3. Render đọc `render.yaml` → hiện service `fapex-backend`.
4. Các biến `sync: false` sẽ được prompt nhập secret khi apply blueprint.
5. Điền: `MONGODB_URI`, `JWT_SECRET`, `RPC_URL`, Pinata, 6 addresses.
6. Tạm SIWE/CORS như Railway → **Apply**.

`render.yaml` đã set sẵn: `NODE_ENV=production`, `CHAIN_ID=11155111`, Docker, health `/health`, plan `free`.

### 6.1b — Web Service thủ công (không dùng Blueprint)

1. **New** → **Web Service** → connect repo `blockchain-backend`.
2. **Branch:** `main`.
3. **Runtime:** Docker.
4. **Dockerfile path:** `./Dockerfile`.
5. **Health Check Path:** `/health`.
6. **Instance type:** Free (nếu đủ).
7. Thêm env vars giống bảng mục 3.

### 6.2 — URL public

Sau deploy, URL dạng:

```
https://fapex-backend.onrender.com
```

(hoặc tên service bạn đặt)

### 6.3 — Cập nhật SIWE + CORS + Redeploy

```env
SIWE_DOMAIN=fapex-backend.onrender.com
APP_URL=https://fapex-backend.onrender.com
ALLOWED_ORIGINS=https://your-frontend.vercel.app
```

**Manual Deploy** → **Deploy latest commit**.

> **Free tier Render:** service **sleep** sau ~15 phút không traffic. Request đầu có thể mất **30–60 giây** (cold start). Health check có thể fail lần đầu — đợi và thử lại.

---

## 7. Kiểm tra sau deploy

Thay `YOUR_URL` bằng URL HTTPS thật (không slash cuối).

### 7.1 — Health check

**PowerShell:**

```powershell
curl.exe https://YOUR_URL/health
```

**Kết quả mong đợi (200):**

```json
{
  "status": "ok",
  "timestamp": "...",
  "uptime": 12.34,
  "environment": "production",
  "mongodb": "connected"
}
```

| `mongodb` | Ý nghĩa |
|-----------|---------|
| `"connected"` | Atlas OK — tiếp tục test auth |
| `"disconnected"` | Sai `MONGODB_URI` hoặc Network Access — xem mục 8 |

### 7.2 — Trang ký SIWE

Mở trình duyệt (Chrome + MetaMask, mạng **Sepolia**):

```
https://YOUR_URL/siwe-sign.html
```

1. Xác nhận banner **SIWE Sign Page v4**.
2. **Kết nối MetaMask**.
3. **Lấy nonce từ API** (cần `mongodb: connected`).
4. Kiểm tra `domain` = `SIWE_DOMAIN`, `uri` = `APP_URL`.
5. **Ký với MetaMask** → **Copy JSON cho Postman**.

### 7.3 — Postman / REST Client — nonce + verify

Chi tiết đầy đủ: [postman-walkthrough-vi.md](./postman-walkthrough-vi.md).

**Bước 1 — Nonce**

```http
POST https://YOUR_URL/api/auth/nonce
Content-Type: application/json

{
  "walletAddress": "0x523eBd853a1638065f148A05c0Ca423E490D92f7"
}
```

Response 200:

```json
{
  "success": true,
  "nonce": "a1b2c3d4...",
  "walletAddress": "0x523eBd853a1638065f148A05c0Ca423E490D92f7",
  "domain": "your-host.up.railway.app",
  "appUrl": "https://your-host.up.railway.app",
  "chainId": 11155111
}
```

**Bước 2 — Ký** trên `https://YOUR_URL/siwe-sign.html` (không sửa message sau khi ký).

**Bước 3 — Verify**

```http
POST https://YOUR_URL/api/auth/verify
Content-Type: application/json

{
  "message": "<chuỗi SIWE đầy đủ nhiều dòng>",
  "signature": "0x..."
}
```

Response 200 → copy `token` → header `Authorization: Bearer <token>` cho các API khác.

**Bước 4 — Me (tùy chọn)**

```http
GET https://YOUR_URL/api/auth/me
Authorization: Bearer <token>
```

### 7.4 — Checklist hoàn tất

- [ ] `GET /health` → 200, `mongodb: connected`
- [ ] `SIWE_DOMAIN` / `APP_URL` khớp URL public
- [ ] `POST /api/auth/nonce` → 200
- [ ] `POST /api/auth/verify` → 200 + JWT
- [ ] `/siwe-sign.html` load và ký được
- [ ] `ALLOWED_ORIGINS` có URL frontend (khi có frontend)
- [ ] `POST /api/ipfs/upload/metadata` (cần JWT + Pinata) — nếu test full flow
- [ ] Log không spam RPC rate limit (hoặc đã set `ENABLE_EVENT_INDEXER=false`)

---

## 8. Lỗi thường gặp và cách sửa

### 8.1 — `mongodb: disconnected` / MongoDB connection refused

| Nguyên nhân | Cách sửa |
|-------------|----------|
| Chưa thêm IP `0.0.0.0/0` trong Atlas **Network Access** | Add IP → Allow from anywhere → đợi Active |
| Sai username/password trong URI | Tạo lại user hoặc reset password |
| Password có `@`, `#`, `%` chưa encode | URL-encode (ví dụ `@` → `%40`) |
| Thiếu tên DB trong path | Thêm `/freelance-platform` trước `?` |
| Cluster chưa Active | Đợi Atlas provisioning xong |

Log deploy thường có: `MongoDB unavailable (...). Server will run without DB`.

### 8.2 — SIWE verify failed / EIP-55

| Triệu chứng | Cách sửa |
|-------------|----------|
| `Invalid address` / EIP-55 | Dùng `siwe-sign.html` v4 — địa chỉ phải checksum, ví dụ `0x523eBd853a1638065f148A05c0Ca423E490D92f7` |
| `Domain does not match` | `SIWE_DOMAIN` = hostname **không** có `https://` |
| `Nonce does not match` | Gọi lại `POST /api/auth/nonce`, ký ngay, không dùng nonce cũ |
| `Signature does not match` | Không sửa message sau khi ký; copy JSON từ trang helper |
| `Invalid chain ID` | `CHAIN_ID=11155111`, MetaMask chọn Sepolia |

### 8.3 — CORS error từ frontend

```
Access to fetch ... has been blocked by CORS policy
```

- Thêm **đúng** origin frontend vào `ALLOWED_ORIGINS`: `https://myapp.vercel.app`
- **Không** trailing slash: dùng `https://app.com` không phải `https://app.com/`
- Nhiều origin: phân tách bằng dấu phẩy, không space thừa
- Redeploy sau khi đổi env

### 8.4 — Infura / RPC rate limit

Log: `too many requests`, code `-32005`, hoặc indexer retry liên tục.

**Giải pháp nhanh (free tier):**

```env
ENABLE_EVENT_INDEXER=false
```

Redeploy. Auth và jobs vẫn chạy; chỉ tắt đồng bộ event on-chain → MongoDB.

Tùy chọn thêm: tăng `INDEXER_RPC_DELAY_MS=2000`, nâng cấp Infura, hoặc dùng Alchemy.

### 8.5 — Build Docker fail `npm ci`

- Đảm bảo `package-lock.json` có trong repo
- Không copy `node_modules` vào image (`.dockerignore` đã loại)

### 8.6 — Health check fail / timeout

| Platform | Ghi chú |
|----------|---------|
| Render free | Cold start 30–60s — tăng patience hoặc ping định kỳ |
| Railway | `healthcheckTimeout = 120` trong `railway.toml` |
| MongoDB chậm | Server vẫn listen; `/health` trả 200 ngay cả khi DB disconnected |

### 8.7 — Pinata / IPFS upload lỗi

- Thiếu `PINATA_JWT` hoặc key pair
- JWT hết hạn — tạo key mới trên Pinata
- Kiểm tra quyền `pinJSONToIPFS`

### 8.8 — `RPC_URL or SEPOLIA_RPC_URL is not defined`

Thêm `RPC_URL` trong Variables → redeploy.

---

## 9. Docker smoke test local (tùy chọn)

Trước khi deploy cloud, test image giống production:

```powershell
cd d:\projects\Blockchain\backend
docker build -t fapex-backend .
docker run --rm -p 5000:5000 --env-file .env fapex-backend
```

Mở `http://127.0.0.1:5000/health`. File `.env` local cần `MONGODB_URI` Atlas nếu muốn `mongodb: connected`.

---

## 10. Tài liệu liên quan

| Tài liệu | Nội dung |
|----------|----------|
| `backend/.env.example` | Template tất cả biến |
| `backend/ENV_SETUP.md` | Giải thích `.env` tiếng Việt |
| [postman-walkthrough-vi.md](./postman-walkthrough-vi.md) | Test API đầy đủ sau deploy |
| `deployments/sepolia.json` | Địa chỉ contract Sepolia |
| `backend/Dockerfile` | Image production |
| `backend/railway.toml` | Cấu hình Railway |
| `backend/render.yaml` | Blueprint Render |

---

*Cập nhật: 2026-06-24 — đồng bộ sau merge monorepo dev→main và backend dev→main.*

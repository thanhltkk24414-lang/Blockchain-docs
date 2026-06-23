# Hướng dẫn chi tiết test API Fapex (tiếng Việt)

Tài liệu này hướng dẫn **từng bước** luồng kiểm thử backend Fapex: khởi động môi trường → gọi API → đăng nhập ví (SIWE) → upload IPFS → tạo job → xem danh sách job.

> **Mục tiêu:** Sau khi làm theo guide, bạn có thể gọi API thành công mà hiểu **tại sao** mỗi bước cần thiết.

---

## ⚠️ KHÔNG dùng extension Postman trong VS Code / Cursor

Extension **Postman** trên VS Code/Cursor **thường báo lỗi** `Could not import collection` với project này — kể cả file collection tối giản. **Đừng mất thời gian sửa import.**

**Dùng một trong ba cách sau (theo thứ tự khuyến nghị):**

| Cách | File / công cụ | Ai nên dùng |
|------|----------------|-------------|
| **1. REST Client** (khuyến nghị trong Cursor) | `backend/api-tests.http` + extension [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) | Làm việc trong VS Code/Cursor, không cần Postman |
| **2. Postman Desktop** | `backend/postman/*.json` | Quen giao diện Postman, import ổn định |
| **3. PowerShell script** | `backend/scripts/test-api-flow.ps1` | Windows, chạy cả luồng tương tác |

> **REST Client:** Cài extension → mở `backend/api-tests.http` → sửa biến `@baseUrl`, `@walletAddress` ở đầu file → nhấn **Send Request** phía trên mỗi khối `###`.

---

## Tổng quan luồng

```mermaid
flowchart LR
  A[MongoDB + Backend] --> B[Health Check]
  B --> C[POST nonce]
  C --> D[Ký SIWE bằng MetaMask]
  D --> E[POST verify → JWT]
  E --> F[Upload IPFS metadata]
  F --> G[POST tạo job]
  G --> H[GET danh sách jobs]
```

| Bước | API | Cần gì |
|------|-----|--------|
| 0 | Chuẩn bị | Docker MongoDB, backend chạy, chọn công cụ test (REST Client / Postman Desktop / PowerShell) |
| 1 | Cấu hình biến | `baseUrl`, `walletAddress` (trong `api-tests.http` hoặc Postman environment) |
| 3 | `GET /health` | Không cần DB (nhưng nên có MongoDB) |
| 4 | `POST /api/auth/nonce` | MongoDB |
| 5 | Ký SIWE (thủ công) | MetaMask + Sepolia |
| 6 | `POST /api/auth/verify` | Message + chữ ký từ bước 5 |
| 7 | Copy `authToken` | JWT từ response verify |
| 8 | `POST /api/ipfs/upload/metadata` | Bearer token + Pinata |
| 9 | `POST /api/jobs` | Token + MongoDB + Pinata + RPC Sepolia |
| 10 | `GET /api/jobs` | MongoDB |

---

## Phần A — Điều kiện tiên quyết

### A.1. Phần mềm cần cài

| Phần mềm | Mục đích |
|----------|----------|
| [Node.js](https://nodejs.org/) (LTS) | Chạy backend |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | Chạy MongoDB local |
| Extension **REST Client** (`humao.rest-client`) hoặc [Postman Desktop](https://www.postman.com/downloads/) | Gọi API |
| [MetaMask](https://metamask.io/) | Ví Ethereum, ký SIWE |
| Tài khoản Sepolia | Ví có ETH testnet (faucet Sepolia) |

### A.2. Biến môi trường backend (`backend/.env`)

Sao chép file mẫu và điền giá trị:

```bash
cd backend
cp .env.example .env
```

Các biến **bắt buộc** cho luồng Postman đầy đủ:

```env
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/freelance-platform
JWT_SECRET=chuoi_bi_mat_dai_ngau_nhien
PINATA_JWT=eyJ...   # lấy từ https://app.pinata.cloud
RPC_URL=https://sepolia.infura.io/v3/YOUR_PROJECT_ID

SIWE_DOMAIN=localhost
APP_URL=http://localhost:3000
CHAIN_ID=11155111

# Khuyến nghị khi test Postman — tắt indexer để tránh lỗi Infura rate limit
ENABLE_EVENT_INDEXER=false
```

> **Lưu ý Windows:** Luôn dùng `127.0.0.1` thay vì `localhost` trong `MONGODB_URI` và `baseUrl` Postman — tránh lỗi IPv6 `ECONNREFUSED ::1:27017`.

### A.3. Khởi động MongoDB (Docker)

Mở terminal tại thư mục `backend`:

```bash
npm run docker:mongo
```

Script sẽ:
- Tạo container MongoDB **chạy nền** (detached) tên `mongo`
- In connection string và **thoát ngay** — terminal không bị treo

Kết quả mong đợi:

```
✅ Created and started MongoDB container "mongo" (detached).

Connection string for backend/.env:
  MONGODB_URI=mongodb://127.0.0.1:27017/freelance-platform
```

Nếu container đã tồn tại, script chỉ **start** lại và báo trạng thái.

**Lệnh hữu ích:**

```bash
docker ps              # xem container đang chạy
docker stop mongo      # dừng MongoDB
docker rm -f mongo     # xóa container (chạy lại docker:mongo để tạo mới)
```

### A.4. Khởi động backend

```bash
cd backend
npm install
npm start
# hoặc phát triển: npm run dev
```

Khi thành công, log hiển thị:

```
Server running on port 5000
```

Nếu `ENABLE_EVENT_INDEXER=false`, bạn sẽ thấy:

```
Event indexer disabled (ENABLE_EVENT_INDEXER=false)
```

Điều này **bình thường** khi test Postman — indexer không cần thiết cho auth/jobs.

---

## Phần B — Chọn công cụ gọi API

### Cách 1 — REST Client trong VS Code / Cursor (khuyến nghị)

1. Mở **Extensions** (`Ctrl+Shift+X`)
2. Tìm và cài **REST Client** (tác giả: **Huachao Mao**, ID: `humao.rest-client`)
3. Mở file `backend/api-tests.http` trong editor
4. Sửa các biến ở **đầu file**:

| Biến | Giá trị | Giải thích |
|------|---------|------------|
| `@baseUrl` | `http://127.0.0.1:5000` | URL backend (khớp `PORT` trong `.env`) |
| `@walletAddress` | `0xABC...` | Địa chỉ ví MetaMask Sepolia |
| `@authToken` | *(để trống ban đầu)* | JWT sau bước verify — dán vào đây |
| `@siweMessage` | *(sau khi ký MetaMask)* | Chuỗi message SIWE đầy đủ |
| `@siweSignature` | *(sau khi ký MetaMask)* | Chữ ký hex `0x...` |

5. Cuộn tới request cần gọi (ví dụ `### 01 — GET /health`)
6. Nhấn link **Send Request** xuất hiện **ngay phía trên** dòng `GET` hoặc `POST`
7. Kết quả hiện ở panel bên phải (hoặc tab mới tùy cấu hình)

**Thứ tự request trong file:** Health → Nonce → Verify → Me → IPFS metadata → POST job → GET jobs.

> REST Client **không có sidebar** như Postman — mọi request nằm trong một file `.http`. Đó là cách chuẩn của VS Code.

### Cách 2 — Postman Desktop

1. Tải và cài [Postman Desktop](https://www.postman.com/downloads/)
2. Mở **Postman Desktop** (không phải extension VS Code)
3. Nhấn **Import** → chọn **hai file**:

   | File | Vai trò |
   |------|---------|
   | `backend/postman/Fapex.postman_collection.json` | Danh sách request API |
   | `backend/postman/Fapex-Local.postman_environment.json` | Biến môi trường |

4. Chọn environment **Fapex — Local** (góc trên phải)
5. Sửa biến `baseUrl`, `walletAddress` trong environment

### Cách 3 — PowerShell script (Windows)

```powershell
cd backend
.\scripts\test-api-flow.ps1
```

Script sẽ: health → nonce → **hướng dẫn ký SIWE** → verify → me → IPFS → POST job → GET jobs.

Chạy lại khi đã có JWT:

```powershell
.\scripts\test-api-flow.ps1 -WalletAddress 0xYour... -AuthToken eyJ...
```

Bỏ qua bước tốn Pinata/RPC:

```powershell
.\scripts\test-api-flow.ps1 -SkipIpfs -SkipJob
```

### ❌ Không dùng: Extension Postman trong VS Code / Cursor

Extension Postman sidebar trong editor **bị lỗi import** (`Could not import collection`) với collection của project này. File `Fapex-minimal.postman_collection.json` (định dạng v2.0) chỉ để thử trên **Postman Desktop** nếu cần — **không** dùng để debug extension VS Code.

---

## Phần C — Chạy từng request

> Áp dụng cho **REST Client** (`api-tests.http`), **Postman Desktop**, hoặc **PowerShell** — thao tác gửi request khác nhau nhưng body/response giống nhau.

### Bước 3: Health Check

**REST Client:** nhấn **Send Request** trên khối `### 01 — GET /health`

**Postman Desktop:** folder **01 — Health** → **GET /health** → **Send**

**Response mong đợi (200):**

```json
{
  "status": "ok",
  "mongodb": "connected",
  ...
}
```

| Trường `mongodb` | Ý nghĩa |
|------------------|---------|
| `"connected"` | MongoDB OK — tiếp tục test nonce/jobs |
| `"disconnected"` | Chạy `npm run docker:mongo` và kiểm tra `MONGODB_URI` |

> Health check **không cần** MongoDB để trả 200, nhưng các bước sau **cần** MongoDB connected.

---

### Bước 4: Auth — Lấy nonce

**REST Client:** khối `### 02 — POST /api/auth/nonce` → **Send Request** (đảm bảo `@walletAddress` đã sửa)

**Postman:** folder **02 — Auth** → **POST /api/auth/nonce** → **Send**

**Response mong đợi (200):**

```json
{
  "success": true,
  "nonce": "a1b2c3d4e5f6...",
  "walletAddress": "0x...",
  "domain": "localhost",
  "chainId": 11155111
}
```

Copy thủ công từ response:
- `nonce`, `chainId`, `domain` → dùng khi ký SIWE (bước 5); REST Client: dán vào `@siweMessage` / `@siweSignature` sau khi ký

**Nonce chỉ dùng một lần** — hết hạn sau khi verify thành công hoặc khi gọi nonce mới.

---

### Bước 5: Ký SIWE bằng MetaMask (bước thủ công — quan trọng)

REST Client / Postman / PowerShell **không thể** ký thay ví. Bạn phải tạo message SIWE (chuẩn EIP-4361) và ký bằng MetaMask.

#### SIWE là gì?

- **Sign-In With Ethereum** — đăng nhập bằng chữ ký ví thay mật khẩu
- Backend phát `nonce` → bạn ký message chứa nonce → backend xác minh → trả JWT

#### Cách 1 — Console trình duyệt (khuyến nghị)

1. Cài MetaMask, chuyển mạng **Sepolia**
2. Mở trang bất kỳ (ví dụ `https://example.com` hoặc tab trống `about:blank`)
3. Mở **DevTools** (F12) → tab **Console**
4. Dán và chạy script sau — **sửa các giá trị** theo response nonce và `.env` backend:

```javascript
// === SỬA CÁC GIÁ TRỊ NÀY ===
const walletAddress = '0xYourWalletAddress';  // giống walletAddress trong Postman
const nonce = 'PASTE_NONCE_FROM_STEP_4';      // từ POST /api/auth/nonce
const domain = 'localhost';                    // khớp SIWE_DOMAIN trong backend/.env
const chainId = 11155111;                      // khớp CHAIN_ID
const uri = 'http://localhost:3000';           // khớp APP_URL trong backend/.env
const statement = 'Sign in to Fapex';

// === KÝ SIWE ===
const { SiweMessage } = await import('https://esm.sh/siwe@3');

const message = new SiweMessage({
  domain,
  address: walletAddress,
  statement,
  uri,
  version: '1',
  chainId,
  nonce,
});

const prepared = message.prepareMessage();
console.log('--- SIWE MESSAGE (copy to Postman siweMessage) ---');
console.log(prepared);

const signature = await ethereum.request({
  method: 'personal_sign',
  params: [prepared, walletAddress],
});

console.log('--- SIGNATURE (copy to Postman siweSignature) ---');
console.log(signature);
```

5. MetaMask hiện popup **Sign message** → xác nhận
6. Copy hai giá trị từ console:
   - Toàn bộ **message** (nhiều dòng) → REST Client: `@siweMessage` · Postman: `siweMessage`
   - **signature** (chuỗi hex `0x...`) → REST Client: `@siweSignature` · Postman: `siweSignature`

#### Cách 2 — Snippet HTML local (nếu console bị chặn import)

Tạo file `siwe-sign.html` trên máy:

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>SIWE Sign</title></head>
<body>
  <p>Mở file này trong Chrome, kết nối MetaMask Sepolia, điền form và ký.</p>
  <script type="module">
    import { SiweMessage } from 'https://esm.sh/siwe@3';
    window.signSiwe = async () => {
      const walletAddress = document.getElementById('addr').value;
      const nonce = document.getElementById('nonce').value;
      const domain = 'localhost';
      const chainId = 11155111;
      const uri = 'http://localhost:3000';
      const msg = new SiweMessage({
        domain, address: walletAddress, statement: 'Sign in to Fapex',
        uri, version: '1', chainId, nonce,
      });
      const prepared = msg.prepareMessage();
      const signature = await ethereum.request({
        method: 'personal_sign',
        params: [prepared, walletAddress],
      });
      document.getElementById('out').textContent =
        'MESSAGE:\n' + prepared + '\n\nSIGNATURE:\n' + signature;
    };
  </script>
  <label>Địa chỉ ví: <input id="addr" size="50"></label><br>
  <label>Nonce: <input id="nonce" size="50"></label><br>
  <button onclick="signSiwe()">Ký với MetaMask</button>
  <pre id="out"></pre>
</body>
</html>
```

Mở file bằng trình duyệt → nhập địa chỉ + nonce → **Ký với MetaMask** → copy kết quả.

#### Lưu ý khi ký SIWE

| Tham số | Phải khớp với |
|---------|---------------|
| `domain` | `SIWE_DOMAIN` trong `backend/.env` |
| `uri` | `APP_URL` trong `backend/.env` |
| `chainId` | `CHAIN_ID` (11155111 = Sepolia) |
| `nonce` | Giá trị vừa nhận từ POST nonce |
| `address` | Ví đang kết nối MetaMask |

Sai một trường → `SIWE verification failed` ở bước verify.

---

### Bước 6: Verify — đổi nonce lấy JWT

1. Đảm bảo `siweMessage` và `siweSignature` đã điền trong environment
2. **POST /api/auth/verify** → **Send**

Body mẫu (Postman tự điền biến):

```json
{
  "message": "{{siweMessage}}",
  "signature": "{{siweSignature}}"
}
```

**Response thành công (200):**

```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "walletAddress": "0x...", ... }
}
```

Script Tests **không còn** trong collection — copy thủ công `token` từ response vào biến `authToken`.

### Bước 7: Kiểm tra token (tùy chọn nhưng nên làm)

1. **GET /api/auth/me**
2. Header: `Authorization: Bearer {{authToken}}`
3. Response 200 với thông tin user → token hợp lệ

Nếu `authToken` trống, mở environment → dán thủ công giá trị `token` từ response verify.

---

### Bước 8: Upload metadata lên IPFS

1. Folder **03 — IPFS** → **POST /api/ipfs/upload/metadata**
2. Cần header `Authorization: Bearer {{authToken}}`
3. Body mẫu đã có sẵn (title, description, skills…)
4. **Send**

**Response thành công:**

```json
{
  "success": true,
  "metadataCID": "Qm..."
}
```

Copy `metadataCID` từ response vào biến environment thủ công.

**Lỗi thường gặp:** `Không thể upload metadata` → kiểm tra `PINATA_JWT` trong `.env`.

---

### Bước 9: Tạo job (POST)

1. Folder **04 — Jobs** → **POST /api/jobs**
2. Cần: Bearer token, MongoDB, Pinata, RPC Sepolia
3. Body ví dụ:

```json
{
  "title": "Smart Contract Audit",
  "description": "Audit Solidity escrow contracts on Sepolia testnet for security vulnerabilities.",
  "category": "development",
  "contractValue": 100,
  "duration": 604800,
  "skills": ["Solidity", "Security"],
  "deliverables": "PDF audit report with findings and remediation plan.",
  "acceptanceCriteria": "All critical and high issues documented with reproducible PoCs."
}
```

4. **Send**

Backend sẽ:
- Upload metadata lên IPFS (nếu chưa có)
- Gọi contract `createJob` trên Sepolia qua `RPC_URL`
- Lưu job vào MongoDB

> User phải đã đăng nhập (có bản ghi từ nonce/verify) và không ở tier `Restricted`.

---

### Bước 10: Xem danh sách jobs (GET)

1. **GET /api/jobs**
2. Không cần auth (public)
3. Query: `page=1`, `limit=20`
4. **Send** → danh sách job trong MongoDB

---

### Bước phụ — Trạng thái trọng tài (không bắt buộc)

**05 — Arbitrator** → **GET /api/arbitrator/:address/status**

- Chỉ cần RPC Sepolia, không cần MongoDB
- Kiểm tra `stakedAmount` và `isValid` (tối thiểu 50 USDC stake)

---

## Phần D — Xử lý sự cố (Troubleshooting)

### MongoDB

| Triệu chứng | Nguyên nhân | Cách sửa |
|-------------|-------------|----------|
| `mongodb: disconnected` trong /health | Container chưa chạy | `npm run docker:mongo` |
| `ECONNREFUSED ::1:27017` | Windows resolve localhost → IPv6 | Dùng `127.0.0.1` trong `MONGODB_URI` |
| `buffering timed out` | MongoDB không kết nối được | `docker ps` xem container `mongo`; `docker start mongo` |

### Infura / RPC rate limit

| Triệu chứng | Nguyên nhân | Cách sửa |
|-------------|-------------|----------|
| Log `Too Many Requests` code `-32005` | Infura free tier giới hạn `eth_getLogs` | Đặt `ENABLE_EVENT_INDEXER=false` trong `.env` khi test Postman |
| `Index JobStatusUpdated error` | Event indexer gọi RPC quá nhiều | Tắt indexer (trên) hoặc dùng RPC provider khác (Alchemy) |
| Request API chậm | Indexer đang poll blockchain | Tắt indexer cho local dev |

**Khuyến nghị cho test Postman:**

```env
ENABLE_EVENT_INDEXER=false
```

Khởi động lại backend sau khi sửa `.env`.

### SIWE / Auth

| Lỗi | Cách sửa |
|-----|----------|
| `SIWE verification failed` | Kiểm tra `domain`, `uri`, `chainId`, `nonce` khớp `.env`; gọi nonce mới |
| `Invalid or expired nonce` | Chạy lại POST nonce, ký lại message mới |
| `JWT_SECRET is not defined` | Thêm `JWT_SECRET` vào `.env` |

### Công cụ test / Windows

| Lỗi | Cách sửa |
|-----|----------|
| `Could not import collection` (extension Postman VS Code) | **Bỏ extension Postman** — dùng REST Client (`api-tests.http`) hoặc Postman Desktop |
| Không thấy nút Send trong Cursor | Cài extension REST Client (`humao.rest-client`); mở file `.http` không phải `.json` |
| Sidebar Postman trống | Chỉ áp dụng Postman Desktop: nhấn **Collections** ở thanh trái |
| Request treo mãi | Đổi `baseUrl` / `@baseUrl` từ `localhost` → `127.0.0.1` |
| 401 trên route có Bearer | Kiểm tra `@authToken` / `authToken`; chạy lại verify |

### Pinata / IPFS

| Lỗi | Cách sửa |
|-----|----------|
| Upload metadata thất bại | Tạo JWT tại [Pinata API Keys](https://app.pinata.cloud/developers/api-keys) → `PINATA_JWT` |

---

## Phần E — Kiểm tra nhanh (không cần Postman)

### REST Client (file có sẵn)

```text
backend/api-tests.http
```

Cài extension REST Client → mở file → Send Request từng khối.

### PowerShell script (luồng đầy đủ)

```powershell
cd backend
.\scripts\test-api-flow.ps1
```

### Script npm (smoke test)

```bash
cd backend
npm run test:api              # GET /health (server phải chạy)
npm run test:api -- --with-nonce   # health + nonce (cần MongoDB)
npm run test:auth               # load module JWT/SIWE
```

### curl / PowerShell (từng bước thay Postman)

Đặt biến (PowerShell):

```powershell
$baseUrl = "http://127.0.0.1:5000"
$wallet = "0xYourWalletAddressHere"
```

**Bước 1 — Health**

```powershell
Invoke-RestMethod -Uri "$baseUrl/health"
```

```bash
curl -s "$baseUrl/health"
```

**Bước 2 — Nonce** (cần MongoDB)

```powershell
$nonceBody = @{ walletAddress = $wallet } | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/api/auth/nonce" -Method POST -Body $nonceBody -ContentType "application/json"
```

```bash
curl -s -X POST "$baseUrl/api/auth/nonce" -H "Content-Type: application/json" -d "{\"walletAddress\":\"$wallet\"}"
```

Lưu `nonce`, `domain`, `chainId` từ response → ký SIWE (Phần C, Bước 5).

**Bước 3 — Verify** (sau khi có `siweMessage` và `siweSignature`)

```powershell
$verifyBody = @{ message = $siweMessage; signature = $siweSignature } | ConvertTo-Json
$r = Invoke-RestMethod -Uri "$baseUrl/api/auth/verify" -Method POST -Body $verifyBody -ContentType "application/json"
$token = $r.token
```

```bash
curl -s -X POST "$baseUrl/api/auth/verify" -H "Content-Type: application/json" \
  -d "{\"message\":\"...\",\"signature\":\"0x...\"}"
```

**Bước 4 — Me** (kiểm tra JWT)

```powershell
Invoke-RestMethod -Uri "$baseUrl/api/auth/me" -Headers @{ Authorization = "Bearer $token" }
```

**Bước 5 — IPFS metadata** (cần Pinata + token)

```powershell
$metaBody = Get-Content -Raw -Path metadata-sample.json   # hoặc tạo JSON inline
Invoke-RestMethod -Uri "$baseUrl/api/ipfs/upload/metadata" -Method POST -Body $metaBody -ContentType "application/json" -Headers @{ Authorization = "Bearer $token" }
```

**Bước 6 — GET jobs**

```powershell
Invoke-RestMethod -Uri "$baseUrl/api/jobs?page=1&limit=20"
```

**Bước 7 — POST job** (cần token, MongoDB, Pinata, RPC)

```powershell
$jobBody = @{
  title = "Smart Contract Audit"
  description = "Audit Solidity escrow contracts on Sepolia testnet."
  category = "development"
  contractValue = 100
  duration = 604800
  skills = @("Solidity", "Security")
  deliverables = "PDF audit report."
  acceptanceCriteria = "All critical issues documented."
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/api/jobs" -Method POST -Body $jobBody -ContentType "application/json" -Headers @{ Authorization = "Bearer $token" }
```

**Bước 8 — Arbitrator status**

```powershell
Invoke-RestMethod -Uri "$baseUrl/api/arbitrator/$wallet/status"
```

---

## Tóm tắt checklist

- [ ] Docker Desktop đang chạy
- [ ] `npm run docker:mongo` → thông báo container detached
- [ ] `backend/.env` đã điền: MongoDB, JWT, Pinata, RPC
- [ ] `ENABLE_EVENT_INDEXER=false` (khi test local)
- [ ] `npm start` → Server port 5000
- [ ] Chọn công cụ: REST Client (`api-tests.http`) **hoặc** Postman Desktop **hoặc** `test-api-flow.ps1`
- [ ] `baseUrl` / `@baseUrl` = `http://127.0.0.1:5000`, `walletAddress` = ví Sepolia
- [ ] Health → nonce → ký SIWE MetaMask → verify → `authToken`
- [ ] IPFS upload → POST job → GET jobs

---

**Tài liệu liên quan:** [postman-testing.md](./postman-testing.md) · [auth-api.md](./auth-api.md)

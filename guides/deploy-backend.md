# Deploy Fapex Backend (Railway & Render)

Step-by-step guide to deploy the **blockchain-backend** API to [Railway](https://railway.app) or [Render](https://render.com). No account automation — you connect the GitHub repo and paste secrets in each platform's dashboard.

**Repo:** [thanhltkk24414-lang/blockchain-backend](https://github.com/thanhltkk24414-lang/blockchain-backend)  
**Entry point:** `src/server.js` (`npm start`)  
**Health check:** `GET /health` (works without MongoDB)

---

## Prerequisites

| Service | Purpose | Free tier |
|---------|---------|-----------|
| **MongoDB Atlas** | Users, jobs, auth, event indexer state | M0 cluster |
| **Infura / Alchemy** | Sepolia `RPC_URL` | Yes |
| **Pinata** | IPFS uploads (`PINATA_JWT` or API key pair) | Yes |
| **GitHub** | Connect repo to Railway/Render | — |

Copy contract addresses from `deployments/sepolia.json` in the monorepo root (or use values in `backend/.env.example`).

Generate a strong `JWT_SECRET` (≥ 32 random characters). **Never commit secrets.**

---

## Required environment variables (production)

Set these in the Railway or Render **Environment** tab. See `backend/.env.example` and `backend/ENV_SETUP.md` for full descriptions.

| Variable | Example / notes | Required |
|----------|-----------------|----------|
| `NODE_ENV` | `production` | Yes |
| `PORT` | Set automatically by Railway/Render — do not hardcode in dashboard unless needed | Auto |
| `MONGODB_URI` | `mongodb+srv://user:pass@cluster.mongodb.net/freelance-platform?retryWrites=true&w=majority` | Yes |
| `JWT_SECRET` | Long random string | Yes |
| `RPC_URL` | `https://sepolia.infura.io/v3/YOUR_PROJECT_ID` | Yes |
| `ALLOWED_ORIGINS` | Comma-separated frontend URLs, e.g. `https://your-app.vercel.app` | Yes (when frontend exists) |
| `SIWE_DOMAIN` | Hostname only, **no** `https://` — e.g. `fapex-backend.up.railway.app` | Yes (SIWE auth) |
| `APP_URL` | Full public backend URL, e.g. `https://fapex-backend.up.railway.app` | Yes (SIWE auth) |
| `CHAIN_ID` | `11155111` (Sepolia) | Yes |
| `MOCK_USDC_ADDRESS` | From `deployments/sepolia.json` | Yes |
| `REPUTATION_STORE_ADDRESS` | From deployments | Yes |
| `PLATFORM_TREASURY_ADDRESS` | From deployments | Yes |
| `JOB_REGISTRY_ADDRESS` | From deployments | Yes |
| `ARBITRATOR_PANEL_ADDRESS` | From deployments | Yes |
| `ESCROW_VAULT_ADDRESS` | From deployments | Yes |
| `PINATA_JWT` | Pinata JWT **or** use `PINATA_API_KEY` + `PINATA_SECRET_API_KEY` | Yes (IPFS) |
| `IPFS_GATEWAY_URL` | `https://gateway.pinata.cloud` | Recommended |
| `ENABLE_EVENT_INDEXER` | `true` (default) or `false` to reduce RPC usage on free tier | Optional |
| `INDEXER_PRIVATE_KEY` | Sepolia wallet key for on-chain indexer txs | Optional |
| `SEPOLIA_WSS_URL` | WebSocket RPC for realtime listener | Optional |

**SIWE after deploy:** `SIWE_DOMAIN` must match the hostname users see in MetaMask (your Railway/Render URL). `APP_URL` is the full HTTPS URL used as SIWE `uri`. Update `ALLOWED_ORIGINS` when you deploy the frontend.

**MongoDB Atlas:** In Network Access, allow `0.0.0.0/0` (or Railway/Render egress IPs) so the cloud host can connect.

---

## Option A — Railway

Railway reads `railway.toml` and `Dockerfile` from the repo root of **blockchain-backend**.

### 1. Create project

1. Log in at [railway.app](https://railway.app).
2. **New Project** → **Deploy from GitHub repo**.
3. Select `thanhltkk24414-lang/blockchain-backend`.
4. Branch: **`main`** (after deploy PR is merged) or **`dev`** for testing.

### 2. Configure service

- **Builder:** Dockerfile (auto-detected via `railway.toml`).
- **Health check:** path `/health` (configured in `railway.toml`).
- **Root directory:** `/` (backend repo root — not the monorepo root).

### 3. Environment variables

In the service → **Variables**, add every variable from the table above. Railway sets `PORT` automatically.

### 4. Generate domain

1. **Settings** → **Networking** → **Generate Domain**.
2. Copy the URL (e.g. `https://blockchain-backend-production-xxxx.up.railway.app`).
3. Set:
   - `SIWE_DOMAIN` = hostname only (`blockchain-backend-production-xxxx.up.railway.app`)
   - `APP_URL` = full URL with `https://`
   - `ALLOWED_ORIGINS` = your frontend origin(s)

Redeploy after changing env vars if the service does not restart automatically.

### 5. Verify

```bash
curl https://YOUR-RAILWAY-URL.up.railway.app/health
```

Expected JSON: `"status":"ok"` and `"mongodb":"connected"` when Atlas is configured.

SIWE helper page: `https://YOUR-RAILWAY-URL.up.railway.app/siwe-sign.html`

---

## Option B — Render

Render can use `render.yaml` (Blueprint) or a manual Web Service with Docker.

### 1a. Blueprint (recommended)

1. [dashboard.render.com](https://dashboard.render.com) → **New** → **Blueprint**.
2. Connect GitHub repo `thanhltkk24414-lang/blockchain-backend`.
3. Render detects `render.yaml` and prompts for secret env vars (`sync: false`).
4. Fill in MongoDB URI, JWT, RPC, Pinata, contract addresses, SIWE/ CORS URLs after first deploy when you know the public URL.

### 1b. Manual Web Service

1. **New** → **Web Service** → connect `blockchain-backend`.
2. **Runtime:** Docker.
3. **Dockerfile path:** `./Dockerfile`.
4. **Health check path:** `/health`.
5. Branch: `main` (or `dev` for staging).

### 2. Environment

Add the same variables as in the Railway table. Render injects `PORT`; the app uses `process.env.PORT || 5000`.

### 3. Custom URL

After deploy, note the `*.onrender.com` URL and set `SIWE_DOMAIN`, `APP_URL`, and `ALLOWED_ORIGINS` accordingly. Trigger a manual redeploy.

### 4. Verify

```bash
curl https://YOUR-SERVICE.onrender.com/health
```

**Free tier:** service may sleep after inactivity; first request can take ~30s.

---

## Local Docker smoke test (optional)

From `backend/`:

```powershell
cd d:\projects\Blockchain\backend
docker build -t fapex-backend .
docker run --rm -p 5000:5000 --env-file .env fapex-backend
```

Visit `http://127.0.0.1:5000/health`. Use a local `.env` with Atlas `MONGODB_URI` if you want `mongodb: connected`.

---

## Post-deploy checklist

- [ ] `GET /health` returns 200 and `mongodb: connected`
- [ ] `SIWE_DOMAIN` / `APP_URL` match the public HTTPS URL
- [ ] `POST /api/auth/nonce` works from Postman (`{{baseUrl}}` = production URL)
- [ ] SIWE sign page loads: `/siwe-sign.html`
- [ ] CORS: frontend origin listed in `ALLOWED_ORIGINS`
- [ ] Pinata upload: `POST /api/ipfs/upload/metadata` (with JWT auth)
- [ ] Event indexer: check logs for RPC rate limits; set `ENABLE_EVENT_INDEXER=false` if needed on free tier

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Build fails on `npm ci` | Ensure `package-lock.json` is committed; do not copy `node_modules` into Docker context |
| Health check fails | Wait for cold start; confirm `PORT` is used (app already reads `process.env.PORT`) |
| `mongodb: disconnected` | Atlas IP allowlist, correct `MONGODB_URI`, database user password URL-encoded |
| SIWE verify fails | `SIWE_DOMAIN` must be hostname only; message address must be EIP-55; re-fetch nonce after deploy URL change |
| CORS errors | Add frontend URL to `ALLOWED_ORIGINS` (comma-separated, no trailing slash) |

---

## Related docs

- `backend/ENV_SETUP.md` — detailed `.env` reference (Vietnamese)
- `backend/.env.example` — template for all variables
- [Postman walkthrough (VI)](postman-walkthrough-vi.md) — API testing against deployed URL

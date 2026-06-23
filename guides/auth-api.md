# SIWE + JWT Authentication

Backend auth for wallet sign-in (Sepolia / Ethereum).

## Environment

Set in `backend/.env` (see `.env.example`):

| Variable | Example | Purpose |
|----------|---------|---------|
| `JWT_SECRET` | long random string | Signs API tokens |
| `JWT_EXPIRES_IN` | `7d` | Token lifetime |
| `SIWE_DOMAIN` | `localhost` | Domain in SIWE message |
| `APP_URL` | `http://localhost:3000` | URI in SIWE message |
| `CHAIN_ID` | `11155111` | Sepolia |

## Endpoints

### `POST /api/auth/nonce`

Request a one-time nonce before signing.

```json
{ "walletAddress": "0xYourWallet..." }
```

Response:

```json
{
  "success": true,
  "nonce": "abc123...",
  "walletAddress": "0x...",
  "domain": "localhost",
  "chainId": 11155111
}
```

### `POST /api/auth/verify`

Verify SIWE signature and receive JWT.

```json
{
  "message": "<EIP-4361 SIWE message string>",
  "signature": "0x..."
}
```

Response:

```json
{
  "success": true,
  "token": "eyJ...",
  "expiresIn": "7d",
  "user": { "walletAddress": "0x...", "username": "...", "role": "client" }
}
```

### `GET /api/auth/me`

Protected. Header: `Authorization: Bearer <token>`

## Protected routes

- `POST /api/jobs`
- `PATCH /api/jobs/:id/status`
- `POST /api/bids`
- `PATCH /api/bids/:id/accept`
- `PATCH /api/bids/:id/reject`
- `POST /api/ipfs/upload/metadata`
- `POST /api/ipfs/upload/file`

Public: `GET /health`, `GET /api/jobs`, `GET /api/jobs/:id`

## Frontend flow

1. `POST /api/auth/nonce` with wallet address
2. Build `SiweMessage` (domain, chainId, nonce, uri from env)
3. `signer.signMessage(message.prepareMessage())`
4. `POST /api/auth/verify` with message + signature
5. Store `token`; send `Authorization: Bearer <token>` on protected calls

## Smoke test

```bash
cd backend
npm run test:auth
```

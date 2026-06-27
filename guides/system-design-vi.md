# Thiết kế hệ thống — FAPEX (VI)

```mermaid
flowchart TB
  subgraph client [Client Browser]
    FE[Vite React + wagmi]
  end
  subgraph backend [Backend Railway]
    API[Express REST]
    IDX[Event Indexer]
    RT[Realtime WSS Listener]
    DB[(MongoDB)]
  end
  subgraph chain [Sepolia]
    JR[JobRegistry]
    EV[EscrowVault]
    AP[ArbitratorPanel]
    RS[ReputationStore]
    CL[Chainlink ETH/USD]
  end
  subgraph storage [IPFS]
    PIN[Pinata]
  end
  FE -->|SIWE JWT| API
  FE -->|read/write tx| chain
  FE -->|ETH/USD| CL
  API --> DB
  API --> PIN
  IDX -->|eth_getLogs| chain
  IDX --> DB
  RT --> chain
  chain -->|events| IDX
```

## Lớp

1. **Presentation:** `frontend/` — RainbowKit MetaMask, landing, dashboards
2. **API:** `backend/` — auth, jobs, bids, IPFS proxy, arbitrator status
3. **Indexer:** `eventIndexer.js` — batch poll, `IndexerState.lastBlock` checkpoint
4. **Contracts:** escrow 3% fee, dispute commit-reveal, reputation soulbound

## Cấu hình

- `GET /health` — contracts env, MongoDB
- `GET /api/config` — addresses cho frontend (FE-5)
- `deployments/sepolia.json` — địa chỉ canonical

## v2

- The Graph thay một phần indexer cho query công khai
- Chainlink VRF thay `prevrandao` sortition

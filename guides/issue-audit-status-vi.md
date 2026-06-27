# Ma trận audit issue — FAPEX (2026-06)

| ID | Mô tả | Trạng thái | Hành động |
|----|--------|------------|-----------|
| SC-1 | Dispute windows baked in bytecode | **Done (partial)** | `DisputeTimings.demo.sol` / `.prod.sol` + `prepare-dispute-timings.js`. Không redeploy Sepolia. v2: `setDisputeWindow` governance |
| SC-2 | Reentrancy escrow release | **Done** | CEI trong `_splitAndPayout`; USDC không callback. Không cần ReentrancyGuard trên deploy hiện tại |
| SC-3 | Centralized arbitrator pool | **Done (documented)** | Stake ≥50 USDC, slash no-reveal, reputation penalty. v2: VRF sortition |
| SC-5 | Reputation off-chain only | **Done** | `ReputationStore` + `useReputation` + `ReputationBadge` |
| SC-6 | Manual export ABIs | **Done** | Hardhat postcompile → `export-abis.js` → backend + frontend |
| BE-1 | Backend source of truth | **Done (arch)** | Event indexer + realtime listener = cache MongoDB. Chain = truth. v2: The Graph |
| BE-2 | SIWE not Web3-native | **Done** | EIP-4361 `/api/auth/nonce` + `/verify`, frontend SIWE, `siwe-sign.html` |
| BE-3 | Centralized deliverables | **Done** | Pinata upload API → CID on-chain (`metadataCID`, `deliverableCID`, evidence `ipfsHash`) |
| BE-4 | Missed blocks on restart | **Done** | `IndexerState` (`lastBlock`) persist + replay từ block đã lưu |
| FE-1 | Network mismatch | **Done** | `NetworkBanner` + `chainChanged` + `switchChain` |
| FE-2 | Tx pending UX | **Done** | `TxStatusModal`, disable buttons khi pending, Etherscan link |
| FE-3 | RPC fallback | **Done** | wagmi `fallback([primary, secondary])` |
| FE-4 | Gas estimate display | **Done (partial)** | `useContractTx` + gas line trong modal; mở rộng dần các hook |
| FE-5 | Hardcoded ABI | **Done (partial)** | `deployments-sepolia.json` + `GET /api/config` |
| CL-1 | ETH/USD price feed | **Done** | Chainlink Sepolia `0x694AA…` — `useEthUsdPrice` |
| CL-2 | VRF sortition | **Deferred v2** | Document + stub; hiện `prevrandao` |
| Brand | Landing + logos | **Done** | HYVE-style `/`, `fapex-icon.svg`, `fapex-wordmark.svg` |

## Deferred v2

- Chainlink VRF cho arbitrator sortition
- The Graph subgraph
- CCIP / Chainlink Functions
- On-chain `setDisputeWindow` governance (nếu cần đổi timing không redeploy)
- OpenZeppelin ReentrancyGuard (optional hardening)

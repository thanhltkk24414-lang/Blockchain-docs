# Chainlink — FAPEX (VI)

## v1 — Price Feeds (đã triển khai)

| Feed | Mạng | Địa chỉ |
|------|------|---------|
| ETH/USD | Sepolia | `0x694AA1769357215DE4FAC081bf1f309aDC325306` |

**Frontend:** `useEthUsdPrice` đọc `latestRoundData` qua wagmi/viem.

- Hiển thị giá ETH trên landing và header app
- Ước tính chi phí gas (kết hợp `formatGasWithUsd` trong tx modal)

**Env (tùy chọn):** `VITE_ETH_USD_FEED=0x694AA1769357215DE4FAC081bf1f309aDC325306`

## v1 strong — VRF sortition (deferred)

**Hiện tại:** `ArbitratorPanel.setupDisputePanel` chọn arbitrator bằng `block.prevrandao` (MVP, có stake gate).

**v2 đề xuất:**

1. Deploy Chainlink VRF Coordinator trên Sepolia (subscription)
2. `requestRandomWords` khi `raiseDispute`
3. Callback chọn 5 arbitrator từ pool (weighted by stake)
4. Stub contract: `contracts/chainlink/VRFSortitionStub.sol` (chưa wire production)

```solidity
// v2 sketch — không deploy trên Sepolia hiện tại
interface IVRFConsumer {
    function requestArbitratorSortition(uint256 jobId) external;
}
```

## Không nằm trong v1

- CCIP cross-chain
- Chainlink Functions (off-chain compute)

## Kiểm thử price feed

1. `npm run dev` trong `frontend/`
2. Mở `/` — badge **ETH $…** góc hero (cần RPC Sepolia)
3. Header app (sau khi rời landing) cũng hiển thị ETH/USD

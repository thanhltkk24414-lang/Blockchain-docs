# Fapex vs Kleros — Tranh chấp & minh bạch

Tài liệu ngắn so sánh luồng tranh chấp Fapex với Kleros, ghi nhận cơ chế phạt arbitrator, và các cải tiến minh bạch đã triển khai.

## 1. So sánh nhanh

| Khía cạnh | Kleros | Fapex (Sepolia demo) |
|-----------|--------|----------------------|
| Chọn hội đồng | Sortition stake-weighted (PNK) | Random 5/ pool, stake cố định 50 USDC |
| Vote | Commit–reveal | Commit–reveal (`keccak256(choice, salt)`) |
| Quorum | Theo subcourt | ≥3/5 vote reveal hợp lệ |
| Appeal | Nhiều vòng, stake loser | 1 vòng appeal (round 2), phí ≈1.3× |
| Phạt vote sai | Coherent penalty — minority mất stake | **Không** — chỉ phạt no-reveal |
| Phạt no-reveal | Có | Slash 5 USDC + reputation −10 |
| Bằng chứng on-chain | Metadata court | `bytes32` = `keccak256(CID)` — CID lưu off-chain |
| Thực thi ruling | Tự động / executor | Manual `executeArbitrationResult` |
| Fallback quorum fail | Appeal / stake redistribution | `adminForceResolve` (admin) |

## 2. Luồng Fapex (tóm tắt)

```
raiseDispute (EscrowVault, phí 2% cap 50 USDC)
  → setupDisputePanel: sortition 5 arbitrator
  → submitEvidence (0–30 phút demo / 0–120h prod): parties, hash CID
  → commitVote → revealVote
  → finalizeDisputeVoting: slash no-reveal, tally
  → [optional] fileAppeal → round 2
  → executeArbitrationResult: payout + reward arbitrator đúng kết quả
```

**Timing demo Sepolia:** xem `docs/guides/demo-seed-data-vi.md` và `DisputeTimings.demo.sol`.

## 3. Phạt arbitrator — on-chain

| Hành vi | Stake | ReputationStore |
|---------|-------|-----------------|
| Commit nhưng không reveal | Slash **5 USDC** → `totalReserveFund` | **−10** điểm |
| Không commit, không reveal | Không phạt | — |
| Vote thiểu số (sai kết quả) | Không phạt | — |
| Vote đúng kết quả cuối | Reward từ 50% dispute fee | — |
| Join pool | Stake ≥50 USDC | Score ≥80 (Normal+) |

**Lưu ý:** Không có “coherent vote penalty” kiểu Kleros. Slashed USDC không chuyển cho juror đa số — reward đến từ pool phí tranh chấp.

## 4. Bằng chứng công khai (đã fix)

- On-chain chỉ lưu `keccak256(CID)` — **không đảo ngược** được CID.
- **Giải pháp Fapex:**
  1. FE upload IPFS → `submitEvidence(hash)` → **POST API** lưu CID + metadata MongoDB.
  2. Indexer sync `EvidenceSubmitted` → gắn `onChainHash` với bản ghi off-chain.
  3. API public: `GET /api/disputes/onchain/:jobId/evidences` — hydrate JSON từ Pinata.
  4. FE merge on-chain + off-chain; fallback match theo `submitter` nếu thiếu hash map.

## 5. Minh bạch đã có / vừa bổ sung

| Tính năng | Trạng thái |
|-----------|------------|
| Bằng chứng xem công khai (mô tả, URL, ảnh IPFS) | ✅ DisputeEvidencePanel — mọi viewer |
| Phase timeline on-chain | ✅ DisputeStatusPanel |
| Vote tally sau reveal | ✅ Công khai trên DisputeStatusPanel + ArbitratorDisputePanel |
| Browse filter DISPUTED | ✅ + badge “Tranh chấp” trên JobCard |
| Reputation score/tier (ReputationStore) | ✅ Profile + header badge |
| Kết quả dispute sau finalize | ✅ DisputeResultPanel |

## 6. Gap audit — cần lưu ý

1. **`EVIDENCE_INITIAL_END` không enforce** — parties nộp evidence cả giai đoạn 0–120h (chỉ check rebuttal end).
2. **Indexer lag** — job/dispute Mongo có thể chậm vài phút so với chain.
3. **Evidence cũ chỉ on-chain** — không khôi phục CID nếu chưa từng POST API.
4. **Sortition seed** — `prevrandao + timestamp` (không VRF).
5. **Wrong vote** — không bị phạt stake/reputation.
6. **Quorum fail** — admin `adminForceResolve` thay vì cơ chế phi tập trung.
7. **`searchJobs` mặc định OPEN** — khi search có budget, cần truyền `status=DISPUTED` nếu muốn lọc disputed.

## 7. Hướng cải thiện tiếp (optional)

- Emit CID string trong event (contract upgrade) hoặc mapping `hash→CID` bắt buộc trước `submitEvidence`.
- Coherent penalty cho arbitrator vote sai (design + ReputationStore).
- VRF sortition (Chainlink).
- Auto-execute sau appeal window.
- Indexer backfill evidence từ historical tx + API logs.

---

*Contract ref: `contracts/FreelanceSystem.sol`, `docs/guides/contract-interaction.md` §4.4–4.5.*

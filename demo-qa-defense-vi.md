# Demo QA — Luồng kháng cáo (Appeal)

## Hành vi on-chain (EscrowVault + ArbitratorPanel)

Sau khi **Finalize voting** hoàn tất vòng 1 (cần ≥3 reveal hợp lệ):

1. **Cửa sổ appeal** mở trong `appealWindowMin` (demo Sepolia: **30 phút** sau `resultAt`).
2. Client hoặc freelancer gọi `EscrowVault.fileAppeal(jobId)`:
   - Thu phí appeal ≈ **1.3× dispute fee** (USDC — cần `approve` EscrowVault trước).
   - Emit `AppealFiled`.
   - Gọi `ArbitratorPanel.startAppealRound` → emit `AppealRoundStarted` với **5 arbitrator mới**.
3. **Vòng 2** chạy lại đầy đủ: evidence (nếu trong window) → commit → reveal → finalize.
4. **Không có vòng 3** — `AppealNotAllowed` nếu `round > 1`.
5. Nếu **không appeal**: sau cửa sổ appeal, bất kỳ ai cũng có thể `executeArbitrationResult` để giải ngân.

## Lỗi thường gặp khi demo

| Revert / lỗi | Nguyên nhân | Cách xử lý |
|--------------|-------------|------------|
| `VotingNotFinalized` | Chưa finalize vòng 1 | Gọi **Finalize voting** khi đủ quorum |
| `AppealWindowClosed` | Quá 30 phút sau finalize | Demo lại hoặc execute kết quả vòng 1 |
| `TransferFailed` / Panic | Thiếu USDC hoặc chưa `approve` appeal fee | Mint MockUSDC + approve 1.3× dispute fee |
| `AppealAlreadyFiled` | Đã appeal | Chuyển sang panel arbitrator vòng 2 |
| Metadata save failed | MongoDB chưa có `Dispute` | Backend tự **upsert** dispute theo `onchainJobId` (đã sửa) |

## UI FAPEX

- **DisputeResultPanel**: mô tả đúng luồng fee → round 2 panel mới.
- **fileAppeal** trong app: tự approve USDC appeal fee trước khi gửi tx.
- Bằng chứng dispute + deliverable: hiển thị công khai trên job SUBMITTED / DISPUTED / COMPLETED.

## Quorum fail (admin)

Khi reveal window kết thúc mà `<3` reveal: Admin dashboard liệt kê job tại **Quorum failed — needs force resolve**, kèm evidence hydrated; nút **Pre-fill force resolve** điền `adminForceResolve`.

# Admin cheatsheet — Demo FAPEX (1 trang)

**Route:** `/admin` · **Chain:** Sepolia · **UI:** English · **Auth:** ví MetaMask (on-chain roles)

---

## Trước khi lên sân khấu

```bash
# Monorepo root
npm run seed:arbitrators
npm run check:dispute
```

- Connect ví **deployer** hoặc ví có delegated role.
- Mở `/admin` → kiểm tra badge **Your access**.
- Chuẩn bị nói: *“Governance is wallet-gated on-chain; this is not a custodial backdoor.”*

---

## Ai làm gì (nhớ nhanh)

| Actor | Việc thường | Khẩn cấp |
|-------|-------------|----------|
| Client / Freelancer | escrow, evidence | `fileAppeal` (1 vòng) |
| 5 Arbitrators | commit → reveal | — |
| Anyone | finalize + execute sau timer | — |
| Pauser | — | `setPaused` |
| Force resolver | — | `adminForceResolve` **chỉ quorum fail** |
| Contract admin | grantRole, joinPool | all above |

**Quorum:** ≥3 valid reveals sau reveal window.

---

## Demo flow tranh chấp (normal path)

1. Client `raiseDispute` trên job SUBMITTED/IN_PROGRESS.
2. Chờ phase timer (demo: evidence 0–10m, commit 10–13m, reveal 13–16m).
3. ≥3 arbitrator reveal → `finalizeDisputeVoting` → `executeArbitrationResult`.
4. **Không mở** force resolve panel trừ khi cố tình demo quorum fail.

---

## Force resolve — khi nào nói / khi nào im

**Nói khi bị hỏi “admin chọn winner?”**

- Arbitrators quyết định trong luồng chuẩn.
- Force resolve = **fallback** khi &lt;3 reveals sau reveal — UI **disable** nếu chưa đủ điều kiện.
- Event `AdminForceResolved` trên EscrowVault — Etherscan.

**Đừng demo force resolve trừ khi đã setup quorum fail** — dễ bị hiểu nhầm thao túng.

**Nếu phải demo:** nhập job ID → đọc snapshot (phase, reveal count, arbs) → checkbox + gõ `FORCE`.

---

## Các panel `/admin`

| Panel | Ghi chú demo |
|-------|----------------|
| How governance works | Mở đầu — normal vs emergency |
| Emergency pause | Modal confirm; chỉ pause khi giải thích incident |
| Grant roles | Cảnh báo đỏ với force_resolver → “production = multisig only” |
| Force resolve | Cuối; chỉ khi quorum fail |

---

## Câu trả lời one-liner

| Câu hỏi | Trả lời |
|---------|---------|
| Admin RBAC Mongo? | Không — chỉ on-chain |
| Ai vào `/admin`? | `admin()`, deployer, delegated roles |
| MVP vs production? | Single deployer → multisig + timelock |
| Force resolve = thao túng? | Fallback quorum fail + event công khai + UI guard |

---

## Liên kết

- [demo-qa-defense-vi.md](demo-qa-defense-vi.md) — mục *Force resolve vs thao túng*
- [contract-interaction.md](contract-interaction.md)
- Frontend copy: `frontend/docs/admin-roles-vi.md`

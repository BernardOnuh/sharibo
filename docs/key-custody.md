# Key custody for circle admins and `fee_recipient` (#552)

**Status:** Reference — documents who holds the two honored addresses on a circle, how they
are protected, and how a circle admin is rotated. Feeds the incident-response plan in
[#551](https://github.com/crackedstudio/sharibo/issues/551).

## Why this matters

A circle stores two `Address` values, both chosen by the creator at `create_circle`
(`contracts/sharibo/src/lib.rs`):

| Address         | Set at           | Privileges                                                  | Signs?  | Immutable?                          |
| --------------- | ---------------- | ----------------------------------------------------------- | ------- | ----------------------------------- |
| `Circle.admin`  | `create_circle`  | `cancel_circle`, `propose_admin` (rotation step 1)          | Yes     | No — rotatable via two-step transfer |
| `fee_recipient` | `create_circle`  | Receives `fee_bps` of every `claim` payout; no auth power   | No      | Yes — sealed at creation (ADR 007)   |

Admin rights are **not** checked by `fund`, `claim`, `expire_round`, or `get_circle`
(`expire_round` is time-gated only). The admin's real blast radius is: cancel a circle
(refund all current-round contributors and close it permanently) and rotate the admin key.

## 1. `Circle.admin`

### Custody model: a Stellar multisig account

The contract authenticates the admin with `circle.admin.require_auth()` (`lib.rs:985`,
`lib.rs:1155`). For an `Address::Account`, `require_auth` is satisfied by the Stellar
account's signer set and threshold — so **native account multisig works with no contract
change**. Recommended:

- A **2-of-3** account: three signers on separate hardware wallets, held by three mutually
  distrusting parties. Anything that needs to happen (rotate, cancel) requires two of them.
- Choose threshold ≥ 2 so a single compromised key cannot act alone; note that the same
  threshold governs rotation, so choose one that is memorable but not single-point.

The admin key is **not** load-bearing for the pot: `fund`, `claim`, and `expire_round`
don't consult it. A lost admin key strands the circle's *control* (no cancel, no rotation),
not its funds. A compromised admin can close circles (griefing) but cannot move a pot to
itself — `claim` is proof-gated and `cancel_circle` refunds the same contributors it
closes. See [§5 loss scenarios](#5-loss-and-compromise-scenarios).

### Rotation runbook

Admin transfer is two-step and each step is signed by a different account:

1. **`propose_admin(circle_id, new_admin)`** — signed by the **current** admin
   (`lib.rs:982`). Seeds `DataKey::PendingAdmin(circle_id)`, overwritable by another
   `propose_admin` call before acceptance. Rejects on a cancelled circle.
2. **`accept_admin(circle_id)`** — signed by the **nominated** `new_admin`
   (`lib.rs:1014`). Atomically sets `Circle.admin`, clears the pending slot, and emits
   `AdminAccepted`.

The pending proposal has **no in-contract deadline**: it lives as long as its persistent
entry's TTL (extended to `LEDGER_EXTEND_TO` ≈ 500,000 ledgers ≈ 29 days at propose time),
and the current admin can overwrite it anytime before `accept_admin`. If the pending key is
ever archived by the storage-lifetime mechanism (`docs/adr/004-storage-archival.md`),
`accept_admin` reverts `CircleNotFound`; recovery is a short-lived "re-propose + accept".

There is **no SDK or UI wrapper** for `propose_admin`/`accept_admin` yet
([#485](https://github.com/crackedstudio/sharibo/issues/485)) — rotations go through the
CLI today:

```bash
# step 1 — current admin signs
stellar contract invoke --id <CONTRACT_ID> --network mainnet \
  --source admin-multisig-or-one-signer -- propose_admin \
  --circle_id <ID> --new_admin <G...G>

# step 2 — the nominated admin signs (any signer-count meeting the account threshold)
stellar contract invoke --id <CONTRACT_ID> --network mainnet \
  --source new-admin -- accept_admin --circle_id <ID>
```

Each step emits an event (`docs/events.md`: `AdminProposed`, `AdminAccepted`); watch for
them as part of monitoring ([#550](https://github.com/crackedstudio/sharibo/issues/550)).

## 2. `fee_recipient`

`fee_recipient` is **receive-only and immutable after `create_circle`** (ADR 007,
`docs/adr/007-protocol-fees.md`). It is paid `fee_bps` of each claim; it never signs, has
no auth power over the circle, and cannot be changed later.

Consequences for custody:

- **Choose it before `create_circle`.** Because it is sealed at creation, the address must
  be one you intend to keep for the circle's whole life. Prefer the same multisig
  treasury that controls the admin, so fee revenue lands in the same accountable place.
- **Losing its key does not endanger the pot.** Payouts to claim-winners are unaffected;
  the loss only strands *future fee revenue* at an unrecoverable address.
- **A compromised `fee_recipient` can only spend accrued fees**, not touch pots. This is a
  revenue-integrity risk, not a fund-safety one.

There is no on-chain remedy for either case (no setter, no rotation). Unlike the admin,
`fee_recipient` cannot be recovered by any operator action.

## 3. Pre-creation checklist

Before real funds flow into a circle:

- [ ] **Admin account is a Stellar multisig** with threshold ≥ 2, signers on separate
      hardware devices, key-record/secrets handled by the parties, and a testnet
      dry-run of the `propose_admin` → `accept_admin` flow completed.
- [ ] **`fee_recipient` chosen and recorded** (immutable, so it *must* be decided first)
      — same treasury/multisig, not an ad-hoc address.
- [ ] `fee_bps` decided; the fee is visible on-chain up front (ADR 007).
- [ ] Admin + `fee_recipient` published in the deploy record matching
      [`docs/deployments.md`](deployments.md), and confirmed off-chain with the held keys.
- [ ] Monitoring covers admin events (`AdminProposed`/`AdminAccepted`) —
      tracked by [#550](https://github.com/crackedstudio/sharibo/issues/550).

## 4. Rotation policy

- **Rotate on personnel change or suspected exposure** of any signer, mirroring the 2-of-3
  signer set. Two-step transfer keeps the new key unusable until it signs `accept_admin`.
- **No rotation for `fee_recipient`** — a change costs a cancel + recreate, so the policy
  is "get the address right at creation."
- **Cancelled circles cannot be rotated** (`propose_admin`/`accept_admin` revert on a
  cancelled circle), consistent with the circle being permanently closed.

## 5. Loss and compromise scenarios

These scenarios feed directly into the incident-response plan
([#551](https://github.com/crackedstudio/sharibo/issues/551)).

| Scenario               | On-chain consequence                                                                 | Remedy / response                                                                                     |
| ---------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| Admin key lost         | Circle cannot be cancelled or rotated; funds are not at risk                          | Threshold 2-of-3 + documented holder of each signer; rotation impossible on-chain, respond at org level |
| Admin compromised      | Attacker can cancel circles (refund + close) and rotate admin to itself              | Watch `AdminProposed`/`AdminAccepted`; rotate promptly on suspicion; incident response (#551)          |
| Pending transfer lost  | `PendingAdmin` entry archived → `accept_admin` reverts `CircleNotFound`              | Current admin re-proposes (overwrites) and new admin accepts                                         |
| `fee_recipient` lost   | Future fees stranded at an unrecoverable address; pot unaffected                     | No on-chain fix (immutable); mitigate by choosing it correctly in §3                                  |
| `fee_recipient` compromised | Attacker spends accrued fee revenue; pots unaffected                            | Monitor fee-account balance; no rotation, escalate via #551                                           |

## References

- [`docs/adr/001-upgradeability.md`](adr/001-upgradeability.md) — immutability decision; this doc supersedes the "rotation deferred" framing.
- [`docs/adr/007-protocol-fees.md`](adr/007-protocol-fees.md) — fee `fee_bps`/`fee_recipient`, immutability.
- [`docs/adr/004-storage-archival.md`](adr/004-storage-archival.md) — persistent-entry TTL/archival that bounds pending transfers.
- [`docs/events.md`](events.md) — `AdminProposed` / `AdminAccepted` event schemas.
- [`docs/mainnet-readiness.md`](mainnet-readiness.md) — §3 where this item is tracked.
- Incident-response plan: [#551](https://github.com/crackedstudio/sharibo/issues/551).
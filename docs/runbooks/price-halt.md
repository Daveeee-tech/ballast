# Runbook: handling a PriceGuard halt

Audience: the on-call risk operator. This is the checklist for a `Halted` guarded price
(PRD §6.4: "no liquidations until a human confirms"). The one write in this runbook is
`confirm_halt_cleared`; everything before it is read-only.

A halt is a **sticky latch**. While it is set, `guarded_price` returns `Halted` and keeps
returning the last good value regardless of a fresh read. Draws, exits, repos and
liquidations are blocked; repayment, release and withdrawal are not. Only the `risk` role can
clear it, and clearing it asserts that the large move was real, not manipulation.

> [!WARNING]
> `confirm_halt_cleared` promotes the value that **tripped** the halt
> (`PendingHaltValue`) to the new baseline, not the price at the moment you confirm. If the
> spike has reverted since the halt, confirming sets the baseline to the spike. The next
> ordinary price then looks like a large drop against that baseline and can halt again, and
> collateral is valued at the spike in the meantime. Always re-read the values immediately
> before confirming (step 4). This is a known behaviour of the current contract, not a
> documentation mistake; it has no public issue number yet (tracked in the repository's
> backlog).

---

## 1. Detect the halt

The `price-guard` service and `services/indexer` alert on `status_changed` to `Halted` and on
the `halt_latched` event. Confirm it on-chain rather than trusting the alert alone. Reads are
simulations: they persist nothing and are safe to repeat.

```bash
source .env.testnet-deploy        # PRICE_GUARD_CID, RWA_SAC, RISK
export NETWORK=testnet

stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- guarded_price --asset "$RWA_SAC"

stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- halt_info --asset "$RWA_SAC"
```

`guarded_price` returns `{ value, status, ts, sources }`. `halt_info` returns `None` when the
asset is not halted, otherwise `{ last_good, pending }`:

- `last_good` is the baseline `guarded_price` is holding while halted.
- `pending` is the median that tripped the halt, and the value `confirm_halt_cleared` will
  promote to `LastGood`.

Also read the configured thresholds, so you know what "too large" means for this asset:

```bash
stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- asset_config --asset "$RWA_SAC"
```

## 2. Check the move against independent sources

The guard's median is built from pushed readings: the issuer NAV (`issuer`), an independent
SEP-40 feed such as RedStone, and the last observed redemption price. Verify the move against
each of them **outside** the contract before doing anything else:

| Check | Where | What you are looking for |
|---|---|---|
| Issuer NAV | Issuer's NAV page or verified endpoint; `nav_published` event for the same `ts`/value | The signed NAV moved by a similar amount, and the reading is inside `max_staleness_s` |
| Independent feed | RedStone (or the configured SEP-40 feed) dashboard and its `source_published` event | The feed moved too, and agrees with the NAV within `band_bps` |
| Redemption price | Issuer redemption records, or the `redemp` source the indexer publishes | Actual redemptions cleared at or near the new value |
| DEX / RFQ price | Classic DEX, Octarine | **Monitoring only.** A move here that the other three do not share is a manipulation signal, not a reason to confirm |

Useful read-only follow-ups:

```bash
# Current baseline, independent of the latch.
stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- last_good --asset "$RWA_SAC"

# Pending configuration changes (e.g. a band change) already in the 48h timelock.
stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- pending_config --id <CONFIG_ID>
```

Two common false alarms are documented in [`TROUBLESHOOTING.md`](../../TROUBLESHOOTING.md):
the RedStone feed updates every 1–2 hours, and a long publication outage makes an ordinary
move look like a jump on the first publish afterwards.

## 3. Decide — do not confirm unless every line is true

- [ ] `pending` is within `band_bps` of **two** independent, fresh sources (not just one).
- [ ] The issuing source is the issuer's verified NAV page/endpoint or a reading you have
      verified the signature on — not a screenshot or a forwarded number.
- [ ] The redemption price is consistent with the new value, or redemptions are explicitly
      suspended with a documented reason.
- [ ] The size and timing of the move are explained by a real event (rate decision, NAV
      publication, credit event), not by a feed outage or a single-source jump.
- [ ] There is no sign of publisher or issuer-key compromise: check who published the
      reading (`nav_published` / `source_published` events) and that the publishing account
      is the expected `price_publisher`.
- [ ] You re-read `halt_info` and `guarded_price` **now**, and `pending` still matches the
      current median (see the warning above).

If any line is false, **do not confirm**. Keep the halt in place (that is the safe state),
record what failed, and escalate (step 6).

## 4. Confirm the halt

Both `risk` and `asset` are explicit arguments; `--source risk` is the identity that signs.
Replace the values with the ones from `.env.<network>-deploy`.

```bash
export NETWORK=testnet
source ".env.${NETWORK}-deploy"

# Final check: halt_info.pending must equal the current guarded_price.value (the median).
stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- halt_info --asset "$RWA_SAC"

stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- confirm_halt_cleared --risk "$RISK" --asset "$RWA_SAC"
```

What the call changes:

1. `LastGood` becomes `pending` (with the current ledger timestamp).
2. `RollingReference` is reset to `pending`, so the slow-drift check starts from the new
   level instead of the old one.
3. `PendingHaltValue` is deleted and the `Halted` latch is removed.
4. A `halt_cleared` event is emitted with the new `LastGood`.

It does **not** publish a price and does not change the current status by itself. The next
publish (or `guarded_price` call) re-evaluates the sources; if they still agree within the
bands, the status becomes `Ok` and collateral is valued at the new baseline. Expect that
transition to appear as `status_changed (Halted → Ok)` in the indexer.

## 5. Verify

```bash
stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- halt_info --asset "$RWA_SAC"          # must return null/None

stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- guarded_price --asset "$RWA_SAC"       # expect status Ok, value == the promoted pending

stellar contract invoke \
  --id "$PRICE_GUARD_CID" --network "$NETWORK" --source risk \
  -- last_good --asset "$RWA_SAC"           # must equal the promoted pending
```

Then check that the monitoring alert has cleared, and note the new baseline in the incident
record so the next on-call knows why the value moved.

## 6. Escalate

Escalate instead of confirming when:

- the sources disagree, any of them is stale, or only the DEX price moved;
- the pending value keeps changing between reads (a live/rolling spike);
- you suspect the publisher or issuer key is compromised — **do not confirm**; involve the
  admin. A publisher rotation is the 48h `queue_role_change` path plus `configure_issuer_key`
  (see [`contracts/README.md`](../../contracts/README.md)); the halt stays in place while that
  runs;
- a liquidation is already overdue: it remains blocked until the halt is cleared, so the
  lender and the keeper job need to know the timeline;
- the move is real but you believe the asset's bands are wrong. Bands can only be **tightened**
  immediately; loosening one goes through the 48h `queue_config` timelock, so do not widen a
  band just to make an alert go away.

Record in the incident: the asset, the `halt_latched` transaction/ledger, `last_good`,
`pending`, the independent values you checked with links/timestamps, the decision, the
`confirm_halt_cleared` transaction hash if you called it, and the resulting baseline.

---

## References

- PRD §6.4 (price-guard rules) and §11 (monitoring alerts).
- [`contracts/price-guard/src/lib.rs`](../../contracts/price-guard/src/lib.rs) —
  `guarded_price`, `halt_info`, `last_good`, `confirm_halt_cleared`.
- [`contracts/price-guard/README.md`](../../contracts/price-guard/README.md) — events and the
  halt cycle.
- [`contracts/AUDIT_NOTES.md`](../../contracts/AUDIT_NOTES.md) — what `LastGood` carries and
  what the guard assumes.

## Verification status

The commands above follow the contract entry points and the existing `stellar contract invoke`
patterns in `scripts/` and `TROUBLESHOOTING.md`, but they were **not executed against testnet
while writing this runbook** (no deployed identities or network access in the authoring
environment). Treat them as untested until an on-call has exercised them on testnet, and
update this note when that happens.

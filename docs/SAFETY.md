# Capital safety and execution controls

This safety document explains how Concordia authorizes, validates, reconciles,
or refuses real-money execution within independently owned member brokerage
accounts. It defines the platform's independent execution gates, deterministic
order identity, broker-side veto, stale-session protection, partial-data
handling, and consequence-based failure behavior. The model assumes scheduled
jobs may execute more than once, upstream feeds may be incomplete, sessions
may remain in intermediate states, and configuration errors will eventually
occur.

## Three authorization conditions and two runtime guards

Real order placement requires three conditions to hold simultaneously:
`EXECUTION_MODE=live` in configuration, an explicit runtime `ARMED` state set
by a human in the administration console, and an invocation that has not been
forced into dry-run mode. If any condition is absent, the executor degrades to
review-only operation: it calculates quantities, submits pre-trade reviews,
and records the proposed actions without placing orders.

Two additional guards can downgrade an otherwise authorized run. A human
quorum floor prevents research-agent votes from creating a real basket when
human participation falls below the configured minimum. A calendar-coverage
check blocks execution when the maintained market-holiday table no longer
covers the current year, because uncertainty about whether a date is tradable
must result in no action rather than an inferred answer.

## Deterministic order identity

Every order's ref-id is a deterministic UUID5 over
`(session_id, member_id, symbol)`:

```python
# backend/app/services/executor.py
_REF_NS = uuid.UUID("c0c0c0c0-0000-4000-8000-000000000000")

def ref_id_for(session_id: int, member_id: int, symbol: str) -> str:
    return str(uuid.uuid5(_REF_NS, f"{session_id}:{member_id}:{symbol}"))
```

If execution stops mid-run and restarts, the retry generates the same reference
identifiers, allowing both the broker and local order ledger to recognize the
duplicate. Before every execution, a reconciliation pass resolves ambiguous
local states against the broker's order feed as the system of record. An order
that failed locally but filled at the broker is marked as placed rather than
retried; estimated quantities are replaced with confirmed fills; and rejected
broker orders are demoted from executable state.

## Broker validation is authoritative

Every buy and sell, in both live and dry-run modes, is submitted to Robinhood's
pre-trade review before placement. A blocking alert terminates the order and
cannot be automatically retried or overridden. The broker has the most current
view of account restrictions, day-trading limits, and market halts, so its
rejection is treated as authoritative.

## Stale-basket refusal

The buy phase runs one trading day after the sell phase to satisfy T+1 cash
settlement and must therefore locate the session it is completing. This design
produced a material near miss: session 14, dated 2026-07-20, remained `closed`
after three order errors prevented transition to `executed`. Weeks later, it
was still the only closed session while production was live and armed. A
subsequent buy-phase invocation could have repurchased a basket the club had
exited six cycles earlier. Concordia now applies a hard age-based refusal:

```python
# backend/app/services/executor.py — buy phase
# A stale basket is never worth buying, so this refuses instead of
# trading and says so loudly. STALE_BASKET_DAYS spans a long weekend
# plus a holiday; anything older is a stuck session to be investigated,
# not an instruction.
age_days = _session_age_days(session)
if age_days is not None and age_days > STALE_BASKET_DAYS:
    note = (f"refusing to buy a stale basket: session {session.get('id')} "
            f"({session.get('trade_date')}) is {age_days} days old and still "
            f"'closed' — investigate that session; it is not today's cycle")
    audit("system", "buy_refused_stale_basket", str(session.get("id")),
          {"trade_date": session.get("trade_date"), "age_days": age_days})
    return {"ok": True, "note": note, "orders": [], "stale_session": session.get("id")}
```

`STALE_BASKET_DAYS` is set to five calendar days, covering a long weekend and
market holiday while rejecting any session old enough to require manual
investigation.

## Preserve participation; fail closed for capital

Failure behavior is selected according to the consequence of uncertainty:

- **Broker outages do not block voting.** Ballots are database-only operations
  with no Robinhood dependency, so members retain access to governance when
  the brokerage is unavailable.
- **Incomplete position data cannot size an order.** The executor derives the
  club's holdings in each member account from the broker order feed. If a
  failed page or invalid cursor makes that derivation incomplete, the member
  is skipped for the current cycle. Treating an incomplete feed as an empty
  position set could omit required sells or repurchase the entire basket:

  ```python
  # backend/app/services/executor.py
  try:
      held = positions.club_shares_derived(mid, client=client, account_number=m["account_number"])
  except positions.DerivationIncomplete as e:
      _skip_member(msum, "transient_rh_error",
                   f"club position feed incomplete — skipping this run: {e}")
      continue
  ```

- **Failed balance reads create gaps rather than false zeros.** The ten-minute
  account recorder rejects `None`, NaN, negative, and implausible observations
  before persistence. A missing point is recoverable; a synthetic zero would
  report a market loss that never occurred. A valid $0.00 balance remains
  acceptable because failed reads return `None`, not numeric zero.

Across these controls, uncertainty reduces system authority, generates an
explicit audit event, and preserves evidence for human review. The principal
regression coverage resides in `test_stale_basket_guard.py` with five refusal
tests, `test_executor_fixes.py` with 28 audited execution edge cases, and the
recorder cases in `test_equity_history.py`, including
`test_failed_broker_read_records_nothing` and
`test_none_balance_records_nothing`.

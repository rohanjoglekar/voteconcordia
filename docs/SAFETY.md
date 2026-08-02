# Money safety

This system places real market orders in other people's brokerage accounts.
That sentence is the whole design constraint. Everything below exists because
I assumed from day one that timers double-fire, feeds return partial data,
sessions get stuck, and I make mistakes at 9am.

## Three switches, and two guards on top

A real order requires three independent conditions to hold at once:
`EXECUTION_MODE=live` in config, the runtime ARMED flag (a human-flipped
switch in the admin console), and the absence of a forced dry-run on the
invocation. Any one of them off and the run degrades to review-only: it still
sizes every order and runs every pre-trade review, it just records instead of
placing. Two more guards can knock a live run back down after the fact: a
human-quorum floor (the research bots vote, but fewer than the minimum number
of *human* voters means no real orders — bots can't auto-buy a basket on a
zero-turnout day), and a calendar-coverage check (if my hand-maintained market
holiday table has lapsed past its last covered year, the system can't know
today is a trading day, so it refuses to trade rather than guess).

## Idempotent by construction

Every order's ref-id is a deterministic UUID5 over
`(session_id, member_id, symbol)`:

```python
# backend/app/services/executor.py
_REF_NS = uuid.UUID("c0c0c0c0-0000-4000-8000-000000000000")

def ref_id_for(session_id: int, member_id: int, symbol: str) -> str:
    return str(uuid.uuid5(_REF_NS, f"{session_id}:{member_id}:{symbol}"))
```

If a run crashes halfway and restarts, the retry produces the same ref-ids, so
the broker and my own placed-order records both recognize the duplicate. A
reconcile pass runs before each execution and settles the ambiguous cases
against the broker's order feed as ground truth: an order that errored on my
side but actually filled at the broker gets promoted to placed (not silently
re-attempted), estimated quantities get rewritten to real fills, and
broker-rejected rows get demoted.

## The broker gets a veto

Every order — sell and buy, live or dry-run — goes through Robinhood's own
pre-trade review before placement, and a blocking alert kills that order
rather than being retried or overridden. I want the layer with the freshest
view of the account (day-trade counters, restrictions, halts) to be able to
say no.

## The stale-basket refusal

The buy leg runs the day after the sell leg (T+1 settlement in a cash
account), so it has to *find* the session it is finishing. That created a real
near-miss: session 14 (2026-07-20) never flipped to `executed` because three
orders errored, and weeks later it was still the only `closed` session in
production — while the club was live and armed. Any buy-phase run would have
re-bought a basket the club had deliberately exited six cycles earlier. The
fix is a refusal, not a heuristic:

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
    audit("system", "buy_refused_stale_basket", str(session.get("id")), ...)
    return {"ok": True, "note": note, "orders": [], "stale_session": session.get("id")}
```

`STALE_BASKET_DAYS` is 5 — long enough for a long weekend plus a holiday,
short enough that anything older is by definition a bug to investigate.

## Fail open for members, fail closed for money

This is the rule I apply whenever a dependency misbehaves, and the direction
flips depending on what's at stake:

- **A broker outage never blocks voting.** Ballots are a pure database
  operation; nothing in the voting path touches Robinhood. Members can always
  vote, even when the brokerage is down.
- **An incomplete position feed never sizes an order.** The executor derives
  what the club holds in a member's account from the broker's order feed. If
  that derivation is incomplete — a failed page, a runaway cursor — the
  executor skips the member for this cycle and retries the next, because
  falling through to an empty holdings set would strand club sells or re-buy
  the entire basket:

  ```python
  # backend/app/services/executor.py
  try:
      held = positions.club_shares_derived(mid, client=client, account_number=m["account_number"])
  except positions.DerivationIncomplete as e:
      _skip_member(msum, "transient_rh_error",
                   f"club position feed incomplete — skipping this run: {e}")
      continue
  ```

- **A failed balance read records nothing, not $0.** The 10-minute account
  recorder validates every observation before writing; `None`, NaN, negative,
  and absurd values — exactly the shapes a failed or partial broker read
  produces — are refused. A gap in the chart is honest; a false $0 would paint
  a crash that never happened. ($0.00 itself is accepted: an empty account
  really is zero, and a failed read reports `None`, never 0.)

The common thread: when the system is unsure, it does less, says why in the
audit log, and leaves a trail a human can act on. Of the 219 backend tests, a
large share exist to pin these behaviors down — including a dedicated suite
for the stale-basket guard, one for executor edge cases found in audits, and
one asserting the recorder's refusal shapes.

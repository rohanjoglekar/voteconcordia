# Concordia — a trading club that votes

**Try it live: [demo.voteconcordia.com](https://demo.voteconcordia.com) — invite code `demo`**

Concordia is a real-money investment club where the portfolio is decided by
vote. Every trading cycle, members allocate their voting power across S&P 500
names; the top picks become the club's basket, and that basket is executed
automatically — **in each member's own Robinhood account**. The club never
pools money, never takes custody, and cannot move funds in or out of anyone's
account.

The real club has been live and trading since July 2026. The demo above is the
full product against a simulated brokerage: everything works — voting, the
basket, research, history, the account dashboard — except orders are fake and
no money moves. Sign-ups on the demo are open to everyone.

---

## How it works

1. **Vote** — each member splits 100% of their voting power across the stocks
   they want the club to hold. Voting power is earned by performance, not
   bought.
2. **Decide** — at the daily close, votes are tallied into one consensus
   basket (top ~20 names, conviction-weighted).
3. **Execute** — the basket is rebalanced into every participating member's
   own brokerage account: names leaving the basket are sold, entrants bought,
   holdovers left untouched (no churn). Sells and buys run on consecutive
   trading days to respect T+1 settlement — a cash account, no margin.
4. **Repeat** — performance is tracked against an S&P 500 buy-and-hold
   baseline, club-wide and per member.

## What's in the product

- **Guided onboarding** — OAuth connect to the member's own brokerage,
  suitability check, explicit stake commitment ("only this slice ever trades").
- **Voting floor** — integrated stock search, weighted ballot with live
  consensus, one-tap research on any name.
- **Research** — per-symbol pages: real intraday/daily charts (drawn from
  recorded observations only — the chart never invents a price), fundamentals,
  52-week context, add-to-ballot.
- **History** — the fund's shared cycle-by-cycle record (baskets, turnout,
  order outcomes), separate from each member's private ballot history.
- **Account** — recorded balance history charted Robinhood-style, holdings,
  profile management, one-switch participation pause, delete-anytime.
- **Admin console** — operator view: arm/disarm switch, cycle state, member
  health, morning digest emails.

## Engineering notes (what I'd want to know if I were reading this)

- **Safety-first executor.** Real orders require three independent switches
  (live mode ∧ armed ∧ not-dry-run), every order passes the broker's own
  pre-trade review, order placement is idempotent by construction
  (deterministic ref-ids), and the buy leg refuses stale baskets outright.
  Trading data is never mutated for display purposes — presentation states are
  derived.
- **Fail-open where it protects members, fail-closed where it protects
  money.** A broker outage can never lock a member out of voting; an
  incomplete position feed can never size an order.
- **Honest charts.** Account history is recorded (10-minute cadence), never
  interpolated; gaps are filled only by club-performance-shaped estimates
  pinned to real observations, flagged as estimates in the payload.
- **The demo is the real codebase** — same app, same executor paths, against
  a deterministic simulated brokerage in an isolated instance with its own
  database and keys. The club record shown on the demo is the real club's,
  mirrored read-only through a file boundary: the public demo process holds no
  credentials for, and no path to, the production database.
- Stack: FastAPI + SQLite (WAL) · Next.js 14 · systemd timers on a single
  box · Robinhood via MCP with OAuth per member · 218 backend tests.

## Status

The production club is private and invite-only (a handful of humans plus a
fleet of research bots that vote but hold no power over real accounts). The
demo is public and resets nothing — your demo account persists.

*Contact: rohannj29@gmail.com*

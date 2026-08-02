# Concordia — a trading club that votes

**Live demo: [demo.voteconcordia.com](https://demo.voteconcordia.com) — invite code `demo`**

Concordia is a real-money investment club where the portfolio is decided by
vote. Each trading cycle, members split 100% of their voting power across S&P
500 names; the top picks become the club's basket, and that basket is executed
automatically in each member's own Robinhood account. The club never pools
money, never takes custody, and has no way to move funds in or out of anyone's
account.

The club has been live and trading real money since July 2026 — private
members trading in their own accounts, alongside research bots that vote but
control no money. The demo above runs the same codebase against a simulated
brokerage: voting, research, the basket, history, and the account dashboard
all work; only the orders are fake. Demo sign-ups are open. Designed, built,
and operated by one person: backend, frontend, execution engine, and the ops
underneath.

## How a cycle works

1. **Vote.** Members allocate voting power across the stocks they want the
   club to hold. Power is earned by performance, not bought.
2. **Decide.** At the close, votes tally into one consensus basket — top ~20
   names, conviction-weighted.
3. **Execute.** The basket is rebalanced into every participating member's own
   account. Names leaving the basket are sold, entrants bought, holdovers left
   alone. Sells and buys run on consecutive trading days so a cash account can
   settle (T+1); no margin anywhere.
4. **Repeat.** Performance is tracked against an S&P 500 buy-and-hold
   baseline, club-wide and per member.

## Where to look

- **[docs/SAFETY.md](docs/SAFETY.md)** — how the system decides when it is
  allowed to spend real money, and the near-miss where a stuck session almost
  re-bought a basket the club had deliberately exited six cycles earlier.
- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — two instances of one
  codebase on one box, and the file boundary that lets the public demo show
  the real club's track record without holding a single credential for the
  production database.
- **[docs/DESIGN.md](docs/DESIGN.md)** — product decisions and why. The
  firmest one: the charts never draw a price that didn't happen. Measured
  against real payloads, the Catmull-Rom smoothing most charts use overshot
  the true high/low on 13 of 15 symbol/range pairs, so it's gone.

The documents quote short excerpts from the private repo. The full source
stays private because it runs a live club with real accounts attached.

## Stack

FastAPI + SQLite (WAL) · Next.js 14 · systemd timers on a single box ·
Robinhood via MCP, OAuth per member · 219 backend tests.

## Status

Production is invite-only and stays that way for now. The demo is public and
persistent — your demo account keeps its history between visits.

Contact: rohannj29@gmail.com

The code excerpts in `docs/` are published for review; all rights reserved.


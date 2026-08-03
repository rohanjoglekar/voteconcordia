# Concordia — vote-governed portfolio execution

**Live demonstration: [demo.voteconcordia.com](https://demo.voteconcordia.com) — invite code `demo`**

Concordia is a production-deployed, real-money investment club in which portfolio construction is governed by member vote. During each trading cycle, participants allocate 100% of their voting power across an S&P 500 universe. The highest-conviction selections form a consensus basket, which Concordia automatically rebalances inside each participant's independently owned Robinhood account. Capital is never pooled, the club never assumes custody, and the platform has no mechanism to deposit, withdraw, or transfer member funds.

The club has operated with real capital since July 2026. Private members participate through their own accounts alongside research agents that may vote but cannot control funds. The public demonstration runs the same application against an isolated, deterministic brokerage simulator: onboarding, voting, research, basket formation, performance history, and account reporting are fully functional, while every order remains simulated. Public demonstration registrations are open.

## Trading cycle

1. **Allocate.** Members distribute voting power across the securities they want the club to hold. Voting power is earned through measured performance; it cannot be purchased.
2. **Form consensus.** At the close of the ballot, Concordia aggregates the votes into an approximately 20-security, conviction-weighted basket.
3. **Rebalance.** The executor independently aligns each participating account with the consensus mandate. Securities leaving the basket are sold, new constituents are purchased, overweight positions are trimmed, and correctly weighted holdovers remain untouched. Sell and buy legs run on consecutive trading days to respect T+1 settlement in cash accounts; the system does not use margin.
4. **Measure.** Club-wide and member-level performance is evaluated against an S&P 500 buy-and-hold benchmark before the process repeats.

## Technical documentation

- **[Safety](docs/SAFETY.md)** — Defines the independent conditions required before the system may place a real order, together with the stale-session guard introduced after a near miss could have repurchased a basket the club had exited six cycles earlier.
- **[Architecture](docs/ARCHITECTURE.md)** — Describes two isolated instances of one codebase on a single server and the one-way file boundary that exposes the live club's aggregate track record to the public demonstration without granting access to production credentials or member data.
- **[Design](docs/DESIGN.md)** — Records the product and data-integrity decisions behind the system, including the removal of Catmull–Rom chart smoothing after measured renderings exceeded the true price range in 13 of 15 symbol-and-window combinations.

These documents include focused excerpts from the private production repository. The complete source remains private because the platform executes real orders through connected member accounts.

## Technology

FastAPI; SQLite in WAL mode; Next.js 14; systemd services and timers on a single Linux server; per-member Robinhood OAuth through MCP; and 219 backend tests.

## Operating status

Production membership remains invitation-only. The public demonstration is persistent, and demonstration accounts retain their voting and performance history between visits.

Contact: rohannj29@gmail.com

Code excerpts in `docs/` are published for technical review; all rights reserved.

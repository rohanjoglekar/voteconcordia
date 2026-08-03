# Design — capital discipline and representational integrity

This design document explains how Concordia translates its capital-safety and
data-integrity requirements into portfolio execution, visual reporting,
onboarding, public demonstration, and interface architecture. It records why
the platform rebalances only changed positions, distinguishes observations
from estimates, limits trading to explicitly committed capital, ships one
frontend across three isolated hosts, and prevents presentation state from
overwriting source trading records.

## Collective intelligence is a testable product thesis

Concordia is designed around a disciplined form of the wisdom of crowds: a
group can combine information held by different participants, but only if the
system preserves diverse inputs, applies a transparent aggregation rule, and
measures outcomes consistently. A raw majority vote does not satisfy those
conditions by itself; popularity can amplify confidence without improving
accuracy.

The product therefore treats influence as accountable state rather than social
status. Voting power blends committed capital with a smoothed measure of each
participant's benchmark-relative historical accuracy. Capital keeps influence
connected to economic commitment, while the accuracy component allows a
participant with repeated signal to earn more weight over time. The accuracy
term phases in as observations accumulate so one early success cannot dominate
the club.

This is a hypothesis under continuous measurement, not a promise of
outperformance. The interface exposes the basket, benchmark, participation,
and per-cycle history so the group process can be evaluated on evidence rather
than justified after the fact. Research agents enter through the same tallying
model under bounded aggregate influence, but they hold no execution authority
or brokerage credentials.

## Rebalance only what changed

A full-liquidation executor would sell and repurchase the entire basket during
every cycle. Although mechanically simple, that approach would generate
unnecessary turnover, multiply spread costs, and create taxable transactions
for holdings whose investment mandate had not changed. Concordia instead
computes a portfolio difference: departing constituents are sold, overweight
positions are reduced, entrants and underweights are funded, and a holdover at
target weight generates no order.

Sell and buy phases run on consecutive trading days because members use cash
accounts and sale proceeds require T+1 settlement. This separation introduces
the stale-session risk documented in [SAFETY.md](SAFETY.md), which is mitigated
through a hard age-based refusal rather than by reverting to unnecessary
liquidation.

## Charts may not exceed observed data

Three visualization rules protect the distinction between measurement,
estimation, and presentation.

**No smoothing.** The original price charts used Catmull–Rom splines, a common
choice in consumer finance interfaces. Measurement showed that the
interpolation created prices outside the source data. A synthetic series
bounded from 0 to 100 produced control-point values as high as 116.7. Across
real payloads, the rendered curve exceeded the observed high or low in 13 of
15 symbol-and-range combinations. XOM's three-month range of [131.08, 148.67]
rendered as [130.08, 149.24], displaying a high $0.57 above any traded value
while crossing guide lines anchored to the actual extrema. The implementation
is now governed by the following constraint:

```tsx
// web/components/PriceChart.tsx
/** Straight segments through every close — the only shape that cannot lie.
 *  ...
 *  A price chart must never draw a price that did not happen, so there is no
 *  smoothing at all — the vertices ARE the closes. (Robinhood and Yahoo draw
 *  it this way for the same reason.) If softening is ever wanted, it must be
 *  monotone/shape-preserving, never Catmull-Rom. */
```

**Observed and estimated values remain distinguishable.** The account-value
chart is based on real observations recorded every ten minutes; a failed read
creates a gap rather than a synthetic point. Because the recorder was
introduced after the club began operating, early history is sparse. Estimates
may bridge those gaps using the club's performance index and are anchored to
real observations at both ends, but every estimated point is identified in the
payload and rendered distinctly. Estimates never replace recorded values and
retire as direct observations accumulate. The one-day range was removed until
sufficient dense history existed rather than presenting a chart based on only
two measured points.

**Labels identify the actual period.** When a security page displays the prior
session's range before the current market has traded, the interface labels it
“Last session” rather than “Day range.” Presentation language follows the
observation period rather than the layout in which the value appears.

## The public demonstration executes the production codebase

The public application at demo.voteconcordia.com runs the production software
against a deterministic brokerage simulator in an isolated instance with
independent database and cryptographic keys. Visitors can complete onboarding,
vote, inspect research, and observe execution through the same product used by
the private club. Because a simulated brokerage cannot provide a credible
historical record, aggregate production performance crosses a one-way file
boundary described in [ARCHITECTURE.md](ARCHITECTURE.md). Fabricated fixtures
and direct read-only production database access were both rejected.

## The onboarding wizard

Onboarding uses explicit stages for club rules, suitability, capital
commitment, and brokerage OAuth authorization. The capital stage defines the
maximum portion of an account delegated to Concordia. Rebalance budgets are
calculated from that committed amount rather than available buying power,
preventing the club allocation from expanding into undelegated capital as an
account grows. Members may pause participation through a single control and
may delete their account without administrator intervention.

## One build, three hosts

The member application, administration console, and demonstration ship as one
Next.js build, with the hostname selecting the appropriate shell. This keeps
the operational and member interfaces on the same release and establishes
that the demonstration uses the same frontend. Browser storage is isolated by
origin, preventing administration tokens, member sessions, and demonstration
credentials from colliding. The administration host selects a role-gated shell
before rendering, while the demonstration applies explicit simulated-capital
branding throughout the interface.

## Derived presentation state cannot overwrite trading records

Interface state is derived from immutable trading records and is never written
back over source events. Order rows retain their original review, placement,
and error payloads; cycle status, member health, and performance are calculated
at read time. The sole exception is reconciliation of order quantity, where an
estimate may be replaced only by the broker's confirmed fill.

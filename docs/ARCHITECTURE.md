# Architecture

This document maps how Concordia is put together — backend, frontend,
brokerage integration, and the boundary between the live club and the public
demo — and records why each piece is shaped the way it is. Concordia is
deliberately small: one box, two instances of one codebase, no queue, no
Kubernetes, no microservices. A club of this size doesn't need horizontal
scale; it needs to be auditable by one person at 7am when a timer misfires,
so every piece below was chosen to keep the whole system holdable in one
head.

## The shape of it

**Backend** is FastAPI over SQLite in WAL mode, running under systemd as a
dedicated unix user. The database, the encrypted broker-token vault, and the
`.env` are owned by that user and readable by nothing else on the box. There
is no job framework: cycle mechanics (open the ballot, close and
tally, execute sells, execute buys, refresh account snapshots every 10
minutes, mirror the club record, back up the DB) are each a small `jobs/*.py`
entrypoint fired by its own systemd timer. Fourteen timers, each one
inspectable with `systemctl status` and each one restartable in isolation.

**Frontend** is a single Next.js 14 build that serves three hosts. The
hostname decides what shell you get — the member site, the admin console, or
the demo — from one deployed artifact:

```ts
// web/lib/host.ts
/** demo.voteconcordia.com — the public fake-money instance. */
export function isDemoHost(): boolean {
  if (typeof window === "undefined") return false;
  return window.location.hostname.startsWith("demo.");
}
```

An `isAdminHost` twin does the same for `admin.`, which selects the ops
console shell.

**Brokerage** is Robinhood's agentic trading MCP. Each member OAuths their own
account during onboarding; tokens land in the encrypted vault and are only
ever decrypted server-side. Members never share credentials with the club or with
each other, and the club has no account of its own; every order is placed in
the member's account, by the member's token, against the member's committed
stake.

## Two instances, one codebase

The live club and the public demo are the same application. The demo is not a
mock frontend or a video; it runs the real executor, the real voting engine,
the real onboarding wizard. What differs is one flag: in `DEMO_MODE` the vault
hands back a `DemoClient` instead of a Robinhood client — deterministic canned
data (prices derive from the symbol name, so refreshes are stable), every
pre-trade review passes, and nothing external is ever called. `is_live` is
forced false while the flag is on, so a demo box cannot place a real order
even by misconfiguration.

Each instance has its own database, its own session keys, its own systemd
units. They share a box and a git checkout and nothing else.

## The mirror boundary

The demo has its own empty database, which left its club pages — the
cycle-by-cycle history, the decisions-vs-SPY chart — with nothing to show.
Seeding fake fixtures would have invented a track record, which is worse than
showing nothing. So the demo shows the real club's record, and the interesting
part is how that data crosses over.

The obvious design — pointing the demo backend at the live database
read-only — was rejected, because the demo is a public sign-up, and that
design puts it one bug away from members' rows and encrypted broker tokens.
Instead, a job on the *production* side periodically writes a single JSON file
containing only club-level payloads, and the demo reads that file. The demo
process holds no credentials for, and no path to, the production database. If
the demo instance were fully compromised, the attacker would have a JSON file
of data the demo was already serving publicly.

The job also refuses to leak by accident. The club-history payload is
identical bytes for every member (a test asserts that), and a deny-list check
rejects the whole mirror if any member-scoped key ever shows up in a row:

```python
# backend/jobs/mirror_club.py
_FORBIDDEN = {"email", "display_name", "member_id", "members", "name", "votes_by",
              "holdings", "account_number", "token", "password_hash"}

def _assert_clean(rows: list[dict]) -> None:
    for row in rows:
        bad = _FORBIDDEN.intersection(row.keys())
        if bad:
            raise SystemExit(f"refusing to mirror: club history row carries member-scoped keys {sorted(bad)}")

def build(limit: int = 60) -> dict:
    if settings.demo_mode:
        raise SystemExit("refusing to run on a demo instance — this mirrors FROM the real club")
    history = voting.club_history(limit)
    _assert_clean(history)
    # ...
```

The write is an atomic `os.replace`, so the demo can never read a half-written
file. On the demo side the reader fails open to local data: a missing or
malformed mirror means the demo shows its own empty club rather than erroring.

## The executor in one paragraph

Execution is a per-member loop: read the member's live balance and the club's
position in their account, compute a rebalance plan against the consensus
basket (sell what left, trim overweights, top up entrants, leave holdovers
untouched), then run every order through the broker's pre-trade review before
deciding whether to place it. Whether an order is *placed* at all comes down
to one boolean assembled at the top of the run:

```python
# backend/app/services/executor.py
armed = is_armed()
will_place = settings.is_live and armed and not force_dry_run
# Human-quorum floor: never spend real money on a basket chosen
# without human participation.
if will_place and voting.human_voter_count(session["id"]) < settings.min_human_voters:
    will_place, quorum_blocked = False, True
# ...
if will_place and not calendar_covers(_today_et()):
    will_place, calendar_stale = False, True
base = "LIVE" if will_place else ("DRY_RUN(disarmed)" if settings.is_live else "DRY_RUN")
```

Everything downstream of `will_place=False` still runs — reviews, sizing,
records — it just never touches money. That means a disarmed run produces the
exact artifact worth inspecting before arming. The full safety design,
including the stale-basket refusal and the fail-open/fail-closed split, is in
[SAFETY.md](SAFETY.md).

## Display reads never touch the trading path

Opening the account page used to trigger live broker reads on every visit
(about 4 seconds and five round-trips). A snapshot layer now reads each
member's account once per 10-minute interval and serves the UI from a local
row, stale-while-revalidate. The one rule that makes this safe is written at
the top of the module in a box, because it must survive every future refactor:
the snapshot is a *display* cache, and the trading path is forbidden from
reading it. Order sizing always does its own live, uncached broker read. A
member's page can be ten minutes stale; an order never is.

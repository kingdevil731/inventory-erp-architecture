# Retrospective

Why v1 is being rewritten rather than extended.

## What was built

Roughly thirty backend domain modules, 29 migrations, a shared contracts package,
a React frontend. Stock movements as an append-only ledger with derived on-hand,
sales and purchase orders, invoicing, public quotes, customers, suppliers,
assets, work orders, work-order templates, maintenance plans, RBAC, company
membership, subscriptions, notifications, attachments, platform admin.

It works. That is not the problem.

## What was wrong

### The localisation was the product, and it was scheduled for later

v1 is English, desktop-first and online-only, with Mozambican specifics filed as
a future phase.

The intended customers are shops, bars and restaurants in Mozambique. The
operator is behind a counter with a phone, not at a desk with a monitor. The
connection drops several times a day. They think in MZN and read Portuguese.

Built as it was, the product competes with Zoho Inventory, Sortly, inFlow and
Odoo on their terms — a fight with no local advantage, against companies with
more engineers. Built the other way it competes on ground none of them occupy.
The offline, Portuguese, mobile, MZN-priced part is not a localisation layer over
a generic product; it is the only reason to choose this one.

Filing it as phase two was the central error, and everything below is downstream
of it.

### No till

Thirty modules and no cash session. No opening float, no expected-versus-counted
reconciliation, no void, no refund path that touches both stock and cash, no
discount authority limits.

A shop owner's actual nightly question is whether the drawer matches the day's
sales. Inventory accuracy is something they tolerate in service of that. v1
answers the question they ask monthly and not the one they ask every night,
which is the difference between software they try and software they pay for.

### Scope outran validation

Maintenance plans and work-order templates were built before any customer had
paid for stock control. Each is defensible individually. Together they represent
months spent on the fifth-most-important thing.

The rule that would have prevented it: nothing gets built past the core loop
until someone outside the family has paid for the core loop.

### Correctness is untested where it matters most

12 test files. None covering moving-average cost across successive price changes,
unit conversion, receiving into negative stock, or stock-count variance when
sales happen during the count.

These are the places where a bug is unrecoverable rather than annoying. The
ledger is the source of truth for everything derived, so a costing error
propagates into every report and cannot be corrected retroactively without
knowing when it started. Testing infrastructure while leaving inventory
arithmetic untested is the wrong way round.

## What carries forward

The parts worth keeping, and why:

**The movement ledger with derived on-hand.** Correct in principle and worth the
read cost, and better built than I gave it credit for — see
[stock-ledger.md](stock-ledger.md). The rewrite keeps the ledger as truth and adds a maintained balance
projection alongside it — reads hit the projection, and a full ledger replay must
reproduce it exactly. That gives read performance without giving up
reconstructibility, which is the thing a stored mutable quantity gets wrong.

**Composed tenancy predicates**, upgraded from convention to enforcement via a
client extension.

**Module structure.** Router, controller, service, repository, schemas and
OpenAPI per domain held up well across thirty modules.

**The contracts package.** One definition, both sides.

## What the rewrite does differently

Offline-capable and mobile-first from the first commit, not retrofitted.
Portuguese-first. Cash session and reconciliation in the first shippable version,
before assets or maintenance. A correctness suite over inventory and cash
arithmetic that must pass before any customer touches it.

Two things listed here in an earlier draft turned out to be already true of v1
and are carried forward rather than introduced: quantities and money are already
`Decimal`, never float, and movements already carry a `clientRequestId` under a
unique constraint, which is the idempotency key an offline client needs. What v1
lacks is not the key but the offline client that would use it. See
[stock-ledger.md](stock-ledger.md).

And one constraint that shapes the rest: a business outside the family must be
paying before scope expands past the core loop.

## The general lesson

v1 was competent software built for a market that was not the one it was going to
be sold into. Every specific failure — no till, wrong platform, wrong language,
untested arithmetic, premature modules — follows from deciding what to build
before deciding precisely who was going to pay for it.

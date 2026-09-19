# Decisions

Each entry records what was chosen, what was rejected, and what it cost.

## Stock on hand is derived from a ledger, not stored

**Chosen:** an append-only `StockMovement` table; on hand is a sum over it,
computed per query.

**Rejected:** a mutable `quantityOnHand` column updated alongside each movement.
Faster to read and the standard approach, but it is a cache of the history, and
when it diverges there is no way to learn when or to rebuild it. Every derived
report inherits the error permanently.

**Cost:** an aggregate query per read, which degrades with ledger size and
degrades fastest for the busiest customer. Batched per page rather than per row,
which defers the problem without solving it. Detail in
[stock-ledger.md](stock-ledger.md).

## Direction on the line, so a transfer is one movement

**Chosen:** `fromLocationId` and `toLocationId` as nullable references on
`StockMovementLine`.

**Rejected:** representing a transfer as two opposing movements. That creates a
pair that must stay consistent forever, and a half-applied transfer is stock
that exists in neither place.

## Corrections are reversals, not edits

**Chosen:** a correcting movement links back via `reversedFromMovementId` with a
`reversalReason`. The original row stays.

**Why:** it preserves what was believed at the time, not just what turned out to
be true. That is the question that matters when reconciling a dispute months
later.

**Undermined by:** soft delete on the same table — see the last entry.

## Shared schema for multi-tenancy, not schema-per-tenant

**Chosen:** one database, one schema, `companyId` on every tenant-owned row.

**Rejected:** a schema or database per tenant. Stronger isolation, and migration
cost scales with tenant count — unworkable at a small-shop price point with 29
migrations to apply.

**Cost:** isolation becomes a correctness problem in every query rather than an
infrastructure property. Mitigated with composed predicates and middleware, not
enforced by the database. Four known weaknesses in
[multi-tenancy.md](multi-tenancy.md).

## Reservations sit outside the ledger

**Chosen:** `StockReservation` is its own table, with `releasedAt` rather than
deletion. Available stock is on hand minus unreleased reservations.

**Why:** a reservation has not moved anything. Putting it in the ledger would
make a promise indistinguishable from a physical movement, and the physical count
would stop matching the shelf.

## Domain-owned modules with a repository boundary

**Chosen:** each domain keeps its router, controller, service, repository,
schemas and OpenAPI definition together; the repository is the only code touching
the database.

**Why:** that boundary is where tenancy and soft-delete rules can be enforced
once per domain instead of at every call site. The entire value is in that
property, so a service reaching past its repository silently removes it — which
makes this a review rule rather than a convention.

**Also:** the OpenAPI definition lives next to the route it describes. In a
separate tree it drifts within weeks; in the same directory, a stale definition
shows up in the same diff.

## Decimal for quantity and money

**Chosen:** `Decimal(18,3)` for quantity, `Decimal(18,4)` for unit cost and
price.

**Rejected:** float. Accumulated error surfaces as cents that do not reconcile,
and in a derived-balance system that error compounds into every downstream
figure.

## Soft delete on movements — the one I would reverse

**Chosen at the time:** `deletedAt` on `StockMovement`, filtered out of the
derivation.

**Why it was wrong:** it contradicts the append-only design it sits on top of. A
movement can vanish from every balance at once, and correctness now depends on
every query remembering the filter. Reversal already does this job, with an audit
trail and without the footgun.

**Status:** removing it is the first change in the rewrite.

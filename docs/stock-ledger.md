# The stock ledger

The one part of this system worth reading closely. Everything about stock —
what you have, what you had last Tuesday, what it cost you — is derived from a
single append-only table rather than stored.

## Shape

Two tables. `StockMovement` is the event; `StockMovementLine` is what it did to
which product in which place.

```
StockMovement                    StockMovementLine
  type       PURCHASE              productId
             SALE                  fromLocationId   nullable
             ADJUSTMENT            toLocationId     nullable
             TRANSFER              qty        Decimal(18,3)
             STOCK_COUNT           unitCost   Decimal(18,4)
  effectiveAt                      unitPrice  Decimal(18,4)
  clientRequestId
  reversedFromMovementId
  adjustmentReason
```

Direction lives on the line, not the movement, as a pair of nullable location
references. A purchase has `toLocationId` only, a sale has `fromLocationId`
only, and a transfer has both — which means a transfer is _one_ movement with
one line, not two opposing movements that must be kept in step. Nothing can
half-apply.

## Derivation

On hand is a sum over the ledger, computed per query:

```sql
WITH deltas AS (
  SELECT sml."productId", sml."toLocationId" AS "locationId", sml."qty" AS delta
  FROM "StockMovementLine" sml
  JOIN "StockMovement" sm ON sm."id" = sml."movementId"
  WHERE sm."companyId" = $1 AND sm."deletedAt" IS NULL
    AND sml."toLocationId" IS NOT NULL
  UNION ALL
  SELECT sml."productId", sml."fromLocationId", -sml."qty"
  ...
)
SELECT "productId", SUM(delta) FROM deltas GROUP BY ...
```

Inbound lines count positive, outbound negative, and the two halves are unioned
before aggregation. Product listings batch this across a whole page rather than
issuing it per row.

`StockReservation` is deliberately _not_ in the ledger. A reservation against a
sales order has not moved anything; it constrains what can be promised. Available
stock is on hand minus unreleased reservations, so a reservation never distorts
the physical count.

## Why derived rather than stored

A stored `quantityOnHand` column is a cache of the movement history. The failure
mode is not that it goes wrong — it is that when it goes wrong, there is no way
to find out _when_, and no way to rebuild it. Every report downstream inherits
the error silently and permanently.

Deriving it means the ledger is the only truth, any historical balance is
reconstructible by summing to a point in time, and a discrepancy is always
explainable by a specific row.

The cost is real: aggregating a growing table on every read gets slower with age,
and it gets slower fastest for the busiest customer. That is the wrong scaling
shape and it is a main driver of the rewrite. The intended resolution is a
maintained balance projection that reads hit, with the ledger kept as the
reconciliation authority behind it — a full replay must reproduce the projection
exactly. That buys read performance without giving up reconstructibility, which
is precisely what a mutable quantity column gives up.

## Four details that earn their place

**Corrections are reversals, not edits.** `reversedFromMovementId` and
`reversalReason` link a correcting movement back to the one it undoes. The
original row stays. You can therefore ask what someone believed the stock was in
March, not just what it turned out to be — which is the question that matters
when reconciling a dispute.

**`effectiveAt` is separate from `createdAt`.** When a delivery arrives Friday
and is entered Monday, the ledger records both the business date and the entry
date. Collapsing them makes backdated entries indistinguishable from late ones.

**`clientRequestId` is unique per company.** `@@unique([companyId, clientRequestId])`
means a retried submission cannot double-post. This is the idempotency key the
events platform check-in is missing — the same problem, solved correctly here.
It is also the piece an offline client would need, and building that client is
rewrite work; the key itself is already in place.

**Decimal throughout, never float.** `Decimal(18,3)` for quantity, `Decimal(18,4)`
for unit cost and price. Float arithmetic on money accumulates error that surfaces
as cents that do not reconcile, and in a ledger that error compounds across every
derived figure.

## What is wrong with it

**Soft delete contradicts append-only.** `StockMovement` carries `deletedAt`,
`deletedById` and `deletedReason`, and the derivation filters on
`deletedAt IS NULL`. So the ledger is not actually append-only — a movement can
be made to disappear from every balance at once. Worse, correctness now depends
on every query remembering that filter, and a single place that forgets it
reports a different number from everywhere else.

Since reversal already exists and is the better mechanism, soft delete on
movements should not. Removing it is the single change I would make first.

**No balance projection.** Described above. Every on-hand read is an aggregate
over the full history.

**Costing is untested.** `unitCost` is carried per line, but nothing tests
moving-average cost across successive purchases at different prices, and nothing
tests receiving into negative stock or a stock count taken while sales are
happening. These are the cases where an error is unrecoverable rather than
annoying, because the ledger is the source everything else derives from.

**Index coverage for the derivation is unverified.** The aggregate joins
`StockMovementLine` to `StockMovement` and filters by company. Whether the
indexes on the line table support that join shape at size was never measured.

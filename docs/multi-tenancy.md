# Multi-tenancy

Shared PostgreSQL, shared schema, `companyId` on every tenant-owned row. Not a
database or schema per tenant.

## Why shared schema

The alternatives were a schema per tenant or a database per tenant. Both give
stronger isolation, and both were rejected for the same reason: the target
customer is a small shop paying a small monthly fee, and migration cost scales
with tenant count. Applying 29 migrations across hundreds of schemas is an
operational burden the price point does not support.

Shared schema moves that burden into the query layer, where it becomes a
correctness problem instead. That is the trade — one migration to run, and every
single query has to be right.

## How scoping is enforced

Three layers, because any one of them alone fails.

**Middleware** — `companyAccess` resolves the caller's membership and company
context; `requireRole` gates by role; `requirePlatformAdmin` marks the deliberate
bypass.

**Composed predicates in repositories** — scoping is a reusable predicate rather
than a hand-written `where` clause:

```ts
...withNotDeleted(withCompany(companyId))
```

**Explicit `companyId` in raw SQL** — where aggregation over the movement ledger
makes the ORM the wrong tool, the parameter is carried through the query
explicitly:

```sql
WHERE sm."companyId" = ${companyId}
```

via Prisma's tagged-template interpolation, which parameterises rather than
concatenating.

## The reasoning behind composition

Tenant leaks do not come from a developer deciding isolation is unimportant. They
come from one query written quickly, in a hurry, that omits a clause — usually a
new endpoint on an existing model, or a join that scopes the outer table and not
the inner one.

Making the scoped form a composable helper means the correct version is shorter
than the incorrect one, which is the only reliable defence. Documentation and
review both fail eventually; ergonomics do not.

Soft deletion travels with scoping deliberately. `withNotDeleted(withCompany(id))`
composes the two because they are always wanted together, and separating them
creates a second thing to forget.

## Known weaknesses

**Nothing prevents an unscoped query.** The helpers are available and
conventional, not enforced. A raw `prisma.model.findMany()` compiles and runs.
The fix is a Prisma client extension that requires an explicit opt-out, so
bypassing tenancy is a visible decision in the diff rather than an omission.

**Isolation is not tested.** There is no test asserting that a request in
company A cannot read company B's data, per endpoint. That is the single test
worth having in a shared-schema system and it does not exist. With roughly thirty
domain modules it should be generated from the route table, not hand-written.

**Index leading columns are not verified.** In a shared-schema design every
composite index should lead with `companyId` — without it, queries scan across
tenants and the system gets _slower_ as customers are added, which is the worst
possible scaling shape. Whether that holds across all 29 migrations was never
audited.

**Platform admin bypass is marked but not consistently audited.**
`requirePlatformAdmin` identifies the intentional bypass, which is right. Every
use of it should produce an audit record; that is not guaranteed.

All four are addressed in the rewrite. They are listed here rather than omitted
because a multi-tenancy document that only describes the mechanism, without
saying what would let it fail, is not worth reading.

# Inventory & Operations Platform

Architecture showcase for a multi-tenant inventory, asset and operations system.
The implementation is private; this repository documents the design.

**Status: v1 complete, being rewritten.** It is documented here because the
rewrite is a direct consequence of what v1 got wrong, and that is the more
useful thing to read. See [retrospective](docs/retrospective.md).

---

## Scope

A pnpm monorepo — `apps/backend`, `apps/frontend`, `packages/contracts` — with
roughly thirty backend domain modules covering more ground than "inventory"
suggests:

**Stock** — products, categories, stock movements, stock-on-hand, reservations,
stock counts, locations, suppliers
**Commerce** — sales orders, purchase orders, invoices, public quotes, customers,
payments
**Operations** — assets, work orders, work-order templates, maintenance plans
**Platform** — companies, company membership, company users, company settings,
roles, subscriptions, notifications, attachments, dashboard, platform admin

29 Prisma migrations against PostgreSQL. Contracts shared between backend and
frontend via `packages/contracts`, organised by domain.

## Two things it gets right

### Stock on hand is derived, not stored

Quantity on hand is computed by aggregating the stock-movement ledger in SQL
rather than maintained as a mutable column on the product.

This is the decision the whole domain rests on. A stored quantity is a cache of
the movement history, and any bug that updates one without the other produces a
number that is wrong with no way to discover when it went wrong. Deriving it
means the ledger is the only truth and on-hand is always reconstructible.

The cost is real and shows up quickly: on-hand becomes an aggregate query per
read, and product lists need it for every row. That is handled by batching —
`getOnHandByProductIds` computes it for a whole page in one query rather than
per product — but the underlying scaling problem is unsolved and is a main driver
of the rewrite.

### Tenancy is composed at the query layer, not remembered

Scoping is not left to each call site to get right. Repositories compose
predicates:

```ts
...withNotDeleted(withCompany(companyId))
```

with `companyAccess`, `requireRole` and `requirePlatformAdmin` middleware in
front of it. Tenant isolation fails through the one query somebody wrote in a
hurry, so the goal is to make the scoped version the path of least resistance.

→ [Multi-tenancy](docs/multi-tenancy.md)

## Structure

Each domain module owns its router, controller, service, repository, Zod schemas
and OpenAPI definition:

```
api/product/
  productRouter.ts
  productController.ts
  productService.ts
  productRepository.ts
  productSchemas.ts
  productOpenApi.ts
  __tests__/
```

Prisma for typical access; raw parameterised SQL where aggregation over the
movement ledger makes the ORM the wrong tool.

## What v1 got wrong

Summarised here, in detail in the [retrospective](docs/retrospective.md).

**It was built for the wrong market.** v1 is English, desktop-first and
online-only. The intended customers are Mozambican shops, bars and restaurants,
where the operator is on a phone, the connection drops, the language is
Portuguese, and prices are in MZN. The localisation is not a feature to add
later — it is the product, and treating it as a phase-two concern was the central
error.

**It has no till.** Thirty modules of inventory, assets and maintenance, and no
cash session, no reconciliation, no void, no discount authority. For a shop
owner the question that matters is whether the money in the drawer matches what
was sold today, and v1 cannot answer it. Inventory is what they tolerate; cash
control is what they buy.

**Scope outran validation.** Maintenance plans and work-order templates were
built before a single customer had paid for stock control.

**12 test files across 29 migrations of domain**, and none over the inventory
arithmetic — costing across price changes, unit conversion, count variance while
sales are happening. That is exactly the code where a silent error is
unrecoverable, because it corrupts the ledger everything else derives from.

## Roadmap

The rewrite is scoped around what v1 proved, not around adding to it.

|                                                         | Status                         |
| ------------------------------------------------------- | ------------------------------ |
| Stock ledger, on-hand derivation, multi-tenancy, RBAC   | v1, works, carried forward     |
| Sales/purchase orders, invoicing, quotes                | v1, carried forward            |
| Assets, work orders, maintenance plans                  | v1, deferred — built too early |
| Cash session, reconciliation, voids, discount authority | Rewrite, P0                    |
| Offline-capable, mobile-first operation                 | Rewrite, P0                    |
| Portuguese-first UI, MZN                                | Rewrite, P0                    |
| Correctness suite over inventory and cash arithmetic    | Rewrite, blocking              |

## Documents

|                                        |                                                                      |
| -------------------------------------- | -------------------------------------------------------------------- |
| [Architecture](docs/architecture.md)   | Module shape, the movement ledger and why on-hand is derived         |
| [Multi-tenancy](docs/multi-tenancy.md) | Shared-schema scoping, how it is enforced, and four known weaknesses |
| [Retrospective](docs/retrospective.md) | Why v1 is being rewritten rather than extended                       |

---

## Repository note

This repository contains architecture documentation only. The implementation is
private.

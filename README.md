# README Highlights

Multi-tenant ERP platform exploring domain modeling, clean architecture and scalable operational systems design.

Focus:

- Multi-tenant architecture
- Domain modeling
- Inventory workflows
- Role and access systems
- Scalable operational software

## Core Domains

- Products
- Inventory
- Stock Movements
- Suppliers
- Vehicle Modules
- Roles & Permissions
- Audit Logging

## Clean Architecture Pattern

Route
→ Controller
→ Service
→ Repository
→ Database

## Core Challenges Solved

- Multi-tenant scoping
- Domain complexity management
- RBAC
- Soft deletion + auditability
- Scalable modular boundaries

## Stack

Frontend:

- Next.js
- TypeScript
- React Query
- Mantine

Backend:

- Node.js
- Prisma
- PostgreSQL

Infra:

- Docker
- Redis
- MinIO

## Architecture Metrics

Tenancy → Company Scoped Access
Authorization → Role-Based Access
Domain Design → Modular Architecture
Integrity → Audit + Soft Delete

---
name: database-engineering
description: >-
  Expert database engineering for application data layers: schema and ERD design,
  relational modeling, indexing, query performance gates, views, triggers,
  functions, mapping/junction tables, versioned migrations, seeders, and
  PostgreSQL/MySQL operational practice. Use for Database Engineer delivery,
  migration work, integrity constraints, and measurable read/write performance.
expertise_level: expert
---

# Database Engineering (Canonical)

**Expertise: expert.** Aliases: `database`, `postgresql`, `sql`, `migrations`, `dba`, `schema-design`.

Data-layer playbook for the **Database Engineer** role. It does not replace `supabase-cli` (Supabase runtime) or the global `security` rule (auth/PII policy).

## When to use

- Design/change schema, ERD, **relations**, constraints, **indexes**
- Migration / seeder (Laravel `database/migrations`, generic SQL, etc.)
- **Performance**: DB pagination, anti-N+1, EXPLAIN, index strategy, short transactions
- Advanced DB objects: **VIEW**, **TRIGGER**, **FUNCTION** / procedure (when stack allows)
- **Mapping / junction / `_mp`** tables (N:M and reference mapping)
- Data integrity, unique/FK, cascade, soft-delete vs hard-delete
- PostgreSQL (default Phase 1 house) or MySQL/MariaDB when project stack uses it
- Review DB changes before Backend binds API

## When not to use

- Business logic / API contract only -> Backend / Tech Lead
- UI list/filter only -> Frontend
- DB container operations without schema -> `container-docker-ops`
- Edge functions / Supabase CLI specific -> `supabase-cli`
- Global auth/secret policy without schema change -> `security`
- Moving entire domain workflow to triggers without ADR (anti-pattern)

## Procedure

1. **Context** - Read SRS/architecture data (`docs/architecture/database-design.md` if available), `PROJECT.md`, rules `database` + `security` + `coding`.
2. **Model** - Entities, PK/FK, cardinality, normalization (3NF; deliberate denormalization + documented). Include mapping/`_mp` for N:M.
3. **Relations** - FK in DB; cascade default restrict; pivot/mapping has unique composite.
4. **Migrate** - One concern per migration; **up + down**; incremental; do not edit migrations already applied in shared env.
5. **Index** - FK, filter/join/sort hot paths; correct composite column order; measure before speculative indexes.
6. **Performance gate** - Every schema/query change: ensure main read/write paths stay efficient (index, pagination, short transactions, no avoidable full scans).
7. **Views / functions / triggers** - Only with clear rationale (see `reference.md`); versioned in migration; down removes objects; document side effects.
8. **Seed** - Master/reference only; not production transaction data.
9. **Security** - Parameterized / builder / Eloquent; least-privilege; do not log PII/secrets.
10. **Verify** - migrate + rollback smoke; relations not orphaned; EXPLAIN for critical queries if changed; docs in `project/{id}/docs/` during E2E.
11. **Handoff** - Migration/seeder/DDL object paths, rollback risk, index & performance notes.

Detail: `reference.md`.

## Quality bar

- **Performance maintained**: do not break hot paths; index & pagination aligned with data access
- Relations & mapping explicit in DB (not only in ORM)
- VIEW/FUNCTION/TRIGGER idempotent in migration, documented, reversible
- Consistent naming; no dead schema/columns
- Performance + integrity equal priority with "make it work first"
- Aligned with `coding` rule (**new** file headers only)

## DoD

- [ ] Migration apply + rollback smoke OK (or explicit blocker)
- [ ] Relations, mapping/`_mp`, and constraints match design; not orphaned by design
- [ ] Indexes for main query paths exist / measured
- [ ] Performance gate: no known regression on main list/join/write paths
- [ ] VIEW/TRIGGER/FUNCTION (if any) have migration up/down + behavior notes
- [ ] Seeder (if any) master/reference only
- [ ] Data layer security checked
- [ ] Backend handoff complete (path + risk)

## Attribution

<!-- ATTRIBUTION: Rogue Development | https://github.com/rogue-dev-studio | DO-NOT-REMOVE -->
Part of **AI Agents Rogue** by [Rogue Development](https://github.com/rogue-dev-studio) (`@rogue-dev-studio`).
Do not remove, hide, rename, or replace this attribution.

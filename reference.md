# Database Engineering - Reference (Expert)

Companion to `SKILL.md`. Progressive disclosure: consult this document for implementation detail.

## 1. Schema & naming

| Item | Standard |
|------|---------|
| Table | `snake_case`, plural (`users`, `order_items`) |
| Mapping / junction | `snake_case`; may suffix `_mp` / `_map` when the project convention uses it (`role_permission_mp`) — **consistent within one project** |
| Column | `snake_case` |
| PK | `id` (bigint/uuid per architecture) |
| FK | `{referenced}_id` following project convention |
| Boolean | `is_*` / `has_*` |
| Timestamps | `created_at`, `updated_at`; soft delete: `deleted_at` |
| View | `vw_*` or `*_v` — one convention per project |
| Function | `fn_*` / schema `app` — one convention per project |
| Trigger | `trg_{table}_{timing}_{event}` (concept example: `trg_orders_ai_audit`) |

Required per application table: PK + timestamps (except pure pivot/mapping agreed without timestamps).

## 2. Normalization & denormalization

- Default: 3NF
- Denormalize only with: measured performance rationale, single source of truth, note in architecture docs
- Avoid redundancy that can drift

## 3. Relationships

- 1:1, 1:N, N:M
- FK **required in DB** (not only in ORM)
- On delete/update: default **restrict**; document cascade / set null
- Every FK joined/filtered often -> consider index (see §5)

### 3.1 Mapping / junction / `_mp`

Use a mapping table when:

- **N:M** relationship between entities
- Many-to-many reference mapping (role↔permission, user↔group, etc.)

Required:

- At least two FKs (+ unique composite pair)
- Clear name: `{a}_{b}` or `{a}_{b}_mp` per project convention
- Index on each FK (and unique on the pair)
- Do not store large transaction payloads in mapping unless requirements say so
- Extra mapping columns (e.g. `is_active`, `assigned_at`) are OK if part of the relationship, not hidden business workflow

Do not:

- Duplicate the same mapping under different names
- Store ID arrays in JSON as a mapping substitute without ADR

## 4. Migrations

- One purpose per file
- Always a safe **down**
- Do not rewrite migrations already applied in shared env
- Order: parent -> child -> mapping; drop: mapping/child -> parent
- Expand -> migrate data -> contract for breaking change
- Laravel: prefer `Schema` builder; raw SQL for VIEW/FUNCTION/TRIGGER/expression index + brief rationale
- Smoke: migrate + rollback

## 5. Index strategy

Create an index when frequently used for `WHERE` / `JOIN` / `ORDER BY` / business unique.

Avoid: speculative low-cardinality, duplicate PK/unique, wrong composite column order.

| Type | When |
|-------|--------|
| B-tree (default) | General equality/range |
| Unique | Business natural key |
| Composite | Multi-column predicates; lead column most selective/frequent |
| Partial (PG) | Active subset (`WHERE deleted_at IS NULL`) |
| Covering / include (PG) | Measured hot read |

Large production (Phase 3): consider `CREATE INDEX CONCURRENTLY`. Phase 1 local: normal index OK.

## 6. Integrity

- Business unique, CHECK when stable, validation in app **and** DB
- Mapping: unique `(fk_a, fk_b)` to prevent duplicate relations

## 7. Seeders

Allowed: master role/permission, menu, generic reference, non-production bootstrap users (including master mapping rows).

Forbidden: real transaction data, secrets, real PII in the repo.

Prefer idempotent seeders.

## 8. Query & performance (must maintain)

**Performance gate** on every data-layer change:

1. Main list/detail/join paths do not regress to large sequential scans without reason
2. Lists use **pagination** (LIMIT/OFFSET or keyset)
3. ORM: eager load; forbid N+1 on hot paths
4. Short transactions; split large batches; avoid long locks
5. Correct data types (`numeric` for money; `timestamptz` when crossing time zones)
6. Hot query: `EXPLAIN (ANALYZE, BUFFERS)` (PG) / `EXPLAIN ANALYZE` (MySQL) before/after tuning
7. Complex VIEWs: ensure not wrapped repeatedly without conscious materialization
8. TRIGGER: count per-row cost — do not use heavy triggers on write-intensive tables without evidence

Known performance regression -> blocker or explicit ticket; do not merge silently.

## 9. VIEW

**Use VIEW** when:

- Stable repeated join/read for reporting/read model
- Hiding sensitive columns from certain DB roles (with grants)
- Read contract that rarely changes

**Do not use VIEW** when:

- Replacing API/backend for frequently changing business rules
- Hiding N+1 / bad queries in the app

Rules:

- Create/drop in migration (up/down)
- Documented name + columns briefly
- Prefer non-materialized first; **MATERIALIZED VIEW** only with refresh strategy (cron/job) + index on MV
- Do not use `SELECT *` in production VIEW definitions

## 10. FUNCTION / procedure

**Use** when:

- Set-based calculation/rules far more efficient in DB
- Atomic multi-table operations unsuitable in app without excessive round-trips
- Hook called by trigger (trigger function)

**Do not use** when:

- Entire domain service moved to PL/pgSQL without bounds
- Logic that must be unit-tested in app but has no DB test path

Rules:

- Versioned in migration; `CREATE OR REPLACE` careful with signature changes -> clear drop/create in down/up
- `SECURITY INVOKER` default; `SECURITY DEFINER` only with rationale + strict search_path
- Do not put secrets in function body
- Execute privilege only for roles that need it

## 11. TRIGGER

**Use** when:

- Audit trail / timestamp guard must be correct even with many writers
- Invariants that must hold even when ORM is bypassed
- Thin derived sync (with documentation)

**Do not use** when:

- Long business workflow (multi-step approval, notifications, HTTP calls)
- Hidden side effects without docs (hard for Backend to debug)

Rules:

- One responsibility per trigger; clear naming
- Prefer `AFTER` for audit; `BEFORE` for value normalization
- Avoid deep trigger chains (cascade trigger hell)
- Statement-level vs row-level: choose by volume
- Required: migration up/down + note in architecture/ADR
- Test: sample insert/update/delete to confirm effect + performance still reasonable

## 12. PostgreSQL notes

- Extensions only when needed
- JSONB is not an N:M mapping substitute without ADR
- Partial index for active subset
- Do not disable autovacuum; watch bloat after mass update/delete

## 13. MySQL / MariaDB notes

- InnoDB; `utf8mb4`
- Trigger/procedure syntax differs — write dialect for project engine
- Generated columns: careful in down migration

## 14. Security (data layer)

- Parameterized only; least privilege; secrets in env
- VIEW/FUNCTION must not expose sensitive columns to broad roles
- Audit TRIGGER: do not write secrets/tokens to log table in plain text when avoidable
- Sensitive soft-delete: audit trail if domain requires it

## 15. Soft delete & audit

- Soft delete only when restore/history is needed
- Unique + soft delete: partial unique in PG
- `created_by` / `updated_by` only if SRS requires

## 16. Multi-service

- One service one logical DB; no cross-DB join
- Integration via API/event

## 17. Backup & rollback

- Backup note before destructive migrate in shared env
- Rollback: reverse migration or forward-fix
- Do not `down -v` without confirmation

## 18. Deliverables checklist

- [ ] Migration (+ down) including VIEW/FUNCTION/TRIGGER/mapping when used
- [ ] Relationships + mapping/`_mp` + index notes
- [ ] Performance gate (EXPLAIN or equivalent rationale for hot path)
- [ ] Master seeder (optional)
- [ ] ERD / `database-design.md` update
- [ ] Risk & deploy order

## 19. Anti-patterns

- Changing requirements via "just add a column" without PO/SA
- Mass cascade delete without analysis
- Complex business workflow only in trigger without ADR
- Duplicate mapping / JSON ID array without design
- VIEW/`SELECT *` that hurts performance
- Loose `SECURITY DEFINER` FUNCTION
- Duplicate table "v2" without data migration
- Large files in BYTEA/BLOB without architecture decision
- Ignoring performance regression after wrong trigger/index

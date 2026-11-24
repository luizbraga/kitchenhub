# KitchenHub: Design

Status: approved. Date: 2026-11-24.

Items marked **[call]** were chosen by the designer rather than requested by the project owner.

## 1. Intent

KitchenHub manages recipes, ingredients and pantry stock, and tells users what they can cook.
It is a backend engineering portfolio project. Its value lies in domain modeling, consistency,
clean boundaries and tests.

**Goals**
- Recipes with search, collections and scaling
- A shared ingredient catalog and correct measurement conversion
- Batch-level pantry stock with an immutable movement ledger
- Recipe matching, atomic cooking and shopping lists
- Private data per user, plus optional shared Homes

**Non-goals:** frontend, microservices, email delivery, account deletion, undoing a cook,
shared recipes or shopping lists, automatic stocking of purchased shopping items.

**Anti-goals:** generic CRUD; business rules inside routes or ORM models; guessed conversions;
stock changes that bypass the ledger; abstractions without a purpose.

## 2. Constraints

Python 3.12+, FastAPI, PostgreSQL, SQLAlchemy 2.0, Alembic, Pydantic, Redis, Celery, Pytest,
Docker Compose, Ruff, Mypy and GitHub Actions, in a modular monolith. Code follows PEP 8 and PEP 20.

## 3. Approaches

| Approach | Gain | Cost |
|---|---|---|
| **A. Ledger plus batch balances (chosen)** | Fast reads, row locks, non-negative check | Balance duplicates ledger; needs reconciliation |
| B. Ledger only | Nothing can drift | Reads slow with history; pantry-wide locking |
| C. Full event sourcing | Replay and time travel | Versioning, projections, eventual consistency |

The owner picked A for simple locking and fast matching. C was judged overengineered.

## 4. Architecture

```
src/kitchenhub/
  api/             routes, schemas, dependencies, error mapping
  application/     use cases; each owns one transaction
  domain/          measurements, ingredients, recipes, pantry, homes, shopping_lists
  infrastructure/  database models, repositories, security, logging
  workers/         Celery app and tasks
  main.py
tests/{unit,integration,e2e}/
migrations/
docs/{architecture.md,decisions/}
```

- Dependencies point inward. `domain` imports neither FastAPI nor SQLAlchemy; a test enforces this. **[call]**
- `application` defines repository protocols only for stock, recipes and shopping lists. Simpler
  modules use SQLAlchemy sessions directly. **[call]**
- The API uses async SQLAlchemy. Celery tasks run the same use cases through `asyncio.run`, with an
  engine created for that run. **[call]**
- The `src/kitchenhub` package replaces the suggested bare `src` layout to avoid top-level names
  such as `api`. **[call]**

## 5. Accounts and access

- Email and password, hashed with Argon2. Access JWTs last 15 minutes. Rotating opaque refresh tokens
  last 30 days and are stored hashed; reuse of a rotated token revokes its family. **[call]**
- Redis limits login attempts. If Redis is down, login still works and a warning is logged. **[call]**
- Each user has a timezone and a conversion mode (`strict` or `convert`, default `strict`). **[call: default]**
- Inaccessible resources return 404, so their existence is not revealed. **[call]**

## 6. Homes and pantries

- A pantry belongs to exactly one user or one Home, enforced by a check constraint. Each has a timezone.
- Each user always has a private pantry. Homes are opt-in, and a user may join several. **[call: several]**
- Joining moves no stock. Leaving moves no stock. Private pantries are visible only to their owner.
- Users explicitly transfer stock between pantries they can access, in either direction. **[call: both ways]**
- Roles are owner and member. Owners create single-use invitation codes, valid for 7 days, stored hashed,
  and passed on outside the system. **[call]**
- An owner must hand over ownership before leaving. A sole member archives the Home instead, which is
  refused while its pantry contains stock. **[call]**
- Every stock use case locks the membership row with `FOR SHARE`, so it cannot race with removal.
- Movements in a Home pantry record the acting member.

## 7. Measurements

| Dimension | Canonical unit | Units |
|---|---|---|
| Mass | g | g, kg |
| Volume | ml | ml, l, tsp (5), tbsp (15), cup (240) |
| Count | piece | piece |

- `Quantity(Decimal, Unit)` is an immutable value object. Floats are rejected.
- JSON quantities are strings. Storage is `NUMERIC(18,6)`, rounded half-even when stored. **[call]**
- Conversion within a dimension is always allowed.
- Across dimensions, `convert` mode uses grams per ml and grams per piece. A user's override wins over
  the ingredient's value. Results carry `approximate: true`. **[call: grams per piece]**
- `strict` mode, or a missing factor, yields `cannot_compare`, never a guess.
- The account mode may be overridden per request. The acting user's overrides apply in Home pantries. **[call]**

## 8. Ingredients

- Fields: name, description, category, dimension, default unit, optional density, grams per piece, nutrition per 100 g.
- The default unit must belong to the ingredient's dimension. Recipes and batches may use any dimension.
- The shared catalog is seeded by a command and read-only for users. Users may create private ingredients.
- Names are unique, ignoring case, within the catalog and within each user's private ingredients.
- A private ingredient can be read by its owner, anyone with access to a pantry holding it, and anyone
  whose recipe uses it. This keeps Home sharing workable. **[call]**
- Once others use it, the owner may edit only its description and nutrition. Referenced ingredients cannot be deleted. **[call]**

## 9. Recipes

- Fields: name, description, ordered steps, servings, preparation and cooking minutes, difficulty, cuisine,
  and ordered ingredient lines (ingredient, quantity, unit).
- Collections are many-to-many and cover categories too. **[call]**
- Servings and quantities are positive; minutes are non-negative.
- Updates send a version. A stale version returns 409. **[call]**
- Search by name (`pg_trgm`), ingredient, cuisine and maximum preparation time, with pagination.
- Any recipe can be scaled up or down, by target servings (a positive integer) or by a multiplier
  (a positive decimal such as 0.5 or 2); exactly one is given. **[call: multiplier]**
- Scaling is a pure domain function: `quantity * factor`, where the factor is `target / original` or the
  multiplier. Pieces may be fractional. Scaling never changes the stored recipe. Preparation and cooking
  times are not scaled. Cooking, matching and shopping lists accept the same scaling input. **[call]**

## 10. Inventory

**Batch:** pantry, ingredient, dimension, remaining canonical quantity, optional expiry date, created time.
A check constraint keeps the remaining quantity at or above zero.

**Movement:** batch, type, signed quantity, balance after, operation id, actor, entered quantity and unit,
conversion factor used, reason, optional recipe id and name snapshot, time.

Types are `addition`, `consumption`, `adjustment`, `discard`, `transfer_out` and `transfer_in`.
A database trigger rejects updates and deletes on movements, so recipe ids are stored without a foreign key.

Rules:
1. A batch balance changes only together with a movement, in one transaction.
2. Affected batches are locked with `FOR UPDATE` in id order, then allocated in domain order.
3. Allocation goes earliest expiry first; undated batches last; ties by creation time, then id.
4. Expired batches are never available. A batch is usable through its expiry date in the pantry's timezone.
5. Insufficient stock rejects the whole operation with a list of shortfalls.
6. A transfer creates a new destination batch with the same expiry. Both movements share an operation id.
7. Adjustments require a reason. Expired stock leaves only when a user confirms a discard.
8. Expiry dates can be corrected; this clears the batch's alerts and is logged. **[call]**

**Idempotency:** stock changes, cooking and shopping-list generation require `Idempotency-Key`.
A record unique on `(user, key)` is inserted in the same transaction, with a request hash and response.
Access is checked before replay. A repeat returns the stored response; a different payload returns 422.
Failed operations roll back and leave the key unused. Records are kept 7 days. **[call]**

## 11. Allocation planner

Matching, cooking and shopping lists share one pure planner, so they cannot disagree.

- Input: requirement lines, batches, conversion mode and factors, and today's date.
- Lines draw from one working snapshot in order, so stock is never counted twice.
- Each remaining requirement is converted into the batch's dimension. Taking the full amount completes the
  line without converting back, which avoids rounding residue.
- Output per line: allocations, shortfall, approximate flag, and a status of `available`, `partial`,
  `missing` or `cannot_compare`. The last applies when a shortfall remains and usable stock could not be compared.

## 12. Cooking, matching and shopping lists

**Cook:** choose recipe, scaling and one pantry. All ingredients are deducted or none. A
`cannot_compare` line rejects the cook with 409.

**Match:** compares the user's recipes with one chosen pantry. `cannot_compare` counts as missing. Ranked by
number of missing ingredients, then coverage (mean of available divided by required, capped at 1),
then name. `max_missing` filters. Recipes and stock each load in one query. **[call: one pantry]**

**Shopping lists:** private, generated from recipes with scaling and an optional pantry.
- The list is a snapshot. Selecting a recipe twice doubles it.
- Shortfalls merge per ingredient; amounts that cannot be converted stay as one line per dimension.
- `cannot_compare` lines are marked "check pantry".
- Generated and manual amounts are stored separately on the same line, so there are no duplicates.
- Regenerating keeps manual amounts and removed lines stay removed. Free-text items are allowed. **[call]**

## 13. Background jobs

Celery beat uses Redis as its broker.
- **Daily, 06:00 UTC:** record `expiring_soon` (within 3 days) and `expired` alerts. Unique on `(batch, kind)`,
  so retries are harmless. Stock is untouched. **[call: time and window]**
- **Daily:** compare batch balances with ledger sums. Drift is logged as an error, never auto-corrected.
- **Daily:** purge old idempotency records and refresh tokens. **[call]**

## 14. Errors, logging, indexes

- Errors use RFC 9457 problem details. Domain errors map to HTTP in one place: 404, 409 or 422.
- Logs are JSON with request, user and operation ids, and never contain tokens or passwords.
- Indexes: active batches on `(pantry, ingredient, expiry)` where remaining is above zero; movements on
  `(batch, time)` and operation; recipe lines on ingredient; trigram on recipe name; foreign keys used in lookups.

## 15. Testing and delivery

- **Unit:** conversions, scaling, planner, permissions. Property tests check that allocation never exceeds stock. **[call: Hypothesis]**
- **Integration, real PostgreSQL:** constraints, ledger immutability, repository queries, concurrent consumption,
  duplicate keys, and leaving a Home during a transfer.
- **E2E:** register, stock, match, cook, shop; create a Home, join, share, leave.
- CI runs Ruff, strict Mypy, migrations up and down, and all tests. Compose runs API, worker, beat, PostgreSQL and Redis.
- Decision records cover the ledger, conversion policy, pantry ownership and idempotency.

## 16. Build order

Foundation and accounts; ingredients and units; recipes and scaling; pantry and ledger; Homes;
planner, cooking and matching; shopping lists; background jobs; final documentation.

## 17. Left to the builder

Endpoint paths, schema names, page sizes, seed contents, exact columns and library choices not named here.

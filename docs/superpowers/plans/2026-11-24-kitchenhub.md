# KitchenHub Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the KitchenHub API: recipes, ingredients, ledger-backed pantries, Homes, cooking, matching and shopping lists.

**Architecture:** A modular monolith whose pure domain layer holds conversion, scaling, allocation and stock rules. Application use cases each own one PostgreSQL transaction. FastAPI and Celery are thin callers.

**Tech Stack:** Python 3.12+, FastAPI, SQLAlchemy 2.0 async + asyncpg, Alembic, Pydantic, PostgreSQL, Redis, Celery, PyJWT, argon2-cffi, structlog, Pytest, Hypothesis, httpx, uv.

**Spec:** `docs/superpowers/specs/2026-11-24-kitchenhub-design.md`

## Global Constraints

- Package root is `src/kitchenhub`. `domain` imports neither FastAPI nor SQLAlchemy.
- Quantities are `Decimal`, JSON strings, and `NUMERIC(18,6)`, rounded half-even when stored. Never float.
- Amounts, scaled results and balances are below 10^12; larger values are rejected with 422.
- Units: g, kg, ml, l, tsp (5 ml), tbsp (15 ml), cup (240 ml), piece.
- Account conversion mode is `strict` or `convert`; default `strict`.
- Inaccessible resources return 404. Errors use RFC 9457 problem details.
- Batch balances change only with a movement in the same transaction.
- Stock changes, cooking and shopping-list generation require `Idempotency-Key`.
- Lock order for stock: membership `FOR SHARE`, idempotency record, batches `FOR UPDATE` by id. Home changes lock the Home row first, then memberships, and read roles only after both locks.
- Time comes from the `get_now() -> datetime` dependency (created in Task 1) and is passed to use cases and domain functions as `now`. JWT expiry alone uses real time.
- Transactions: a use case takes `session: AsyncSession` and opens `async with session.begin()` once. Functions suffixed `_in_tx` run inside a caller's transaction. Authentication uses its own session.
- All routes are under `/api/v1`. Page size defaults to 20, maximum 100, using `limit` and `offset`.
- Logs never contain tokens or passwords.
- PEP 8; Ruff and strict Mypy pass before every commit.
- Commits have Luiz Braga as sole author, with no AI attribution or co-author trailers.

## Review Focus

1. Quantities such as `NaN`, `Infinity`, `0`, or values that round to zero at 6 decimals: rejected with 422 (Task 2, Task 6).
2. A batch added with a past expiry date: recorded, but never available (Task 8).
3. A transfer whose source and destination are the same pantry: 422, no movements (Task 10).
4. A pantry east or west of UTC around midnight: usability follows the pantry's local date (Task 7, Task 8).
5. A shopping list regenerated after one of its recipes was deleted: succeeds using the remaining recipes (Task 14).

## File Map

- `domain/measurements/{units,quantity,conversion}.py`: units, value object, policies
- `domain/recipes/{models,scaling}.py`: recipe invariants and scaling
- `domain/pantry/{batch,planner}.py`: balances, movements, allocation
- `domain/homes/rules.py`, `domain/shopping_lists/lines.py`, `domain/errors.py`
- `application/<module>/use_cases.py`: one per accounts, ingredients, recipes, pantry, homes, matching, shopping_lists, maintenance
- `application/{access,idempotency}.py`: shared access and replay rules
- `application/{recipes,pantry,shopping_lists}/ports.py`: repository protocols (`RecipeRepository`, `StockRepository`, `ShoppingListRepository`)
- `infrastructure/database/models/<module>.py`, `infrastructure/repositories/{stock,recipes,shopping_lists}.py`
- `api/routes/<module>.py`, `api/schemas/<module>.py`
- Each task adds its own Alembic revision under `migrations/versions/`

From Task 1 Step 3 onward, commands assume `docker compose up -d postgres redis`, with `TEST_DATABASE_URL` and `TEST_REDIS_URL` set. Every task ends with this gate, called **GATE**: `uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q`; expected: all succeed.

---

### Task 1: Foundation

**Files:** Create `pyproject.toml`, `uv.lock`, `.gitignore`, `.env.example`, `Dockerfile`, `compose.yaml`, `.github/workflows/ci.yml`, `alembic.ini`, `migrations/env.py`, `src/kitchenhub/{main,config}.py`, `domain/errors.py`, `infrastructure/{logging.py,database/session.py,database/base.py}`, `api/{errors,dependencies}.py`, `api/routes/health.py`, `tests/conftest.py`. Test `tests/unit/test_architecture.py`, `tests/integration/test_health.py`.

**Interfaces, produces:**
- `create_app() -> FastAPI`; `Settings` with `database_url, redis_url, jwt_secret, environment`
- `DomainError(code: str, detail: str, extra: dict[str, object])`; subclasses `NotFound` (404), `Conflict` (409), `Invalid` (422), `Unauthorized` (401), `TooManyRequests` (429)
- `get_session() -> AsyncIterator[AsyncSession]`; fixtures `session`, `session_factory`, `client`. The test session runs `alembic upgrade head`; between tests, fixtures truncate only tables in the model metadata and flush the test Redis database. Fixture `frozen_clock` overrides `get_now`.

- [ ] **Step 1: Write failing tests**

```python
def test_domain_has_no_framework_imports():
    # walk ast of every file in src/kitchenhub/domain
    assert not ({"fastapi", "sqlalchemy", "pydantic"} & imported_top_level_modules)

async def test_health_and_problem_details(client):
    assert (await client.get("/api/v1/health")).json() == {"status": "ok"}
    missing = await client.get("/api/v1/nope")
    assert missing.status_code == 404
    assert missing.headers["content-type"] == "application/problem+json"
    assert missing.headers["x-request-id"]
```

- [ ] **Step 2:** Run `uv run pytest -q`; expected: FAIL, missing package.
- [ ] **Step 3: Implement** the files. `imported_top_level_modules` is a helper in the test. Map `DomainError`, HTTP errors and validation errors in `api/errors.py` only. Request middleware binds `request_id` (authentication later binds `user_id`, stock operations `operation_id`) to structlog JSON output. Compose has `api`, `worker`, `beat`, `postgres:16`, `redis:7` and a one-shot `migrate` service that `api` waits for; `worker` and `beat` use the Celery module from Task 15 and sit behind Compose profile `jobs` until then.
- [ ] **Step 4:** CI runs GATE plus `alembic upgrade head && alembic downgrade base && alembic upgrade head` with PostgreSQL and Redis services.
- [ ] **Step 5:** Run GATE. Commit `chore: add project foundation`.

### Task 2: Measurements

**Files:** Create `domain/measurements/{units,quantity,conversion}.py`. Test `tests/unit/test_measurements.py`.

**Interfaces, produces:**
- `Dimension(StrEnum)`: MASS, VOLUME, COUNT; `Unit(StrEnum)` with `.dimension`, `.factor`; `canonical_unit(dimension) -> Unit`
- `quantize(value: Decimal) -> Decimal` (6 places, half-even)
- `Quantity(amount: Decimal, unit: Unit)`, frozen: `.to(unit) -> Quantity`, `.canonical() -> Quantity`
- `ConversionMode(StrEnum)`: STRICT, CONVERT; `Factors(grams_per_ml: Decimal | None, grams_per_piece: Decimal | None)`
- `Converted(quantity: Quantity, approximate: bool, factor: Decimal | None)`; `convert(quantity, target: Dimension, mode, factors) -> Converted | None` where `None` means cannot compare
- `convert_amount(amount: Decimal, source: Dimension, target: Dimension, mode, factors) -> tuple[Decimal, Decimal | None] | None`: unvalidated canonical amounts and factor, for working values that may be tiny. `convert` wraps it.

- [ ] **Step 1: Write failing tests**

```python
def test_exact_conversion():
    assert Quantity(D("1.5"), Unit.KG).canonical() == Quantity(D("1500"), Unit.G)
    assert Quantity(D("1"), Unit.CUP).to(Unit.TBSP).amount == D("16")

@pytest.mark.parametrize("bad", [D("NaN"), D("Infinity"), D("0"), D("-1"), D("0.0000001"), D("1E+20"), 1.5])
def test_rejects_bad_amounts(bad):
    with pytest.raises(Invalid): Quantity(bad, Unit.G)

def test_cross_dimension():
    cup = Quantity(D("1"), Unit.CUP)
    flour = Factors(D("0.5"), None)
    result = convert(cup, Dimension.MASS, ConversionMode.CONVERT, flour)
    assert (result.quantity.amount, result.approximate) == (D("120"), True)
    assert convert(cup, Dimension.MASS, ConversionMode.STRICT, flour) is None
    assert convert(cup, Dimension.MASS, ConversionMode.CONVERT, Factors(None, None)) is None

def test_count_to_volume_goes_through_mass():
    r = convert(Quantity(D("2"), Unit.PIECE), Dimension.VOLUME, ConversionMode.CONVERT, Factors(D("0.5"), D("50")))
    assert r.quantity == Quantity(D("200"), Unit.ML)
```

- [ ] **Step 2:** Run the file; expected: FAIL on import.
- [ ] **Step 3: Implement.** `factor` is canonical target units per canonical source unit. Same-dimension conversion gives `approximate=False, factor=None`. Compute at precision 28; quantize only results that are stored.
- [ ] **Step 4:** Run GATE. Commit `feat: add measurement conversion`.

### Task 3: Accounts

**Files:** Create accounts `models`, `use_cases`, `routes`, `schemas`, `infrastructure/security.py`, `infrastructure/rate_limit.py`, migration. Test `tests/integration/test_accounts.py`.

**Interfaces, produces:**
- Tables `users` (email stored lowercase, unique; password hash; timezone; conversion_mode), `token_families` (user_id, revoked_at), `refresh_tokens` (family_id, token_hash, expires_at, rotated_at), `pantries` (owner_user_id, home_id without foreign key until Task 9, timezone, check exactly one set)
- `CurrentUser(id: UUID, timezone: str, conversion_mode: ConversionMode)`; dependency `get_current_user() -> CurrentUser`
- Fixture `make_user(email) -> AuthedUser(id, pantry_id, headers)`
- Routes: POST `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`; GET and PATCH `/me`

- [ ] **Step 1: Write failing tests**

```python
async def test_register_creates_private_pantry(client):     # 201; body pantry_id not None
async def test_duplicate_email_ignores_case(client):        # second register == 409
async def test_bad_password_and_unknown_email_match(client): # identical 401 bodies
async def test_refresh_rotates_and_reuse_revokes_family(client):
    # old token reused == 401, then newest token == 401
async def test_expired_access_token_is_401(client):
async def test_expired_refresh_token_is_401(client, frozen_clock):
async def test_reuse_during_rotation_leaves_no_valid_token(session_factory):
async def test_logs_contain_no_password_or_token(client, caplog): # register, login, refresh
async def test_invalid_timezone_is_422(client):             # "Mars/Base"
async def test_sixth_login_attempt_in_a_minute_is_429(client):
async def test_login_works_when_redis_is_down(client, broken_redis): # 200
```

Each comment is the required assertion; `broken_redis` overrides the Redis client to raise `ConnectionError`.

- [ ] **Step 2:** Run the file; expected: FAIL with 404s.
- [ ] **Step 3: Implement.** Access JWT: HS256, 15 minutes, `sub` and `exp`. Refresh: 32 random bytes, SHA-256 stored, 30 days. Passwords 12 to 128 characters. Limit is 5 attempts per minute per email and IP. Rotation, reuse handling and logout lock the `token_families` row `FOR UPDATE`, then re-read the token; a token is valid only while its family is not revoked. Logout takes the refresh token in its body. Authentication binds `user_id` to the log context. Revocation on reuse must commit before the 401 is raised. PATCH `/me` timezone also updates the private pantry.
- [ ] **Step 4:** Run GATE. Commit `feat: add accounts`.

### Task 4: Ingredients

**Files:** Create ingredients `models`, `use_cases`, `routes`, `schemas`, `infrastructure/seed.py`, migration. Test `tests/integration/test_ingredients.py`.

**Interfaces, produces:**
- Tables `ingredients` (owner_user_id nullable, name, description, category, dimension, default_unit, grams_per_ml, grams_per_piece, nutrition JSONB with optional decimal-string keys energy_kcal, protein_g, fat_g, carbs_g), `ingredient_factors` (user_id, ingredient_id, both factors)
- `visible_ingredient_clause(user_id) -> ColumnElement[bool]` (global or owned for now)
- `load_factors(session, user_id, ingredient_ids: set[UUID]) -> dict[UUID, Factors]` with overrides applied field by field
- Routes: `/ingredients` CRUD and search; PUT and DELETE `/ingredients/{id}/factors`; `python -m kitchenhub.infrastructure.seed`

- [ ] **Step 1: Write failing tests**

```python
async def test_catalog_is_read_only(client):            # PATCH global ingredient == 404
async def test_private_ingredient_hidden_from_others(client): # GET by other == 404
async def test_name_unique_ignoring_case(client):       # same owner 409; different owner 201
async def test_default_unit_must_match_dimension(client): # mass + ml == 422
async def test_override_replaces_only_given_factor(session):
async def test_seed_is_repeatable(session):             # row count equal after second run
```

- [ ] **Step 2:** Run the file; expected: FAIL.
- [ ] **Step 3: Implement.** Uniqueness uses two partial indexes on `lower(name)`. Categories: vegetables, fruits, meat, seafood, dairy, grains, spices, oils, baking, other. Seed at least 50 ingredients, with density or piece weight for common ones. Factors are positive. Only the owner may PATCH or DELETE.
- [ ] **Step 4:** Run GATE. Commit `feat: add ingredient catalog`.

### Task 5: Recipes

**Files:** Create `domain/recipes/models.py`, recipes `models`, `use_cases`, `routes`, `schemas`, `infrastructure/repositories/recipes.py`, migration. Test `tests/unit/test_recipe.py`, `tests/integration/test_recipes.py`.

**Interfaces, produces:**
- `RecipeLine(ingredient_id: UUID, quantity: Quantity)`; `Recipe(id, owner_id, name, description, steps, servings, prep_minutes, cook_minutes, difficulty, cuisine, lines, version)`
- `RecipeRepository.get(recipe_id, owner_id) -> Recipe | None`, `.list_for_owner(owner_id) -> list[Recipe]` (three queries: recipes, steps, lines)
- Tables `recipes`, `recipe_steps`, `recipe_lines`, `collections`, `collection_recipes`
- Routes: `/recipes` CRUD and search (`name`, `ingredient_id`, `cuisine`, `max_prep_minutes`); `/collections` CRUD; PUT and DELETE `/collections/{id}/recipes/{recipe_id}`

- [ ] **Step 1: Write failing tests**

```python
def test_invariants():  # servings 0, negative minutes, empty name each raise Invalid
async def test_steps_and_lines_keep_order(client):
async def test_stale_version_is_409(client):
async def test_simultaneous_updates_one_wins(session_factory): # one succeeds, one Conflict
async def test_other_users_recipe_is_404(client):        # GET, PATCH, DELETE
async def test_unknown_or_hidden_ingredient_is_422(client):
async def test_search_combines_filters(client):          # all four filters return only the match
async def test_search_matches_misspelled_name(client):   # "spagetti" finds "Spaghetti"
async def test_collection_rejects_others_recipe(client): # 404
async def test_referenced_ingredient_cannot_be_deleted(client): # 409
```

- [ ] **Step 2:** Run both; expected: FAIL.
- [ ] **Step 3: Implement.** Difficulty: easy, medium, hard. Updates use `UPDATE ... WHERE version = :version`. Define `RecipeRepository` as a protocol in `application/recipes/ports.py` and modify the ingredients module for visibility. Cuisine is stored trimmed and lowercase. Enable `pg_trgm` with a GIN index on name; index `recipe_lines.ingredient_id`. Name search combines `ILIKE` with the `%` similarity operator. The same ingredient may appear on several lines. Extend `visible_ingredient_clause` with ingredients used by the user's own recipes.
- [ ] **Step 4:** Run GATE. Commit `feat: add recipes and collections`.

### Task 6: Scaling up and down

**Files:** Create `domain/recipes/scaling.py`; modify recipes `use_cases`, `routes`, `schemas`. Test `tests/unit/test_scaling.py`, extend `test_recipes.py`.

**Interfaces, produces:**
- `Scaling(servings: int | None = None, multiplier: Decimal | None = None)`; exactly one set
- `scale_lines(recipe: Recipe, scaling: Scaling | None) -> tuple[RecipeLine, ...]`; `None` means unscaled
- Schema `ScalingInput`, reused by Tasks 12 to 14; route POST `/recipes/{id}/scale`

- [ ] **Step 1: Write failing tests**

```python
@pytest.mark.parametrize("scaling,expected", [
    (Scaling(servings=6), ["600", "300", "150"]),
    (Scaling(servings=2), ["200", "100", "50"]),
    (Scaling(multiplier=D("0.5")), ["200", "100", "50"]),
    (Scaling(multiplier=D("2")), ["800", "400", "200"]),
])
def test_scales_up_and_down(pasta, scaling, expected):  # pasta: 4 servings, 400/200/100 g
    assert [l.quantity.amount for l in scale_lines(pasta, scaling)] == [D(x) for x in expected]

@pytest.mark.parametrize("kw", [{}, {"servings": 2, "multiplier": D("2")}, {"servings": 0},
                                {"multiplier": D("0")}, {"multiplier": D("NaN")}])
def test_invalid_scaling(kw):
    with pytest.raises(Invalid): Scaling(**kw)

def test_result_rounding_to_zero_is_invalid():  # 0.000001 g times 0.1 raises Invalid
async def test_scaling_does_not_change_recipe(client):  # GET before == GET after
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** Compute `amount * target / original`, multiplying first, then quantize. Keep each line's unit. Times are unchanged.
- [ ] **Step 4:** Run GATE. Commit `feat: scale recipes up and down`.

### Task 7: Batches and allocation planner

**Files:** Create `domain/pantry/{batch,planner}.py`. Test `tests/unit/test_batch.py`, `tests/unit/test_planner.py`.

**Interfaces, produces:**
- `MovementType(StrEnum)`: ADDITION, CONSUMPTION, ADJUSTMENT, DISCARD, TRANSFER_OUT, TRANSFER_IN
- `Batch(id, pantry_id, ingredient_id, dimension, remaining: Decimal, expires_on: date | None, created_at)`; `.is_usable(today) -> bool`; `.apply(delta: Decimal) -> Batch` raises `Conflict("insufficient_stock")` below zero
- `local_today(timezone: str, now: datetime) -> date`
- `Allocation(batch_id, amount: Decimal, factor: Decimal | None)`
- `LineStatus(StrEnum)`: AVAILABLE, PARTIAL, MISSING, CANNOT_COMPARE
- `LineResult(line: RecipeLine, allocations, shortfall: Decimal, approximate: bool, status)`; shortfall uses the line's canonical unit
- `plan(lines, batches, mode, factors: Mapping[UUID, Factors], today) -> list[LineResult]`

- [ ] **Step 1: Write failing tests**

```python
def test_earliest_expiry_first_undated_last():
def test_expired_yesterday_skipped_but_today_usable():
def test_local_today_differs_by_timezone():
    now = datetime(2026, 1, 1, 23, 30, tzinfo=UTC)
    assert local_today("Asia/Tokyo", now) == date(2026, 1, 2)
    assert local_today("America/Sao_Paulo", now) == date(2026, 1, 1)
def test_two_lines_do_not_double_count():       # 100 g stock; 80 g + 80 g -> AVAILABLE, PARTIAL shortfall 60
def test_strict_flags_incomparable_stock():     # ml line, only g stock -> CANNOT_COMPARE
def test_strict_sufficient_same_dimension_is_available():
def test_convert_allocates_in_batch_dimension(): # 1 cup, density .5 -> allocation 120, approximate
def test_full_conversion_leaves_no_residue():   # density 0.333333 -> shortfall == 0
def test_no_stock_is_missing():
def test_tiny_first_batch_does_not_raise():     # 1 piece at 1500 g/piece; batches 0.0005 g then 2000 g -> AVAILABLE

@given(stock_and_lines())
def test_never_allocates_more_than_stock(case):
    # per batch: sum(allocations) <= remaining; every allocation > 0
```

- [ ] **Step 2:** Run; expected: FAIL on import.
- [ ] **Step 3: Implement `plan`:**

```
snapshot = {batch.id: remaining}
for line: need = canonical amount; blocked = False
  for batch in usable batches of ingredient, ordered (expires_on or date.max, created_at, id):
    skip if snapshot is 0; stop if need is 0
    converted = convert_amount(need, line dimension, batch.dimension, ...); None -> blocked = True; continue
    wanted = quantize(converted amount)
    if wanted <= snapshot: take wanted; need = 0
    else: take snapshot; need -= the taken amount converted back; need never below 0
    record take when > 0
  shortfall = quantize(need)   # zero is allowed here
  status: shortfall 0 AVAILABLE; blocked CANNOT_COMPARE; any allocation PARTIAL; else MISSING
```

- [ ] **Step 4:** Run GATE. Commit `feat: add batch rules and allocation planner`.

### Task 8: Ledger, idempotency and stock API

**Files:** Create `application/{access,idempotency}.py`, pantry `models`, `use_cases`, `routes`, `schemas`, `infrastructure/repositories/stock.py`, migration. Test `tests/integration/{test_ledger,test_idempotency,test_stock_concurrency}.py`.

**Interfaces, produces:**
- Tables `batches`, `movements` (fields in spec section 10), `idempotency_records` (user_id, key, request_hash, response JSONB, status, created_at)
- `require_pantry(session, user_id, pantry_id) -> Pantry` raises `NotFound`; locks membership once Homes exist
- `run_idempotent(session, user_id, key, fingerprint: str, status: int, work: Callable[[], Awaitable[dict[str, object]]]) -> StoredResponse`, in caller's transaction
- `StockRepository.lock_batches(pantry_id, ingredient_ids)`, `.lock_batch(pantry_id, batch_id)`, `.read_batches(pantry_id)`, `.add_batch(batch)`, `.record(batch, MovementDraft) -> Batch`. `record` is the only writer of balances. The protocol lives in `application/pantry/ports.py`.
- `MovementDraft(type, delta: Decimal, operation_id, actor_id, entered: Quantity | None, factor: Decimal | None, reason: str | None, recipe_id, recipe_name)`. When one line spans batches, every movement repeats that line's entered quantity.
- `/consume` body: `lines` (ingredient_id, quantity, unit), optional `mode`, optional `reason`
- `consume_in_tx(session, user, pantry, lines, mode, reason, recipe: Recipe | None, now: datetime) -> ConsumeResult(operation_id, results: list[LineResult])`
- Routes: GET `/pantries` (all accessible); under `/pantries/{id}`: GET and POST `/batches`; POST `/consume`; POST `/batches/{id}/adjust`, `/batches/{id}/discard`; PATCH `/batches/{id}` (expiry); GET `/movements`

- [ ] **Step 1: Write failing tests**

```python
async def test_every_change_matches_ledger(client, session):  # add, consume, adjust, discard; remaining == sum(delta)
async def test_movements_reject_update_and_delete(session):   # both raise DBAPIError
async def test_negative_remaining_rejected_by_database(session):
async def test_shortfall_rejects_whole_operation(client):     # 409 lists shortfalls; movement count unchanged
async def test_past_expiry_batch_recorded_but_unavailable(client): # add 201; consume 409
async def test_expiry_uses_pantry_timezone(client, frozen_clock):
async def test_adjustment_requires_reason(client):            # 422
async def test_bad_adjustment_delta_is_422(client):           # NaN, Infinity, 0, 1E+20
async def test_balance_over_limit_is_422(client)
async def test_expired_batch_rejects_negative_adjustment(client) # 409
async def test_discarding_empty_batch_is_409(client)
async def test_stock_log_has_operation_id(client, caplog)
async def test_other_users_pantry_is_404(client):
async def test_missing_key_is_422(client):
async def test_replay_returns_same_body_and_one_movement(client):
async def test_same_key_different_payload_is_422(client):
async def test_failed_operation_leaves_key_unused(client):
async def test_parallel_consumes_never_overdraw(session_factory): # 100 g, 5 x 30 g: 3 succeed, 2 Conflict, remaining 10
async def test_parallel_same_key_applies_once(session_factory)
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** The record is inserted with `ON CONFLICT DO NOTHING`; on conflict, read and compare the SHA-256 of method, path and canonical JSON body. A trigger rejects UPDATE and DELETE on `movements`. Adjust takes a finite, nonzero signed delta with a unit in the batch's dimension; a negative adjustment of an expired batch is refused. Discard takes the full remainder. Modify the ingredients module so visibility also covers ingredients with remaining stock in accessible pantries. Create the partial index `(pantry_id, ingredient_id, expires_on) WHERE remaining > 0` and indexes on `(batch_id, created_at)` and `operation_id`. Expiry edits are logged.
- [ ] **Step 4:** Run GATE. Commit `feat: add inventory ledger`.

### Task 9: Homes

**Files:** Create `domain/homes/rules.py`, homes `models`, `use_cases`, `routes`, `schemas`, migration; modify `application/access.py`, ingredient visibility. Test `tests/unit/test_home_rules.py`, `tests/integration/test_homes.py`.

**Interfaces, produces:**
- Tables `homes` (archived_at), `memberships` (unique home and user, role), `invitations` (code_hash, expires_at, used_at); foreign key on `pantries.home_id`
- `require_membership(session, user_id, home_id, *, for_update=False) -> Membership`
- `domain/homes/rules.py`: `can_invite(role)`, `can_remove(actor_role, is_self)`, `can_leave(role)`, `can_archive(role, member_count, has_stock)`, `can_transfer(role, target_is_member)`, all `-> bool`
- Fixture `make_home(owner, *members) -> HomeRef(id, pantry_id)`
- Routes: `/homes` create, list, get; POST `/homes/{id}/invitations`, `/invitations/accept`, `/homes/{id}/transfer-ownership`, `/homes/{id}/leave`, `/homes/{id}/archive`; DELETE `/homes/{id}/members/{user_id}`

- [ ] **Step 1: Write failing tests**

```python
async def test_create_makes_shared_pantry_and_moves_nothing(client):
async def test_join_by_code_then_code_is_spent(client):       # second use 409
async def test_expired_code_is_409(client, frozen_clock):     # 7 days + 1 second
async def test_member_cannot_invite_or_remove(client):        # 409
async def test_private_pantry_hidden_from_home_members(client): # 404
async def test_shared_movement_records_acting_member(client)
async def test_leaving_keeps_stock_and_ends_access(client):   # leaver 404; batch remains
async def test_owner_cannot_leave(client):                    # 409
async def test_handover_then_former_owner_leaves(client)
async def test_handover_racing_leave_keeps_one_owner(session_factory)
async def test_removing_self_is_409(client)
async def test_accept_when_member_is_409_and_code_unspent(client)
async def test_archive_racing_add_never_strands_stock(session_factory) # archive or add fails
async def test_home_timezone_defaults_to_creator_and_can_change(client)
def test_rule_table():  # every rule, for owner and member
async def test_archive_needs_sole_member_and_empty_pantry(client) # 409 each; 204 when both hold
async def test_private_ingredient_visible_through_home_pantry(client)
async def test_shared_ingredient_core_fields_locked(client):  # density 409; description 200
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** Codes are 16 random URL-safe bytes, SHA-256 stored. Accept, leave, remove, transfer and archive lock the Home row `FOR UPDATE`, then affected membership rows `FOR UPDATE`, then read roles. Archive locks the owner's membership before checking stock. `require_pantry` takes `FOR SHARE` on membership, then reads `archived_at` in a separate statement and returns 404 if set. PATCH `/homes/{id}` lets the owner change name and timezone. A private ingredient's core fields lock once another user's recipe uses it, or once it has ever been held in a pantry of a Home with another member. Stock through a Home pantry is added with the Task 8 routes.
- [ ] **Step 4:** Run GATE. Commit `feat: add Homes`.

### Task 10: Sharing stock between pantries

**Files:** Modify pantry `use_cases`, `routes`, `schemas`. Test `tests/integration/test_transfers.py`.

**Interfaces, produces:** POST `/pantries/{id}/batches/{batch_id}/transfer` with `destination_pantry_id`, `quantity` (decimal string) and `unit`.

- [ ] **Step 1: Write failing tests**

```python
async def test_share_creates_linked_movements(client):  # out -500, in +500, same operation_id, same expiry
async def test_member_can_move_stock_back(client)
async def test_nonmember_destination_is_404(client)
async def test_same_pantry_is_422_without_movements(client)
async def test_over_remaining_is_409_and_moves_nothing(client)
async def test_different_dimension_is_422(client)
async def test_expired_batch_cannot_be_transferred(client)  # 409
async def test_leave_waits_for_running_transfer(session_factory):
    # transfer holds membership lock; leave completes only after commit; both movements exist
async def test_transfer_replay_moves_once(client)
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** Check access to both pantries in id order before locking the source batch. Always create a new destination batch.
- [ ] **Step 4:** Run GATE. Commit `feat: share stock with Homes`.

### Task 11: Query counts and indexes

**Files:** Create `tests/integration/test_query_counts.py`, `tests/integration/test_index_usage.py`.

**Interfaces, produces:** fixture `count_queries()` yielding a counter with `.value`.

- [ ] **Step 1: Write tests**

```python
async def test_recipe_list_query_count_is_constant(client): # 1 recipe and 20 recipes: equal count
async def test_batch_list_query_count_is_constant(client)
async def test_active_batch_index_is_usable(session):       # EXPLAIN, enable_seqscan=off, names the index
async def test_recipe_name_index_is_usable(session)         # EXPLAIN with only the name predicate
```

- [ ] **Step 2:** Run; fix any failing query until both files pass.
- [ ] **Step 3:** Run GATE. Commit `test: pin query counts and indexes`.

### Task 12: Cooking

**Files:** Modify recipes `use_cases`, `routes`, `schemas`. Test `tests/integration/test_cooking.py`.

**Interfaces:** Consumes `scale_lines`, `consume_in_tx`, `run_idempotent`. Produces POST `/recipes/{id}/cook` with `pantry_id`, optional `scaling`, optional `mode`.

- [ ] **Step 1: Write failing tests**

```python
async def test_cook_deducts_every_line(client)
async def test_one_shortfall_deducts_nothing(client)        # 409; balances unchanged
async def test_half_multiplier_deducts_half(client)
async def test_cannot_compare_is_409_then_convert_succeeds(client) # movement keeps factor
async def test_cook_from_home_pantry_records_actor(client)
async def test_retry_deducts_once(client)
async def test_movement_survives_recipe_deletion(client)    # id and name snapshot remain
async def test_retry_after_recipe_deletion_replays(client)
async def test_recipe_without_lines_is_409(client)
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement** by loading the recipe inside `work` (after the replay check), scaling, then calling `consume_in_tx`. Default mode is the user's.
- [ ] **Step 4:** Run GATE. Commit `feat: cook recipes from a pantry`.

### Task 13: Matching

**Files:** Create `domain/recipes/matching.py`, matching `use_cases`, `routes`, `schemas`. Test `tests/unit/test_matching.py`, `tests/integration/test_matching.py`.

**Interfaces, produces:**
- `RecipeMatch(recipe, results, missing_count: int, coverage: Decimal, approximate: bool)`; `match(recipe, results) -> RecipeMatch`; `rank(matches) -> list[RecipeMatch]`
- GET `/pantries/{id}/matches` with `max_missing`, `mode`, optional `servings` or `multiplier`

- [ ] **Step 1: Write failing tests**

```python
def test_missing_counts_distinct_ingredients()   # two short lines, same ingredient -> 1
def test_cannot_compare_counts_as_missing()
def test_coverage():                             # 100/100 and 50/100 -> 0.75
def test_coverage_ignores_line_order():          # lines 10 and 90, stock 50 -> 0.5 both ways
def test_rank_order():                           # missing, coverage descending, name, recipe id
async def test_max_missing_and_shortfalls(client)
async def test_recipe_without_lines_is_excluded(client)
async def test_query_count_is_constant(client)   # 1 vs 20 recipes
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** Coverage per ingredient is `(sum required - sum shortfall) / sum required`, computed per dimension and averaged if an ingredient has lines in several. A recipe whose scaled line rounds to zero is excluded. Run the planner separately per recipe on the same batches. Paginate after ranking.
- [ ] **Step 4:** Run GATE. Commit `feat: match recipes to pantry stock`.

### Task 14: Shopping lists

**Files:** Create `domain/shopping_lists/lines.py`, shopping_lists `models`, `use_cases`, `routes`, `schemas`, `repositories/shopping_lists.py`, migration. Test `tests/unit/test_shopping_lines.py`, `tests/integration/test_shopping_lists.py`.

**Interfaces, produces:**
- `ShoppingLine(ingredient_id | None, label | None, dimension | None, generated, manual, check_pantry, approximate, removed, checked)`; total is `generated + manual`
- `merge_shortfalls(results, mode, factors, ingredient_dimensions) -> list[ShoppingLine]`
- Tables `shopping_lists` (owner, name, pantry_id, mode), `shopping_list_recipes` (recipe_id, servings, multiplier, position; recipe deletion cascades), `shopping_lines` (the `ShoppingLine` fields; partial unique on list, ingredient, dimension)
- Generate body: `name`, `recipes` (recipe_id, optional scaling), optional `pantry_id`, optional `mode`. PATCH item body: `manual`, `checked`. `ShoppingListRepository` protocol in `application/shopping_lists/ports.py`
- Routes: `/shopping-lists` generate, list, get, delete; POST `/{id}/regenerate`, `/{id}/items`; PATCH and DELETE `/{id}/items/{item_id}`

- [ ] **Step 1: Write failing tests**

```python
def test_merges_same_ingredient_across_recipes()  # 200 g + 300 g -> one 500 g line
def test_unconvertible_dimensions_stay_separate()
async def test_subtracts_pantry_once_across_recipes(client) # 100 g stock, 80 + 80 -> buy 60
async def test_without_pantry_lists_everything(client)
async def test_selecting_recipe_twice_doubles(client)
async def test_scaled_selection(client)                     # 0.5 -> half
async def test_cannot_compare_is_marked(client)
async def test_manual_amount_joins_generated_line(client)
async def test_regenerate_keeps_manual_and_removed(client)
async def test_regenerate_after_recipe_deleted(client)      # 200; remaining recipe only
async def test_free_text_item(client)
async def test_manual_other_dimension_is_separate_in_strict(client)
async def test_readding_removed_item_starts_from_new_amount(client)
async def test_fully_stocked_line_is_hidden(client)
async def test_regenerate_racing_manual_edit_keeps_both(session_factory)
async def test_list_is_private_even_in_home(client)         # 404
async def test_generation_replay_creates_one_list(client)
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** One planner call receives all selections' lines in selection order. In `convert` mode merge into the ingredient's dimension when possible. Regenerate rechecks pantry access and resets only generated fields. DELETE sets `removed`; re-adding clears it, sets `manual` to the new amount and `generated` to 0. A manual amount merges into the ingredient's dimension in `convert` mode when a factor exists; otherwise it uses its own dimension's line. Responses omit removed lines and ingredient lines whose total is zero. Every write locks the list row `FOR UPDATE`. Regenerate requires `Idempotency-Key`.
- [ ] **Step 4:** Run GATE. Commit `feat: add shopping lists`.

### Task 15: Background jobs

**Files:** Create `workers/{celery_app,tasks}.py`, `application/maintenance/use_cases.py`, alert model, migration; modify `compose.yaml`, pantry `use_cases` and `routes`. Test `tests/integration/test_maintenance.py`.

**Interfaces, produces:** `scan_expiry(session, now) -> int`, `find_drift(session) -> list[UUID]`, `purge_expired(session, now) -> int`; table `expiry_alerts` unique on `(batch_id, kind)`; GET `/pantries/{id}/expiry-alerts`.

- [ ] **Step 1: Write failing tests**

```python
async def test_scan_flags_three_days_not_four(session)
async def test_scan_flags_expired_and_leaves_stock(session)  # remaining and movement count unchanged
async def test_scan_twice_creates_one_alert(session)         # second returns 0
async def test_empty_batches_are_ignored(session)
async def test_scan_uses_pantry_timezone(session)
async def test_alerts_hide_consumed_batches(client)
async def test_changing_expiry_clears_alerts(client)
async def test_drift_is_reported_not_corrected(session)      # raw SQL changes remaining; id returned; value kept
async def test_purge_keeps_recent_records(session)           # keys: 8 days deleted, 6 kept; refresh tokens: deleted once expired
def test_beat_schedule():                                    # scan at 06:00 UTC; three daily tasks
```

- [ ] **Step 2:** Run; expected: FAIL.
- [ ] **Step 3: Implement.** Tasks call use cases through `asyncio.run` with a `NullPool` engine created and disposed inside. Tasks retry three times on `OperationalError` or `OSError`. Each pantry's local date decides alerts. Remove the Compose `jobs` profile and give the worker a ping healthcheck; `migrate` is a completed dependency of the worker and beat.
- [ ] **Step 4:** Run GATE, then `docker compose up -d --build --wait && docker compose exec worker celery -A kitchenhub.workers.celery_app inspect ping`; expected: `pong`. Commit `feat: add background jobs`.

### Task 16: End-to-end journeys and documentation

**Files:** Create `tests/e2e/test_journeys.py`, `README.md`, `docs/architecture.md`, `docs/decisions/{0001-ledger,0002-conversion,0003-pantry-ownership,0004-idempotency}.md`.

- [ ] **Step 1: Write tests** over HTTP only:

```python
async def test_private_journey(client):
    # register, add stock, create recipe, match missing_count 0, cook at 0.5,
    # balances fall by half the recipe's unscaled requirement, shopping list at 2 shows exact shortfall
async def test_home_journey(client):
    # create, invite, join, share stock, member cooks, member leaves; stock stays, access 404
async def test_openapi_documents_problem_details_and_idempotency(client)
```

- [ ] **Step 2:** Run `uv run pytest tests/e2e -v`; fix until all pass.
- [ ] **Step 3: Write docs.** The README covers purpose, quick start, test commands and a short API tour. Each decision record has context, decision, alternatives and consequences from the spec.
- [ ] **Step 4:** Follow the README quick start from a clean clone: `/api/v1/health` returns `{"status": "ok"}` and registering a user returns 201.
- [ ] **Step 5:** Run GATE. Commit `docs: add architecture and decision records`.

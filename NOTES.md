# Sanctum Sanctorum — Project Submission Notes

## 🌐 Live Deployment
- **Deployment URL**: `https://sanctum-sanctorum-main-pp3g.onrender.com/`
- **Interactive API Docs (Swagger)**: `https://sanctum-sanctorum-main-pp3g.onrender.com/docs`
- **Hosted Database**: Neon Serverless PostgreSQL
- **Default Seed Accounts**:
  - `wong@example.com` (Supreme Tier)
  - `christine@example.com` (Master Tier)
  - `jonathan@example.com` (Adept Tier)
  - `sara@example.com` (Apprentice Tier)

---

## 🎯 What Was Finished vs. What Was Skipped
- **Books Module (100% Complete)**:
  - ISBN-13 checksum validation and digit normalization.
  - Duplicate ISBN conflict rejection (`409 Conflict`).
  - Search query `q` with case-insensitive substring matching across title and author.
  - Filtering (`restricted`, `min_price`, `max_price`), sorting (`title`, `-title`, `price`, `-price` with `id` tie-breaker), and pagination (`limit`, `offset`, accurate pre-pagination `total` count).
  - Partial updates via `PATCH /books/{id}` (ignoring `isbn` and rejecting nulls).
- **Members Module (100% Complete)**:
  - Whitespace trimming, email normalization (lowercased + regex validated), and duplicate email detection (`409`).
  - Strict tier ranking fix (`tier_at_least` hierarchy).
  - Member orders listing and full member stats calculation (`orders_paid`, `total_spent_cents`, `active_loans`, `overdue_loans`, `late_fees_cents`).
- **Orders Module (100% Complete)**:
  - Input validation (non-empty items, quantity $\ge 1$, duplicate `book_id` rejection).
  - Tier-based permission checks for restricted titles.
  - Atomic stock reservation (all-or-nothing stock decrements).
  - Tiered discounts ($0\%, 5\%, 10\%, 15\%$) + $5\%$ bulk volume discount for orders with $\ge 10$ items (integer floor rounding).
  - Payment and cancellation lifecycle (`pending` $\to$ `paid` / `cancelled` with inventory restoration).
- **Loans Module (100% Complete)**:
  - ORM model completion (`due_at`, `returned_at`, `late_fee_cents`).
  - Tier borrowing limits (Apprentice: 1, Adept: 3, Master: 5, Supreme: unlimited).
  - Dynamic status evaluation (`active`, `overdue`, `returned`) with strict due date boundary logic.
  - Return workflow with late fee calculation at $25¢/\text{day}$ (partial days rounded up to full days via ceiling, capped at the book's current price).
  - Member loans filtering by computed status.
- **Reports Module (100% Complete)**:
  - `GET /reports/top-books` aggregated across paid orders, ordered by `copies_sold DESC, title ASC`.

**Test Suite Coverage**:
- **202 / 202 acceptance tests passing (100%)**.

---

## 🏛️ Architecture & Design Decisions
1. **Layer Separation**:
   - **Routers (`app/routers/`)**: Pure presentation layer responsible only for parameter parsing, dependency injection, and HTTP status codes.
   - **Services (`app/services/`)**: Centralizes all domain business logic, data constraints, pricing formulas, and transactional mutations.
   - **Schemas (`app/schemas.py`)**: Handles data sanitization, field-level normalization (trimming, regex, checksums), and response contract formatting.
   - **Models (`app/models.py`)**: SQLAlchemy 2.x declarative ORM models.
2. **Deterministic Time (`app/clock.py`)**:
   - Strictly enforced `Depends(get_now)` for all timestamping (`created_at`, `borrowed_at`, `due_at`, `returned_at`, and late fee calculations) to guarantee 100% deterministic test executions.
3. **Database Portability**:
   - Maintained zero-configuration SQLite support for local testing (`sqlite://`), while supporting PostgreSQL (`postgresql+psycopg`) in cloud production with proper driver connect args handling.

---

## 🔍 Spec Feedback & Observations
- **Tier Ranking Boundary**: The starter code had `TIER_ORDER.index(tier) > TIER_ORDER.index(minimum)` which prevented members at the exact `minimum` tier (such as `master`) from accessing restricted titles. Correcting this to `>=` aligned the service with the specification requirements.
- **Strict Due Date Boundary**: Spec clarified that a loan checked at the exact second of `due_at` is still considered `active` and owes 0 late fee, which was cleanly handled using strict `now > due_at` comparisons.

---

## 🤖 AI Usage
- **Tools Used**: Antigravity IDE coding assistant (Gemini 3.7 Flash).
- **Use Cases**:
  - Scaffolding the implementation plan and verifying test suite assertions.
  - Generating initial query structures and validating Pydantic v2 field constraint behaviors.
  - Troubleshooting cloud deployment database driver dependencies (`psycopg` vs `psycopg2-binary`).
- **Where AI Was Overridden / Refined**:
  - The default Pydantic model dump included all fields on patch updates; refined it to use `exclude_unset=True` with explicit null rejection validators to strictly follow the PATCH specification.
  - Ensured that `late_fee_cents` calculation uses ceiling division over total seconds (`math.ceil((now - due_at).total_seconds() / 86400)`) rather than integer truncated division so that any partial day is properly counted as a full billable day.

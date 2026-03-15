# Universal AI Coding Protocol

**Version:** 1.0
**Applies to:** All development — human, AI-assisted, and fully autonomous
**Principle:** Code must be correct by default. Silence is not consent. Absence is not evidence. Unknown is not safe.

---

## How to Use This Protocol

This protocol is designed to be dropped into any project that uses AI coding agents. It works with any tech stack and any AI tool.

**Claude Code / Claude Projects:** Place this file as `CODING_PROTOCOL.md` in your project root and reference it in `.claude/CLAUDE.md`:
```
**MANDATORY:** Before writing any code, read and internalize `CODING_PROTOCOL.md` in the project root. Every line of code you produce must comply with it.
```

**Cursor:** Add the cardinal rules (Section 1) to `.cursorrules` in your project root. Reference this file for the full protocol.

**GitHub Copilot:** Use `.github/copilot-instructions.md` to embed the cardinal rules, with a pointer to this file.

**Any LLM-based tool:** Include the cardinal rules in your system prompt. The agent can only follow rules it can see.

---

## 0. Why This Exists

AI coding agents produce sophisticated, well-structured, well-tested code that can still produce wrong results. The problem is never "bad code." The problem is **bad defaults, untested invariants, and optimism where skepticism is required.**

AI agents are trained on millions of examples of working code. This creates systematic biases:

- **Optimism Bias:** Agents underweight failure paths because they are trained on code that works.
- **Completion Bias:** Agents return reasonable-looking defaults rather than raising errors when data is missing.
- **Symmetry Bias:** Agents assign middle values to middle concepts ("skipped" gets 0.5 because it feels halfway between "pass" and "fail").
- **Demo-Path Bias:** Agents build features by imagining the demo, not the adversarial case.

This protocol counteracts those biases. Every rule traces back to a class of bug that AI agents produce repeatedly across teams, stacks, and domains.

---

## 1. The Cardinal Rules

These are non-negotiable. Every code review must verify compliance.

### 1.1 Default to Deny

Every scoring system, validation check, access control, and state transition must default to the restrictive outcome.

```python
# WRONG — treats unknown as trusted
def compute_score(results):
    if not results:
        return 100.0  # "Nothing failed"

# RIGHT — treats unknown as unverified
def compute_score(results):
    if not results:
        return 0.0  # "Nothing was checked"
```

**The test:** If all external data disappears — the database returns empty, the API times out, the cache is cold — does the system lock down or open up? If it opens up, the default is wrong.

### 1.2 Absence Is Not Evidence

"Not found" is not the same as "verified clean." "Skipped" is not the same as "passed." "No error returned" is not the same as "succeeded."

Every function that checks for something must have three distinct return paths:

```python
# WRONG — two-state thinking
if record_exists:
    return MATCH
else:
    return NO_MATCH  # But is this "checked and absent" or "couldn't check"?

# RIGHT — three-state thinking
if record_exists:
    return MATCH
elif lookup_succeeded:
    return NO_MATCH  # We checked. It's not there.
else:
    return UNKNOWN   # We couldn't check. This is NOT a pass.
```

### 1.3 Skipped Work Gets Zero Credit

If a validation rule, test, or check could not execute (missing data, unavailable service, unsupported format), its contribution to any aggregate score must be **zero**, not partial credit.

```python
# WRONG
STATUS_WEIGHTS = {
    "pass": 1.0,
    "warning": 0.7,
    "skipped": 0.5,  # Half credit for doing nothing
    "fail": 0.0,
}

# RIGHT
STATUS_WEIGHTS = {
    "pass": 1.0,      # Full credit: check ran and succeeded
    "warning": 0.7,   # Partial credit: check ran, minor issue found
    "skipped": 0.0,   # ZERO: check did not run; no information gained
    "fail": 0.0,      # Zero: check ran and found a problem
    "error": 0.0,     # Zero: check crashed; treat as unverified
}
# NOTE: "skipped" is 0.0, not 0.5, because a check that didn't execute
# provides NO assurance. The absence of a failure is not a success.
```

### 1.4 Every Button Must Do Something

No UI element should render without a working handler. If the handler isn't implemented yet, the element must be visually disabled with a tooltip explaining why.

```jsx
// WRONG — button exists, does nothing
<button className="btn-primary">Approve</button>

// RIGHT — either wired or explicitly disabled
<button onClick={handleApprove} disabled={isProcessing}>
  {isProcessing ? 'Processing...' : 'Approve'}
</button>

// ALSO RIGHT — if handler isn't built yet
<button disabled title="Approval workflow not yet implemented">
  Approve (Coming Soon)
</button>
```

---

## 2. Defensive Coding Standards

### 2.1 Mutable Defaults

Never use mutable objects as default arguments. Defaults must not carry state between invocations.

```python
# WRONG — shared across all calls
def process(items: list = [], config: dict = {}):

# RIGHT
def process(items: list | None = None, config: dict | None = None):
    items = items if items is not None else []
    config = config if config is not None else {}
```

### 2.2 Float Arithmetic

Never use `==` or `%` with floating-point numbers in business logic. Financial calculations must use `Decimal` or integer cents.

```python
# WRONG — fails for 50000.00000000001
is_round = total % 1000 == 0

# RIGHT
from decimal import Decimal
is_round = Decimal(str(total)) % 1000 == 0
```

### 2.3 Path Traversal

Every file-serving endpoint must validate that the resolved path is within the expected directory. Never trust paths stored in a database.

```python
resolved = Path(user_supplied_path).resolve()
allowed = Path(settings.UPLOAD_DIR).resolve()
if not str(resolved).startswith(str(allowed)):
    raise PermissionError("Path traversal blocked")
```

### 2.4 Cleanup on Unmount

Every component that opens a connection, creates a blob URL, starts a timer, or subscribes to an event must clean up in its teardown/unmount path.

```tsx
// WRONG
useEffect(() => {
  initWebSocket()
  return () => {
    // "Don't destroy — keep connection alive"
  }
}, [])

// RIGHT
useEffect(() => {
  initWebSocket()
  return () => {
    destroyWebSocket()
    revokeAllBlobUrls()
  }
}, [])
```

### 2.5 Closure Freshness

In React effects, never reference state variables in cleanup functions. Use local variables scoped to the effect.

```tsx
// WRONG — pdfUrl is stale in cleanup
useEffect(() => {
  const url = URL.createObjectURL(blob)
  setPdfUrl(url)
  return () => { URL.revokeObjectURL(pdfUrl) }  // stale!
}, [id])

// RIGHT — local variable captures the right value
useEffect(() => {
  let blobUrl: string | null = null
  blobUrl = URL.createObjectURL(blob)
  setPdfUrl(blobUrl)
  return () => { if (blobUrl) URL.revokeObjectURL(blobUrl) }
}, [id])
```

### 2.6 Serialization Consistency

All fields of the same logical type must serialize the same way. If `created_at` uses `.isoformat()`, then `updated_at` must too. No exceptions.

---

## 3. State Machine Discipline

### 3.1 Status Fields Must Be Enums

Never use bare strings for status, state, or type fields. Define exhaustive enums and handle every variant.

### 3.2 Every State Transition Must Be Explicit

If a record moves from "processing" to "completed," there must be a function that performs exactly that transition and nothing else. No implicit status changes buried in unrelated business logic.

### 3.3 Display Must Derive from Data, Not Mode

The UI must show the *result* of an operation, not the *mode* it was configured for.

```
# WRONG — shows mode as if it were result
"Verification Complete" (because mode was set to "verify")

# RIGHT — shows actual outcome
"Verification Failed — reference document not found"
"Verification: 94.2%" (because all checks actually ran and scored)
```

---

## 4. Scoring and Aggregation Systems

Any system that computes a composite score (validation, risk, quality, compliance) must follow these rules:

### 4.1 The Denominator Problem

When aggregating scores, the denominator must only include checks that actually executed. Skipped or errored checks must be excluded from both numerator and denominator.

```python
# WRONG — skipped rules inflate denominator or contribute partial credit
for rule in results:
    total_weight += weights[rule.id]
    if rule.status == "pass": earned += weights[rule.id]
    elif rule.status == "skipped": earned += weights[rule.id] * 0.5

# RIGHT — only count what actually ran
for rule in results:
    if rule.status == "skipped":
        continue  # Does not exist for scoring purposes
    total_weight += weights[rule.id]
    if rule.status == "pass": earned += weights[rule.id]
```

### 4.2 Score Must Carry Metadata

A score of 85 is meaningless without knowing: how many checks ran, how many were skipped, what severity levels were involved.

```python
@dataclass
class ScoreResult:
    score: float
    checks_run: int
    checks_skipped: int
    checks_failed: int
    highest_severity_failure: str | None
    is_complete: bool  # True only if checks_skipped == 0
```

### 4.3 Incomplete Scores Must Be Flagged

If more than 20% of checks were skipped, the score must be marked as "incomplete" and must not be used for automated decisions (auto-approve, touchless processing).

---

## 5. Test Philosophy

### 5.1 Write Invariant Tests First

Before implementing any feature, write tests that assert the *properties* the system must hold — especially negative properties.

```python
# Invariant: A record with no validations must score zero
def test_unvalidated_record_scores_zero():
    score = compute_overall_score([])
    assert score == 0.0  # Not 100.0

# Invariant: A record referencing non-existent data must not claim a match
def test_missing_reference_cannot_match():
    record = create_record(reference="REF-DOES-NOT-EXIST")
    result = run_matching(record)
    assert result.match_score < 50
    assert result.match_type != "verified"

# Invariant: Skipped rules must not inflate scores
def test_skipped_rules_contribute_zero():
    results = [
        {"rule_id": "1.01", "status": "pass"},
        {"rule_id": "1.02", "status": "skipped"},
    ]
    score = compute_overall_score(results)
    # Score should be based only on the one rule that ran
    assert score.checks_run == 1
    assert score.checks_skipped == 1
```

### 5.2 Test the Boundaries, Not the Middle

Every test of a numeric threshold must include: exactly at threshold, one unit below, one unit above, zero, negative (if applicable), and maximum value.

### 5.3 Test with Adversarial Data

For every entity in the system, maintain a "chaos fixture" — a version designed to break assumptions:

```python
CHAOS_FIXTURES = [
    {"reference": "REF-DOES-NOT-EXIST"},     # Non-existent reference
    {"amount": -5000},                        # Negative amount
    {"date": "2099-01-01"},                   # Future date
    {"reference": ""},                        # Empty string (not None)
    {"identifier": "ID-' OR 1=1; --"},        # SQL injection attempt
    {"line_items": []},                       # No line items
    {"currency": "INVALID"},                  # Non-existent currency
    {"amount": 0.0},                          # Zero-value record
    {"amount": 99999999.99},                  # Extremely large value
]
```

### 5.4 Seed Data Is Testable Code

Demo data generators must be covered by assertions that verify the generated data is internally consistent. If a record has `status=pending` but all checks show `PASS`, that's a bug in the seed data.

```python
def test_seed_data_consistency():
    records = generate_demo_data()
    for record in records:
        if record.status in ("pending", "processing"):
            assert not all(r.status == "pass" for r in record.checks), \
                f"Record {record.id} in '{record.status}' should not have all-pass checks"
```

### 5.5 The "What If Nothing Exists" Test

For every feature, write a test where the primary entity exists but all related data is missing. This catches fail-open defaults.

```python
def test_record_with_no_related_data():
    """Record exists but all related data is missing."""
    record = create_bare_record()
    detail = get_record_detail(record.id)

    assert detail.validation_score == 0    # Not 100
    assert detail.match_score == 0         # Not "matched"
    assert detail.risk_level != "low"      # Unknown is not low
    assert detail.status != "approved"     # Cannot auto-approve
```

---

## 6. AI Agent-Specific Safeguards

These rules address biases specific to LLM-driven development.

### 6.1 The Failure Manifest

After completing any feature, the developer (or AI agent) must write a list of at least 5 ways the feature could produce wrong results while appearing to work correctly. Each item must have a corresponding test.

```
# Failure manifest for: Duplicate Detection
#
# 1. Same file uploaded twice → should reject second upload
# 2. Different file, same reference number → should flag
# 3. Same line items from same source → should detect
# 4. Detection service down → should NOT auto-approve
# 5. Empty file uploaded → should reject, not create empty record
# 6. Hash collision (different content) → should compare content
# 7. Upload retry after timeout → should be idempotent
```

### 6.2 Explicit "Data Not Available" Paths

Every function that receives external data must have an explicit "data not available" path that is distinct from "data indicates success."

```python
# The agent WANTS to write this (completion bias):
def get_risk_level(score):
    if score > 80: return "high"
    if score > 50: return "medium"
    return "low"  # Feels complete. Every input produces output.

# The agent MUST write this:
def get_risk_level(score):
    if score is None:
        return "unassessed"  # Explicitly not "low"
    if score > 80: return "high"
    if score > 50: return "medium"
    return "low"
```

### 6.3 Weight Justification

Weight tables, threshold configs, and scoring parameters must include a comment explaining WHY each value was chosen — not just what it is. If the justification is "it felt right" or "it's in the middle," the value is wrong.

### 6.4 Adversarial Demo Data

For every 3 "happy" demo records, there must be at least 1 "broken" demo record that exercises failure handling. Demo data must include: missing references, invalid states, edge-case values.

### 6.5 Naming Must Reflect Missing-Data Behavior

Function names must not imply certainty when the function might not have enough data to be certain.

```python
# WRONG — name implies validation always happens
def validate_record(record):
    if not record.reference:
        return Score(100)  # "Valid" because nothing to check

# RIGHT — name reflects actual behavior
def validate_record_if_data_available(record):
    if not record.reference:
        return IncompleteScore(0, reason="No reference to validate against")
```

### 6.6 Integration Over Isolation

AI agents write and test functions in isolation. Each function works correctly with the inputs it expects. But the COMPOSITION of correct functions can produce incorrect systems when intermediate data doesn't match assumptions.

```python
def test_end_to_end_with_missing_reference():
    """A record referencing non-existent data must NOT reach 'approved' status."""
    record = create_record(reference="REF-DOES-NOT-EXIST")
    process_record(record.id)

    result = get_record(record.id)
    assert result.status != "approved"
    assert result.match_score < 50
```

### 6.7 Argue Against Yourself

After building a feature, explicitly prompt the AI agent: *"Now find 5 ways this could produce a wrong result while appearing to work."* Agents are capable of adversarial analysis — they just don't do it by default because the training loop rewards completion, not skepticism.

---

## 7. Security Defaults

### 7.1 No Hardcoded Credentials

Configuration files must not contain usable default credentials. Use sentinel values that cause immediate, obvious failure:

```python
# WRONG
SECRET_KEY = "change-me-in-production"

# RIGHT
SECRET_KEY = ""  # App refuses to start if empty

def validate_config(self):
    if not self.SECRET_KEY:
        raise ValueError("SECRET_KEY must be set via environment variable")
```

### 7.2 Authorization Is Not Authentication

Verifying that a user is logged in is not the same as verifying they can access a specific resource. Every data-access endpoint must check ownership or role.

```python
# WRONG — any authenticated user can read any record
@router.get("/{record_id}")
async def get_record(record_id, user=Depends(get_current_user)):
    return serve_record(record_id)

# RIGHT — verify the user has access to this specific record
@router.get("/{record_id}")
async def get_record(record_id, user=Depends(get_current_user)):
    record = await db.get(Record, record_id)
    if record.organization_id != user.organization_id:
        raise HTTPException(403, "Access denied")
    return serve_record(record_id)
```

### 7.3 No User Data in LLM Prompts

Never interpolate user-controlled data into LLM system prompts. Pass it as structured context.

```python
# WRONG — user input could contain injection
prompt = f"Process record {record_id} through validation."

# RIGHT — separate data from instructions
prompt = "Process the record provided in the context through validation."
context = {"record_id": record_id}
```

---

## 8. Concurrency and Data Integrity

### 8.1 Read-Then-Write Must Be Atomic

Any pattern that reads a value, computes from it, then writes back is a race condition. Use database-level locking or atomic operations.

```python
# WRONG — two requests can read the same value
current_max = await db.scalar(select(func.max(Sequence.number)))
new_number = (current_max or 0) + 1

# RIGHT — lock the relevant rows first
await db.execute(text("SELECT ... FOR UPDATE"))
current_max = await db.scalar(select(func.max(Sequence.number)))
new_number = (current_max or 0) + 1
```

### 8.2 Connection and Queue Cleanup

When a connection is closed, all associated queues, buffers, and pending messages must be cleared. Stale messages on reconnection cause duplicate operations.

---

## 9. The Null Taxonomy

AI agents systematically confuse different flavors of "nothing." Each has different semantics.

### 9.1 The Seven Nulls

```python
# These are ALL different and must be handled distinctly:
value = None       # Never set — unknown
value = ""         # Set to empty — intentionally blank
value = 0          # Set to zero — a real quantity
value = 0.0        # Set to zero float — a real measurement
value = []         # Set to empty list — checked, found nothing
value = {}         # Set to empty dict — checked, no data
# And: key doesn't exist  # The concept was never considered
```

**The rule:** Never conflate these. `if not value` catches all of them, which is almost always wrong for business logic.

```python
# WRONG — treats "0 records this month" same as "data unavailable"
total = data.get("record_count")
if not total:
    return "No data"

# RIGHT — distinguish between absent and zero
total = data.get("record_count")
if total is None:
    return "Data unavailable"
if total == 0:
    return "No records this period"
return f"{total} records"
```

### 9.2 Empty String vs None in Databases

A blank form field, a field that was never filled, and a field that was intentionally cleared are three different things. Design your schema to distinguish them.

### 9.3 The JavaScript Falsy Problem

In TypeScript/JavaScript, `undefined`, `null`, `0`, `""`, `false`, and `NaN` all behave differently in conditionals. Always use explicit checks.

```tsx
// WRONG — hides real zero values
{record.score && <Badge>{record.score}%</Badge>}
// A score of 0 is falsy — the badge never renders

// RIGHT — only hide when truly absent
{record.score != null && <Badge>{record.score}%</Badge>}
```

---

## 10. Time, Dates, and Timezone Discipline

### 10.1 Store UTC, Display Local

All timestamps stored in databases or transmitted over APIs must be in UTC. Conversion to local time happens exclusively in the UI layer.

```python
# WRONG — stores server's local time
created_at = datetime.now()

# RIGHT — always UTC
from datetime import datetime, timezone
created_at = datetime.now(timezone.utc)
```

### 10.2 Date Boundaries Are Timezone-Dependent

"Records from March 15" means different things in different timezones. Any date-range query must account for the user's timezone.

### 10.3 Never Compare Naive and Aware Datetimes

Mixing timezone-aware and timezone-naive datetimes silently produces wrong results or throws exceptions. Pick one and enforce it everywhere.

### 10.4 Daylight Saving Time

2:30 AM doesn't exist on "spring forward" days. 1:30 AM happens twice on "fall back" days. Any system that schedules events by wall-clock time must handle this.

---

## 11. The N+1 and Performance Trap

### 11.1 The N+1 Query Problem

```python
# WRONG — 1 query for records + N queries for related data
records = await db.execute(select(Record))
for rec in records:
    items = await db.execute(
        select(LineItem).where(LineItem.record_id == rec.id)
    )  # This runs once PER record

# RIGHT — eager load or join in one query
records = await db.execute(
    select(Record).options(selectinload(Record.line_items))
)
```

**The rule:** Any database call inside a loop is a bug until proven otherwise.

### 11.2 Unbounded Queries

```python
# WRONG — loads entire table into memory
all_records = await db.execute(select(Record))

# RIGHT — always paginate
records = await db.execute(
    select(Record).limit(100).offset(page * 100)
)
```

### 11.3 Missing Database Indexes

If you filter or sort by a column, it needs an index. AI agents define models with query patterns but forget to add indexes.

### 11.4 Large Payload Blindness

List endpoints should return summaries. Detail endpoints return full data. Never return full details for every item in a list.

---

## 12. Idempotency and Retry Safety

### 12.1 Every Mutation Must Be Idempotent or Guarded

Network failures cause retries. If an action runs twice, the system must remain consistent.

```python
# WRONG — creates duplicate record on retry
async def approve(record_id):
    db.add(ApprovalAction(record_id=record_id, decision="approved"))
    record.status = "approved"

# RIGHT — idempotent: check current state first
async def approve(record_id):
    if record.status == "approved":
        return  # Already done — safe retry
    if record.status != "pending_approval":
        raise InvalidStateError(f"Cannot approve from '{record.status}' state")
    db.add(ApprovalAction(record_id=record_id, decision="approved"))
    record.status = "approved"
```

### 12.2 Idempotency Keys for API Endpoints

POST endpoints that create resources must accept an idempotency key to prevent duplicate creation on retry.

### 12.3 Retry Storms and Exponential Backoff

When a service fails, every client retrying simultaneously creates a thundering herd. All retry logic must use exponential backoff with jitter.

```python
# WRONG — immediate retry floods the server
for attempt in range(5):
    try: return await call_service()
    except: await asyncio.sleep(1)

# RIGHT — exponential backoff with jitter
for attempt in range(5):
    try: return await call_service()
    except:
        delay = min(2 ** attempt + random.uniform(0, 1), 30)
        await asyncio.sleep(delay)
```

---

## 13. Error Handling That Doesn't Lie

### 13.1 Catch Specific Exceptions

```python
# WRONG — hides real bugs behind a friendly message
try:
    result = await process(record_id)
except:
    return {"status": "ok", "message": "Processing complete"}

# RIGHT — catch what you expect, let everything else surface
try:
    result = await process(record_id)
except RecordNotFoundError:
    raise HTTPException(404, f"Record {record_id} not found")
except ValidationError as e:
    raise HTTPException(422, str(e))
```

### 13.2 Don't Log and Swallow

```python
# WRONG — error is logged but system continues with wrong state
try:
    score = compute_risk_score(record)
except Exception as e:
    logger.warning("Risk scoring failed: %s", e)
    score = 0  # Looks safe but risk check was SKIPPED, not passed

# RIGHT — make the failure visible in the result
try:
    score = compute_risk_score(record)
    risk_status = "completed"
except Exception as e:
    logger.error("Risk scoring failed: %s", e, exc_info=True)
    score = None
    risk_status = "error"  # UI shows "Risk check failed" not "Low risk"
```

### 13.3 Error Messages Must Not Leak Implementation Details

```python
# WRONG — exposes internal database structure
raise HTTPException(500, f"IntegrityError: duplicate key on records_pkey")

# RIGHT — user-friendly message, details in server logs only
logger.error("Duplicate record key: %s", exc)
raise HTTPException(409, "A record with this identifier already exists")
```

### 13.4 Fail-Fast on Startup, Fail-Gracefully at Runtime

Configuration errors should crash the app on startup — not be discovered when the first user request arrives.

```python
# On startup — fail fast
def validate_config():
    assert settings.DATABASE_URL, "DATABASE_URL is required"
    assert settings.SECRET_KEY, "SECRET_KEY is required"

# At runtime — fail gracefully
async def get_exchange_rate(currency):
    try:
        return await forex_service.get_rate(currency)
    except ServiceUnavailable:
        return None  # Caller must handle missing rate
```

---

## 14. Data Integrity Across Boundaries

### 14.1 Validate at Every Boundary

Data must be validated when it enters the system (API), when it crosses service boundaries, and when it leaves the system. Trust nothing.

```
[User Input] → validate → [API Layer] → validate → [Service Layer] → validate → [Database]
```

### 14.2 Foreign Key References Must Be Verified

Before creating a record that references another entity by ID, verify the referenced entity exists and is in a valid state.

### 14.3 Cascading Deletes Must Be Explicit

Define cascades explicitly and test them. Don't rely on database-level cascades that AI agents may have configured incorrectly.

### 14.4 Never Trust Client-Side Calculations

If the frontend computes a total, the backend must recompute and verify. The client is untrusted.

```python
# WRONG — trusts frontend-computed total
record.total = request.total_amount

# RIGHT — recompute from source data
computed_total = sum(item.quantity * item.unit_price for item in request.line_items)
if abs(computed_total - request.total_amount) > 0.01:
    raise HTTPException(422, "Line item totals do not match declared total")
record.total = computed_total
```

---

## 15. The Encoding and Character Trap

### 15.1 Always Declare and Enforce UTF-8

Every file, database connection, and HTTP response must explicitly declare UTF-8.

### 15.2 Test with Real-World Text

```python
CHAOS_STRINGS = [
    "Müller GmbH & Co. KG",              # German umlauts & ampersand
    "東京テクノロジー株式会社",              # Japanese
    "Amount: ₹1,50,000.00",              # Indian Rupee + lakh format
    "O'Brien & Associates",               # Apostrophe
    "<script>alert('xss')</script>",      # XSS attempt
    "'; DROP TABLE records; --",          # SQL injection attempt
]
```

### 15.3 Sanitize Before Rendering, Not Before Storing

Store the original text. Sanitize/escape when rendering in HTML, PDFs, or other output formats.

---

## 16. API Contract Discipline

### 16.1 Never Add Required Fields to Existing Responses

New fields must always be optional with sensible defaults.

### 16.2 Never Change the Type of an Existing Field

If `score` was a number, it must remain a number forever. Add a new field for richer data.

### 16.3 Pagination Must Be Consistent

Use cursor-based pagination for mutable datasets. Offset pagination breaks when data changes between pages.

### 16.4 API Errors Must Be Machine-Readable

```json
// WRONG
{"detail": "Something went wrong"}

// RIGHT
{
  "error_code": "RECORD_DUPLICATE",
  "message": "A record with this identifier already exists",
  "existing_id": "rec_789",
  "field": "identifier"
}
```

---

## 17. Graceful Degradation

### 17.1 Every External Dependency Must Have a Failure Mode

If an external service is down, the system must not silently produce wrong results.

```python
# WRONG — external service failure silently produces wrong results
async def convert_currency(amount, from_curr, to_curr):
    try:
        rate = await forex_api.get_rate(from_curr, to_curr)
    except:
        rate = 1.0  # "Safe" default that's actually wrong
    return amount * rate

# RIGHT — failure is visible to the caller
async def convert_currency(amount, from_curr, to_curr):
    try:
        rate = await forex_api.get_rate(from_curr, to_curr)
    except ExternalServiceError:
        return ConversionResult(
            converted=None,
            status="service_unavailable",
            message="Exchange rate service temporarily unavailable"
        )
    return ConversionResult(converted=amount * rate, status="ok")
```

### 17.2 Circuit Breakers for External Services

After N consecutive failures, stop calling the failing service for a cooldown period.

### 17.3 Feature Flags for New Integrations

New external integrations should be behind feature flags so they can be disabled without a deployment.

---

## 18. Logging and Observability

### 18.1 Never Log Sensitive Data

Passwords, tokens, API keys, PII, and financial identifiers must never appear in logs.

### 18.2 Structured Logging Over String Interpolation

```python
# WRONG — hard to parse, search, and alert on
logger.info(f"Record {record_id} processed in {duration}ms with score {score}")

# RIGHT — structured, queryable, alertable
logger.info(
    "record_processed",
    extra={
        "record_id": record_id,
        "duration_ms": duration,
        "score": score,
        "checks_run": checks_run,
        "checks_skipped": checks_skipped,
    }
)
```

### 18.3 Every Decision Must Be Auditable

If the system auto-approves something, the audit log must record WHY: the score, which rules passed, which threshold was used.

---

## 19. The Copy-Paste Drift Problem

### 19.1 Identical Logic Must Be One Function

If two operations share 80% of their logic, extract the shared part. Don't maintain two copies that will drift.

### 19.2 Serialization Helpers Must Be Centralized

If multiple endpoints serialize the same entity, they must all call the same serialization function.

### 19.3 When You Fix a Bug, Search for Its Siblings

After fixing a bug, search the entire codebase for the same pattern. If one date field was serialized wrong, check every date field. If one button was missing a handler, check every button.

---

## 20. Verification Protocol

After fixing any bug or implementing any feature, the developer (or AI agent) **MUST** verify the fix in the running application before marking it as complete.

### 20.1 For User-Facing Changes

1. Test through the running application
2. Reproduce the original issue to confirm it existed
3. Verify the fix — perform the action that was broken
4. Capture a screenshot showing the fixed behavior
5. Include the evidence in the PR

### 20.2 For Backend-Only Logic

1. Hit the API endpoint directly with test data
2. Capture the request and response
3. Verify with realistic data — use the chaos fixtures from §5.3
4. Include the response showing the fix works

### 20.3 What Does Not Count

- Code inspection alone
- Unit test passing (unit tests can lie — see §5)
- Logical reasoning ("This should work")
- Assumption that "the system was already checked"

---

## 21. The Review Checklist

Every pull request must include a self-review against this checklist.

### Defaults and State
- [ ] Every fallback/default value has been justified in a comment
- [ ] No scoring function defaults to 100 or "pass" on empty input
- [ ] No access control defaults to "allow" on error
- [ ] "Skipped" and "unknown" states are handled distinctly from "success"
- [ ] `None`, `""`, `0`, and `[]` are handled as distinct states where relevant
- [ ] Every UI control has a working handler or is explicitly disabled
- [ ] Every connection/subscription/timer is cleaned up on teardown
- [ ] No mutable default arguments
- [ ] Closures in cleanup functions reference local variables, not state

### Data Integrity
- [ ] Float arithmetic uses Decimal or epsilon comparison for business logic
- [ ] Date serialization is consistent across all fields of the same type
- [ ] All timestamps are stored in UTC
- [ ] File paths from databases are validated against allowed directories
- [ ] No user-controlled data is interpolated into LLM prompts
- [ ] Foreign key references are verified before insert
- [ ] Client-computed values are recomputed and verified on the server

### Scoring and Aggregation
- [ ] Skipped checks are excluded from both numerator AND denominator
- [ ] Scores carry metadata about how many checks ran vs. were skipped
- [ ] Incomplete scores (>20% skipped) are flagged and blocked from automation

### Performance
- [ ] No database queries inside loops (N+1)
- [ ] All list endpoints have pagination with enforced limits
- [ ] Columns used in WHERE/ORDER BY have database indexes
- [ ] Response payloads are sized appropriately (summary vs. detail)

### Error Handling
- [ ] No bare `except:` or `except Exception:` that swallows errors
- [ ] Error messages don't leak implementation details
- [ ] External service failures are visible in the result, not silently defaulted
- [ ] Mutations are idempotent or protected by idempotency keys

### Security
- [ ] No hardcoded credentials in config defaults
- [ ] Every data-access endpoint checks ownership, not just authentication
- [ ] Read-then-write patterns use database locking
- [ ] No sensitive data in log messages

### API Contract
- [ ] New fields in existing responses are optional with defaults
- [ ] No field type changes in existing endpoints
- [ ] Error responses include machine-readable error codes

### Tests
- [ ] At least one test verifies behavior when primary related data is missing
- [ ] At least one test verifies behavior at exact threshold boundaries
- [ ] Seed data is consistent with the states it claims to represent
- [ ] Integration test traces a full user story with realistic (imperfect) data
- [ ] Test fixtures include non-ASCII strings

### AI-Specific
- [ ] A "failure manifest" (5+ wrong-but-plausible results) exists for each feature
- [ ] Every weight/threshold has a comment explaining WHY, not just WHAT
- [ ] Duplicated logic has been extracted into shared functions
- [ ] After fixing a bug, the codebase was searched for sibling instances

### Verification (Section 20)
- [ ] For user-facing changes: screenshot showing the fix working
- [ ] For backend changes: API response showing the fix with realistic data
- [ ] Evidence is included in the PR
- [ ] Original issue was reproduced before fix was verified

---

## Appendix: Quick Reference — Bug Class → Protocol Rule

| Bug Class | Section | Cardinal Rule |
|-----------|---------|---------------|
| Fail-open defaults | 1.1 | Default to Deny |
| Missing data treated as success | 1.2 | Absence Is Not Evidence |
| Skipped checks inflating scores | 1.3, 4.1 | Skipped Work Gets Zero Credit |
| Non-functional UI elements | 1.4 | Every Button Must Do Something |
| Mutable default arguments | 2.1 | — |
| Float precision in business logic | 2.2 | — |
| Path traversal in file serving | 2.3 | — |
| Resource leaks on unmount | 2.4 | — |
| Stale closures in React | 2.5 | — |
| Inconsistent serialization | 2.6 | — |
| Display showing mode not result | 3.3 | — |
| Score denominator pollution | 4.1 | — |
| Null vs empty vs zero confusion | 9 | — |
| Timezone and date boundary bugs | 10 | — |
| N+1 queries and unbounded selects | 11 | — |
| Non-idempotent mutations | 12 | — |
| Swallowed exceptions | 13.2 | — |
| Error messages leaking internals | 13.3 | — |
| Missing input validation | 14.1 | — |
| Dangling foreign key references | 14.2 | — |
| Trusting client-side calculations | 14.4 | — |
| Encoding and character bugs | 15 | — |
| Breaking API contract changes | 16 | — |
| Silent degradation on service failure | 17 | — |
| Sensitive data in logs | 18.1 | — |
| Copy-paste drift | 19 | — |
| Optimism bias (AI) | 6.1 | — |
| Completion bias (AI) | 6.2 | — |
| Symmetry bias (AI) | 6.3 | — |
| Demo-path bias (AI) | 6.4 | — |
| Naming bias (AI) | 6.5 | — |
| Isolation bias (AI) | 6.6 | — |
| Hardcoded credentials | 7.1 | — |
| AuthN vs AuthZ confusion | 7.2 | — |
| Prompt injection | 7.3 | — |
| Race conditions | 8.1 | — |

---

## Enforcement

This protocol is enforced through:

1. **Pre-commit hooks** — scan for known anti-patterns (mutable defaults, `return 100.0` in score functions, float `==` comparisons, bare `except:`, `datetime.now()` without timezone)
2. **CI pipeline** — invariant tests run before unit tests
3. **Code review** — Section 21 checklist on every PR
4. **Verification protocol** — Section 20 evidence required before merge
5. **Quarterly audit** — review scoring systems and default values
6. **Chaos testing** — periodic runs with adversarial fixtures to verify failure paths

Violations of Cardinal Rules (Section 1) are treated as P0 bugs regardless of whether they currently cause user-visible issues.

---

*This protocol is a living document. Every production bug that traces to a pattern not covered here must result in a new rule being added.*

*Created by [SumVec AI](https://sumvec.ai) — Purpose-built AI solutions for the enterprise.*

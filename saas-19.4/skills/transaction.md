# Transaction / Cursor mechanics — candidates (SaaS-19.4)

Scope note: this pass focused deeply on `odoo/odoo/sql_db.py`, `odoo/odoo/orm/environments.py`
(the `Transaction` class), `odoo/odoo/addons/base/models/ir_cron.py`, and
`odoo/odoo/http/router.py` / `odoo/odoo/http/retrying.py`, cross-checked with `git log`/`git show`
on the community repo and usage grep across `addons/`, `enterprise/`, `industry/`,
`design-themes/`. It did **not** do a systematic sweep of every module's own ad-hoc
commit/savepoint usage (e.g. bespoke batch jobs in enterprise verticals) beyond sampling —
treat the 8 candidates below as the highest-confidence, highest-impact findings rather than
an exhaustive catalogue.

---

## 1. `_commit_progress()` replaces raw `cr.commit()` in batch/cron code

**Claim**: In SaaS-19.4, cron server-actions that process records in batches are expected to
call `self.env['ir.cron']._commit_progress(...)` instead of a bare `self.env.cr.commit()`.
This both commits and records progress (done/remaining counters) used by the cron scheduler
to decide whether the job is `fully done`, `partially done` (reschedule ASAP on any worker),
or `failed`. The old `_notify_progress()` method (which only logged progress and required the
caller to still commit separately) is deprecated.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_cron.py:871-916` — `_commit_progress()` definition:
  ```python
  @api.model
  def _commit_progress(self, processed: int = 0, *, remaining: int | None = None, deactivate: bool = False) -> float:
      ...
      progress.write(vals)
      self.env.cr.commit()
      return max(ctx.get('cron_end_time', float('inf')) - time.monotonic(), 0)
  ```
- `odoo/odoo/addons/base/models/ir_cron.py:849-869` — `_notify_progress()` carries
  `@deprecated("Since 19.0, use _commit_progress")`.
- `odoo/odoo/addons/base/models/ir_cron.py:919-921` — companion `_rollback_progress()`
  ("The rollback with the same logic as the commit for cron jobs.").
- Introduced by commit `69dd579c85d2` ("[IMP] base: cron `_commit_progress`", 2025-01-10) in the
  community repo — confirmed via `git log --oneline -S "_commit_progress" -- odoo/addons/base/models/ir_cron.py`.
- Adopted pervasively: 46 files call `_commit_progress` across `addons/` (account, stock, sms,
  mass_mailing, mail, sale, website_sale, base_automation, crm_iap_enrich, auth_signup, …) and
  `enterprise/` (ai, ai_fields, l10n_do_edi, sale_subscription, documents, website_generator*,
  sale_tiktok, account_followup, partner_commission, …), e.g.
  `odoo/addons/account/models/account_move.py:6852-6896`,
  `enterprise/ai/models/ai_embedding.py:116-169`.
- A test explicitly documents that raw `cr.commit()` is now the wrong tool inside cron batch
  code: `odoo/addons/l10n_ar_stock/tests/test_l10n_ar_delivery_guide_batch.py:118-121` —
  comment "stub `_commit_progress` to avoid its forbidden `cr.commit()`" (see also candidate 7).

**Why it matters**: A stale-trained LLM patching a Sentry crash in a batch cron method (e.g. "too
many records committed at once" or "lost progress after a mid-batch exception") will most likely
write `self.env.cr.commit()` directly, as was idiomatic in 16/17. This compiles and "looks right,"
but it silently loses the progress-tracking that drives rescheduling: the job will look like it
always fully completed (or the agent will have to hand-roll its own `ir_cron_progress`-style
bookkeeping), and it also skips the return value used to check remaining time budget
(`ctx.get('cron_end_time', ...)`), so the agent's patch can easily reintroduce timeouts or
infinite-processing-without-yielding bugs that `_commit_progress` was specifically built to
prevent.

**Confidence**: high

---

## 2. Cron jobs now loop internally with a bounded budget and a tri-state completion status

**Claim**: A single cron "run" is no longer "call the server action once." `IrCron._run_job()`
calls the server action repeatedly in the same worker (up to `MIN_RUNS_PER_JOB` times or until
`MIN_TIME_PER_JOB` seconds elapse), and classifies the outcome as `CompletionStatus.FULLY_DONE`,
`PARTIALLY_DONE` (reschedule ASAP, can run on another worker), or `FAILED`, based on the
processed/remaining counts reported via `_commit_progress`.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_cron.py:35-38`:
  ```python
  MIN_RUNS_PER_JOB = 10
  MIN_TIME_PER_JOB = 120  # seconds
  CONSECUTIVE_TIMEOUT_FOR_FAILURE = 3
  MIN_FAILURE_COUNT_BEFORE_DEACTIVATION = 5
  ```
- `odoo/odoo/addons/base/models/ir_cron.py:63-66` — `class CompletionStatus(enum.StrEnum)` with
  `FULLY_DONE`, `PARTIALLY_DONE`, `FAILED`.
- `odoo/odoo/addons/base/models/ir_cron.py:460-571` — `_run_job()`, notably the loop condition
  `while status is None and (loop_count < MIN_RUNS_PER_JOB or time.monotonic() < env.context['cron_end_time'])`
  and the `match (success, done, remaining):` block that derives the completion status per
  iteration.
- `odoo/odoo/addons/base/models/ir_cron.py:401-427` — `_process_job()` docstring spells out the
  three outcomes and how each affects rescheduling (`'partially done'` → `_reschedule_asap`,
  possibly picked up by a *different* cron worker).

**Why it matters**: A stale LLM diagnosing a Sentry issue like "records processed twice" or
"duplicate side effects from cron X" may assume the classic model (cron = one call = one
transaction = one pass over records) and conclude the bug is e.g. a missing lock. In reality,
the same job can legitimately be invoked several times in a row within one worker cycle *and*
picked up again immediately by another worker after a `partially done` result — so idempotency
of each batch iteration (not just of the whole cron) is now a hard requirement. A patch that
"fixes" this by adding a single flag/lock checked once at the top of the method will not survive
the intra-job loop or cross-worker rescheduling.

**Confidence**: high

---

## 3. Committing or rolling back while a `cr.savepoint()` is still open is a hard error

**Claim**: `cr.commit()` and `cr.rollback()` now assert that there is no open (unreleased)
savepoint on the transaction. Calling either while inside a `with cr.savepoint():` block (without
closing it first) raises `AssertionError`, not a silent/ignored no-op.

**Evidence**:
- `odoo/odoo/orm/environments.py:934-937`:
  ```python
  @contextmanager
  def committing(self):
      """ Context for committing the connection. """
      assert not self._state_stack__, "Pending savepoints not released, cannot commit!"
  ```
- `odoo/odoo/orm/environments.py:999-1002`:
  ```python
  @contextmanager
  def rollbacking(self):
      """ Context for rollbacking the connection. """
      assert not self._state_stack__, "Pending savepoints not released, cannot rollback!"
  ```
- `odoo/odoo/sql_db.py:527-538` (`Cursor.commit`) and `:540-562` (`Cursor.rollback`) both go
  through `self.transaction.committing()` / `.rollbacking()` when a `Transaction` is attached,
  so this assertion fires on ordinary `cr.commit()`/`cr.rollback()` calls, not just internal APIs.

**Why it matters**: A very natural "defensive" patch for a Sentry crash inside a `try/except`
around a savepoint (e.g. "catch the error, savepoint-protect the risky call, then commit what we
have so far") will crash with an unrelated `AssertionError` if the fix commits the cursor before
the savepoint's `with` block exits (or before calling `.close()`/`.rollback()` on the `Savepoint`
object explicitly). A stale-knowledge agent, remembering pre-19 semantics where this wasn't
guarded this strictly, could easily introduce exactly this ordering bug while "fixing" the
original crash.

**Confidence**: high

---

## 4. `cr.savepoint()` auto-flushes the ORM and snapshots/restores transaction-level cache state

**Claim**: `Cursor.savepoint()` now defaults to `flush=True`, which uses `_FlushingSavepoint`
instead of the bare SQL `Savepoint`. Entering the savepoint flushes pending ORM writes and calls
`cr.transaction.save_state()`; on rollback it clears `cr.precommit` and calls
`cr.transaction.restore_state()`; on clean close it flushes again and calls
`cr.transaction.merge_state()`. This couples savepoint rollback to the ORM's per-field cache
layering (`ormcaches__` / `TransactionState`), not just to SQL-level `ROLLBACK TO SAVEPOINT`.

**Evidence**:
- `odoo/odoo/sql_db.py:137-164` (`_FlushingSavepoint`):
  ```python
  class _FlushingSavepoint(Savepoint):
      def __init__(self, cr: Cursor):
          cr.flush()
          if cr.transaction is not None:
              cr.transaction.save_state()
          super().__init__(cr)

      def rollback(self):
          cr = self._cr
          super().rollback()
          cr.precommit.clear()  # they were flushed in the init
          if cr.transaction is not None:
              cr.transaction.restore_state()
  ```
- `odoo/odoo/sql_db.py:287-296` — `Cursor.savepoint(self, flush: bool = True)` picks
  `_FlushingSavepoint` vs. plain `Savepoint` based on the `flush` flag, defaulting to flushing.
- `odoo/odoo/orm/environments.py:1014-1060` — `save_state()` / `merge_state()` / `restore_state()`
  implement the `ormcaches__` layer push/pop tied to the savepoint lifecycle.

**Why it matters**: A stale-trained LLM used to Odoo 16/17's much simpler `Savepoint` (pure SQL
`SAVEPOINT`/`ROLLBACK TO SAVEPOINT`, no ORM cache interaction) might assume that rolling back a
savepoint leaves in-memory recordset field values as they were, and add manual
`record.invalidate_recordset()` / `env.clear()` calls "to be safe" around a savepoint-guarded
retry — which is redundant at best, and at worst fights with `restore_state()`/`merge_state()`
(e.g. by clearing `precommit` callbacks or ormcache layers the framework is still tracking),
producing stale-cache bugs that only show up under concurrent access.

**Confidence**: medium

---

## 5. HTTP dispatch already retries serialization failures and converts `IntegrityError` — don't re-implement it

**Claim**: `odoo/odoo/http/retrying.py:retrying()` wraps every controller call at the router
level: it retries the whole request up to `MAX_TRIES_ON_CONCURRENCY_FAILURE` (5) times with
exponential backoff on `psycopg2.OperationalError`/`ConcurrencyError` (which includes
`SerializationFailure`, `LockNotAvailable`, `DeadlockDetected` via
`PG_CONCURRENCY_EXCEPTIONS_TO_RETRY` in `sql_db.py`), rolling back the cursor between attempts,
and it converts any `psycopg2.IntegrityError` into a user-facing `ValidationError` using
`model._sql_error_to_message()` instead of letting the raw DB error propagate. It also performs
the final `cr.commit()` itself after a successful call.

**Evidence**:
- `odoo/odoo/http/retrying.py:26-112` (full function), key excerpts:
  ```python
  def retrying[T](func, env, *, close_on_commit: bool = True) -> T:
      ...
      except (psycopg2.IntegrityError, psycopg2.OperationalError, ConcurrencyError) as exc:
          ...
          if isinstance(exc, psycopg2.IntegrityError):
              ...
              raise ValidationError(message) from exc
          if isinstance(exc, PG_CONCURRENCY_EXCEPTIONS_TO_RETRY):
              ...
          wait_time = random.uniform(0.0, 2 ** tryno)
          ...
      if not env.cr.closed:
          env.cr._closing = close_on_commit
          env.cr.commit()  # effectively commits and executes post-commits
  ```
- `odoo/odoo/sql_db.py:66-70` — `PG_CONCURRENCY_EXCEPTIONS_TO_RETRY` tuple
  (`LockNotAvailable`, `SerializationFailure`, `DeadlockDetected`) consumed by `retrying.py`.
- Called from the router at `odoo/odoo/http/router.py:415` and `:441`.

**Why it matters**: If a Sentry crash report shows a `ValidationError` with a generic "operation
cannot be completed" message instead of the underlying constraint violation, a stale-knowledge
agent may look for a manual `try/except psycopg2.IntegrityError` in application code (as one
might write in 16/17 controllers) and, not finding one, wrongly conclude the error is
unhandled/needs a new try/except — when in fact it's already centrally handled and the real bug
is upstream (why the constraint is violated at all, or why `_sql_error_to_message` produces an
unhelpful message for that specific constraint). Conversely, adding a redundant per-controller
try/except-and-retry loop around a serialization failure duplicates the router's own retry logic.

**Confidence**: medium

---

## 6. HTTP routes can run on a read-only replica cursor and transparently retry on a read/write one

**Claim**: Controllers declare `readonly=True/False` (or a callable) via `@route(...)`. When
`readonly` is true and the current cursor is a read-only replica cursor (`cr.readonly`), the
request is first served on that read-only cursor; if it raises `psycopg2.errors.ReadOnlySqlTransaction`
(i.e. it actually attempted a write), the router closes it, opens a fresh read/write cursor, and
retries the whole request — instead of failing.

**Evidence**:
- `odoo/odoo/http/router.py:410-441`:
  ```python
  if readonly and cr.readonly:
      threading.current_thread().cursor_mode = 'ro'
      try:
          return retrying(serve_func, env=request.env)
      except ReadOnlySqlTransaction as exc:
          _logger.warning("%s, retrying with a read/write cursor", exc.args[0].rstrip(), exc_info=True)
          threading.current_thread().cursor_mode = 'ro->rw'
      ...
  if cr.readonly:
      cr.close()
      cr = request.env.registry.cursor()
  else:
      cr.rollback()
  ```
- `odoo/odoo/sql_db.py:573-575` — `Cursor.readonly` property (`bool(self._cnx.readonly)`);
  `Connection.cursor()` at `:760-769` sets `readonly=self.__pool.readonly` on the psycopg2
  session.
- `odoo/odoo/tools/config.py:413-417` — `--db_replica_host` / `PGHOST_REPLICA` option feeding the
  separate read-only connection pool (`sql_db.py:817-828`, `_Pool_readonly`).
- `odoo/odoo/http/routing_map.py:38,172` — `readonly` typed as
  `bool | Callable[[registry, rule, args], bool]` in the route options.

**Why it matters**: A stale LLM debugging a Sentry error that only reproduces intermittently in
production (e.g. "record not found" or "stale data shown right after write") might not consider
that the request could have executed reads against a lagging read replica, or that a route
marked `readonly=True` silently ran twice (once read-only, once read/write) when it turned out to
write — meaning any non-idempotent side effect before the write attempt (e.g. an external API
call) executed only once but any state mutation logged/asserted "read-only" may need re-checking
under retry. It could also misdiagnose "why does this only fail under load" bugs without knowing
a replica-lag-driven code path exists at all.

**Confidence**: medium

---

## 7. `cr.commit()` / `cr.rollback()` / `cr.close()` are hard-forbidden inside `TransactionCase` tests

**Claim**: Odoo's base test case (`TransactionCase`) monkey-patches `commit`, `rollback`, and
`close` on the shared test cursor to raise `AssertionError` unconditionally. Tests (and any code
exercised by them) must use `cr.savepoint()` for rollback-style isolation instead of touching the
cursor's commit/rollback/close directly.

**Evidence**:
- `odoo/odoo/tests/common.py:1383-1392`:
  ```python
  def forbidden(*args, **kwars):
      traceback.print_stack()
      raise AssertionError('Cannot commit or rollback a cursor from inside a test, this will '
          'lead to a broken cursor when trying to rollback the test. Please rollback to a '
          'specific savepoint instead or open another cursor if really necessary')

  cls.commit_patcher = patch.object(cls.cr, 'commit', forbidden)
  cls.rollback_patcher = patch.object(cls.cr, 'rollback', forbidden)
  cls.close_patcher = patch.object(cls.cr, 'close', forbidden)
  ```
- Real-world consequence documented in
  `odoo/odoo/addons/l10n_ar_stock/tests/test_l10n_ar_delivery_guide_batch.py:118-123`, which has
  to `patch.object(IrCron, '_commit_progress')` specifically "to avoid its forbidden `cr.commit()`"
  when calling cron logic directly from a test.

**Why it matters**: If the agent is asked to write a regression test alongside a Sentry-driven
bug fix, and the buggy/fixed code path calls `cr.commit()` (directly, or transitively via
`_commit_progress()`/`_rollback_progress()`), a naive test will crash with this `AssertionError`
rather than exercising the real bug — the stale-knowledge fix is to mock/patch out the
commit-calling method (as the real codebase does), not to "make the test call commit safely."

**Confidence**: high

---

## 8. `cr.commit()` is now the *only* correct trigger for registry/ormcache signaling — bypassing it desyncs workers

**Claim**: Cache invalidation and cross-worker registry signaling are no longer a loosely-coupled
side concern; they are wired directly into the `commit()` call path via
`Transaction.committing()`, which flushes, decides which cache layers/registry sequence numbers
changed, takes a registry lock, calls `registry._signal_changes(cr, names)` *just before* the
underlying `self._cnx.commit()`, and only then publishes the updated cache layers to the shared
`registry.registry_caches__`. Any code path that commits the underlying psycopg2 connection
without going through `Cursor.commit()` (e.g. manual `cr._cnx.commit()`, or holding a psycopg2
cursor obtained from `_obj` directly) skips this signaling and cache publication entirely.

**Evidence**:
- `odoo/odoo/orm/environments.py:582-588` (class docstring):
  ```
  Signaling summary:
  - Creating the first `Environment` for a cursor triggers checks signaling.
  - `cr.commit` signals changes and resets the transaction.
  - `cr.rollback` resets the transaction without signaling.
  ```
- `odoo/odoo/orm/environments.py:934-997` — `committing()` context manager: flush, compute
  `push_caches`, `with lock: registry._signal_changes(cr, names); yield; registry.registry_caches__.update(push_caches)`.
- `odoo/odoo/sql_db.py:527-538` — `Cursor.commit()` only reaches this logic via
  `self.transaction.committing()`; a bare `self._cnx.commit()` (bypassing `Cursor.commit`) would
  skip it.

**Why it matters**: A stale LLM trying to work around some other bug (e.g. "the cursor's commit
hook is doing something we don't want") might reach for the underlying psycopg2 connection
(`cr._cnx.commit()`) or otherwise dodge the ORM-level `commit()` to "just commit the SQL." That
silently breaks multi-worker cache coherency (other workers/processes won't be told to invalidate
their ormcache / reload the registry), producing exactly the class of hard-to-reproduce
"stale data after save" Sentry issues the agent is trying to fix, or masking the original bug
under a new one.

**Confidence**: medium

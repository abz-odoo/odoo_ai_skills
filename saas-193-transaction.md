# Odoo SaaS-19.3 Transaction Guide

Reference for transaction management, savepoints, and cursor callbacks in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/sql_db.py` and `odoo/orm/environments.py`

> **SaaS-19.3 key additions**
> - `_commit_progress(processed, remaining, deactivate)` — cron batch progress + commit (new)
> - `_rollback_progress()` — cron rollback counterpart (new)
> - `@api.autovacuum` return value can now be `(done, remaining)` tuple
> - `Transaction.save_state()` / `restore_state()` — ORM state stacked per savepoint

---

## Table of Contents

1. [Isolation level and cursor lifecycle](#isolation-level-and-cursor-lifecycle)
2. [commit and rollback](#commit-and-rollback)
3. [Cursor callbacks](#cursor-callbacks)
4. [Savepoints](#savepoints)
5. [Transaction ORM state](#transaction-orm-state)
6. [Cron batch operations — _commit_progress (new)](#cron-batch-operations--_commit_progress-new)
7. [@api.autovacuum](#apiautovacuum)
8. [Common patterns](#common-patterns)

---

## Isolation level and cursor lifecycle

Odoo opens every cursor in **PostgreSQL REPEATABLE READ** with autocommit off:

```python
cnx.set_session(
    isolation_level=ISOLATION_LEVEL_REPEATABLE_READ,
    readonly=pool.readonly,  # True for read-replica cursors (@api.readonly routes)
    autocommit=False,
)
```

A cursor is typically used as a context manager — it commits on clean exit and rolls back
on exception:

```python
with registry.cursor() as cr:
    env = api.Environment(cr, SUPERUSER_ID, {})
    env['my.model'].create({'name': 'test'})
# cr.commit() called automatically here if no exception
# cr.close() called in finally
```

---

## commit and rollback

```python
# Commit — flushes ORM, runs precommit hooks, calls cnx.commit(), runs postcommit hooks
cr.commit()

# Rollback — clears precommit/postcommit, runs prerollback hooks, cnx.rollback(), runs postrollback
cr.rollback()

# flush — flush ORM pending writes + run precommit hooks (without committing)
cr.flush()
```

> `cr.commit()` internally calls `cr.flush()` first — so all pending ORM writes are written
> to the DB before the transaction is committed.

---

## Cursor callbacks

Four hook queues that fire at transaction boundaries:

| Queue | Fires when |
|-------|-----------|
| `cr.precommit` | just before `cnx.commit()` — after ORM flush |
| `cr.postcommit` | just after `cnx.commit()` succeeded |
| `cr.prerollback` | just before `cnx.rollback()` |
| `cr.postrollback` | just after `cnx.rollback()` |

```python
# Register a callback
def notify_external_system():
    requests.post('https://webhook.example.com/notify', json={'id': self.id})

cr.postcommit.add(notify_external_system)

# Callbacks are cleared on the opposite event:
# - precommit / postcommit are cleared on rollback
# - prerollback / postrollback are cleared on commit
```

### Typical use-cases

```python
# Send email only after the transaction commits (avoids sending on rollback)
self.env.cr.postcommit.add(lambda: self._send_confirmation_email())

# Wake up cron worker after commit
self.env.cr.postcommit.add(self._notifydb)

# Pre-commit: flush a specific model before an external read
self.env.cr.precommit.add(lambda: self.env['my.model'].flush_model())
```

---

## Savepoints

A savepoint creates a nested rollback point within the current transaction.
The ORM provides two variants:

```python
# With ORM state save/restore (default, flush=True)
with cr.savepoint() as sp:
    env['my.model'].create({'name': 'maybe'})
    if something_failed:
        sp.rollback()   # rollback to savepoint, ORM state restored
# If an exception escapes the block, the savepoint is rolled back automatically

# Without ORM flush (flush=False) — rare, for performance-critical paths
with cr.savepoint(flush=False) as sp:
    cr.execute("INSERT INTO my_table VALUES (%s)", (value,))
```

### Manual rollback inside savepoint

```python
with cr.savepoint() as sp:
    for record in records:
        try:
            record.do_risky_operation()
        except Exception as e:
            sp.rollback()   # undo this iteration only
            _logger.warning("Skipped record %s: %s", record.id, e)
```

### Savepoint on error

```python
# If an exception propagates out of the `with` block:
try:
    with cr.savepoint():
        raise ValueError("oops")
except ValueError:
    pass
# The savepoint was automatically rolled back — transaction still alive
```

---

## Transaction ORM state

`cr.transaction` (a `Transaction` instance) manages the ORM cache and computed-field
queue. It stacks state when a savepoint is opened:

```python
# When entering cr.savepoint(flush=True):
cr.transaction.save_state()    # pushes a snapshot onto _state_stack

# On savepoint rollback:
cr.transaction.restore_state() # pops snapshot, clears ORM cache, resets registry seq

# On savepoint success:
cr.transaction.merge_state()   # pops snapshot without restoring (merges into parent)
```

You rarely call these directly — `cr.savepoint()` handles them. But understanding the
behaviour explains why ORM cache is correctly invalidated after a savepoint rollback.

### transaction.reset()

Called when the registry is reloaded mid-transaction (rare):

```python
cr.transaction.reset()
# Refreshes registry reference, clears all caches, rebuilds state stack
```

---

## Cron batch operations — _commit_progress (new)

`_commit_progress` / `_rollback_progress` are new in SaaS-19.3 for cron jobs that process
data in batches. They replace the pattern of manually calling `cr.commit()` in a loop.

```python
@api.model
def _process_pending_records(self):
    """Cron job — call from ir.cron action."""
    records = self.search([('state', '=', 'pending')], limit=100)

    for record in records:
        record._do_work()
        # Commit this batch, log progress, return remaining time
        remaining_time = self.env['ir.cron']._commit_progress(
            processed=1,
            # remaining= can be set to update the progress bar
        )
        if remaining_time <= 0:
            break   # cron time budget exhausted, scheduler will re-run
```

### Signature

```python
def _commit_progress(
    self,
    processed: int = 0,
    *,
    remaining: int | None = None,
    deactivate: bool = False,
) -> float:
    """
    processed  — number of items processed in this step (added to done counter)
    remaining  — explicit remaining count (if None, decremented by processed)
    deactivate — deactivate the cron after this run
    returns    — remaining time in seconds (float('inf') if called outside a cron)
    """
```

```python
def _rollback_progress(self) -> None:
    """Rollback the current batch. Use in except blocks."""
```

### Full batch loop pattern

```python
@api.model
def _my_batch_cron(self):
    while True:
        batch = self.search([('state', '=', 'todo')], limit=50)
        if not batch:
            break
        try:
            batch.write({'state': 'done'})
            remaining_time = self.env['ir.cron']._commit_progress(
                processed=len(batch),
            )
        except Exception:
            self.env['ir.cron']._rollback_progress()
            _logger.exception("Batch failed, rolling back")
            break

        if remaining_time <= 0:
            break   # time budget exhausted
```

> Outside a cron context (e.g., called manually from a button), `_commit_progress` just
> calls `cr.commit()` and returns `float('inf')`.

---

## @api.autovacuum

Marks a private method to be called by the daily vacuuming cron (`ir.autovacuum`).
In SaaS-19.3 the return value can signal progress:

```python
@api.autovacuum
def _gc_old_attachments(self):
    """Delete attachments older than 90 days."""
    old = self.search([
        ('create_date', '<', fields.Datetime.now() - timedelta(days=90)),
        ('res_model', '=', 'my.model'),
    ], limit=1000)
    old.unlink()
    # Return (done, remaining) to tell the scheduler there may be more
    return len(old), len(old) == 1000
```

- Method name **must** start with `_`
- No arguments (called by the vacuum cron with no context)
- Return `(done, remaining)` for batched GC — the vacuum cron will re-run if `remaining > 0`

---

## Common patterns

### Wrapping a long operation in savepoints

```python
def process_all(self):
    for chunk in chunked(self.ids, size=100):
        with self.env.cr.savepoint():
            self.browse(chunk)._process_chunk()
        # Each chunk committed via savepoint release
        # If a chunk fails, only that chunk is rolled back
```

### Notify after commit (webhooks, emails)

```python
def write(self, vals):
    res = super().write(vals)
    if vals.get('state') == 'done':
        # Don't send the email if the transaction rolls back
        self.env.cr.postcommit.add(self._send_done_email)
    return res
```

### Read-only transaction via @api.readonly

```python
@api.readonly
def get_dashboard_data(self):
    """Routes to read replica; raises if you accidentally write."""
    return self.env['sale.order'].search_read(
        [('state', '=', 'sale')], ['name', 'amount_total'],
    )
```

### Re-using the test transaction (testing only)

```python
# In tests: open a parallel cursor that shares the test's transaction
with self.enter_registry_test_mode():
    new_cr = self.registry.cursor()
    # Operations here are visible to self.env.cr (same snapshot)
    new_cr.close()
```

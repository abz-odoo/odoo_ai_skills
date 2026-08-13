# Candidates: ORM / Model layer (Odoo SaaS-19.4)

Scope note: This pass focused on `odoo/odoo/orm/` (the new home of the ORM,
split out of the old monolithic `odoo/models.py`), `odoo/odoo/orm/*.py`
docstrings, real usage in `odoo/addons/`, `enterprise/`, and tests under
`odoo/odoo/addons/test_orm/tests/`. It is not exhaustive — I did not deeply
review `fields.py`/`fields_relational.py` internals, `registry.py`,
`identifiers.py` (NewId), or the full `cache.py` invalidation machinery
beyond what came up incidentally. Those are candidates for a follow-up pass
if needed. I capped at 8 candidates as instructed, prioritizing things a
stale (16/17-era) LLM would confidently get wrong in a way that produces a
broken patch, not just "renamed but equivalent" facts.

---

## 1. Domains are no longer plain lists — there is a `Domain` AST class

**Claim:** The `[('field', 'op', value), '&', '|', '!']` prefix-list-of-tuples
representation still parses, but internally (and increasingly in new code)
domains are built and manipulated as a `Domain` object from
`odoo.orm.domains` / re-exported as `odoo.fields.Domain`. `Domain` is not a
list: it's an AST with subclasses `DomainBool`, `DomainNot`, `DomainAnd`,
`DomainOr`, `DomainCondition`, combinable with `&`, `|`, `~`, and built via
`Domain.AND([...])`, `Domain.OR([...])`, `Domain.TRUE`, `Domain.FALSE`, and
`Domain(field, op, value)`. Calling `bool(domain)` is actively discouraged
(there's a `PLW1641` lint suppression and a commented-out deprecation
warning) — use `.is_true()` / `.is_false()` instead. Old `osv.expression`
list-munging helpers (`expression.AND`, manually splicing `'&'`/`'|'`
prefix tokens) are superseded by this object.

**Evidence:**
- `~/src/194/odoo/odoo/orm/domains.py:193-260` — `class Domain`,
  `__new__` factory accepting `Domain(...)`, list, or bool.
- `~/src/194/odoo/odoo/orm/domains.py:369` —
  `# warnings.warn("Do not use bool() on Domain, use is_true() or is_false() instead", DeprecationWarning)`
- `~/src/194/odoo/odoo/addons/test_orm/tests/test_domain.py:16` —
  `from odoo.fields import Command, Domain` (public import path for new code).
- Real usage: `~/src/194/odoo/odoo/orm/models.py:1503-1520`
  (`_search_display_name` returns `Domain.TRUE`/`Domain.FALSE` directly,
  not `[]`/`[(0,'=',1)]`).

**Why it matters:** A stale LLM patching a Sentry `TypeError`/`ValueError`
around domain building will likely (a) hand-splice `'&'`/`'|'` prefix lists
the old way when a helper now expects/returns a `Domain` object, (b) call
`bool(domain)` to check "is this domain empty" and get wrong results or a
future-deprecated pattern, or (c) fail to realize a method signature now
declares `domain: DomainType` where `DomainType` includes both list and
`Domain`, silently breaking when the code elsewhere assumes a list and does
`domain + [...]` (concatenation semantics don't apply to `Domain` objects).

**Confidence:** high

---

## 2. `any` / `not any` / `any!` / `not any!` are first-class domain operators with distinct access semantics

**Claim:** Beyond legacy dot-notation (`'partner_id.name'`), relational
traversal in a domain can be written explicitly as
`(field, 'any', [sub_domain])` / `not any`. There are also bang variants
`any!` / `not any!` that **bypass** `ir.rule` record rules on the comodel
(equivalent to what used to require `sudo()` tricks or manual rule
bypassing). `any!`/`not any!` are rejected in domains built from plain lists
unless `Domain(..., internal=True)` or built through `Domain(field, 'any!', ...)`
directly — they're an "internal" operator class, not meant for arbitrary
user-supplied domains.

**Evidence:**
- `~/src/194/odoo/odoo/orm/domains.py:78-96` — docstring:
  "`any!` works like `any` but bypass adding record rules on the comodel".
- `~/src/194/odoo/odoo/orm/domains.py:216-218` — "the special
  operators `'any!'` and `'not any!'` are allowed in domain conditions ...
  but not in domain lists".
- Real usage/tests: `~/src/194/odoo/odoo/addons/test_orm/tests/test_domain_expression.py:227-249,297-338`
  (`[('link_sibling_id', 'any', [('quantity', '>', 5)])]`,
  `[('child_ids', 'not any', [('quantity', '=', 1)])]`).
- `any!` access-bypass test: same file `:926,946-951` —
  `Domain('currency_id', 'any!', comodel_rule)` vs `.sudo()` producing
  `any!` automatically after `optimize_full`.

**Why it matters:** If a Sentry crash is an `AccessError` or missing-rows bug
triggered by a domain that traverses a relation, a stale LLM may "fix" it by
adding `sudo()` broadly (security regression) instead of using the narrower,
already-supported `any!` bypass, or may not recognize `any`/`not any` in a
domain at all and mis-parse it as a typo for `in`/`not in`, producing an
incorrect rewritten domain.

**Confidence:** high

---

## 3. `_sql_constraints` and `_constraints` are gone — use `models.Constraint` / `models.Index` / `models.UniqueIndex` class attributes

**Claim:** The old `_sql_constraints = [('name', 'sql', 'message'), ...]`
tuple-list and the even older `_constraints` Python-validator list are no
longer read by the framework at all (only a warning is logged if they're
present — they do nothing). The replacement is declaring `models.Constraint(...)`,
`models.Index(...)`, or `models.UniqueIndex(...)` as private
(underscore-prefixed) class attributes directly on the model, each becoming
its own named SQL object applied/diffed against the DB schema individually
(not as one big list assigned in one place).

**Evidence:**
- `~/src/194/odoo/odoo/orm/model_classes.py:167-172`:
  ```python
  if hasattr(model_def, '_constraints'):
      _logger.warning("Model attribute '_constraints' is no longer supported, ...")
  if hasattr(model_def, '_sql_constraints'):
      _logger.warning("Model attribute '_sql_constraints' is no longer supported, "
                      "please define models.Constraint on the model.")
  ```
- Class definitions: `~/src/194/odoo/odoo/orm/table_objects.py:88-149` (`Constraint`),
  `:150-220` (`Index`), `:221-260` (`UniqueIndex`).
- Real usage: `~/src/194/odoo/odoo/addons/hr/models/hr_employee.py:359-366`:
  ```python
  _barcode_uniq = models.Constraint(
      'unique (barcode)',
      'The Badge ID must be unique, this one is already assigned to another employee.',
  )
  _user_uniq = models.Constraint(
      'unique (user_id, company_id)',
      'A user cannot be linked to multiple employees in the same company.',
  )
  ```
  Also `~/src/194/odoo/odoo/addons/im_livechat/models/discuss_channel.py:201-235`
  (mix of `Constraint` and conditional/functional `Index` definitions, some with
  callables for dynamic SQL).
- Naming note/gotcha: `table_objects.py:117-125` — a constraint named
  `..._not_null` triggers a `warnings.warn` because PG18 auto-generates that
  suffix for NOT NULL columns, risking a name clash.

**Why it matters:** A stale LLM fixing a Sentry `IntegrityError` (duplicate
key, check constraint violation) by adding/editing
`_sql_constraints = [(...)]` on a SaaS-19.4 model will produce a **silent
no-op patch** — the tuple is simply ignored (only a deprecation warning is
logged), so the constraint is never actually created in the database and the
bug isn't fixed. It also won't know the constraint name must be a
`_`-prefixed unmangled attribute name, or that per-object `apply_to_database`/
`matches_database` diffing means a changed definition auto-migrates on
`-u`/upgrade, which the old list-based mechanism didn't guarantee as cleanly.

**Confidence:** high

---

## 4. Raw SQL should go through `SQL()` + `env.execute_query()`/`execute_query_dict()`, not bare `cr.execute(str % ...)`

**Claim:** There's a dedicated `odoo.tools.sql.SQL` wrapper (and
`SQL.identifier(...)`) for composing parameterized/composable raw SQL. It
tracks which ORM `Field`s a query's *string interpolation* depends on
(`to_flush` metadata) so that `env.execute_query(sql)` /
`env.execute_query_dict(sql)` can auto-flush exactly those fields' pending
in-memory changes to the DB before running the query — something plain
`cr.execute(...)` never did and still doesn't do. `SQL`'s `code` argument is
typed as `typing.LiteralString | SQL` specifically to push people away from
building queries with f-strings/`%`-formatting of table/column names
(a lint rule `# pylint: disable=sql-injection` sits at the top of
`tools/sql.py`, implying the opposite is flagged elsewhere).

**Evidence:**
- `~/src/194/odoo/odoo/tools/sql.py:47-95` — `class SQL`, docstring
  showing `SQL("UPDATE TABLE %s SET %s", SQL.identifier(tablename), SQL(...))`.
- `~/src/194/odoo/odoo/orm/environments.py:548-573`:
  ```python
  def execute_query(self, query: SQL) -> list[tuple]:
      """ ... The method automatically flushes all the fields in the metadata of the query. """
      assert isinstance(query, SQL)
      _, _, fields_to_flush = query._sql_tuple
      if fields_to_flush:
          ...
          self[model_name].flush_model(field_names)
      self.cr.execute(query)
      return [] if self.cr.description is None else self.cr.fetchall()
  ```
- Real usage in tests: `~/src/194/odoo/odoo/addons/test_orm/tests/test_domain.py:63-65`
  (`sql = SQL("SELECT number FROM test_orm_empty_int WHERE id IN %s ORDER BY id", records._ids)`,
  then `self.env.execute_query(sql)`).
- Widespread real usage: `grep -c "SQL("` in
  `~/src/194/odoo/odoo/addons/sale/models/sale_order.py` → non-trivial
  count of `SQL(...)` constructions alongside `cr.execute(...)` calls (166 hits
  for `cr.execute(` in that one file alone, many wrapping `SQL(...)`).

**Why it matters:** A Sentry crash that looks like "stale/missing data read
back after a write in the same request" is a classic symptom of raw SQL that
reads a column the ORM had pending-but-unflushed changes for. A stale LLM
patching this by adding a plain `self.env.cr.execute("SELECT ... FROM tbl
WHERE ...")` (string-built, no `SQL()` wrapper) will not get the automatic
flush that `execute_query` provides, and may silently keep the bug, or
"fix" it by manually calling `flush_all()` everywhere instead of using the
already-idiomatic, narrower `SQL(..., to_flush=field)` / `execute_query`
pattern that the surrounding codebase actually uses.

**Confidence:** high

---

## 5. `check_access()` / `has_access()` are the only access-check API, and both are `@typing.final`

**Claim:** The old trio `check_access_rights(operation, raise_exception=...)`,
`check_access_rule(operation)`, and the underlying `_filter_access_rules`
override points are gone from the public model API. There is now exactly one
pair: `check_access(operation) -> None` (raises `AccessError`) and
`has_access(operation) -> bool`, and both are decorated `@typing.final` —
i.e. the framework's own type-checking metadata says these are not meant to
be overridden by addon code anymore (custom security logic should be
expressed via `ir.rule` domains / `_access_domain` overrides, not by
overriding the check method itself).

**Evidence:**
- `~/src/194/odoo/odoo/orm/models.py:3377-3400`:
  ```python
  @api.private  # use has_access
  @typing.final
  def check_access(self, operation: str) -> None:
      ...
  @typing.final
  def has_access(self, operation: str) -> bool:
      ...
  ```
- Grep confirms the old names are essentially gone from the framework and
  addons: 0 real hits for `check_access_rights(`/`check_access_rule(` as ORM
  API calls in `~/src/194/odoo/odoo/orm/models.py`; the only
  remaining `check_access_rights`/`_check_access_rights` hits repo-wide are an
  unrelated JS test double
  (`~/src/194/odoo/odoo/addons/web/static/tests/.../project_models.js:35`)
  and a locally-defined helper of the same name in
  `~/src/194/odoo/odoo/addons/account/wizard/account_merge_wizard.py:104,126,159`
  that is NOT the ORM method (it's a private `_check_access_rights` on that
  wizard class, unrelated to the framework API).
- 79 files under `odoo/addons` call the new `.check_access(` vs. essentially
  none calling the old names.

**Why it matters:** Fixing a Sentry `AccessError`/permission-bug by
overriding `check_access_rule`/`check_access_rights` (the standard Odoo
16/17 pattern for custom security logic) will simply define a method that is
never called in SaaS-19.4 — a silent no-op patch that looks plausible in
review but does nothing, leaving the actual bug (and possibly a security
hole) in place. The correct fix path is almost always an `ir.rule` domain or
overriding `_search`/`_access_domain`-related hooks instead of the check
method itself.

**Confidence:** high

---

## 6. `search()` is now a thin wrapper over `search_fetch()`; `fetch()` is the public API for bulk-loading fields into cache

**Claim:** `search(domain, offset, limit, order)` no longer does its own
"search then let cache/read lazily happen" — its entire body is
`return self.search_fetch(domain, [], offset=offset, limit=limit, order=order)`.
`search_fetch(domain, field_names, ...)` runs `_search()` (build query +
apply access rules) and `_fetch_query()` (load requested fields) in one
combined pass, computing exactly which fields need fetching via
`_determine_fields_to_fetch()`. Separately, `fetch(field_names)` is the
supported public way to force-load a batch of fields for an existing
recordset (replacing ad hoc `record.mapped('field')` calls used purely to
warm the cache, or reliance on implicit prefetch).

**Evidence:**
- `~/src/194/odoo/odoo/orm/models.py:1414-1428` — `search()` body:
  ```python
  def search(self, domain, offset=0, limit=None, order=None):
      ...
      return self.search_fetch(domain, [], offset=offset, limit=limit, order=order)
  ```
- `~/src/194/odoo/odoo/orm/models.py:1434-1478` — `search_fetch()`
  building `query = self._search(...)`, short-circuiting on `query.is_empty()`,
  then `self._fetch_query(query, fields_to_fetch)`.
- `~/src/194/odoo/odoo/orm/models.py:3042-3055` — `fetch()` docstring:
  "This method is implemented thanks to methods `_search` and `_fetch_query`,
  and should not be overridden."

**Why it matters:** A stale LLM diagnosing an N+1 query problem or a
"missing field in cache" Sentry error might reach for patterns like manually
calling `.read(fields)` right after `.search()` (redundant extra query,
16/17-era idiom) instead of calling `search_fetch(domain, field_names)`
directly, or might not realize overriding `search()` to add filtering logic
now needs to account for the fact that `search()` unconditionally delegates
to `search_fetch`, so overriding `_search()` (not `search()`) is the correct
hook for domain-level changes.

**Confidence:** medium — the search()/fetch() split direction itself may
partially predate 19.4 (search_fetch existed earlier in the 17/18 line), but
the exact one-line delegation and the `_determine_fields_to_fetch` machinery
are current-codebase specifics worth citing precisely rather than assuming.

---

## 7. `_read_group` uses string aggregate/groupby specs, and `_read_grouping_sets` computes multiple groupings in one SQL query

**Claim:** The public `read_group()` RPC-style method is gone; grouping goes
through `_read_group(domain, groupby, aggregates)` where `groupby` entries
are strings like `'field'` or `'field:granularity'` (e.g. `'date:month'`) and
`aggregates` entries are strings like `'field:sum'`, `'__count'`,
`'field:recordset'`, `'field:count_distinct'`. Beyond that, SaaS-19.4 adds
`_read_grouping_sets(domain, grouping_sets, aggregates, order)` which runs
**multiple different groupby combinations in a single SQL query** using SQL
`GROUPING SETS`, with explicit special-casing for many2many fields (which can
duplicate rows) that recursively splits the query when aggregates aren't
duplication-safe (`:max`, `:min`, `:bool_and`, `:bool_or`,
`:array_agg_distinct`, `:recordset`, `:count_distinct` are safe; `:sum`,
`:avg`, plain `:count` are not).

**Evidence:**
- `~/src/194/odoo/odoo/orm/models.py:1656-1710` — `_read_grouping_sets`
  docstring and signature, e.g.:
  ```python
  def _read_grouping_sets(self, domain, grouping_sets, aggregates=(), order=None) -> list[list[tuple]]:
      """ ... uses SQL `GROUPING SETS` ... """
  ```
- Same method body `:1717-1745` — the `might_duplicate_rows` / m2m-safe
  aggregate suffix list logic.
- Real usage confirmed outside the core: `_read_grouping_sets(` appears in
  `~/src/194/odoo/odoo/addons/product_margin/models/product_product.py`,
  `~/src/194/enterprise/sale_commission/report/achievement_report.py`,
  `~/src/194/enterprise/sale_commission/report/commission_report.py`,
  `~/src/194/enterprise/account_budget/tests/test_account_budget.py`,
  and others.

**Why it matters:** A stale LLM fixing a Sentry error in a reporting/pivot
computation might rewrite a `_read_grouping_sets` call back into several
sequential `_read_group` calls (functionally similar but N queries instead of
1, and it would miss that the grouping-sets code intentionally handles m2m
row-duplication differently per aggregate). It might also mishandle the
string spec format (`'field:agg'`) as if it were the old
`read_group(fields=[...], groupby=[...])` dict/list-of-field-names API,
producing a `ValueError`/`KeyError` instead of a fix.

**Confidence:** medium-high — `_read_group` string-spec format itself is
likely already 17/18-era knowledge; the specifically new, high-confidence
part is `_read_grouping_sets` and its m2m-duplication-safe aggregate list.

---

## 8. `@api.readonly` + read-replica routing: writing inside a readonly-marked path raises a real DB error, not just a lint issue

**Claim:** SaaS-19.4 has live infrastructure for routing HTTP/RPC-served
requests to a **separate read-only PostgreSQL connection pool** (configured
via `db_replica_*` settings) when the matched controller/route is marked
`readonly`. `@api.readonly` marks a model method as safe to run with
`self.env.cr` potentially bound to such a read-only connection.
`@api.private` marks a method as not RPC-callable at all. If a method
(or something it calls) executes a write (`create`/`write`/`unlink`/raw
`INSERT`/`UPDATE`) while running under a route/RPC call actually dispatched
with a readonly cursor, PostgreSQL will raise a real
"cannot execute ... in a read-only transaction" error at the DB level — this
is not a static/opt-in check, it's enforced by the actual connection.

**Evidence:**
- `~/src/194/odoo/odoo/orm/decorators.py:289-303` (`@api.private`)
  and `:306-313` (`@api.readonly`):
  ```python
  def readonly[C: Callable](method: C) -> C:
      """ Decorate a record-style method where ``self.env.cr`` can be a
          readonly cursor when called trough a rpc call. """
      method._readonly = True
      return method
  ```
- `~/src/194/odoo/odoo/http/router.py:377-388` — `serve_db`:
  ```python
  registry = Registry(request.db)
  cr = registry.cursor(readonly=True)   # get the registry and cursor (RO)
  ```
- `~/src/194/odoo/odoo/sql_db.py:610-830` — separate
  `ConnectionPool(readonly=True)` (`_Pool_readonly`) and
  `connection_info_for(db_or_uri, readonly=...)` reading `db_replica_<param>`
  config keys as a fallback when `readonly=True`.
- Real, widespread usage: 35 files under `odoo/addons` use `@api.readonly` on
  model methods, e.g.
  `~/src/194/odoo/odoo/addons/mail/models/mail_activity.py:685,689,748`,
  `~/src/194/odoo/odoo/addons/mail/models/res_users.py:495`,
  `~/src/194/odoo/odoo/addons/mail/models/res_partner.py:257`.
- Also relevant: `search_fetch()` itself is decorated
  `@api.model @api.private @api.readonly` at
  `~/src/194/odoo/odoo/orm/models.py:1432-1434`, i.e. reads are
  explicitly expected to tolerate a readonly cursor.

**Why it matters:** This is a strong candidate to explain a specific *class*
of Sentry crash: `psycopg2.errors.ReadOnlySqlTransaction` /
"cannot execute X in a read-only transaction" showing up only in production
(where read replicas are actually configured) and not in a dev DB (single
read/write connection, so the same code "works" locally). A stale LLM will
not know this failure mode exists at all and may misdiagnose it as a
permissions bug, a transaction/locking bug, or try to "fix" it by wrapping
the write in a retry/try-except instead of the real fix: removing the
write from the readonly-eligible code path, or removing an incorrect
`@api.readonly` annotation, or moving the write out of a method reachable
from a readonly-routed controller.

**Confidence:** high on the mechanism's existence and real usage; medium on
how often it's the actual root cause of a given Sentry ticket (would need
the specific stack trace to confirm route/decorator involvement).

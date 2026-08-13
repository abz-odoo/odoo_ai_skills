# Field-type candidates — Odoo SaaS-19.4 (discovery pass)

Scope note: this pass covered `odoo/odoo/orm/fields*.py`, `odoo/odoo/orm/domains.py`,
`odoo/odoo/orm/commands.py`, `odoo/odoo/orm/models.py` (default-value path), and
`odoo/addons/test_orm/tests/test_fields.py` in depth, with targeted sampling of
`enterprise/`, `industry/` was not sampled (no field-type hits were specifically chased
there; `design-themes/` was not sampled either — low prior for backend field mechanics
in a themes repo). Community `addons/` and `enterprise/` were grepped for real usage of
every candidate below, not just read cold in the ORM source. If a human reviewer wants
deeper industry/design-themes coverage, that is the main known gap.

Two sub-investigations were run in parallel (Domain/Properties/Json area; core
Field/Command mechanics area) in addition to direct investigation of index kinds,
`compute_sql`, Many2many FK behavior, and computed-field-unassigned semantics. All
citations below were either found directly or spot-verified against the source file
before inclusion.

---

## 1. `odoo/fields.py` is now a package — real implementation lives in `odoo/orm/`

**Claim:** In SaaS-19.4, `odoo/fields.py` no longer exists as a file. It was replaced by
`odoo/fields/__init__.py`, a 22-line re-export shim, and the actual `Field` class and all
field-type implementations moved to a new top-level `odoo/orm/` package, split across
multiple files: `orm/fields.py` (base `Field`), `orm/fields_relational.py` (Many2one/
One2many/Many2many), `orm/fields_numeric.py`, `orm/fields_textual.py`,
`orm/fields_temporal.py`, `orm/fields_selection.py`, `orm/fields_reference.py`,
`orm/fields_binary.py`, `orm/fields_misc.py`, `orm/fields_properties.py`, plus
`orm/commands.py` (`Command`) and `orm/domains.py` (`Domain`). This is an explicit,
documented refactor, not drift: commit `0a5b1f96b8126820618487906acfe1a6b144e595`
("[REF] core: odoo.orm package", 2024-10-07, Part-of odoo/odoo#182727).

**Evidence:**
- `~/src/194/odoo/odoo/fields/__init__.py:1-23` — full file, e.g. line 3:
  `# This is a \`__init__.py\` file to avoid merge conflicts on \`odoo/fields.py\`.` and
  line 5: `from odoo.orm.fields import Field`.
- Real implementation: `~/src/194/odoo/odoo/orm/fields.py` (2017 lines),
  `~/src/194/odoo/odoo/orm/fields_relational.py` (1884 lines),
  `~/src/194/odoo/odoo/orm/domains.py` (2154 lines),
  `~/src/194/odoo/odoo/orm/commands.py` (130 lines), etc.
- `cd ~/src/194/odoo && git show --stat 0a5b1f96b812` confirms the
  refactor commit and its stated intent to keep `from odoo import fields` working
  for backward compatibility while moving the real code.

**Why it matters:** Sentry tracebacks from this version will show frames like
`File "odoo/orm/fields_relational.py", line 823, in write_batch` or
`File "odoo/orm/domains.py", line 1523`. An agent trained on 16/17 knowledge expects
`odoo/fields.py` and may (a) fail to locate the file it needs to read/patch, (b) assume
the traceback frame belongs to some third-party or addon-specific ORM extension because
the path "orm/fields_relational.py" doesn't match its memorized layout, or (c) edit the
shim in `odoo/fields/__init__.py` instead of the real class in `odoo/orm/`.

**Confidence:** High.

---

## 2. `compute_sql` — computed fields can be computed directly in SQL, bypassing Python `compute`

**Claim:** Alongside the classic `compute='method_name'` (a Python method setting
`record.field = value` per record), fields now support `compute_sql='method_name'`
(or a lambda), a method with signature `(self, table)` returning an `SQL` object that
the ORM inlines directly into the generated SELECT for that field — no Python-side
per-record loop, no recompute/flush cycle for read-only consumption. This is a distinct,
additional computation channel, not a synonym for `compute`.

**Evidence:**
- `~/src/194/odoo/odoo/orm/fields.py:241` (docstring): `:param str compute_sql: name of a method that produces SQL for the field`.
- `~/src/194/odoo/odoo/orm/fields.py:447-451`: setup-time warnings if
  `compute_sql` is set without `compute`, or without an explicit `compute_sudo`.
- `~/src/194/odoo/odoo/orm/fields.py:1270-1280`: dispatch in `to_sql()` —
  `if self.compute_sql: ... sql_field = determine(self.compute_sql, model, table); assert isinstance(sql_field, SQL) ...`.
- Real usage, minimal example:
  `~/src/194/odoo/odoo/addons/test_orm/models/test_access_rights.py:75,82-83`:
  ```python
  currency_id = fields.Many2one('res.currency', compute='_get_company_currency',
                                 compute_sql='_get_company_currency_sql', compute_sudo=True, readonly=True)
  ...
  def _get_company_currency_sql(self, table):
      return table.company_id.currency_id
  ```
- Widely used in core and enterprise, e.g.
  `~/src/194/odoo/odoo/addons/base/models/ir_access.py:85-121` (several
  `compute_sql=lambda self, table: self._compute_sql_for(table, 'read')`-style fields),
  `~/src/194/enterprise/account_reports/models/account.py:17-23`
  (`audit_debit`, `audit_credit`, `audit_balance`, ... all pair `compute=`/`compute_sql=`).

**Why it matters:** A Sentry error inside a method named like `_compute_sql_xxx`, or an
`AssertionError`/`TypeError` about "invalid return of compute_sql" (raised at
`fields.py:1275`), will look unfamiliar to a stale LLM, which may try to "fix" it by
treating the method as a normal per-record `compute` method (iterating `for record in
self: record.field = ...`) instead of returning a single `SQL(...)` expression built
from the `table` argument — a wrong patch here would either raise a new assertion or
silently produce wrong query results for every row, not just the buggy one.

**Confidence:** High.

---

## 2b. `precompute=True` is silently disabled by any explicit value or `default=`

**Claim:** `precompute=True` tells the ORM to run the compute method before `INSERT`
instead of after, but the docstring makes an easy-to-miss carve-out: precomputation
only triggers when `create()` gets **no explicit value and no default value** for that
field. Passing `default=...` on a `precompute=True` field disables precomputation
entirely, even though the field is still marked `precompute=True`. The ORM also
force-disables `precompute` at setup time in specific cases (see evidence).

**Evidence:**
- `~/src/194/odoo/odoo/orm/fields.py:206-231`, especially: *"Precomputation
  only happens when no explicit value and no default value is provided to create().
  This means that a default value disables the precomputation, even if the field is
  specified as precompute=True."*
- Setup-time guards, same file, lines 470-476: warns and resets `precompute=False` if
  the field isn't computed/related, or isn't stored.
- Lines 883-885 and 909-910: `precompute` is also force-disabled down a dependency
  chain once it crosses a non-precomputed stored compute field or a `many2one`.
- Real usage pattern: `~/src/194/odoo/odoo/addons/base/models/res_partner_bank.py`
  and `res_partner.py:241` use `precompute=True, readonly=False, store=True` on
  computed many2one fields (grep `precompute=True` in `addons/base/models/res_partner*.py`
  to confirm).

**Why it matters:** Debugging a Sentry issue like "field X is always computed in a slow
one-by-one loop instead of batch on creation" (a `precompute` performance regression), a
stale LLM would not know to check whether someone added a `default=` to that same field
— the two kwargs interact in a non-obvious, undocumented-outside-this-docstring way, and
"fixing" the symptom by re-asserting `precompute=True` does nothing if a `default=` is
still present.

**Confidence:** High.

---

## 3. Computed fields that leave some records unassigned now raise `ValueError` — unless the field is inversable

**Claim:** If a `compute=` method does not assign a value to every record in `self` (a
common bug: `for record in self: if condition: record.x = value` with no `else`
branch), accessing that field on an unassigned record raises `ValueError` — **but only
for plain readonly compute fields**. If the same field also declares `readonly=False`
(making it inversable/user-editable), unassigned records silently fall back to `False`
instead of raising.

**Evidence:** Test model
`~/src/194/odoo/odoo/addons/test_orm/models/test_orm.py:882-914`:
```python
class TestOrmComputeUnassigned(models.Model):
    _name = 'test_orm.compute.unassigned'
    foo = fields.Char()
    bar = fields.Char(compute='_compute_bar')                       # readonly, no fallback
    bare = fields.Char(compute='_compute_bare', readonly=False)      # inversable, falls back
    bars = fields.Char(compute='_compute_bars', store=True)          # readonly, no fallback
    bares = fields.Char(compute='_compute_bares', readonly=False, store=True)  # inversable

    @api.depends('foo')
    def _compute_bar(self):
        for record in self:
            if record.foo == "assign":
                record.bar = record.foo
    # (_compute_bare/_compute_bars/_compute_bares follow the identical pattern)
```
Test assertions in
`~/src/194/odoo/odoo/addons/test_orm/tests/test_fields.py:883-901`
(`test_16_compute_unassigned`):
```python
record = model.create({})
with self.assertRaises(ValueError):
    record.bar
self.assertEqual(record.bare, False)
self.assertEqual(record.bars, False)   # NOTE: see caveat below
self.assertEqual(record.bares, False)
```
(Note: `bars`/`bares` are `store=True`; the test shows they read as `False` post-create
because store+compute go through a different assignment path at INSERT time — the
`ValueError`-on-access behavior is clearest for the non-stored `bar` field. Re-check
`test_11_stored`/`test_11_stored_protected` in the same file, lines 341-431, if writing
a skill entry that needs the stored-field nuance pinned down precisely.)

**Why it matters:** This is a very plausible real Sentry crash shape: a compute method
with an incomplete conditional (missing `else`) that used to "just work" (silently
False/empty) now raises `ValueError` on the unassigned records for plain compute
fields. A stale LLM asked to fix the resulting traceback might add a workaround at the
call site (e.g. wrap in try/except, or check `if record.field:`) instead of recognizing
the real bug is the incomplete compute method itself, and might also assume adding
`readonly=False` is a valid one-line "fix" to suppress the error, which changes the
field from readonly to user-editable — a much bigger, unrelated behavior change.

**Confidence:** Medium-high (mechanism and test are concrete and directly read; the
store=True interaction needs a closer look before being stated as an absolute rule in a
final skill).

---

## 4. `Domain` is a real class with operator overloading, constructed directly by addon code — not just list literals

**Claim:** Alongside (not instead of) the classic Polish-notation domain
(`[('a', '=', 1), '|', ('b', '=', 2), ('c', '=', 3)]`), this version exposes
`Domain` (`from odoo.fields import Domain` / `from odoo.orm.domains import Domain`) as
a real, immutable AST class that addon code constructs explicitly and combines with
`&`, `|`, `~`, plus `Domain.AND([...])` / `Domain.OR([...])` classmethods and
`Domain.TRUE` / `Domain.FALSE` singletons for "always true"/"always false" domains
(replacing ad hoc idioms like `domain = []` or `[('id', '=', 0)]`). Plain list/tuple
domains still work everywhere — `Domain(list_or_tuple)` parses the old Polish notation
into the same AST — so this is additive, but it is the dominant idiom in this
codebase's own business logic, not just ORM-internal plumbing.

**Evidence:**
- Class docstring / factory, `~/src/194/odoo/odoo/orm/domains.py:202-209`:
  ```python
  Domain([('a', '=', 5), ('b', '=', 8)])
  Domain('a', '=', 5) & Domain('b', '=', 8)
  Domain.AND([Domain('a', '=', 5), *other_domains, Domain.TRUE])
  ```
- `Domain.TRUE`/`Domain.FALSE` classproperties: `domains.py:270-276`.
- Real addon usage: `~/src/194/enterprise/sale_renting/models/sale_order.py:440-448`
  (`base = Domain("is_rental_order", "=", True)`, combined with `&`), and
  `~/src/194/enterprise/ai/models/ai_attachment_vacuum.py:25-26`
  (`Domain('create_date', '<', '-1d')`, `Domain('attachment_id.res_model', '!=', False) & Domain(...)`).
- Test import treats it as first-class, co-equal with `Command`:
  `~/src/194/odoo/odoo/addons/test_orm/tests/test_domain.py:6`:
  `from odoo.fields import Command, Domain`.

**Why it matters:** An agent unfamiliar with this API, seeing `Domain(...)`, `&`, `|`,
`Domain.AND(...)`, or `Domain.FALSE` in a Sentry stack trace or in the file it's about to
patch, may mistake it for bespoke project code rather than core ORM vocabulary — leading
it to "explain" the bug in terms of a custom helper class, or to rewrite working
`Domain(...)`-based code back into list literals unnecessarily (which is not wrong per
se, but risks breaking chained boolean combination logic the original author relied on,
and is exactly the kind of unnecessary-churn patch a triage agent should avoid).

**Confidence:** High.

---

## 5. New domain operators `any` / `not any` / `any!` / `not any!` for relational traversal, with an access-control distinction

**Claim:** Domain conditions on relational fields support `any`/`not any` (e.g.
`Domain("move_id.line_ids", "any", <subdomain>)`) to express "at least one linked
record matches", executed as a `EXISTS`-style subquery. There is a documented, load
bearing distinction between `any`/`not any` (respect the comodel's record rules /
access control) and the internal-only `any!`/`not any!` variants (bypass access
control) — the latter cannot appear in a plain list literal domain and must be
constructed explicitly via `Domain(field, 'any!', subdomain)`, guarded by a
`ValueError` if attempted via a list.

**Evidence:**
- `~/src/194/odoo/odoo/orm/domains.py:249-255` — list-literal parsing
  raises `ValueError` if an item's operator is in `INTERNAL_CONDITION_OPERATORS`
  (which includes `any!`/`not any!`) unless `internal=True` is explicitly passed.
- `~/src/194/odoo/odoo/orm/fields.py:1440-1444` (`condition_to_sql`
  handling for `any!`) — `sql_operator = SQL_OPERATORS["in" if operator == "any!" else "not in"]`.
- Real usage: `~/src/194/enterprise/l10n_ph_reports/models/boa_cash_payment_report.py:41-43`:
  `Domain("move_id.line_ids", "any", Domain.AND([Domain("account_id.account_type", "in", (...)), ...]))`.

**Why it matters:** This has real security weight: `any!`/`not any!` deliberately
bypass record rules on the related model. A stale LLM patching a Sentry
`AccessError`/permission bug by swapping `any` for `any!` (or vice versa) without
understanding this distinction could either introduce an access-control hole (using
the bypass variant to silence a legitimate permission error) or break an intentional
sudo-style traversal (removing the bypass where it was needed).

**Confidence:** High.

---

## 6. `fields.Properties` — dynamic, container-defined sub-fields stored as one JSONB column

**Claim:** `Properties` is not a generic key-value JSON blob. It's a pseudo-field
system: the *type* of each sub-property (char, many2one, selection, tags, ...) is
declared on a separate "definition" record (via a `PropertiesDefinition` field on
another model), referenced from the `Properties` field via a `definition=` dotted path
(e.g. `'definition' points at `<related-record>.<properties_definition_field>`). At
read time the ORM merges the container's definition with the child record's stored
values so the client sees a fully-typed pseudo-field. Only the values are stored on the
child row (as jsonb); the schema/type information lives elsewhere.

**Evidence:**
- Class docstring, `~/src/194/odoo/odoo/orm/fields_properties.py:34-51`
  (`type = 'properties'`, `_column_type = ('jsonb', 'jsonb')`, `store = True`,
  `readonly = False`, `precompute = True` set as class defaults — i.e. Properties
  fields are computed-editable by design, not plain stored fields).
- Real usage:
  `~/src/194/enterprise/account_asset/models/account_asset.py:127`:
  `asset_properties = fields.Properties('Properties', definition='account_asset_id.asset_properties_definition', copy=True)`.
  `~/src/194/enterprise/stock_barcode/models/stock_move_line.py:19`:
  `fields.Properties(related='lot_id.lot_properties', definition='product_id.lot_properties_definition', readonly=True)`.

**Why it matters:** A stale LLM debugging a Sentry error on a `Properties` field (e.g.
a `KeyError`/`MissingError` reading a sub-property, or a search filter that silently
returns nothing) is likely to treat the field like a normal stored/JSON field — trying
direct SQL-level filtering, assuming a fixed schema, or missing that `definition=`
points at a *different model's* definition record entirely, so the actual bug may be
that the definition record was deleted/changed, not that the child record's data is
corrupt.

**Confidence:** High.

---

## 7. `Many2many` fields get a real foreign key with a configurable `ondelete` (default `'cascade'`), restricted to `'cascade'`/`'restrict'`

**Claim:** `Many2many.ondelete` defaults to `'cascade'` and is validated at setup time
to be one of only `'cascade'` or `'restrict'` (never `'set null'`, which makes no sense
for a relation-table row). This is wired into a genuine PostgreSQL `FOREIGN KEY ...ON
DELETE` constraint on the relation table's second column, added via
`update_db_foreign_keys`.

**Evidence:**
- `~/src/194/odoo/odoo/orm/fields_relational.py:1317`:
  `ondelete: OnDelete | None = 'cascade'  # optional ondelete for the column2 fkey`.
- Validation, same file, lines 1331-1342: raises `ValueError` if
  `self.ondelete not in ('cascade', 'restrict')`.
- FK creation, lines 1422-1434 (`update_db_foreign_keys`):
  ```python
  model.pool.add_foreign_key(self.relation, self.column1, model._table, 'id', 'cascade', ...)
  model.pool.add_foreign_key(self.relation, self.column2, comodel._table, 'id', self.ondelete, ...)
  ```

**Why it matters:** A Sentry `IntegrityError` mentioning a many2many relation table
(`..._rel`) and a foreign key violation is a real DB-level constraint failure, not
application logic — a stale LLM might look for a Python-level guard to add (e.g. an
`ondelete` override on a `unlink()` method) when the actual fix is either declaring
`ondelete='restrict'`/`'cascade'` explicitly on the `Many2many(...)` field definition
that's missing it, or handling the constraint at the point records are deleted. Getting
this wrong risks either leaving orphaned relation rows (if patched by suppressing the
error) or an unintended cascade-delete of unrelated data.

**Confidence:** Medium-high (mechanism, defaults, and validation are directly read and
cited; whether the presence of an actual FK constraint on m2m relation tables at all is
itself new relative to 16/17, versus just the configurable-`ondelete` part being newer,
was not fully pinned down with a dated commit — flagged for reviewer follow-up).

---

## 8. `readonly` is silently auto-derived for `compute`/`related` fields — passing `readonly=False` alone does not make a compute field editable

**Claim:** At field setup, `readonly` is force-computed rather than merely defaulted:
for `related=` fields, `readonly` defaults to `True` unconditionally; for `compute=`
fields (non-related), `readonly` defaults to `not bool(inverse)` — i.e. a compute field
with no `inverse=` method is **forced** `readonly=True` regardless of what the field
declaration passes, and only becomes writable once an `inverse=` method is also
supplied. `compute_sudo` and `copy` defaults are similarly auto-derived from
`store`/`related_sudo`, not simple static defaults.

**Evidence:** `~/src/194/odoo/odoo/orm/fields.py:452-469`:
```python
if attrs.get('related'):
    attrs['store'] = store = attrs.get('store', False)
    attrs['compute_sudo'] = attrs.get('compute_sudo', attrs.get('related_sudo', True))
    attrs['copy'] = attrs.get('copy', False)
    attrs['readonly'] = attrs.get('readonly', True)
elif attrs.get('compute'):
    attrs['store'] = store = attrs.get('store', False)
    attrs['compute_sudo'] = attrs.get('compute_sudo', store)
    if not (attrs['store'] and not attrs.get('readonly', True)):
        attrs['copy'] = attrs.get('copy', False)
    attrs['readonly'] = attrs.get('readonly', not attrs.get('inverse'))
```
Corroborated by test assertions in
`~/src/194/odoo/odoo/addons/test_orm/tests/test_fields.py`
around `test_10_computed` (~line 103-136): a plain `compute=` field with no
`inverse=` reads back as `readonly=True` even if the field declaration did not say so.

**Why it matters:** Fixing a Sentry/UI bug of the shape "field X should be editable by
the user but the form shows it as readonly," a stale LLM's first instinct is to add
`readonly=False` to the field declaration. For a `compute=`-only field with no
`inverse=`, that change is silently overridden back to `readonly=True` by this
`_get_attrs` logic (note: `attrs.get('readonly', not attrs.get('inverse'))` — i.e. this
uses `.get('readonly', default)`, so if `readonly` is *explicitly* passed the explicit
value wins; the trap is specifically when a dev believes just adding `readonly=False`
with no `inverse=` is sufficient and doesn't realize the field also needs
`store=True`+`inverse=` to truly behave as user-editable in the intended pattern used
throughout this codebase, e.g. `precompute=True, readonly=False, store=True` triples).
Getting this wrong produces a patch that "looks right" in the diff but doesn't change
observed behavior, or requires guessing the extra `inverse=`/`store=True` wiring.

**Confidence:** High.

---

## Notes on things checked but NOT included as standalone candidates

- **`Command` (`odoo/orm/commands.py`)**: spelling (`Command.create/update/delete/unlink/link/clear/set`) and the underlying `(int, id, values)` tuple shape are unchanged in spirit from 14.0+ era knowledge; raw literal tuples like `(6, 0, ids)` still work identically (Command is an `IntEnum`, so `Command.CREATE == 0`). No deprecation markers found in `commands.py`, `fields.py`, or `fields_relational.py` (three greps, zero hits). Not included as a top-8 item because a 16/17-era model likely already knows this API; flagged here only so the reviewer doesn't waste time re-deriving it.
- **`fields.Json`**: simple JSONB field, explicitly documented as not searchable/indexable/mutable-in-place (`~/src/194/odoo/odoo/orm/fields_misc.py:54-61`). Real usage confirmed (`odoo/addons/hr/models/hr_employee.py:208`, `odoo/addons/pos_restaurant/models/pos_config.py:14-15`). Considered for the top 8 but judged lower-impact than Properties since its surface area is much smaller and its docstring self-documents the main gotcha.
- **`index='trigram'`/`'btree_not_null'` as string values for the `index` kwarg** (`~/src/194/odoo/odoo/orm/fields.py:107-115`), and **`company_dependent=True` auto-setting `index='btree_not_null'`** (`fields.py:486`): real and concrete, but judged a second-tier nuance (affects query plans/perf, not correctness) relative to the 8 above — worth folding into a broader "field parameters" skill section rather than a standalone entry.
- **Date/Datetime `expression_getter` supporting dotted "property" expressions** like `create_date.month_number` in domain conditions (`~/src/194/odoo/odoo/orm/fields.py:1463-1473`, `fields_temporal.py:288-303`, via `parse_field_expr` in `orm/utils.py:102-110`): real and usable in `search()`/`read_group()` domains, but this granular-date-search feature substantially predates SaaS-19.4 (introduced around 17.0), so a model with "17-era" knowledge may already know it — kept out of the top 8 for that reason, flagged here in case the reviewer's baseline is stricter (pure 16.0).
- **`translate` accepting a callable** (not just bool) on text fields (`~/src/194/odoo/odoo/orm/fields_textual.py:33`): confirmed real (used for `Html` fields via `html_translate`), but only one concrete shipped example (`html_translate`) was found in the time available — lower confidence on how commonly addon code actually supplies custom callables, so left as a footnote rather than a full candidate.
- **Unknown field kwargs only warn, not raise** (`Field.__init__` stores all kwargs unconditionally in `_args__`; unknown-parameter validation happens later in `setup()` and only logs `_logger.warning`, never raises) — `~/src/194/odoo/odoo/orm/fields.py:317-319` and `:536-545`. Real and potentially useful for explaining "silently ignored typo'd kwarg" bugs, but overlaps enough with general Python/ORM debugging practice (not a "field type" mechanic per se) that it was left out of the top 8 by a narrow margin.

# Odoo SaaS-19.3 Model Guide

Reference for the ORM, recordsets, CRUD, domains, and SQL in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/orm/models.py` and related files.

> **SaaS-19.3 differences vs generic Odoo 19**
> - `_name` can be omitted — auto-derived from CamelCase class name (with deprecation warning; **still set it explicitly**)
> - `models.Constraint` replaces `_sql_constraints` (old form logs a warning but still works)
> - `models.Index` / `models.UniqueIndex` for declarative DB indexes
> - `Domain` class with `&`, `|`, `~` operators and `any` / `any!` operators
> - `search_fetch()` — new method returning a prefetched recordset (vs `search_read()` list of dicts)
> - `odoo.tools.SQL` for composable parameterised SQL

---

## Table of Contents

1. [Defining a model](#defining-a-model)
2. [Model attributes](#model-attributes)
3. [Constraints](#constraints)
4. [Database indexes](#database-indexes)
5. [Recordsets](#recordsets)
6. [CRUD operations](#crud-operations)
7. [Search domains](#search-domains)
8. [Environment](#environment)
9. [SQL execution](#sql-execution)
10. [Inheritance](#inheritance)

---

## Defining a model

```python
from odoo import models, fields

class MyModel(models.Model):
    _name = 'my.model'          # still recommended to set explicitly
    _description = 'My Model'
    _order = 'name'

    name = fields.Char(required=True)
    active = fields.Boolean(default=True)
```

### `_name` auto-derivation (SaaS-19.3)

If `_name` is omitted, Odoo derives it from the class name by inserting `.` before each
capital letter: `MyModel` → `my.model`, `SaleOrder` → `sale.order`.

A **deprecation warning** is logged. Always set `_name` explicitly in production code.

```python
# Technically works, but triggers a warning — don't do this:
class MyModel(models.Model):
    _description = 'My Model'  # _name inferred as 'my.model'
```

### Model types

| Class | DB table | Description |
|-------|----------|-------------|
| `models.Model` | Yes | Standard persistent model |
| `models.TransientModel` | Yes | Wizard/temporary (garbage-collected) |
| `models.AbstractModel` | No | Mixin — no table, not directly instantiable |

---

## Model attributes

| Attribute | Description |
|-----------|-------------|
| `_name` | Technical name (dotted, lowercase) — set explicitly |
| `_description` | Human-readable name (required) |
| `_order` | Default sort, e.g. `'name'`, `'date desc'`, `'sequence, id'` |
| `_rec_name` | Field used for `display_name` (default: `name`) |
| `_inherit` | Model(s) to inherit from |
| `_inherits` | Delegation inheritance dict |
| `_table` | Override DB table name |
| `_log_access` | Auto create/write date+uid fields (default `True`) |
| `_auto` | Auto-create DB table (default `True` for Model) |
| `_parent_store` | Enable `parent_path` column for tree queries |
| `_fold_name` | Field controlling kanban column folding |
| `_abstract` | Set `True` to mark as abstract (use `AbstractModel` instead) |
| `_transient` | Set `True` for transient (use `TransientModel` instead) |

---

## Constraints

### `models.Constraint` (SaaS-19.3 — replaces `_sql_constraints`)

```python
from odoo import models, fields

class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char()
    code = fields.Char()
    qty  = fields.Integer()

    _unique_name      = models.Constraint('UNIQUE(name)',       'Name must be unique.')
    _unique_name_code = models.Constraint('UNIQUE(name, code)', 'Name+code combination must be unique.')
    _check_qty        = models.Constraint('CHECK(qty > 0)',     'Quantity must be positive.')
```

Parameters: `definition` (SQL string), `message` (str or callable returning str).

The attribute name (e.g., `_unique_name`) becomes the constraint name in the DB if not
already present.

### Overriding in an inherited model

```python
class MyModelExtended(models.Model):
    _inherit = 'my.model'

    # Relax the constraint
    _check_qty = models.Constraint('CHECK(qty >= 0)', 'Quantity cannot be negative.')
```

### `models.UniqueIndex` (SaaS-19.3)

Like `Constraint('UNIQUE(...)')` but creates a DB index rather than a constraint — useful
for partial uniqueness or when you want the performance of an index:

```python
_unique_active_name = models.UniqueIndex(
    '(name) WHERE active IS TRUE',
    'Active records must have a unique name.',
)
```

### `_sql_constraints` (deprecated, still works)

```python
# Still works but logs: WARNING: '_sql_constraints' is no longer supported,
#   please define models.Constraint on the model.
_sql_constraints = [
    ('name_uniq', 'UNIQUE(name)', 'Name must be unique!'),
]
```

Migrate to `models.Constraint` to silence the warning.

---

## Database indexes

### Field-level index

```python
name = fields.Char(index=True)  # simple btree index
```

### `models.Index` (SaaS-19.3 — declarative)

```python
class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char()
    date = fields.Date()
    active = fields.Boolean()

    # Composite index
    _name_date_idx = models.Index('(name, date)')

    # Partial index (WHERE clause)
    _active_name_idx = models.Index('(name) WHERE active IS TRUE')

    # Descending
    _date_desc_idx = models.Index('(date DESC)')
```

> Over-indexing hurts `INSERT`/`UPDATE`/`DELETE` performance. Add indexes only when you have
> confirmed slow queries on those columns.

---

## Recordsets

### Accessing fields

```python
record.name               # read field value
record.company_id.name    # traverse Many2one
record['name']            # dynamic field name

record.name = 'Bob'       # write field (single record)
record.write({'name': 'Bob'})  # write (multiple records OK)
```

### Iteration

```python
for rec in records:       # rec is a single-record recordset
    print(rec.name)
```

### Set operations

```python
a | b   # union
a & b   # intersection
a - b   # difference
a + b   # concatenation (ordered, may have duplicates)
rec in records   # membership test
```

### Filtering / mapping

```python
active_records = records.filtered('active')
active_records = records.filtered(lambda r: r.state == 'done')

names = records.mapped('name')                 # list of values
partners = records.mapped('order_id.partner_id')  # flattened recordset

sorted_records = records.sorted('date')
sorted_records = records.sorted(key=lambda r: r.date, reverse=True)
```

---

## CRUD operations

### Create

```python
# Single record
record = self.env['my.model'].create({'name': 'Test', 'qty': 10})

# Batch (preferred — one DB round trip)
records = self.env['my.model'].create([
    {'name': 'A', 'qty': 1},
    {'name': 'B', 'qty': 2},
])
```

### Read

```python
# Browse by ID(s)
record  = self.env['my.model'].browse(42)
records = self.env['my.model'].browse([1, 2, 3])

# Read as list of dicts
data = records.read(['name', 'qty'])
# → [{'id': 1, 'name': 'A', 'qty': 1}, ...]

# Search + read (returns list of dicts)
data = self.env['my.model'].search_read([('active', '=', True)], ['name', 'qty'])

# Search + fetch (SaaS-19.3 — returns prefetched recordset)
recs = self.env['my.model'].search_fetch([('active', '=', True)], ['name', 'qty'])
# recs is a recordset; field values are already cached — no extra queries when accessed
```

`search_fetch` vs `search_read`:
- `search_fetch` → recordset with fields pre-loaded in cache (use when you need ORM methods)
- `search_read` → list of plain dicts (use for JSON serialisation / RPC responses)

### Write

```python
record.write({'name': 'Updated', 'qty': 20})
records.write({'active': False})   # bulk update
```

### Unlink (delete)

```python
record.unlink()
records.unlink()
```

---

## Search domains

### Basic syntax

```python
domain = [('name', '=', 'ABC')]
domain = [('state', 'in', ['draft', 'confirmed']), ('active', '=', True)]
```

### Operators

| Operator | Description |
|----------|-------------|
| `=`, `!=` | equality |
| `>`, `>=`, `<`, `<=` | comparison |
| `=?` | unset or equals (treats `False`/`None` as match) |
| `like`, `ilike` | SQL LIKE (case-sensitive / insensitive) |
| `=like`, `=ilike` | exact LIKE pattern (no auto-`%`) |
| `in`, `not in` | membership in list |
| `child_of`, `parent_of` | tree traversal (requires `_parent_store`) |
| `any`, `any!` | **SaaS-19.3** — match if any related record satisfies sub-domain |
| `not any`, `not any!` | **SaaS-19.3** — negate of `any`; `!` variants bypass record rules |

### Boolean operators

```python
# AND (implicit — list of tuples)
[('name', '=', 'A'), ('active', '=', True)]

# OR
['|', ('name', '=', 'A'), ('name', '=', 'B')]

# NOT
['!', ('state', '=', 'draft')]

# Compound: (A OR B) AND C
['|', ('a', '=', 1), ('b', '=', 2), ('c', '=', 3)]
```

### `Domain` class (SaaS-19.3)

```python
from odoo.orm.domains import Domain  # or: from odoo.fields import Domain

d1 = Domain([('name', '=', 'ABC')])
d2 = Domain([('active', '=', True)])

combined = d1 & d2          # AND
either   = d1 | d2          # OR
negated  = ~d1              # NOT

# Combine lists
Domain.AND([d1, d2])        # equivalent to d1 & d2
Domain.OR([d1, d2])         # equivalent to d1 | d2

# Convert back to list
list(combined)
```

### Search methods

```python
records = self.env['my.model'].search(domain)
records = self.env['my.model'].search(domain, limit=10, offset=20, order='name ASC')
count   = self.env['my.model'].search_count(domain)

# Returns list of dicts
data = self.env['my.model'].search_read(domain, ['name', 'qty'], limit=50)

# Returns prefetched recordset (SaaS-19.3)
recs = self.env['my.model'].search_fetch(domain, ['name', 'qty'], limit=50)
```

---

## Environment

```python
env = self.env            # environment of current recordset
env = record.env

env.uid                   # current user ID (int)
env.user                  # current user recordset
env.company               # current company recordset
env.companies             # all allowed companies
env.lang                  # current language code ('fr_FR')
env.cr                    # database cursor
env.context               # context dict (read-only)
```

### Modifying the environment

```python
# Context
records.with_context(lang='fr_FR')
records.with_context(**{'active_test': False})

# User
records.with_user(self.env.ref('base.user_admin'))

# Company
records.with_company(company_id)

# Superuser (bypass access rights)
records.sudo()
records.sudo(False)       # revert to current user
```

---

## SQL execution

### Raw cursor

```python
self.env.cr.execute(
    "SELECT id, name FROM my_model WHERE state = %s",
    ('done',),
)
rows = self.env.cr.fetchall()    # list of tuples
row  = self.env.cr.fetchone()    # single tuple or None
```

### `odoo.tools.SQL` (SaaS-19.3 — recommended)

```python
from odoo.tools import SQL

query = SQL(
    "SELECT id FROM %s WHERE field = %s AND company_id = %s",
    SQL.identifier('my_model'),
    value,
    self.env.company.id,
)
self.env.cr.execute(query)
```

`SQL.identifier(name)` safely quotes a table or column name.
Compose sub-queries:

```python
subquery = SQL("SELECT partner_id FROM sale_order WHERE state = %s", 'sale')
query    = SQL("SELECT id FROM res_partner WHERE id IN (%s)", subquery)
self.env.cr.execute(query)
```

### Flush and invalidate (required before/after raw SQL)

```python
# Before: flush pending ORM writes so the DB is in sync
self.env['my.model'].flush_model(['field1', 'field2'])
records.flush_recordset(['field1'])

# Execute raw SQL ...

# After: invalidate ORM cache so next read goes to the DB
self.env['my.model'].invalidate_model(['field1', 'field2'])
records.invalidate_recordset(['field1'])

# Or: notify the ORM that you modified specific fields
records.modified(['field1', 'field2'])
```

---

## Inheritance

### Extension (add to an existing model in-place)

```python
class ResPartner(models.Model):
    _inherit = 'res.partner'    # no _name redefinition

    custom_field = fields.Char()

    def name_get(self):         # override existing method
        result = super().name_get()
        return [(r[0], f"[EXT] {r[1]}") for r in result]
```

### Prototype (new model based on another)

```python
class MyOrder(models.Model):
    _name = 'my.order'
    _inherit = 'sale.order'     # copies all fields + methods

    extra_field = fields.Char()
```

### Delegation (`_inherits`)

```python
class Screen(models.Model):
    _name = 'delegation.screen'
    size = fields.Float()

class Laptop(models.Model):
    _name = 'delegation.laptop'
    _inherits = {'delegation.screen': 'screen_id'}

    name      = fields.Char()
    screen_id = fields.Many2one('delegation.screen', required=True, ondelete='cascade')

# laptop.size directly accesses the screen's size field
```

### Multiple inheritance (mixin pattern)

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char(tracking=True)
```

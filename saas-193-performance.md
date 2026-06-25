# Odoo SaaS-19.3 Performance Guide

Reference for ORM performance, caching, and query optimisation in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/orm/` and `/home/achraf/src/193/odoo/addons/web/models/`

> **SaaS-19.3 key additions**
> - `search_fetch()` — combined search + prefetch in one call (replaces search + read pattern)
> - `@api.readonly` — marks method as read-only, enables read-replica routing
> - `web_read()` / `web_search_read()` — client-optimised nested field loading
> - `PREFETCH_MAX = 1000` — batch size constant
> - `@tools.ormcache` with `cache='stable'` option for long-lived caches

---

## Table of Contents

1. [search_fetch — the right way to search + read](#search_fetch)
2. [@api.readonly — read replica routing](#apireadonly)
3. [flush and invalidate — cache management](#flush-and-invalidate)
4. [@tools.ormcache — method-level LRU cache](#toolsormcache)
5. [with_prefetch — prefetch context control](#with_prefetch)
6. [_read_group — aggregate queries](#_read_group)
7. [web_read / web_search_read — nested field loading](#web_read--web_search_read)
8. [N+1 query anti-patterns and fixes](#n1-query-anti-patterns-and-fixes)
9. [Field-level prefetch control](#field-level-prefetch-control)
10. [Monitoring](#monitoring)

---

## search_fetch

**Preferred method for searching + reading fields.** Combines the search query and the
field prefetch in a single database round trip — never use `search()` + `read()` separately.

```python
def search_fetch(
    self,
    domain: DomainType,
    field_names: Sequence[str] | None = None,
    offset: int = 0,
    limit: int | None = None,
    order: str | None = None,
) -> Self:
```

```python
# GOOD — one DB round trip, field values pre-loaded in cache
records = self.env['sale.order'].search_fetch(
    [('state', '=', 'sale')],
    ['name', 'partner_id', 'amount_total'],
    limit=50,
    order='date_order DESC',
)
for rec in records:
    print(rec.name, rec.amount_total)   # No extra queries

# BAD — two round trips
records = self.env['sale.order'].search([('state', '=', 'sale')], limit=50)
data    = records.read(['name', 'partner_id', 'amount_total'])  # Second query
```

`search_fetch` vs `search_read`:
- `search_fetch` → recordset (use when you need ORM methods, computed fields, or method calls)
- `search_read` → list of dicts (use for JSON serialisation / RPC responses)

---

## @api.readonly

Marks a method as read-only. The DB cursor is opened in read-only mode, which:
1. Allows routing to a **read replica** (if the deployment has one)
2. **Raises an error** if the method accidentally tries to write

```python
from odoo import api, models

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.readonly
    def get_summary(self):
        """Safe for read replicas — only reads data."""
        return {
            'name': self.name,
            'total': self.amount_total,
            'lines': self.order_line.mapped('name'),
        }
```

Also applies to HTTP controllers:

```python
@http.route('/api/orders', auth='user', type='json', readonly=True)
def list_orders(self):
    return request.env['sale.order'].search_read([], ['name', 'state'])
```

> Do NOT use `@api.readonly` if the method calls `write()`, `create()`, `unlink()`, or any
> method that modifies database state — the transaction will fail with a ReadOnlyError.

---

## flush and invalidate

Required **before and after raw SQL** to keep the ORM cache consistent with the database.

```python
# flush_model — write all pending ORM changes for this model to DB
self.env['my.model'].flush_model()
self.env['my.model'].flush_model(['field1', 'field2'])  # only specific fields

# flush_recordset — only flush pending changes on records in self
records.flush_recordset()
records.flush_recordset(['name', 'state'])

# invalidate_model — clear ORM cache for all records of the model
self.env['my.model'].invalidate_model()
self.env['my.model'].invalidate_model(['field1'], flush=False)  # skip pre-flush

# invalidate_recordset — clear ORM cache for records in self only
records.invalidate_recordset()
records.invalidate_recordset(['computed_field'], flush=False)
```

### Canonical pattern around raw SQL

```python
def _sync_from_external(self):
    # 1. Flush pending ORM writes so DB is up to date
    self.env['my.model'].flush_model(['amount', 'state'])

    # 2. Raw SQL update
    self.env.cr.execute("""
        UPDATE my_model
        SET amount = amount * 1.1
        WHERE state = 'active'
    """)

    # 3. Invalidate so next ORM read fetches fresh data
    self.env['my.model'].invalidate_model(['amount'])
```

---

## @tools.ormcache

LRU cache for expensive model methods. Scoped per registry (i.e., per database + process).
Cache key is always `(model_name, *evaluated_args)`.

```python
from odoo import tools

class MyModel(models.Model):
    _name = 'my.model'

    @tools.ormcache('self.env.uid', 'self.env.lang')
    def get_user_config(self):
        """Cached per uid + lang combination."""
        return self.env['my.config'].search_read(
            [('user_id', '=', self.env.uid)],
            ['name', 'value'],
        )

    @tools.ormcache('model_name', cache='stable')
    def _get_defaults_for_model(self, model_name: str):
        """Long-lived cache ('stable' survives module upgrades)."""
        return {
            r['key']: r['value']
            for r in self.env[model_name].search_read([], ['key', 'value'])
        }
```

**Rules:**
- Methods must return **simple values** — dicts, lists, strings, numbers, tuples
- **Never return recordsets** (the cursor used to build the recordset will be closed)
- `cache='stable'` = only cleared on registry reload or explicit call to `registry.clear_cache('stable')`
- Default `cache='default'` = cleared at end of each request

### Cache invalidation

```python
# Clear all ormcache (default + stable)
self.env.registry.clear_cache()

# Clear only the default cache
self.env.registry.clear_cache('default')

# Pattern: invalidate on write
def write(self, vals):
    result = super().write(vals)
    if 'config_key' in vals:
        self.env.registry.clear_cache()
    return result
```

---

## with_prefetch

Controls which record IDs are batched together when the ORM triggers a prefetch.

```python
# Default: search() already sets up prefetch over all result IDs
orders = self.env['sale.order'].search([])
for order in orders:
    _ = order.partner_id  # batches all partner_ids in one query

# When building a custom list of IDs, prefetch is empty by default
order = self.env['sale.order'].browse(42)
_ = order.partner_id   # single query — no prefetch context

# Fix: pass prefetch_ids explicitly
ids = [42, 43, 44]
orders = self.env['sale.order'].browse(ids).with_prefetch(ids)
for order in orders:
    _ = order.partner_id  # batched across all 3 orders
```

---

## _read_group

Aggregate query — replaces loops with `sum()`/`len()` in Python.

```python
def _read_group(
    self,
    domain: DomainType,
    groupby: Sequence[str] = (),
    aggregates: Sequence[str] = (),
    having: DomainType = (),
    offset: int = 0,
    limit: int | None = None,
    order: str | None = None,
) -> list[tuple]:
```

```python
# Count orders per partner, filtered to 'sale' state
rows = self.env['sale.order']._read_group(
    domain=[('state', '=', 'sale')],
    groupby=['partner_id'],
    aggregates=['__count', 'amount_total:sum'],
    order='amount_total:sum DESC',
    limit=10,
)
# rows = [(partner_record, count, total), ...]
for partner, count, total in rows:
    print(f"{partner.name}: {count} orders, {total:.2f}")
```

### Supported aggregate specs

| Spec | SQL | Notes |
|------|-----|-------|
| `field:sum` | `SUM(field)` | |
| `field:avg` | `AVG(field)` | |
| `field:min` / `field:max` | `MIN` / `MAX` | |
| `field:count_distinct` | `COUNT(DISTINCT field)` | |
| `__count` | `COUNT(*)` | rows in group |
| `field:recordset` | `ARRAY_AGG(id)` → recordset | returns a recordset |

### Date groupby granularities

```python
groupby=['date_order:day']     # 'YYYY-MM-DD'
groupby=['date_order:week']    # ISO week
groupby=['date_order:month']   # 'YYYY-MM-01'
groupby=['date_order:quarter'] # 'YYYY-MM-01' (Q start)
groupby=['date_order:year']    # 'YYYY-01-01'
```

---

## web_read / web_search_read

Client-optimised reading with nested field specifications. Minimises round trips for
hierarchical data (e.g., form views with One2many lines).

```python
@api.readonly
def web_search_read(self, domain, specification, offset=0, limit=None, order=None, count_limit=None):
    ...

@api.readonly
def web_read(self, specification: dict[str, dict]) -> list[dict]:
    ...
```

```python
specification = {
    'id': {},
    'name': {},
    'partner_id': {
        'fields': {
            'id':    {},
            'name':  {},
            'email': {},
        }
    },
    'order_line': {
        'fields': {
            'product_id': {
                'fields': {'name': {}, 'list_price': {}}
            },
            'product_uom_qty': {},
            'price_unit': {},
        }
    },
}

# Fetch with nested structure in one operation
result = self.env['sale.order'].web_search_read(
    [('state', '=', 'sale')],
    specification,
    limit=10,
)
# result['records'] = [{id, name, partner_id: {id, name, email}, order_line: [...]}]
```

---

## N+1 query anti-patterns and fixes

### Anti-pattern 1 — relational access inside a loop

```python
# BAD — triggers a SQL query for each partner
for order in orders:
    print(order.partner_id.name)  # N extra queries

# GOOD — search_fetch pre-loads partner_id into cache
orders = self.env['sale.order'].search_fetch(
    [], ['name', 'partner_id']
)
for order in orders:
    print(order.partner_id.name)  # cache hit, no extra query
```

### Anti-pattern 2 — search inside a loop

```python
# BAD — one query per partner
for partner in partners:
    orders = self.env['sale.order'].search([('partner_id', '=', partner.id)])

# GOOD — one query for all partners
orders = self.env['sale.order'].search([('partner_id', 'in', partners.ids)])
```

### Anti-pattern 3 — Python aggregation instead of SQL

```python
# BAD — loads all records into memory
total = sum(order.amount_total for order in self.env['sale.order'].search([]))

# GOOD — single SQL aggregate
rows = self.env['sale.order']._read_group([], aggregates=['amount_total:sum'])
total = rows[0][0]
```

### Anti-pattern 4 — repeated field access without prefetch

```python
# BAD — each .read() triggers a SQL query
for product in products:
    cost  = product.standard_price
    price = product.list_price

# GOOD — prefetch both fields at once
products.read(['standard_price', 'list_price'])
for product in products:
    cost  = product.standard_price  # cache hits
    price = product.list_price
```

### Anti-pattern 5 — `mapped()` on deep paths without prefetch

```python
# BAD — N queries for partner country names
names = orders.mapped('partner_id.country_id.name')

# GOOD — ensure fields are prefetched first
orders = self.env['sale.order'].search_fetch(
    [], ['partner_id']
)
# partner_id is prefetched; country_id will still batch via ORM prefetch
names = orders.mapped('partner_id.country_id.name')
```

---

## Field-level prefetch control

```python
class MyModel(models.Model):
    _name = 'my.model'

    # Default — included in the standard prefetch group
    name = fields.Char(prefetch=True)

    # Custom group — prefetched separately (e.g., company-dependent fields)
    company_note = fields.Char(prefetch='company_dependent')

    # Never prefetch — avoid loading large fields unless explicitly requested
    document_content = fields.Binary(prefetch=False)
    raw_html = fields.Html(prefetch=False)
```

`PREFETCH_MAX = 1000` — the ORM batches prefetch queries in groups of at most 1000 IDs.

---

## Monitoring

```bash
# Send SIGUSR1 to the Odoo process to dump ormcache stats to the log
kill -USR1 <odoo_pid>

# Output shows per-cache: Entries, Hit, Miss, Hit Ratio, TX Hit Ratio
# Low hit ratio → cache invalidated too often or key too granular
```

For query-level profiling, enable the `profile` module or use `--log-level=debug` with
`--log-handler=odoo.sql_db:DEBUG` to log every SQL statement with its duration.

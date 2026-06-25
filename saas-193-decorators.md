# Odoo SaaS-19.3 API Decorators Guide

Reference for `@api.*` decorators in the SaaS-19.3 codebase.
Source of truth: `/home/achraf/src/193/odoo/odoo/api/decorators.py`

> **SaaS-19.3 differences vs generic Odoo 19**
> - `@api.depends_context` is **new** — explicitly declares context keys that trigger recomputation
> - `@api.readonly` is **new** — marks RPC-callable methods that run on a read-only cursor
> - `@api.private` is present (introduced in Odoo 19) — blocks RPC access to a public method
> - `@api.returns` has been **removed** (was Odoo ≤17 only)

---

## Table of Contents

1. [Method decorators overview](#method-decorators-overview)
2. [@api.depends](#apidepends)
3. [@api.depends_context](#apidepends_context)
4. [@api.constrains](#apiconstrains)
5. [@api.onchange](#apionchange)
6. [@api.ondelete](#apiondelete)
7. [@api.model](#apimodel)
8. [@api.model_create_multi](#apimodel_create_multi)
9. [@api.autovacuum](#apiautovacuum)
10. [@api.private](#apiprivate)
11. [@api.readonly](#apireadonly)
12. [Decision tree](#decision-tree)

---

## Method decorators overview

```python
from odoo import api, models

class MyModel(models.Model):
    _name = 'my.model'

    @api.depends('amount', 'tax_rate')
    def _compute_total(self):
        for rec in self:
            rec.total = rec.amount * (1 + rec.tax_rate)
```

All decorators live in `odoo.api`. Import with `from odoo import api`.

---

## @api.depends

Marks a compute method and declares which fields trigger recomputation.

```python
total = fields.Float(compute='_compute_total', store=True)

@api.depends('amount', 'tax_rate')
def _compute_total(self):
    for rec in self:
        rec.total = rec.amount * (1 + rec.tax_rate)
```

### Dotted paths (relational traversal)

```python
@api.depends('partner_id.email', 'partner_id.country_id.code')
def _compute_partner_info(self):
    for rec in self:
        rec.partner_email = rec.partner_id.email
        rec.partner_country = rec.partner_id.country_id.code
```

Dotted paths subscribe to changes on related records — the field is recomputed whenever
`partner_id`, `partner_id.email`, or `partner_id.country_id.code` changes.

### Multiple computed fields in one method

```python
discount_value = fields.Float(compute='_apply_discount')
total = fields.Float(compute='_apply_discount')

@api.depends('amount', 'discount')
def _apply_discount(self):
    for rec in self:
        discount = rec.amount * rec.discount
        rec.discount_value = discount
        rec.total = rec.amount - discount
```

### Inverse (writable computed field)

```python
duration_hours = fields.Float(compute='_compute_duration', inverse='_inverse_duration', store=True)

@api.depends('date_start', 'date_stop')
def _compute_duration(self):
    for rec in self:
        if rec.date_start and rec.date_stop:
            delta = rec.date_stop - rec.date_start
            rec.duration_hours = delta.total_seconds() / 3600

def _inverse_duration(self):
    for rec in self:
        if rec.date_start:
            rec.date_stop = rec.date_start + timedelta(hours=rec.duration_hours)
```

### Search on a computed field

```python
upper_name = fields.Char(compute='_compute_upper', search='_search_upper')

@api.depends('name')
def _compute_upper(self):
    for rec in self:
        rec.upper_name = (rec.name or '').upper()

def _search_upper(self, operator, value):
    if operator == 'like':
        operator = 'ilike'
    return [('name', operator, value)]
```

> **Note**: `@api.depends('id')` raises `NotImplementedError` — never depend on `id`.

---

## @api.depends_context

**SaaS-19.3 — new decorator.**  
Declares context keys that trigger recomputation of a non-stored computed field.  
Without this, context changes are silently ignored by the ORM cache.

```python
price_display = fields.Float(compute='_compute_price_display')

@api.depends('list_price')
@api.depends_context('pricelist_id', 'currency_id')
def _compute_price_display(self):
    pricelist = self.env.context.get('pricelist_id')
    for rec in self:
        if pricelist:
            rec.price_display = self.env['product.pricelist'].browse(pricelist)._get_product_price(rec, 1)
        else:
            rec.price_display = rec.list_price
```

### Special context keys

Some keys have built-in semantics:

| Key | Behaviour |
|-----|-----------|
| `'company'` | Recompute when the active company changes (`allowed_company_ids` in context) |
| `'uid'` | Recompute when the current user changes |
| `'active_test'` | Recompute when the active_test context flag changes |

```python
@api.depends('name')
@api.depends_context('company')
def _compute_localized_name(self):
    company = self.env.company
    for rec in self:
        rec.localized_name = f"{rec.name} ({company.name})"
```

> Use `@api.depends_context` instead of passing context values through dotted `@api.depends` paths —
> it is more explicit and avoids cache invalidation surprises.

---

## @api.constrains

Called on `create` and `write` to validate field values. Raises `ValidationError` to block the operation.

```python
from odoo.exceptions import ValidationError

@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for rec in self:
        if rec.date_end and rec.date_end < rec.date_start:
            raise ValidationError("End date must be after start date.")
```

> **Cannot use dotted paths** — unlike `@api.depends`, you can only name direct fields on the model.

```python
# WRONG — dotted path not supported in @api.constrains
@api.constrains('partner_id.email')
def _check_partner_email(self):
    pass

# RIGHT — check the relational field itself
@api.constrains('partner_id')
def _check_partner_email(self):
    for rec in self:
        if not rec.partner_id.email:
            raise ValidationError("Partner must have an email.")
```

---

## @api.onchange

Triggered in the form view when a field value changes. Updates the in-memory record before save.

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.email = self.partner_id.email
        self.phone = self.partner_id.phone
```

### Return a warning

```python
@api.onchange('amount')
def _onchange_amount(self):
    if self.amount < 0:
        return {
            'warning': {
                'title': "Negative amount",
                'message': "Amount should not be negative.",
            }
        }
```

> **No CRUD in onchange** — do not call `.create()`, `.write()`, or `.unlink()` inside an onchange.
> The record is not yet saved; any DB writes would be orphaned or cause inconsistencies.

---

## @api.ondelete

Validates whether records can be deleted. Called before `unlink`. Preferred over overriding `unlink()`.

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_draft(self):
    if any(rec.state != 'draft' for rec in self):
        raise UserError("Only draft records can be deleted.")
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| `at_uninstall` | `True` = also run when the module is uninstalled. `False` = skip during uninstall (usual choice). |

> **Prefer `@api.ondelete` over overriding `unlink()`** — overriding `unlink()` with a raise
> blocks module uninstall, which calls `unlink()` to clean up demo/init data.

---

## @api.model

Marks a method as model-level — `self` is any recordset (may be empty). The method doesn't
operate on specific records, just on the model.

```python
@api.model
def get_defaults(self):
    return {
        'currency_id': self.env.company.currency_id.id,
        'company_id': self.env.company.id,
    }
```

Can be called on any recordset:

```python
self.env['my.model'].get_defaults()
self.env['my.model'].browse([]).get_defaults()  # same thing
```

> When you name a method `create` and decorate it with `@api.model`, the ORM automatically
> redirects single-dict calls through `model_create_multi`. You don't need both decorators on `create`.

---

## @api.model_create_multi

Handles both single-dict and list-of-dicts `create` calls. Required on any `create` override in Odoo 17+.

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals.setdefault('state', 'draft')
        if not vals.get('reference'):
            vals['reference'] = self.env['ir.sequence'].next_by_code('my.model')
    return super().create(vals_list)
```

The decorator normalises a single `dict` to `[dict]` before calling the method, so the method
always receives a list.

---

## @api.autovacuum

Marks a method to be run by the daily garbage-collection cron (`ir.autovacuum`). Used for
periodic cleanup that doesn't need its own `ir.cron` record.

```python
@api.autovacuum
def _gc_old_logs(self):
    cutoff = fields.Datetime.now() - timedelta(days=90)
    self.search([('create_date', '<', cutoff)]).unlink()
```

> The method is called once per day per database. Keep it idempotent and non-blocking.

---

## @api.private

**Introduced in Odoo 19, present in SaaS-19.3.**  
Prevents the method from being called via XML-RPC or JSON-RPC, even if it has a public name.

```python
@api.private
def compute_sensitive_data(self):
    return self._fetch_internal_keys()
```

### `@api.private` vs underscore convention

| | `_underscore_method` | `@api.private` |
|-|----------------------|----------------|
| RPC access blocked | By convention only | Enforced by ORM |
| Callable from action buttons | No (by convention) | No (enforced) |
| Use when | Normal private helper | Public name but must not be exposed |

---

## @api.readonly

**SaaS-19.3 — new decorator.**  
Marks a method that can be called via RPC using a **read-only database cursor**.
The client must explicitly request a read-only call; the server validates it at the ORM level.

```python
@api.readonly
def get_summary_stats(self):
    return {
        'count': self.search_count([]),
        'total': sum(self.search([]).mapped('amount')),
    }
```

Use for:
- Pure read endpoints exposed via JSON-RPC / REST
- Methods called from dashboards or reporting that never write
- Pairing with `@http.route(..., readonly=True)` on the controller side

> `@api.readonly` on the model and `readonly=True` on the route are **independent** but
> complementary. The route-level `readonly` controls the DB cursor at the HTTP layer;
> `@api.readonly` declares intent at the ORM layer and enables the client to opt in.

---

## Decision tree

```
Need to react to field value changes?
├── Recompute a stored/unstored field → @api.depends
│   └── Result also depends on context keys → add @api.depends_context
├── Validate data before save → @api.constrains (no dotted paths)
├── Update other fields on the same record in the form → @api.onchange (no CRUD)
└── Block deletion → @api.ondelete(at_uninstall=False)

Need to define method behaviour?
├── Method doesn't operate on specific records → @api.model
├── Override create() → @api.model_create_multi
├── Daily cleanup/GC → @api.autovacuum
├── Public name but must not be callable via RPC → @api.private
└── Read-only RPC method → @api.readonly
```

---

## Common patterns

### Computed field that varies by company (SaaS-19.3)

```python
display_price = fields.Float(compute='_compute_display_price')

@api.depends('list_price')
@api.depends_context('company')
def _compute_display_price(self):
    company = self.env.company
    for rec in self:
        rec.display_price = rec.list_price * company.currency_id.rate
```

### Safe delete guard

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_not_locked(self):
    if any(rec.state == 'locked' for rec in self):
        raise UserError("Locked records cannot be deleted.")
```

### Batch create with sequence

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if not vals.get('name') or vals['name'] == '/':
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    return super().create(vals_list)
```

### Read-only RPC summary (SaaS-19.3)

```python
@api.readonly
def dashboard_data(self):
    domain = [('company_id', '=', self.env.company.id)]
    return {
        'total_orders': self.env['sale.order'].search_count(domain),
        'total_revenue': sum(
            self.env['sale.order'].search(domain + [('state', '=', 'sale')]).mapped('amount_total')
        ),
    }
```

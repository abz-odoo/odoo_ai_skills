# Odoo SaaS-19.3 Field Guide

Reference for field types and parameters in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/orm/fields*.py`
Exports: `/home/achraf/src/193/odoo/odoo/fields/__init__.py`

> **SaaS-19.3 differences vs generic Odoo 19**
> - `fields.Json` — new field backed by PostgreSQL `jsonb`
> - `fields.Properties` + `fields.PropertiesDefinition` — new dynamic property system
> - `fields.Float` gains `min_display_digits` parameter
> - `fields.Html` gains granular sanitize options + `sanitize_overridable`
> - `fields.Char` has `trim=True` by default (server-side trimming)
> - `track_visibility` is **deprecated** — use `tracking=True` on the field
> - Field source split across `odoo/orm/fields_*.py` (not a single `fields.py`)

---

## Table of Contents

1. [Type summary](#type-summary)
2. [Basic fields](#basic-fields)
3. [Text fields](#text-fields)
4. [Numeric fields](#numeric-fields)
5. [Date fields](#date-fields)
6. [Relational fields](#relational-fields)
7. [Binary fields](#binary-fields)
8. [New in SaaS-19.3: Json](#new-fieldsjson)
9. [New in SaaS-19.3: Properties](#new-fieldsproperties)
10. [Computed fields](#computed-fields)
11. [Related fields](#related-fields)
12. [Field parameters reference](#field-parameters-reference)
13. [Reserved field names](#reserved-field-names)
14. [ORM Commands](#orm-commands)

---

## Type summary

| Category | Types |
|----------|-------|
| Basic | `Boolean`, `Char`, `Integer`, `Float` |
| Text | `Text`, `Html`, `Monetary`, `Selection` |
| Date | `Date`, `Datetime` |
| Binary | `Binary`, `Image` |
| Relational | `Many2one`, `One2many`, `Many2many` |
| **New SaaS-19.3** | `Json`, `Properties`, `PropertiesDefinition` |
| Pseudo | `Reference`, `Many2oneReference` |

---

## Basic fields

### Boolean

```python
active = fields.Boolean(default=True)
is_company = fields.Boolean(string="Is Company")
```

### Char

```python
name = fields.Char(required=True, index=True)
code = fields.Char(size=10)  # max DB length
reference = fields.Char(trim=False)  # disable server-side trimming (default trim=True)
```

> `trim=True` (default): the server strips leading/trailing whitespace on write.

### Integer

```python
sequence = fields.Integer(default=10)
count = fields.Integer(string="Count")
```

### Float

```python
price = fields.Float(digits='Product Price')        # reference a decimal.precision record
weight = fields.Float(digits=(12, 3))               # (total digits, decimal places)
duration = fields.Float(
    digits=(5, 2),
    min_display_digits=1,  # SaaS-19.3: minimum decimal digits shown in UI
)
```

`min_display_digits` — new in SaaS-19.3. Ensures the UI always shows at least N decimal
digits, even when the value would display fewer. Can also reference a `decimal.precision` name.

---

## Text fields

### Text

```python
description = fields.Text()
notes = fields.Text(string="Internal Notes", translate=True)
```

### Html

```python
body = fields.Html(string="Body")

# Granular sanitize options (all default as shown):
body_full = fields.Html(
    sanitize=True,                      # master switch
    sanitize_overridable=False,         # True = users in base.group_sanitize_override can bypass
    sanitize_tags=True,                 # strip disallowed HTML tags
    sanitize_attributes=True,           # strip disallowed attributes
    sanitize_style=False,               # strip style= attributes
    sanitize_form=True,                 # strip <form> elements
    sanitize_conditional_comments=True, # remove IE conditional comments
    sanitize_output_method='html',      # 'html' or 'xhtml'
    strip_style=False,                  # strip all style attributes
    strip_classes=False,                # strip all class attributes
)

# Preset for email content:
email_body = fields.Html(sanitize='email_outgoing')
```

### Monetary

```python
amount = fields.Monetary(currency_field='currency_id')  # default field name
price_tax = fields.Monetary(string="Tax", currency_field='currency_id')
```

Requires a `Many2one('res.currency')` field named `currency_id` on the model (or the field
named in `currency_field`).

### Selection

```python
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirmed', 'Confirmed'),
    ('done', 'Done'),
], default='draft', tracking=True)

# Dynamic selection from a method:
type = fields.Selection(selection='_get_type_selection')

@api.model
def _get_type_selection(self):
    return [('a', 'Type A'), ('b', 'Type B')]
```

Extending a selection in an inherited model:

```python
state = fields.Selection(
    selection_add=[('new_state', 'New State')],
    ondelete={'new_state': 'set default'},
)
```

`ondelete` values: `'set null'`, `'cascade'`, `'set default'`, `'set VALUE'`, or a callable.

---

## Date fields

### Date

```python
date = fields.Date()
deadline = fields.Date(default=fields.Date.context_today)
```

```python
from odoo.fields import Date
today = Date.context_today(self)
next_week = Date.add(Date.today(), weeks=1)
last_month = Date.subtract(Date.today(), months=1)
start = Date.start_of(Date.today(), 'month')
end = Date.end_of(Date.today(), 'month')
```

### Datetime

```python
scheduled_at = fields.Datetime()
created_at = fields.Datetime(default=fields.Datetime.now)
```

```python
from odoo.fields import Datetime
now = Datetime.now()
ts = Datetime.context_timestamp(self, datetime_value)  # convert UTC → user TZ
next_hour = Datetime.add(Datetime.now(), hours=1)
start = Datetime.start_of(Datetime.now(), 'day')
end = Datetime.end_of(Datetime.now(), 'day')
```

Datetime values are always stored as UTC. Client-side conversion uses the user's timezone.

---

## Relational fields

### Many2one

```python
partner_id = fields.Many2one('res.partner', ondelete='restrict')
user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
company_id = fields.Many2one('res.company', default=lambda self: self.env.company)
```

| Parameter | Description |
|-----------|-------------|
| `comodel_name` | Related model (positional arg) |
| `ondelete` | `'set null'` (default), `'cascade'`, `'restrict'` |
| `domain` | Filter domain for the field |
| `context` | Extra context for searches |
| `check_company` | Enforce `company_id` match |
| `index` | Add DB index (recommended for FK fields) |
| `delegate` | Expose parent fields directly on this model |

### One2many

```python
line_ids = fields.One2many('sale.order.line', 'order_id', string="Lines")
```

The `inverse_name` (second arg) is the `Many2one` field on the comodel pointing back.

### Many2many

```python
# Simple (auto relation table name)
tag_ids = fields.Many2many('crm.tag', string="Tags")

# Explicit relation table (required when two Many2many link the same pair of models)
product_ids = fields.Many2many(
    'product.product',
    'my_model_product_rel',  # relation table
    'my_model_id',           # column for this model
    'product_id',            # column for product.product
    string="Products",
)
```

`ondelete` on Many2many accepts only `'cascade'` (default) or `'restrict'`.

---

## Binary fields

### Binary

```python
file_data = fields.Binary(string="File", attachment=True)
```

`attachment=True` stores the binary in `ir.attachment` instead of the DB column (recommended
for large files).

### Image

```python
image_1920 = fields.Image(string="Image")
image_128 = fields.Image(
    max_width=128,
    max_height=128,
    related='image_1920',
    store=True,
)
```

`Image` automatically resizes on write, keeping aspect ratio. `verify_resolution=True` (default)
rejects images exceeding the maximum resolution.

---

## New: fields.Json

**SaaS-19.3** — backed by a PostgreSQL `jsonb` column.

```python
from odoo import fields

metadata = fields.Json(string="Metadata")
config = fields.Json(string="Configuration")
```

- Stored as `jsonb` in PostgreSQL — supports GIN indexes and jsonpath queries at DB level
- Python value is a native `dict` / `list` / scalar (auto-serialised via `json.dumps`/`json.loads`)
- No ORM search, filtering, or domain support — for structured data you control entirely
- No domain operators work on Json fields through the ORM

```python
# Reading
record.metadata  # → dict, list, or None

# Writing
record.metadata = {'key': 'value', 'count': 42}

# Updating (must reassign — no in-place mutation)
data = dict(record.metadata or {})
data['new_key'] = 'new_value'
record.metadata = data
```

---

## New: fields.Properties

**SaaS-19.3** — dynamic user-defined fields stored as `jsonb`, without DB schema changes.

```python
from odoo import fields

# On the definition model (e.g., project.project):
task_properties_definition = fields.PropertiesDefinition(string="Task Properties")

# On the record model (e.g., project.task):
task_properties = fields.Properties(
    definition='project_id.task_properties_definition',
    string="Properties",
)
```

`Properties` links to a `PropertiesDefinition` field on a parent record (via `definition`).
The definition record holds the schema; each task record holds its values.

### Sub-types supported in Properties definitions

`'boolean'`, `'integer'`, `'float'`, `'char'`, `'text'`, `'html'`, `'date'`, `'datetime'`,
`'monetary'`, `'signature'`, `'many2one'`, `'many2many'`, `'selection'`, `'tags'`, `'separator'`

### PropertiesDefinition

```python
task_properties_definition = fields.PropertiesDefinition(
    string="Task Properties",
)
```

Stored as `jsonb`. Each entry in the definition has keys:
`name`, `string`, `type`, `comodel` (for relational), `default`, `selection`/`tags` (for those types),
`domain`, `hidden`, `view_in_cards`, `fold_by_default`.

```python
# Programmatically hide/show a property:
project.task_properties_definition = fields.PropertiesDefinition.set_properties_visibility(
    project.task_properties_definition,
    property_names=['my_prop'],
    hidden=True,
)
```

---

## Computed fields

```python
total = fields.Float(compute='_compute_total', store=True)

@api.depends('price', 'qty')
def _compute_total(self):
    for rec in self:
        rec.total = rec.price * rec.qty
```

With context dependency (SaaS-19.3):

```python
display_price = fields.Float(compute='_compute_display_price')

@api.depends('list_price')
@api.depends_context('pricelist_id')
def _compute_display_price(self):
    ...
```

With inverse (writable):

```python
full_name = fields.Char(compute='_compute_full_name', inverse='_inverse_full_name')
```

With search:

```python
upper_name = fields.Char(compute='_compute_upper', search='_search_upper')

def _search_upper(self, operator, value):
    if operator == 'like':
        operator = 'ilike'
    return [('name', operator, value)]
```

---

## Related fields

```python
partner_name = fields.Char(related='partner_id.name', store=True)
partner_email = fields.Char(related='partner_id.email', readonly=True)
currency_name = fields.Char(related='currency_id.name', store=True, depends=['currency_id'])
```

`store=True` on a related field persists the value in the DB and adds an index, making it
searchable and sortable. Without `store`, it recomputes on every read.

---

## Field parameters reference

### Common parameters (all field types)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `string` | field name | UI label |
| `required` | `False` | Block save if empty |
| `readonly` | `False` | Not editable in UI |
| `index` | `False` | Add DB index |
| `default` | `None` | Value or callable |
| `help` | `''` | Tooltip text |
| `groups` | `''` | Comma-separated group XML IDs restricting visibility |
| `copy` | `True` | Include when duplicating record |
| `tracking` | `False` | Log changes to chatter (`True` or integer priority) |
| `translate` | `False` | Enable translation (Char, Text, Html) |
| `compute` | — | Method name or lambda |
| `store` | `False` for computed | Persist computed value in DB |
| `related` | — | Dotted path to a field on a related record |

> `tracking=True` replaces the deprecated `track_visibility='always'`.
> `tracking=<int>` sets a priority: lower number = shown higher in the chatter summary.

---

## Reserved field names

| Name | Description |
|------|-------------|
| `id` | Record identifier (auto, never declare) |
| `display_name` | `_rec_name` field value (computed) |
| `create_date`, `create_uid` | Creation timestamp and author |
| `write_date`, `write_uid` | Last write timestamp and author |
| `name` | Default `_rec_name` field |
| `active` | When `False`, record is archived (hidden from searches) |
| `state` | Lifecycle stages (used by chatter/kanban) |
| `parent_id` | Tree structure parent |
| `parent_path` | Materialized path for tree queries |
| `company_id` | Multi-company isolation field |
| `sequence` | Integer for drag-and-drop ordering |
| `color` | Integer color index for kanban views |
| `priority` | Selection for star rating widgets |

---

## ORM Commands

Use `Command` class for `One2many` / `Many2many` write operations:

```python
from odoo.fields import Command

record.write({
    'line_ids': [
        Command.create({'product_id': 1, 'qty': 10}),   # (0, 0, vals)
        Command.update(line.id, {'qty': 20}),            # (1, id, vals)
        Command.delete(line.id),                         # (2, id)
        Command.unlink(tag.id),                          # (3, id) — M2M only: remove link
        Command.link(tag.id),                            # (4, id) — M2M only: add link
        Command.clear(),                                 # (5,) — remove all
        Command.set([id1, id2]),                         # (6, 0, ids) — replace set
    ]
})
```

`Command.delete` removes the record; `Command.unlink` only removes the M2M link.

# Odoo SaaS-19.3 Migration Guide

Reference for writing migration scripts in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/modules/migration.py`

> **SaaS-19.3 notes vs generic Odoo 19**
> - Version string is `saas~19.3` (not `19.0`) — affects version comparisons
> - Migration directories use the **module's own semantic version**, not the Odoo server version
> - Three phases unchanged: `pre`, `post`, `end`
> - `base64` type in XML data files deprecated — use `file` + read the bytes directly
> - `_sql_constraints` logs a warning; migrate to `models.Constraint`

---

## Table of Contents

1. [When to write a migration script](#when-to-write-a-migration-script)
2. [Directory structure](#directory-structure)
3. [Script phases](#script-phases)
4. [Script template](#script-template)
5. [Key changes to migrate from Odoo 17/18](#key-changes-to-migrate-from-odoo-1718)
6. [Common migration patterns](#common-migration-patterns)
7. [Module hooks](#module-hooks)
8. [Testing](#testing)

---

## When to write a migration script

Write a migration script when upgrading a module introduces changes that require transforming
**existing data** in the database:

- Renaming a field or model
- Splitting or merging a field
- Changing a selection value key
- Initialising a new non-nullable column from existing data
- Cleaning up orphaned records

Do **not** write a migration script for:
- Adding a new nullable field with a default — the ORM handles that
- Adding a new model — the ORM creates the table
- Data that only matters for new installations (use `data/` files)

---

## Directory structure

Migration scripts live under a `migrations/` subdirectory of the module.
The subdirectory name is the **module's own semantic version** (not the Odoo server version):

```
my_module/
└── migrations/
    ├── 1.1/
    │   ├── pre-migration.py     # runs before module update
    │   └── post-migration.py    # runs after module update
    ├── 1.2/
    │   └── post-migration.py
    └── 0.0.0/
        └── post-migration.py    # runs on ANY version change
```

The `__manifest__.py` version is what drives which scripts run:

```python
# __manifest__.py
'version': '1.2',  # bumping from 1.1 → 1.2 triggers migrations/1.2/*
```

For SaaS-specific migrations the version may use the SaaS format:

```python
'version': 'saas~19.3.1.0',  # migrations/saas~19.3.1.0/ or migrations/1.0/
```

> The migration loader compares the installed version with the new version using a version
> comparator that understands both `x.y.z` and `saas~x.y.z` formats.

---

## Script phases

| Script name | When it runs |
|-------------|--------------|
| `pre-migration.py` | Before the module is updated (before ORM applies schema changes) |
| `post-migration.py` | After the module is updated (after ORM, before next module) |
| `end-migration.py` | After all modules have been updated |

The `0.0.0/` directory is special: scripts there run on **any** version change, regardless of
the old and new version numbers. Useful for maintenance scripts that should always run.

---

## Script template

Every migration script must expose a `migrate(cr, version)` function:

```python
import logging

_logger = logging.getLogger(__name__)


def migrate(cr, version):
    """
    Args:
        cr: database cursor (psycopg2)
        version: installed version before this migration (string), or None on first install
    """
    if not version:
        return  # fresh install, nothing to migrate

    _logger.info("Migrating %s → ...", version)

    cr.execute("""
        UPDATE my_model
        SET new_field = old_field
        WHERE new_field IS NULL
    """)
    _logger.info("Migrated %d rows", cr.rowcount)
```

> `cr` is a raw psycopg2 cursor — no ORM available. Use `%s` placeholders, never f-strings
> or `.format()` for values.

For `post-migration.py` where the ORM is available via the registry:

```python
from odoo import api, SUPERUSER_ID


def migrate(cr, version):
    env = api.Environment(cr, SUPERUSER_ID, {})
    records = env['my.model'].search([('state', '=', 'old_value')])
    records.write({'state': 'new_value'})
```

---

## Key changes to migrate from Odoo 17/18

### `<tree>` → `<list>` in views

```xml
<!-- Before (Odoo 17 and earlier) -->
<tree string="Records">
    <field name="name"/>
</tree>

<!-- After (Odoo 18+, mandatory in SaaS-19.3) -->
<list string="Records">
    <field name="name"/>
</list>
```

### `attrs` → direct attributes

```xml
<!-- Before -->
<field name="amount" attrs="{'invisible': [('state', '!=', 'done')], 'required': [('state', '=', 'done')]}"/>

<!-- After -->
<field name="amount"
    invisible="state != 'done'"
    required="state == 'done'"/>
```

### `unlink()` override → `@api.ondelete`

```python
# Before (Odoo 17 and earlier) — breaks module uninstall
def unlink(self):
    if any(r.state != 'draft' for r in self):
        raise UserError("Only draft records can be deleted.")
    return super().unlink()

# After (Odoo 18+ / SaaS-19.3)
@api.ondelete(at_uninstall=False)
def _unlink_if_draft(self):
    if any(r.state != 'draft' for r in self):
        raise UserError("Only draft records can be deleted.")
```

### `group_operator` → `aggregator`

```python
# Before (Odoo 17 and earlier)
amount_total = fields.Monetary(group_operator="sum")

# After (Odoo 18+ / SaaS-19.3)
amount_total = fields.Monetary(aggregator="sum")
```

### `_sql_constraints` → `models.Constraint` (SaaS-19.3)

`_sql_constraints` still works but logs a deprecation warning. Migrate to `models.Constraint`:

```python
# Before — still works but logs WARNING in SaaS-19.3
class MyModel(models.Model):
    _name = 'my.model'
    _sql_constraints = [
        ('name_uniq', 'UNIQUE(name)', 'Name must be unique!'),
        ('qty_positive', 'CHECK(qty > 0)', 'Qty must be positive!'),
    ]

# After (SaaS-19.3)
class MyModel(models.Model):
    _name = 'my.model'

    _name_uniq = models.Constraint('UNIQUE(name)', 'Name must be unique!')
    _qty_positive = models.Constraint('CHECK(qty > 0)', 'Qty must be positive!')
```

### Raw SQL → `odoo.tools.SQL`

```python
# Before — fragile string formatting
cr.execute("SELECT id FROM %s WHERE field = '%s'" % (table, value))  # SQL injection risk

# After (SaaS-19.3) — composable, parameterised
from odoo.tools import SQL
cr.execute(SQL("SELECT id FROM %s WHERE field = %s", SQL.identifier(table), value))
```

### `track_visibility` → `tracking`

```python
# Before (Odoo 16 and earlier)
name = fields.Char(track_visibility='onchange')

# After (Odoo 17+ / SaaS-19.3)
name = fields.Char(tracking=True)
```

---

## Common migration patterns

### Rename a field

```python
def migrate(cr, version):
    # The ORM will add the new column; we copy data from the old column first
    cr.execute("""
        ALTER TABLE my_model
        ADD COLUMN IF NOT EXISTS new_name VARCHAR
    """)
    cr.execute("""
        UPDATE my_model
        SET new_name = old_name
        WHERE new_name IS NULL AND old_name IS NOT NULL
    """)
```

### Change a selection value key

```python
def migrate(cr, version):
    cr.execute("""
        UPDATE my_model
        SET state = 'confirmed'
        WHERE state = 'open'
    """)
```

### Delete orphaned records

```python
def migrate(cr, version):
    cr.execute("""
        DELETE FROM my_model
        WHERE parent_id IS NOT NULL
          AND NOT EXISTS (
            SELECT 1 FROM my_parent WHERE id = my_model.parent_id
          )
    """)
```

### Initialise a new column from another

```python
from odoo import api, SUPERUSER_ID

def migrate(cr, version):
    env = api.Environment(cr, SUPERUSER_ID, {})
    # Flush to ensure pending writes are committed before raw SQL
    env['my.model'].flush_model()

    cr.execute("""
        UPDATE my_model
        SET computed_field = price * quantity
        WHERE computed_field IS NULL
    """)

    env['my.model'].invalidate_model(['computed_field'])
```

### Remove a module from depends

If your module removed a dependency, existing databases may still have the old module installed.
Clean up in `post-migration.py`:

```python
def migrate(cr, version):
    # Mark old bridge module as to uninstall
    cr.execute("""
        UPDATE ir_module_module
        SET state = 'to remove'
        WHERE name = 'old_bridge_module'
          AND state = 'installed'
    """)
```

---

## Module hooks

Hooks run via Python (with `env`) and are registered in `__manifest__.py`.
Use them for operations that cannot be expressed as data files or migration scripts.

```python
# In __init__.py

def post_init_hook(env):
    """Runs after module installation. env = Odoo Environment (SUPERUSER)."""
    # Initialise sequence numbers, create default records, etc.
    env['my.model'].search([]).write({'initialised': True})


def uninstall_hook(env):
    """Runs after module uninstallation."""
    # Remove external service config, custom DB objects, etc.
    env.cr.execute("DROP TABLE IF EXISTS my_custom_cache")
```

```python
# In __manifest__.py
'post_init_hook': 'post_init_hook',
'uninstall_hook': 'uninstall_hook',
```

> `post_load` (no `env`, runs before registry) is for server-wide Python setup only — see the
> manifest skill.

---

## Testing

```python
from odoo.tests import TransactionCase


class TestMigration(TransactionCase):

    def test_state_renamed(self):
        # Create a record in the old state (simulate pre-migration data)
        record = self.env['my.model'].create({'name': 'test', 'state': 'confirmed'})
        # Verify the migration result
        self.assertEqual(record.state, 'confirmed')

    def test_field_copied(self):
        record = self.env['my.model'].create({'name': 'test', 'old_field': 'value'})
        self.assertEqual(record.new_field, 'value')
```

For migration scripts specifically, run the module upgrade in a test database and verify
data integrity manually or with post-upgrade assertions in `end-migration.py`.

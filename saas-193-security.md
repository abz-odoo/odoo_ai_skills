# Odoo SaaS-19.3 Security Guide

Reference for access control (ACLs), record rules, groups, and sudo() in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/addons/base/models/`

> **SaaS-19.3 key notes**
> - Global rules (no `group_id`) are **deprecated** — log a warning; always specify a group
> - `all_implied_ids` / `all_implied_by_ids` are now **recursive** (transitive closure)
> - `company_ids` in `domain_force` = list of **all active company IDs** (from context)
> - `Domain.TRUE` returned by `_compute_domain` when superuser → no restrictions

---

## Table of Contents

1. [Two-level security model](#two-level-security-model)
2. [Access Control Lists (ACLs)](#access-control-lists-acls)
3. [Record rules](#record-rules)
4. [Rule combination logic](#rule-combination-logic)
5. [Group hierarchy](#group-hierarchy)
6. [Base groups reference](#base-groups-reference)
7. [Multi-company rules](#multi-company-rules)
8. [sudo()](#sudo)
9. [Security in code](#security-in-code)
10. [Common patterns](#common-patterns)

---

## Two-level security model

Every database operation goes through **two independent checks**. Both must pass:

```
Request
  │
  ▼
1. ACL check (ir.model.access)
   Does this GROUP have this OPERATION on this MODEL?
   If NO → AccessError (model level)
  │
  ▼
2. Record rule check (ir.rule)
   Does the DOMAIN FILTER allow this USER to see this RECORD?
   If NO → AccessError or filtered out (record level)
  │
  ▼
 Allowed
```

`sudo()` bypasses **both** levels.

---

## Access Control Lists (ACLs)

Defined in `ir.model.access` — usually via a CSV file.

### CSV format

File: `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model user,model_my_model,base.group_user,1,0,0,0
access_my_model_manager,my.model manager,model_my_model,my_module.group_my_manager,1,1,1,1
```

| Column | Description |
|--------|-------------|
| `id` | XML ID for this ACL record |
| `name` | Human-readable label |
| `model_id:id` | XML ID of the model — `model_` + model name with `.` → `_` |
| `group_id:id` | XML ID of the group (leave empty for global — **deprecated**) |
| `perm_read` | 1 = allowed, 0 = denied |
| `perm_write` | 1 = allowed, 0 = denied |
| `perm_create` | 1 = allowed, 0 = denied |
| `perm_unlink` | 1 = allowed, 0 = denied |

> `model_id` derivation: `my.model` → `model_my_model`, `sale.order` → `model_sale_order`

### XML format (for non-CSV scenarios)

```xml
<record id="access_my_model_manager" model="ir.model.access">
    <field name="name">my.model manager</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="group_id" ref="my_module.group_my_manager"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="True"/>
</record>
```

---

## Record rules

Defined in `ir.rule` — filter records a user can see or modify.

```xml
<!-- security/record_rules.xml -->

<!-- Example 1: user can only see their own records -->
<record id="rule_my_model_own" model="ir.rule">
    <field name="name">my.model: own records only</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="groups" eval="[Command.set([ref('base.group_user')])]"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="False"/>
    <field name="perm_unlink" eval="False"/>
</record>

<!-- Example 2: managers see everything (no domain restriction) -->
<record id="rule_my_model_manager" model="ir.rule">
    <field name="name">my.model: manager full access</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="groups" eval="[Command.set([ref('my_module.group_my_manager')])]"/>
    <field name="domain_force">[(1, '=', 1)]</field><!-- always true -->
</record>
```

### `domain_force` available variables

| Variable | Type | Value |
|----------|------|-------|
| `user` | `res.users` record | Current user (without context) |
| `user.id` | int | Current user's ID |
| `user.partner_id` | record | User's partner |
| `user.commercial_partner_id` | record | Commercial partner (for portal) |
| `company_id` | int | Current company ID |
| `company_ids` | list of int | All active company IDs (from context) |

### Permissions on the rule

Each `ir.rule` has 4 permission flags (`perm_read`, `perm_write`, `perm_create`, `perm_unlink`),
all defaulting to `True`. Set only the ones you want the rule to apply to.

---

## Rule combination logic

```
Result domain = (global_rule_1 AND global_rule_2 AND ...)
                AND
                (group_rule_A OR group_rule_B OR ...)
```

- **Global rules** (no `groups`) — combined with AND — ALL must pass
- **Group rules** (with `groups`) — combined with OR among rules whose group the user belongs to

> Global rules without a group are **deprecated** in SaaS-19.3 (warning logged).
> Always specify a group, or use `base.group_everyone` for universal rules.

### Worked example

Rules on model `my.model`:
1. Global rule: `[('active', '=', True)]` ← deprecated, prefer group_everyone
2. Group rule (group_user): `[('user_id', '=', user.id)]`
3. Group rule (group_manager): `[(1, '=', 1)]` (always true)

For a **regular user** (has `group_user`, not `group_manager`):
```
active = True  AND  (user_id = uid)
```

For a **manager** (has both `group_user` and `group_manager`):
```
active = True  AND  (user_id = uid  OR  1 = 1)
# → active = True  AND  True  → sees all active records
```

---

## Group hierarchy

```xml
<!-- security/groups.xml -->
<record model="res.groups" id="group_my_user">
    <field name="name">My Module / User</field>
    <field name="category_id" ref="base.module_category_my_module"/>
    <field name="implied_ids" eval="[Command.link(ref('base.group_user'))]"/>
</record>

<record model="res.groups" id="group_my_manager">
    <field name="name">My Module / Manager</field>
    <field name="category_id" ref="base.module_category_my_module"/>
    <field name="implied_ids" eval="[Command.link(ref('my_module.group_my_user'))]"/>
</record>
```

`implied_ids` is the direct parent. `all_implied_ids` is the full transitive closure.
When checking rules, Odoo uses `all_implied_ids` — so a manager automatically satisfies
any rule requiring `group_my_user`.

---

## Base groups reference

| XML ID | Name | Who has it |
|--------|------|-----------|
| `base.group_system` | Administrator | System admins |
| `base.group_erp_manager` | ERP Manager | Implied by `group_system` |
| `base.group_user` | Internal User | All logged-in internal users |
| `base.group_portal` | Portal | Customer portal users |
| `base.group_public` | Public | Unauthenticated visitors |
| `base.group_everyone` | Everyone | Implied by all three above |

Hierarchy:
```
group_system
  └─ group_erp_manager
      └─ group_user
          └─ group_everyone

group_portal  └─ group_everyone
group_public  └─ group_everyone
```

---

## Multi-company rules

Standard pattern: allow records that belong to the user's active companies, or are shared
(no `company_id`):

```xml
<record id="rule_my_model_company" model="ir.rule">
    <field name="name">my.model: multi-company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="groups" eval="[Command.set([ref('base.group_user')])]"/>
    <field name="domain_force">
        ['|',
            ('company_id', 'parent_of', company_ids),
            ('company_id', '=', False)
        ]
    </field>
</record>
```

- `('company_id', 'parent_of', company_ids)` — record's company is one of the user's
  active companies or a parent of them
- `('company_id', '=', False)` — shared/global records (no company)

The model must have a `company_id` field for this to work. Add it to the model and make
the form view respect it.

---

## sudo()

`sudo()` enables superuser mode — **bypasses both ACLs and record rules**.

```python
# Read with superuser — no record rule filtering
all_records = self.env['my.model'].sudo().search([])

# Write as superuser — no access check
record.sudo().write({'internal_field': value})

# Revert to current user after sudo
record.sudo().with_user(self.env.user)

# Explicit False = revert to non-sudo
record.sudo(False)
```

### When to use sudo()

| Scenario | Use sudo? |
|----------|-----------|
| Background job / cron reading all companies' data | Yes |
| Install hook initialising default data | Yes |
| User action reading their own records | No |
| User action writing records they should own | No |
| Checking existence of a record across companies | Yes (read-only) |
| Bypassing record rules to log/audit | Yes |

**Warning**: `sudo()` can silently cross company boundaries. In multi-company setups, a
`sudo().search()` returns records from all companies — be explicit with `company_id`
filters when using sudo in multi-company contexts.

---

## Security in code

### Checking access explicitly

```python
# Check if current user can read this model
self.env['my.model'].check_access('read')   # raises AccessError if denied

# Check without raising
can_read = bool(self.env['my.model'].has_access('read'))

# Filter a recordset to only accessible records
accessible = records._filtered_access('write')
```

### @api.ondelete — secure delete guards

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_draft(self):
    """Prevent deletion of non-draft records."""
    if any(r.state != 'draft' for r in self):
        raise UserError("Only draft records can be deleted.")
```

Use `at_uninstall=False` to skip the guard during module uninstallation (avoids blocking
data cleanup).

### Hiding fields in views

```xml
<!-- Hide field from portal users -->
<field name="internal_note" groups="base.group_user"/>

<!-- Show field only to managers -->
<field name="cost_price" groups="my_module.group_my_manager"/>

<!-- Hide a whole section -->
<group groups="base.group_user">
    <field name="internal_ref"/>
    <field name="vendor_id"/>
</group>
```

The `groups` attribute takes a comma-separated list of XML IDs. The field is removed from
the view for users not in any of the listed groups.

---

## Common patterns

### Module category + group definition

```xml
<record model="ir.module.category" id="module_category_my_module">
    <field name="name">My Module</field>
    <field name="sequence">50</field>
</record>

<record model="res.groups" id="group_my_user">
    <field name="name">User</field>
    <field name="category_id" ref="module_category_my_module"/>
    <field name="implied_ids" eval="[Command.link(ref('base.group_user'))]"/>
</record>

<record model="res.groups" id="group_my_manager">
    <field name="name">Manager</field>
    <field name="category_id" ref="module_category_my_module"/>
    <field name="implied_ids" eval="[Command.link(ref('my_module.group_my_user'))]"/>
</record>
```

### Portal access pattern

```xml
<!-- ACL: portal users can read -->
access_my_model_portal,my.model portal,model_my_model,base.group_portal,1,0,0,0

<!-- Rule: portal users see only their own partner's records -->
<record id="rule_my_model_portal" model="ir.rule">
    <field name="name">my.model: portal own</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="groups" eval="[Command.set([ref('base.group_portal')])]"/>
    <field name="domain_force">[('partner_id', 'child_of', user.commercial_partner_id.id)]</field>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="False"/>
    <field name="perm_create" eval="False"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

### Restricting a field to managers only (model level)

```python
class MyModel(models.Model):
    _name = 'my.model'

    cost = fields.Monetary(groups='my_module.group_my_manager')
```

When `groups` is set on a field, non-group users:
- Cannot read or write the field via RPC
- See the field as `False` in `read()`
- Cannot search on it

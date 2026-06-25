# Odoo SaaS-19.3 Actions Guide

Reference for `ir.actions.*` in the SaaS-19.3 codebase.
Source of truth: `/home/achraf/src/193/odoo/odoo/addons/base/models/ir_actions.py`
and `/home/achraf/src/193/odoo/odoo/addons/base/models/ir_cron.py`

> **SaaS-19.3 differences vs generic Odoo 19**
> - `ir.actions.server` gains a `webhook` state and an `object_copy` state
> - `ir.actions.server` has a `usage` field distinguishing server actions from cron actions
> - `ir.cron._notify_progress()` is **deprecated** since 19.0 — use `_commit_progress()` instead
> - `_rollback_progress()` method added alongside `_commit_progress()`

---

## Table of Contents

1. [Window Actions](#window-actions)
2. [URL Actions](#url-actions)
3. [Server Actions](#server-actions)
4. [Report Actions](#report-actions)
5. [Client Actions](#client-actions)
6. [Scheduled Actions (ir.cron)](#scheduled-actions-ircron)
7. [Action Bindings](#action-bindings)

---

## Window Actions

`ir.actions.act_window` — opens views for a model.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `res_model` | Char | Model to display (required) |
| `view_mode` | Char | Comma-separated view types, e.g. `list,form` |
| `view_id` | Many2one `ir.ui.view` | Specific view to load |
| `domain` | Char | Python expression filtering records |
| `context` | Char | Python dict expression, default `{}` |
| `res_id` | Integer | Record to open in form view |
| `target` | Selection | `current` \| `new` \| `fullscreen` \| `main` |
| `limit` | Integer | Records shown in list (default 80) |
| `search_view_id` | Many2one | Specific search view |

### View types

`list`, `form`, `kanban`, `graph`, `pivot`, `calendar`, `gantt`, `map`, `activity`

> **Note**: `list` replaced `tree` in Odoo 17+. Never use `tree` in SaaS-19.3.

### XML example

```xml
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">My Records</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">list,form</field>
    <field name="domain">[('state', '=', 'active')]</field>
    <field name="context">{'default_company_id': company_id}</field>
</record>
```

### Python dict (returned from a method)

```python
return {
    'type': 'ir.actions.act_window',
    'res_model': 'sale.order',
    'view_mode': 'form',
    'res_id': self.id,
    'target': 'current',
}
```

### Open in dialog

```python
return {
    'type': 'ir.actions.act_window',
    'res_model': 'product.product',
    'views': [[False, 'form']],
    'res_id': product_id,
    'target': 'new',
}
```

### Specific view via ir.actions.act_window.view

```xml
<record model="ir.actions.act_window.view" id="action_my_model_list">
    <field name="sequence" eval="1"/>
    <field name="view_mode">list</field>
    <field name="view_id" ref="view_my_model_list"/>
    <field name="act_window_id" ref="action_my_model"/>
</record>
```

---

## URL Actions

`ir.actions.act_url` — opens a web URL.

| Field | Type | Description |
|-------|------|-------------|
| `url` | Char | URL to open (required) |
| `target` | Selection | `new` (default) \| `self` \| `download` |

```xml
<record id="action_open_docs" model="ir.actions.act_url">
    <field name="name">Documentation</field>
    <field name="url">https://odoo.com</field>
    <field name="target">new</field>
</record>
```

---

## Server Actions

`ir.actions.server` — executes code or structured operations.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `model_id` | Many2one `ir.model` | Target model (required) |
| `state` | Selection | Type of action (see below) |
| `usage` | Selection | `ir_actions_server` or `ir_cron` |
| `code` | Text | Python code (only for `code` state) |
| `binding_model_id` | Many2one | Model for contextual binding |
| `binding_type` | Selection | `action` or `report` |
| `binding_view_types` | Char | `list`, `form`, or `list,form` |

### State values (SaaS-19.3)

| State | Description |
|-------|-------------|
| `code` | Execute Python code |
| `object_create` | Create a new record |
| `object_write` | Update current record(s) |
| `object_copy` | **SaaS-19.3** Duplicate current record(s) |
| `webhook` | **SaaS-19.3** Send a webhook POST request |
| `multi` | Execute multiple child actions |

### `code` state

```xml
<record model="ir.actions.server" id="action_mark_done">
    <field name="name">Mark as Done</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">
        for rec in records:
            rec.write({'state': 'done'})
    </field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="binding_view_types">list</field>
</record>
```

Returning a next action from code:

```python
# Inside the code field — set the `action` variable to return it
action = {
    'type': 'ir.actions.act_window',
    'res_model': record._name,
    'view_mode': 'form',
    'res_id': record.id,
}
```

### `webhook` state (SaaS-19.3 specific)

Sends an HTTP POST to an external URL with selected record fields as JSON payload.

| Field | Description |
|-------|-------------|
| `webhook_url` | Destination URL |
| `webhook_field_ids` | Fields to include in the payload |
| `webhook_sample_payload` | Computed preview of the JSON payload |

```xml
<record model="ir.actions.server" id="action_notify_webhook">
    <field name="name">Notify External System</field>
    <field name="model_id" ref="model_sale_order"/>
    <field name="state">webhook</field>
    <field name="webhook_url">https://example.com/hook</field>
</record>
```

### `multi` state

```xml
<record model="ir.actions.server" id="action_multi">
    <field name="name">Multi Action</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">multi</field>
    <field name="child_ids" eval="[ref('action_step1'), ref('action_step2')]"/>
</record>
```

### Evaluation context for `code` state

| Variable | Description |
|----------|-------------|
| `model` | The model linked via `model_id` |
| `record` | Single record (when triggered on one) |
| `records` | Recordset (all selected records) |
| `env` | Odoo Environment |
| `datetime`, `dateutil`, `time`, `timezone` | Python stdlib |
| `log(message, level='info')` | Write to `ir.logging` |
| `Warning` | Raises a user-visible warning |

---

## Report Actions

`ir.actions.report` — generates PDF or HTML reports.

| Field | Type | Description |
|-------|------|-------------|
| `model` | Char | Model the report is about (required) |
| `report_type` | Selection | `qweb-pdf` or `qweb-html` |
| `report_name` | Char | External ID of the QWeb template (required) |
| `print_report_name` | Char | Python expression for file name |
| `paperformat_id` | Many2one | Paper format |
| `attachment_use` | Boolean | Cache report as attachment |
| `attachment` | Char | Python expression for attachment name |
| `binding_model_id` | Many2one | Binds to Print menu of this model |

```xml
<report
    id="report_my_document"
    model="my.model"
    string="My Document"
    report_type="qweb-pdf"
    name="my_module.report_my_document_template"
    print_report_name="'Doc-%s' % object.name"
    binding_model_id="model_my_model"
/>
```

---

## Client Actions

`ir.actions.client` — triggers a JavaScript action in the web client.

| Field | Type | Description |
|-------|------|-------------|
| `tag` | Char | Client-side action identifier (required) |
| `params` | Json | Extra data sent to the client |
| `target` | Selection | `current` \| `new` \| `fullscreen` \| `main` |

```python
return {
    'type': 'ir.actions.client',
    'tag': 'reload',
}
```

```python
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'title': 'Done',
        'message': 'Operation successful',
        'type': 'success',
    },
}
```

---

## Scheduled Actions (ir.cron)

`ir.cron` — recurring automated actions.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | Char | Display name |
| `model_id` | Many2one `ir.model` | Model to call |
| `code` | Text | Python code (`model.my_method()`) |
| `interval_number` | Integer | Frequency count |
| `interval_type` | Selection | `minutes` \| `hours` \| `days` \| `weeks` \| `months` |
| `nextcall` | Datetime | Next planned execution |
| `priority` | Integer | Lower = higher priority |
| `numbercall` | Integer | `-1` = unlimited |
| `doall` | Boolean | Run missed executions on restart |

```xml
<record id="cron_my_job" model="ir.cron">
    <field name="name">My Daily Job</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">model._cron_my_job()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>
    <field name="active" eval="True"/>
</record>
```

### Progress tracking (SaaS-19.3)

Use `_commit_progress()` inside cron methods to allow the scheduler to batch work safely.

```python
def _cron_process_records(self, *, limit=500):
    domain = [('state', '=', 'pending')]
    records = self.search(domain, limit=limit)
    for record in records:
        record._do_work()
    remaining = 0 if len(records) < limit else self.search_count(domain)
    self.env['ir.cron']._commit_progress(len(records), remaining=remaining)
```

`_commit_progress(done, *, remaining=0)` :
- commits the current transaction
- records progress in `ir.cron.progress`
- returns remaining seconds for this cron run (use to break early)

```python
def _cron_process_records(self):
    domain = [('state', '=', 'pending')]
    records = self.search(domain, limit=1000)
    self.env['ir.cron']._commit_progress(remaining=len(records))
    for record in records:
        try:
            record._do_work()
            if not self.env['ir.cron']._commit_progress(1):
                break  # time budget exhausted
        except Exception:
            self.env.cr.rollback()
```

> **DEPRECATED**: `_notify_progress(done, remaining)` was removed in 19.0.
> Always use `_commit_progress()` in SaaS-19.3.

`_rollback_progress()` — rolls back the cursor with the same cron-job bookkeeping as `_commit_progress()`.

### Triggering crons from code

```python
# Trigger at a specific time
cron_record._trigger(at=fields.Datetime.now())

# For tests only — runs synchronously
cron_record.method_direct_trigger()
```

### Security

- **3 consecutive errors or timeouts** → current execution skipped
- **5 failures over 7 days** → cron deactivated, DB admin notified

---

## Action Bindings

Bind actions to the contextual menu of a model.

| Attribute | Description |
|-----------|-------------|
| `binding_model_id` | Model whose views show this action |
| `binding_type` | `action` → Action menu \| `report` → Print menu |
| `binding_view_types` | `list`, `form`, or `list,form` (default) |

```xml
<!-- Appears in the Action menu on list view -->
<record model="ir.actions.server" id="action_export">
    <field name="name">Export Selection</field>
    <field name="model_id" ref="model_sale_order"/>
    <field name="state">code</field>
    <field name="code">records.action_export()</field>
    <field name="binding_model_id" ref="model_sale_order"/>
    <field name="binding_type">action</field>
    <field name="binding_view_types">list</field>
</record>
```

---

## Quick reference: returning actions from Python methods

```python
# Open list + form
return {
    'type': 'ir.actions.act_window',
    'res_model': 'my.model',
    'view_mode': 'list,form',
    'domain': [('partner_id', '=', self.partner_id.id)],
}

# Close dialog
return {'type': 'ir.actions.act_window_close'}

# Reload full client
return {'type': 'ir.actions.client', 'tag': 'reload'}

# Success notification (no reload)
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {'message': 'Done!', 'type': 'success', 'sticky': False},
}
```

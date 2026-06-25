# Odoo SaaS-19.3 Mixins Guide

Reference for Odoo mixins in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/addons/mail/models/`

> **SaaS-19.3 differences vs generic Odoo 19**
> - `mail.alias.mixin.optional` is a new distinct variant (alias is optional)
> - `mail.alias.mixin` now **inherits** `mail.alias.mixin.optional` (alias required)
> - `get_alias_model_name()` and `get_alias_values()` are **removed** — use `_alias_get_creation_values()`
> - `message_post()` has new params: `outgoing_email_to`, `incoming_email_to`, `incoming_email_cc`
> - `mail.tracking.duration.mixin` — **new** — tracks time spent per stage ("rotting")
> - `mail.thread.main.attachment` — **new** — manages the main attachment of a thread
> - `mail.thread.subject.suggested` — **new** — provides suggested subject for chatter
> - Activity plans system: `activity_plans_ids` computed field + `mail.activity.plan` model

---

## Table of Contents

1. [mail.thread](#mailthread)
2. [mail.activity.mixin](#mailactivitymixin)
3. [mail.alias.mixin and mail.alias.mixin.optional](#mailaliasmixin-and-mailaliasmixinoptional)
4. [mail.tracking.duration.mixin (new)](#mailtrackingdurationmixin-new)
5. [mail.thread.main.attachment (new)](#mailthreadmainattachment-new)
6. [utm.mixin](#utmmixin)
7. [portal.mixin](#portalmixin)
8. [website.published.mixin](#websitepublishedmixin)
9. [rating.mixin](#ratingmixin)

---

## mail.thread

Adds the chatter (message thread, followers, field tracking) to a model.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread']
    _description = 'My Model'

    name = fields.Char(tracking=True)
    state = fields.Selection([
        ('draft', 'Draft'),
        ('done', 'Done'),
    ], tracking=True)
    partner_id = fields.Many2one('res.partner', tracking=1)  # integer = tracking priority
```

### Form view chatter

```xml
<form string="My Model">
    <sheet>
        <!-- fields -->
    </sheet>
    <chatter/>
</form>
```

Chatter attributes:
- `open_attachments="True"` — show attachments section open by default
- `reload_on_post="True"` — reload form on new message
- `reload_on_attachment="True"` — reload on attachment change
- `reload_on_follower="True"` — reload on follower change

### Posting messages

```python
# Plain notification (no email)
record.message_post(
    body='Status changed to Done.',
    message_type='notification',
    subtype_xmlid='mail.mt_note',
)

# Comment visible to followers (sends email)
record.message_post(
    body='Please review this record.',
    subject='Review needed',
    message_type='comment',
    subtype_xmlid='mail.mt_comment',
)

# HTML body (use Markup to avoid double-escaping)
from markupsafe import Markup
record.message_post(body=Markup('<b>Important:</b> check the amount.'))
```

### New `message_post` parameters (SaaS-19.3)

```python
record.message_post(
    body='...',
    outgoing_email_to='user@example.com,other@example.com',  # NEW — extra recipients
    incoming_email_to='original@example.com',                # NEW — already notified (incoming)
    incoming_email_cc='cc@example.com',                      # NEW — already notified (CC)
)
```

`outgoing_email_to` is experimental in SaaS-19.3 — use with care.

### Receiving messages (email aliases)

```python
def message_new(self, msg_dict, custom_values=None):
    """Called when a new email arrives for an alias of this model."""
    return super().message_new(msg_dict, {
        'name': msg_dict.get('subject', 'New'),
        **(custom_values or {}),
    })

def message_update(self, msg_dict, update_vals=None):
    """Called when an email reply arrives for an existing thread."""
    return super().message_update(msg_dict, update_vals)
```

### Followers

```python
# Subscribe
record.message_subscribe(partner_ids=[p.id for p in partners])
record.message_subscribe(partner_ids=[pid], subtype_ids=[subtype.id])

# Unsubscribe
record.message_unsubscribe(partner_ids=[pid])
record.message_unsubscribe_users()   # unsubscribe current user
```

### Subtypes

Log an internal note:

```python
record.message_post(
    body='Internal note.',
    message_type='notification',
    subtype_xmlid='mail.mt_note',       # internal note (not emailed to followers)
)
```

Post a public comment (emails followers):

```python
record.message_post(
    body='Comment.',
    message_type='comment',
    subtype_xmlid='mail.mt_comment',
)
```

---

## mail.activity.mixin

Adds the activity (to-do) system.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.activity.mixin', 'mail.thread']
```

### Scheduling activities

```python
record.activity_schedule(
    'mail.mail_activity_data_todo',     # activity type XML ID
    user_id=self.env.user.id,
    summary='Please review',
    note='Additional context here.',
    date_deadline=fields.Date.today(),
)
```

Common activity type XML IDs:
- `mail.mail_activity_data_todo` — To-Do
- `mail.mail_activity_data_call` — Phone Call
- `mail.mail_activity_data_meeting` — Meeting
- `mail.mail_activity_data_email` — Email

### Completing activities

```python
record.activity_ids.action_done()
record.activity_ids.action_feedback(feedback='Done — reviewed and approved.')
```

### Activity plans (SaaS-19.3)

A new system for pre-defined activity sequences (`mail.activity.plan`):

```python
# The mixin exposes a computed field:
record.activity_plans_ids   # Many2many of mail.activity.plan records applicable to this model

# Schedule a full plan
plan = self.env.ref('my_module.my_activity_plan')
record.activity_schedule_with_view(...)   # or use the UI plan button
```

---

## mail.alias.mixin and mail.alias.mixin.optional

### SaaS-19.3 hierarchy

```
mail.alias.mixin.optional   ← alias_id optional (required=False)
        └── mail.alias.mixin  ← alias_id required (required=True)
```

Use `mail.alias.mixin.optional` when the alias is optional (e.g., a project that may or may
not have an email alias). Use `mail.alias.mixin` (required) for models that always have one.

### mail.alias.mixin (required alias)

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.alias.mixin', 'mail.thread']

    name = fields.Char()

    def _alias_get_creation_values(self):
        """SaaS-19.3: replaces get_alias_values()."""
        values = super()._alias_get_creation_values()
        values['alias_defaults'] = {'partner_id': self.partner_id.id}
        return values
```

> **BREAKING**: `get_alias_model_name()` and `get_alias_values()` are **removed** in SaaS-19.3.
> Replace with `_alias_get_creation_values()`.

### mail.alias.mixin.optional (optional alias)

```python
class MyTeam(models.Model):
    _name = 'my.team'
    _inherit = ['mail.alias.mixin.optional', 'mail.thread']

    name = fields.Char()
    # alias_id is optional — may be False if team has no alias
```

### Creating the alias record in data files

```xml
<record id="alias_my_model" model="mail.alias">
    <field name="alias_name">my-model</field>
    <field name="alias_model_id" ref="model_my_model"/>
    <field name="alias_user_id" ref="base.user_admin"/>
</record>
```

---

## mail.tracking.duration.mixin (new)

**SaaS-19.3 — new mixin.** Tracks how long records stay in each stage and flags "rotting"
records (stale beyond a threshold).

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.tracking.duration.mixin', 'mail.thread']

    stage_id = fields.Many2one('my.stage')
    # The mixin requires a stage field — configure via _track_duration_field
    _track_duration_field = 'stage_id'
```

Fields added by the mixin:

| Field | Type | Description |
|-------|------|-------------|
| `duration_tracking` | Json (stored) | Dict of `{stage_id: seconds_spent}` |
| `rotting_days` | Integer | Days in current stage (from duration_tracking) |
| `is_rotting` | Boolean | True when rotting_days exceeds stage threshold |

`duration_tracking` is **stored** (as of SaaS-19.3) — values persist across server restarts.

Useful for CRM/Helpdesk-style views that show how long a record has been idle.

---

## mail.thread.main.attachment (new)

**SaaS-19.3 — new mixin.** Manages `message_main_attachment_id` — the "primary" attachment
of a thread (e.g., the PDF of an invoice).

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['mail.thread.main.attachment', 'mail.thread']
```

Fields added:

| Field | Type | Description |
|-------|------|-------------|
| `message_main_attachment_id` | Many2one `ir.attachment` | Primary attachment of the thread |

The mixin automatically promotes the first PDF attachment posted in the thread to
`message_main_attachment_id`. Useful for document-centric models.

---

## utm.mixin

Adds UTM campaign tracking fields.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['utm.mixin']
```

Fields added: `campaign_id`, `source_id`, `medium_id` (all Many2one to their UTM models).

---

## portal.mixin

Adds customer portal access.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['portal.mixin']

    partner_id = fields.Many2one('res.partner')

    def _compute_access_url(self):
        super()._compute_access_url()
        for rec in self:
            rec.access_url = f'/my/model/{rec.id}'

    def _get_portal_return_action(self):
        return self.env.ref('my_module.action_my_model')
```

Fields added: `access_url`, `access_token`, `access_warning`.

---

## website.published.mixin

Adds website visibility control.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['website.published.mixin']
```

Fields added: `website_published` (Boolean), `website_url` (Char, computed).

Methods: `website_publish_button()` — toggles `website_published`.

---

## rating.mixin

Adds customer satisfaction rating.

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['rating.mixin', 'mail.thread']
```

Key methods:

```python
# Send a rating request via email
record.rating_send_request(
    rating_template='mail.mail_template_data_rating',
)

# Aggregate stats
stats = record.rating_get_stats()  # {'avg': 4.2, 'total': 12, ...}
```

---

## Quick reference: which mixin for what

| Goal | Mixin(s) |
|------|----------|
| Add chatter + followers | `mail.thread` |
| Track field changes in chatter | `mail.thread` + `tracking=True` on fields |
| Add to-do activities | `mail.activity.mixin` |
| Receive emails via alias (required) | `mail.alias.mixin` + `mail.thread` |
| Receive emails via alias (optional) | `mail.alias.mixin.optional` + `mail.thread` |
| Track time per stage / rotting | `mail.tracking.duration.mixin` + `mail.thread` |
| Manage main attachment | `mail.thread.main.attachment` + `mail.thread` |
| Customer portal access | `portal.mixin` |
| Website publish/unpublish | `website.published.mixin` |
| UTM campaign tracking | `utm.mixin` |
| Customer ratings | `rating.mixin` + `mail.thread` |

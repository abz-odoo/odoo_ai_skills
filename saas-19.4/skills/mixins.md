# Mixins — candidate nuances for Odoo SaaS-19.4 (mail addon)

Scope note: this pass focused on `odoo/addons/mail/models/*.py` and
`odoo/addons/mail/tests/*` (community core), cross-checked with real usages in
`enterprise/`, `industry/`-adjacent addons under community (crm, project,
hr_recruitment, documents, sign) and git history (`odoo/` is a git repo with
tags `17.0`/`18.0`, so commit ranges like `18.0..HEAD` could be used to date
features precisely). `industry/` and `design-themes/` were only grepped for
mixin usage, not read in depth — if the review needs industry-specific mixin
quirks, that needs a follow-up pass. 7 candidates below (not padded to 8 —
several other differences found, e.g. `check_access()` replacing
`check_access_rights`/`check_access_rule`, or the `odoo/fields.py` →
`odoo/orm/fields.py` package split, were judged ORM-wide rather than
mixin-specific and dropped).

---

## 1. `mail.thread`'s tracking logic was extracted into a standalone `mail.track.mixin`

**Claim**: In this codebase, value-change tracking (the machinery behind `tracking=True` fields, `mail.tracking.value` creation, `_track_prepare`/`_track_finalize`) is **no longer built into `mail.thread`** — it lives in a separate `AbstractModel` called `mail.track.mixin`, and `mail.thread` merely inherits it. Models that only want field tracking (no chatter, no followers, no message_post) can inherit `mail.track.mixin` alone, or even call its methods off the pool without inheriting it at all.

**Evidence**:
- `odoo/addons/mail/models/mail_thread.py:135-140`:
  ```python
  _name = 'mail.thread'
  _inherit = [
      'bus.listener.mixin',
      'mail.track.mixin',  # values tracking basic capabilities
  ]
  ```
- `odoo/addons/mail/models/mail_track_mixin.py:18-23` — the whole file is a new `AbstractModel` `mail.track.mixin` holding `_track_prepare`, `_track_finalize`, `_track_get_fields`, `_mail_track`, `_create_mail_tracking_values`, etc. (previously this logic lived directly inside `mail_thread.py`).
- Git evidence: commit `d4e5b45d3e89` "`[MOV] mail: move values tracking in a new mixin`" (author date 2026-03-04) — commit message explicitly says: *"Some models (try to) track value changes without really inheriting or using mail.thread capabilities ... In order to enable value tracking ... without inheriting from the whole mail.thread stack we add a new mixin ..."* This is very recent (post `18.0` tag) core work, done specifically for saas-19.
- Real usage confirming the new pattern: `enterprise/sign/models/sign_request.py:698-712` calls `self.pool['mail.track.mixin']._create_mail_tracking_values(target_record, old_val, new_val, fname, field_info)` **as a static/pool-level utility**, on a record that may not even inherit any mail mixin — with an explicit code comment: *"It lives on the abstract `mail.track.mixin` but its logic is generic, so we call it off the mixin class with the target record. This works whether or not the target model inherits the mixin."*

**Why it matters**: A stale-trained LLM believes tracking-value creation (`_create_mail_tracking_values`, `_track_get_fields`, etc.) is private/internal to `mail.thread` and only reachable by fully inheriting `mail.thread`. Fixing a Sentry crash like "tracking of field X fails on model Y that doesn't have chatter" could lead the LLM to (a) wrongly conclude the model must inherit full `mail.thread` (over-engineering, adding chatter/followers nobody wants) or (b) reimplement tracking-value formatting from scratch instead of reusing `mail.track.mixin`, or (c) miss that `self.pool['mail.track.mixin']` is a valid, intended way to reuse this code off-model.

**Confidence**: high

---

## 2. `mail.tracking.duration.mixin` gained a "rotting" feature (`is_rotting`, `rotting_days`) — not just duration tracking

**Claim**: The mixin used for stage-duration widgets (`_track_duration_field`, `duration_tracking` JSON field) now also implements a "rotting" concept: `is_rotting` (boolean, searchable via SQL) and `rotting_days` (integer), driven by a `rotting_threshold_days` field expected on the tracked stage model and a `date_last_stage_update` field expected on the inheriting model. This is a distinct, newer feature bolted onto the mixin, with its own configuration contract (`_is_rotting_feature_enabled`, `_get_rotting_domain`, `_get_rotting_depends_fields`) and a raw-SQL `_search_is_rotting` implementation.

**Evidence**:
- `odoo/addons/mail/models/mail_tracking_duration_mixin.py:19-24` (fields `rotting_days`, `is_rotting`) and lines 60-186 (`_is_rotting_feature_enabled`, `_get_rotting_domain`, `_compute_rotting`, `_search_is_rotting` with a hand-written `SQL()` query joining the stage table).
- `_search_is_rotting` explicitly raises if operator is not `in`/`not in` (`odoo/addons/mail/models/mail_tracking_duration_mixin.py:130-133`): *"For performance reasons, use `=` operators on rotting fields"* — i.e. `record.is_rotting == True` domain filters will raise `ValueError`, only `in`/`not in` work.
- Git evidence: the mixin itself is old (created 2023, commit `d31b3ecb8444`), but rotting was added later: `git log --oneline 18.0..HEAD -- addons/mail/models/mail_tracking_duration_mixin.py` shows `1aa378213cd5 [IMP] mail, crm: add rotting logic & fields`, `31ab32ebe7e1 [FIX] mail: fix traceback on rotting filters`, `cd9ed5788036 [FIX] mail,*: use many2one field widget for rotting in list view` — all strictly **after** the `18.0` tag, i.e. new in saas-19.
- Real usage: `odoo/addons/crm/models/crm_lead.py:89-100` inherits `mail.tracking.duration.mixin` with `_track_duration_field = 'stage_id'`, and `crm.stage` carries `rotting_threshold_days` to drive `crm.lead.is_rotting`.

**Why it matters**: A stale LLM investigating a Sentry crash on `crm.lead` / `project.task` / `helpdesk.ticket` search or list-view rendering involving `is_rotting`/`rotting_days` won't know these fields exist at all (they postdate its training), and would likely (a) not recognize `is_rotting` as a mixin-provided computed+searchable field and try to add it as a stored field/migration, or (b) if told to "fix filtering on is_rotting", write a domain using `=` which will raise the explicit `ValueError` guard in `_search_is_rotting`, or (c) miss that the feature is opt-in per-model (requires `rotting_threshold_days` on the stage model and `date_last_stage_update` on self) and "fix" a bug by assuming rotting is always active.

**Confidence**: high

---

## 3. Activities can be assigned to a `res.role` instead of a `res.users` — `mail.activity.user_id` can be legitimately empty

**Claim**: `mail.activity` now has a `role_id` (Many2one to `res.role`) field as an alternative responsible to `user_id`. Activities can be created/scheduled with only a role set and no specific user; `mail.activity.mixin`'s `activity_schedule()` explicitly treats `role_id` as an alternative to `user_id` when deciding whether to fall back to the activity type's default user.

**Evidence**:
- `odoo/addons/mail/models/mail_activity.py:127-130` — new field:
  ```python
  role_id = fields.Many2one(
      ...
      compute='_compute_role_id', precompute=True, store=True, readonly=False)
  ```
  and `_unassigned_role_idx = models.Index("(role_id) WHERE user_id IS NULL")` (line 159) — a dedicated partial index specifically for the "no user, only role" state.
- `odoo/addons/mail/models/mail_activity_mixin.py:393-410` (`activity_schedule`) — responsible-user fallback logic now checks `role_id` at every step: `if not create_vals.get('user_id') and not create_vals.get('role_id') and activity_user_id_fname:` and again for `activity_type.default_user_id`.
- `odoo/addons/mail/models/mail_activity.py:858` — `role_to_assign_ids = ongoing.filtered(lambda a: not a.user_id).role_id.ids` — confirms activities with `user_id=False` but `role_id` set are an expected, handled state, not a data anomaly.
- Git evidence: commit `a50844982557` "`[IMP] mail, hr: allow assigning activities to roles`" (author date 2026-06-11), commit message: *"Previously, activities had to be assigned to a specific user ... This commit introduces the ability to assign activities to a specific role instead."* — post-`18.0`, i.e. saas-19-only.
- `odoo/addons/mail/models/mail_activity_type.py:67` — `default_role_id = fields.Many2one("res.role", ...)` on the activity type too.

**Why it matters**: A stale LLM fixing a Sentry crash such as "AttributeError / NoneType" from code that assumes `activity.user_id` is always set (e.g. `activity.user_id.partner_id`, or code that does `if not activity.user_id: raise`) would "fix" it by adding a hard requirement that `user_id` be set, or by defaulting it to `env.user`/`activity_type.default_user_id` — silently breaking the role-based-assignment feature and potentially reassigning role-targeted activities to the wrong person. Any code branching on activity responsibility must now consider `role_id` as a valid alternative to `user_id`.

**Confidence**: high

---

## 4. `message_post`/`message_notify` gained explicit To/Cc email-recipient params, and forbid mixing message-only kwargs

**Claim**: `message_post` now accepts `partner_cc_ids`, `outgoing_email_to`, `incoming_email_to`, `incoming_email_cc` as first-class keyword parameters (in addition to `partner_ids`), documented as letting non-partner email addresses receive/have received notifications tied to a message. Docstring calls `outgoing_email_to` explicitly "Experimental support as of Odoo v19". Conversely, `message_notify` (the no-document notification path) explicitly **forbids** the incoming/outgoing email kwargs via `_raise_for_invalid_parameters`.

**Evidence**:
- `odoo/addons/mail/models/mail_thread.py:2240-2248` — signature now includes `partner_cc_ids=None, outgoing_email_to=False, incoming_email_to=False, incoming_email_cc=False` alongside `partner_ids`.
- `odoo/addons/mail/models/mail_thread.py:2269-2270` (docstring): *"`outgoing_email_to`: comma-separated list of emails to notify in addition to `partner_ids`. Experimental support as of Odoo v19"*.
- `odoo/addons/mail/models/mail_thread.py:2336-2338` — new merge/precedence rule: *"If a partner is in 'To' and 'Cc', 'To' wins."* (`partner_cc_ids = [pid for pid in list(partner_cc_ids or []) if pid not in partner_ids]`).
- `odoo/addons/mail/models/mail_thread.py:2836-2842` (`message_notify`) — `_raise_for_invalid_parameters(..., forbidden_names={'incoming_email_cc', 'incoming_email_to', 'message_id', 'message_type', 'outgoing_email_to', 'parent_id'})`, i.e. passing these to `message_notify` (as opposed to `message_post`) raises.
- Git evidence: commit `25cf9d05f500` "`[IMP] mail: add to / cc on message model`" (2024-12-19) and `94d9461890b1` "`[IMP] mail: propagate email_cc / to from gateway to mail.message`" (2024-12-19), both post-`18.0` tag work, task reference `Task-4286232: [mail] Store email{cc/to} on mail.message`.

**Why it matters**: A stale LLM fixing "customer CC'd on incoming email isn't notified"/"reply-all not working" bugs would not know these parameters exist and would likely invent ad-hoc solutions (e.g., manually parsing `Cc` headers into extra `message_post(partner_ids=...)` calls by fabricating partners for every email, or overriding `_notify_get_recipients`), rather than using the built-in `partner_cc_ids`/`incoming_email_cc` plumbing — and might also wrongly pass these new kwargs to `message_notify`, which will raise `ValueError` since they are on the forbidden list there.

**Confidence**: high

---

## 5. `mail.alias.mixin` no longer stands alone — it is built on top of `mail.alias.mixin.optional`, and the "optional" variant is the one most new/enterprise models should use

**Claim**: There are now two alias mixins with different semantics. `mail.alias.mixin` (classic) still uses `_inherits = {'mail.alias': 'alias_id'}` (mandatory one-to-one alias per record via `_inherits`), but it now itself inherits `mail.alias.mixin.optional`, which is the newer, more commonly-used mixin: it does NOT use `_inherits`, keeps `alias_id` optional/nullable, and only creates the underlying `mail.alias` record lazily, the first time `alias_name` is actually given (in `create`/`write`), specifically to avoid populating the alias table with void/inactive aliases.

**Evidence**:
- `odoo/addons/mail/models/mail_alias_mixin.py:13-18`:
  ```python
  _name = 'mail.alias.mixin'
  _inherit = ['mail.alias.mixin.optional']
  _inherits = {'mail.alias': 'alias_id'}
  ...
  alias_id = fields.Many2one(required=True, index=True)
  ```
- `odoo/addons/mail/models/mail_alias_mixin_optional.py:10-20` — docstring: *"A mixin for models that handles underlying 'mail.alias' records to use the mail gateway. Field is not mandatory and its creation is done dynamically based on given 'alias_name', allowing to gradually populate the alias table without having void aliases..."*, and `alias_id = fields.Many2one(..., required=False, ...)`.
- `odoo/addons/mail/models/mail_alias_mixin_optional.py:153-157` (`_require_new_alias`) vs. `odoo/addons/mail/models/mail_alias_mixin.py:26-28` — the two mixins override this hook oppositely: optional creates an alias only `if record_vals.get('alias_name')`; classic (`_inherits`-based) requires one *always* (`return not record_vals.get('alias_id')`).
- Git evidence: `mail.alias.mixin.optional` was introduced by commit `1af84e28ee8c` "`[IMP] mail: introduce 'mail.alias.mixin.optional' with optional aliases`" (2023-08-07) — i.e. it is genuinely newer than the classic mixin (2020) and postdates most 16-era training data, though it likely landed in the 17.0 branch, so confidence is medium rather than high.
- Real usage of the newer mixin in production code: `enterprise/documents/models/documents_document.py:39` — `_inherit = ['mail.thread', 'mail.activity.mixin', 'mail.alias.mixin.optional']`; also `odoo/addons/account/models/account_journal.py`.

**Why it matters**: A stale LLM asked to fix an alias-related bug (e.g., "alias created even when no incoming-mail feature is configured", "cannot unlink record because linked alias blocks it") might assume the classic `_inherits`-based mixin semantics (alias always exists, `alias_id` always set, deleting the record cascades in lockstep) even on a model that actually inherits `mail.alias.mixin.optional`, where `alias_id` can be `False`/absent and the CRUD contract (`_alias_get_creation_values`, `_require_new_alias`, `ALIAS_WRITEABLE_FIELDS`) is different. This could produce a patch that dereferences `record.alias_id` unconditionally, or "fixes" a missing alias by forcing `_inherits`, breaking the lazy-creation design.

**Confidence**: medium

---

## 6. Chatter/frontend sync mostly no longer needs per-model `_to_store` overrides — generic `Store` handles plain records and relations

**Claim**: The `Store` helper class (used to push chatter/follower/activity data to the web client, e.g. via `message_post`'s bus notification) was refactored so that `_to_store` overrides are now optional on models: any recordset can be handed to `Store.add()` and it falls back to `_read_format()`-based generic serialization; `_to_store` is only needed for custom/relational logic, and relation declarations changed from ad-hoc tuples to dedicated classes (`Store.One`/`Store.Many`/attribute-style relation objects). Consequently, most models integrating with chatter/mixins in this codebase do not define `_to_store` at all — it is not a mandatory part of "hooking a model into the chatter", contrary to defensive patterns from older code generations.

**Evidence**:
- `odoo/addons/mail/tools/discuss.py:249` — `def add(self, records, fields, *, as_thread=False, fields_params=None, ignore_empty=False):` docstring (lines following) states fields can be a string (method name), callable, list of field names ("Data for fields are coming from `_read_format()`"), or dict of static values — i.e. no `_to_store` required for the common cases.
- Git evidence: commit `3d7130f9e3c7` "`[REF] mail, *: clean Store, simplify _to_store and improve relation handling`" (2024-12-09, post-18.0), commit message point 4: *"`_to_store` is now optional. Any record can now be given to Store methods, and it will be added following ORM `_read_format`. `_to_store` is now only used to add custom code..."*
- Cross-check: in the entire community `addons/` tree (excluding `addons/mail/` itself), only `addons/im_livechat/models/discuss_channel.py` defines a `_to_store` override — confirming it's the exception, not something every chatter-integrated model (e.g. `project.task`, `crm.lead`) needs to define.
- `odoo/addons/mail/tools/discuss.py:195-236` — `Store.__init__` shows the class can auto-dispatch over the bus via `bus_channel._bus_send(...)` (tying back to `bus.listener.mixin`, candidate #1's evidence), i.e. this is the same delivery pipe `mail.thread` uses for real-time chatter/follower updates.

**Why it matters**: If a Sentry crash traces into chatter data not refreshing / a `KeyError` in frontend store data for some model, a stale LLM might assume a `_to_store` override is missing and must be added from scratch (based on older patterns it may have seen), producing an unnecessarily large, error-prone patch, when the actual bug is more likely in what fields/relations are passed to `Store.add()`, or in the relation-class usage (`Store.One`/`Store.Many` replaced older tuple-based relation returns), not the absence of `_to_store` itself.

**Confidence**: medium

---

## 7. `mail.track.mixin` restricts field parameters to `tracking` only — `track_visibility` is fully dead, including compatibility fallbacks

**Claim**: The only field-level kwarg the mail tracking machinery recognizes is `tracking` (an int/bool). The legacy `track_visibility` kwarg (pre-v13 API) is not just "unused" — as of a late-2024 cleanup, even the internal fallback code that used to check `getattr(field, 'track_visibility', None)` as a secondary lookup was deleted from `mail.thread`/`mail.track.mixin`, so nothing in the mixin chain looks at that attribute name anymore, anywhere.

**Evidence**:
- `odoo/addons/mail/models/mail_track_mixin.py:33-35`:
  ```python
  def _valid_field_parameter(self, field, name):
      # allow tracking on models inheriting from 'mail.thread'
      return name == 'tracking' or super()._valid_field_parameter(field, name)
  ```
- `odoo/addons/mail/models/mail_track_mixin.py:189-195` (`_track_get_fields`) — `model_fields = {name for name, field in self._fields.items() if getattr(field, 'tracking', None)}` — no `track_visibility` fallback.
- Git evidence: commit `e6c4fa5870a2` "`[IMP] mail,test_mail: remove last remains of track_visibility`" (2024-12-05, post-18.0) — diff shows the removed fallback was exactly `getattr(field, 'tracking', None) or getattr(field, 'track_visibility', None)` → now just `getattr(field, 'tracking', None)`, plus a doc-comment change from *"Since OpenERP v7 ... `track_visibility` set"* to *"... `tracking` set"*.

**Why it matters**: This is a smaller/lower-severity item than the others (using an unknown field kwarg only produces a logged warning at registry setup, per `odoo/orm/fields.py:536-545`, not a crash) but a stale LLM "fixing" a tracking bug by porting an old snippet that sets `track_visibility='onchange'` (a pattern that was common in 8.0-12.0-era code some LLMs may still have memorized, and that a 16/17-era LLM might reach for as a "safe legacy fallback") will silently produce a field that is never tracked, since only the `tracking` parameter name is recognized end-to-end now — with no compatibility shim left to catch it.

**Confidence**: medium

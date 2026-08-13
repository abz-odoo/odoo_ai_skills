# Security candidates — Odoo SaaS-19.4 vs stale (16/17-era) knowledge

Scope note: this pass focused deeply on the single biggest, highest-impact
change (the `ir.model.access` + `ir.rule` merger into `ir.access`, which
cascades into several sub-mechanics) plus a handful of adjacent group/API
renames. It did **not** do a broad sweep of every enterprise/industry module's
security folder, nor did it investigate `record rule` performance/caching
internals, portal/website-specific access mechanics, or field-level groups
(`groups=` attribute) in depth. Treat this as a solid but partial pass — a
human reviewer should expect more candidates exist in this area than the 8
below, especially around per-module security XML idioms in enterprise/.

---

## 1. `ir.model.access` and `ir.rule` are merged into a single model `ir.access`

**Title**: Unified `ir.access` model replaces `ir.model.access` + `ir.rule`

**Claim**: In Odoo 16/17, model-level CRUD permissions (`ir.model.access`,
loaded from `ir.model.access.csv`) and record-level domain rules (`ir.rule`,
usually defined as `<record model="ir.rule">` in XML) were two separate
models with separate semantics (permissions are OR'ed, rules are AND'ed per
group, global vs. group-specific rules, etc.). In 19.4 these two models are
**gone** and have been replaced by a single model `ir.access` that
represents both a "permission" (when `group_id` is set) and a "restriction"
(when `group_id` is empty/false), unified by one `operation` field (a
selection of CRUD-letter combos like `'crud'`, `'ru'`, `'r'`, ...) and one
`domain` field. A migration script converts all existing modules'
`ir.model.access.csv` + `<record model="ir.rule">` XML into
`security/ir.access.csv` rows automatically on upgrade.

**Evidence**:
- New model definition: `~/src/194/odoo/odoo/addons/base/models/ir_access.py:64-93`
  ```python
  class IrAccess(models.Model):
      """ Access control records with domains. """
      _name = 'ir.access'
      ...
      group_id = fields.Many2one('res.groups', string="Group", ondelete='cascade', index=True)
      operation = fields.Selection(list(CRUD_SELECTION.items()), required=True, ...)
      domain = fields.Char(...)
      kind = fields.Selection([('permission', 'Permission'), ('restriction', 'Restriction')], ...)
  ```
- `git log --oneline` on the relevant files, in
  `~/src/194/odoo` (git repo, branch `saas-19.4`):
  - `77c17e7111c1 [ADD] core: new model ir.access that merges ir.model.access and ir.rule`
  - `26cc6ac18fe9 [IMP] base: adapt code to use new model ir.access`
  - `a994829b1478 [IMP] *: adapt code to use new model ir.access`
- Zero hits for `class IrRule` / `class IrModelAccess` / `_name = 'ir.rule'` /
  `_name = 'ir.model.access'` anywhere under `~/src/194/odoo/odoo/`.
- Explicit migration/upgrade tooling that only exists because of this merge:
  `~/src/194/odoo/odoo/upgrade_code/19.4-00-ir-access.py` (entire
  file converts `ir.model.access` + `ir.rule` records into `ir.access`
  records; see the module docstring at lines 278-289: "This script converts
  ir.rule records to corresponding ir.access records...").
- Real-world confirmation across enterprise: `grep` counts in
  `~/src/194/enterprise`:
  - `ir.model.access.csv` referenced in manifests: **0**
  - `ir.access.csv` referenced in manifests: **267**
  - `<record model="ir.rule">` in XML: **0**
  - `<record model="ir.access">` in XML: **3** (e.g.
    `~/src/194/enterprise/hr_payroll/security/hr_payroll_security.xml:43-45`
    uses `<record id="hr.access_hr_employee_departure" model="ir.access"><field name="active">False</field></record>` to disable an inherited base access)
- Canonical CSV format sample:
  `~/src/194/odoo/odoo/addons/base/security/ir.access.csv:1-4`
  ```
  id,name,model_id,group_id/id,operation,domain
  access_decimal_precision_config,decimal.precision configuration,decimal.precision,base.group_system,ru,
  access_ir_access_group_erp_manager,ir_access_group_erp_manager,ir.access,base.group_erp_manager,crud,
  ir_attachment_public_rule,read public attachments,ir.attachment,base.group_user,r,"[('public', '=', True)]"
  ```
  Note the single `operation` column uses CRUD-letter strings (`crud`, `ru`,
  `r`, ...) instead of the old four separate boolean columns
  (`perm_read`/`perm_write`/`perm_create`/`perm_unlink`), and a domain column
  is present even for what used to be pure `ir.model.access` rows (empty
  domain = permission, non-empty domain = permission restricted to those
  records, `group_id` empty = global restriction/rule).

**Why it matters**: A stale-trained LLM asked to fix a Sentry `AccessError`
or "can't see records"/"missing permission" bug will very likely try to (a)
add a row to a `ir.model.access.csv` file that either doesn't exist in the
module anymore or won't be loaded correctly if fabricated fresh (module
manifest expects `security/ir.access.csv`), or (b) add a
`<record model="ir.rule">` XML record, which the ORM no longer recognizes as
a model at all — the patch would fail to install / silently do nothing
because `ir.rule` isn't a valid model reference, or worse, the agent invents
a spurious new model file. Any patch touching model or record-level access
in 19.4 must target `ir.access` (CSV or XML), with the unified
`operation` letter-string and a single `domain` field.

**Confidence**: high

---

## 2. `check_access_rights()` / `check_access_rule()` are gone; replaced by unified `check_access()` / `has_access()` / `_filtered_access()`

**Title**: Unified access-check API on recordsets

**Claim**: Odoo 16/17 code (and most tutorials/stale training data) checks
access in two separate calls: `self.check_access_rights('write')` (model-level
ACL check, no records needed) and `records.check_access_rule('write')`
(record-rule check on an existing recordset), each with their own
`raise_exception` parameter. In 19.4, both are consolidated into three
`@typing.final` methods that take a single `operation` string and handle
both aspects (ACL + record rule) together: `check_access(operation)` (raises
`AccessError`), `has_access(operation)` (returns bool), and
`_filtered_access(operation)` (returns the accessible subset). Calling
`records.browse().check_access(op)` (empty recordset) is the new idiom for a
"model-level-only" check.

**Evidence**:
- New methods: `~/src/194/odoo/odoo/orm/models.py:3379-3440`
  ```python
  def check_access(self, operation: str) -> None: ...
  @typing.final
  def has_access(self, operation: str) -> bool: ...
  @typing.final
  def _filtered_access(self, operation: str) -> typing.Self: ...
  ```
- Explicit deprecation markers pointing at the successors, same file:
  - `~/src/194/odoo/odoo/orm/models.py:3506` —
    `@deprecated("Since 20.0, use Model._access_domain instead")` on
    `_check_access()`.
  - `~/src/194/odoo/odoo/orm/models.py:2732` —
    `@deprecated("Since 20.0, use check_field_access()")` on `_check_field_access()`.
- Zero occurrences of `.check_access_rights(` or `.check_access_rule(` in
  `~/src/194/odoo`, `~/src/194/enterprise`,
  `~/src/194/industry`, `~/src/194/design-themes`
  (all four repos combined).
- 227 real call sites of `.check_access(` in `~/src/194/odoo` and
  142 in `~/src/194/enterprise`, confirming this is the live,
  actively-used API, not a rarely-touched corner.
- Canonical test usage patterns:
  `~/src/194/odoo/odoo/addons/test_orm/tests/test_access.py:49-68`
  (helper `assertAccess` calls `self.model.has_access(operation)`,
  `record.check_access(operation)`, `self.records._filtered_access(operation)`)
  and `:110-133` (`test_sudo`/`test_no_access` exercising all three methods
  for `'read'/'write'/'create'/'unlink'`).

**Why it matters**: A stale LLM patching a Sentry crash whose traceback shows
"AccessError" or "user cannot perform operation" will likely reach for
`records.check_access_rule('write')` or
`self.env['model'].check_access_rights('read', raise_exception=False)` —
both AttributeErrors in 19.4, since these methods don't exist. This would
produce a patch that raises a *new*, unrelated crash (AttributeError) instead
of fixing the access issue, or an agent might invent an incorrect
compatibility shim.

**Confidence**: high

---

## 3. New domain operator `'access'` for expressing access on a related model inside an `ir.access` domain

**Title**: `('field', 'access', 'read'/'write'/...)` domain operator

**Claim**: To express "the current user's access to model B should gate
access to model A through a many2one/one2many/many2many field", 19.4
introduces a first-class domain operator `access` usable directly inside
`ir.access` domains (and general domains), e.g.
`[('categ_id', 'access', 'read')]`. This did not exist as a domain operator
in 16/17; such cross-model access gating had to be hand-rolled (subqueries,
custom rule domains referencing `ir.rule`/`ir.model.access` tables directly,
or Python overrides).

**Evidence**:
- Operator implementation:
  `~/src/194/odoo/odoo/orm/domains.py:1900-1939`
  ```python
  @operator_optimization(['access'], level=OptimizationLevel.DYNAMIC_VALUES)
  def _operator_access_rule_domain(condition, model):
      ...
      condition._raise("The 'access' operator works only for many2one and 'id' fields")
      ...
      access_domain = comodel._access_domain(operation)
  ```
- Consumed inside `ir.access`'s own group-resolution logic:
  `~/src/194/odoo/odoo/addons/base/models/ir_access.py:398-437`
  (`_get_groups_with_access`, which walks `'access'` conditions recursively
  via `groups_satisfying(domain)`).
- Test coverage exercising it directly:
  `~/src/194/odoo/odoo/addons/test_orm/tests/test_access.py:341-368`
  ```python
  rule = self.make_access("Delegated permission", group=everyone)
  rule.domain = str([('categ_id', 'access', 'read')])
  ```
  and combinations with `&`/`|`/`!` at lines 353-368.

**Why it matters**: A stale LLM asked to write a record rule/access domain
that should depend on whether the user can access a related record (a common
real-world security bug: "user sees parent record but shouldn't because they
can't access the linked partner/project") will not know this operator
exists, and will instead write a subquery-style domain (e.g.
`[('partner_id.user_ids', 'in', ...)]`) that doesn't actually enforce access
control equivalence, silently under- or over-restricting records — exactly
the kind of subtle bug that produces new Sentry AccessErrors or data leaks
rather than fixing the reported one.

**Confidence**: medium (the operator's existence and mechanics are firmly
confirmed by code + tests; how *commonly* real modules should use it, versus
plain `any`/`in` domains, is less clear from this pass).

---

## 4. Group mutual-exclusivity: `category_id`-based typing replaced by `privilege_id` (`res.groups.privilege`) + explicit `disjoint_ids` constraint

**Title**: `res.groups.privilege` + `disjoint_ids` replace `category_id`-based group typing

**Claim**: In 16/17, `res.groups.category_id` (an `ir.module.category`) was
overloaded both for UI grouping ("Sales", "Accounting", ...) and, for the
three special "user type" groups (`group_user`/Internal, `group_portal`,
`group_public`), for enforcing that a user can only belong to one of the
three. In 19.4, `res.groups` has a new `privilege_id` field (Many2one to a
new model `res.groups.privilege`, described as "Scope") replacing that role,
plus an explicit computed field `disjoint_ids` and constraint
`_check_disjoint_groups()` that hard-enforces mutual exclusivity for the
three user-type groups (and any other declared disjoint sets), independent
of category. `res.groups.category_id` no longer exists as a field on
`res.groups` at all.

**Evidence**:
- New field replacing category on `res.groups`:
  `~/src/194/odoo/odoo/addons/base/models/res_groups.py:40`
  ```python
  privilege_id = fields.Many2one('res.groups.privilege', string='Scope', index=True)
  ```
  and `disjoint_ids` field/constraint at lines 83-116:
  ```python
  disjoint_ids = fields.Many2many('res.groups', string='Disjoint Groups',
      help="Users in the current group cannot be added in disjoint groups. "
           "For example, a user cannot be in the \"Internal\" and \"Portal\" groups at the same time.",
      compute='_compute_disjoint_ids')

  @api.constrains('implied_ids', 'implied_by_ids')
  def _check_disjoint_groups(self):
      ...
  ```
- `grep` confirms `category_id` is fully absent from `res_groups.py` except
  as a nested reference to the *privilege's* category
  (`~/src/194/odoo/odoo/addons/base/models/res_groups.py:389`,
  `'category_id': privilege.category_id.id`) — i.e. category now lives one
  level indirected through `res.groups.privilege`, not on the group itself.
- New model: `~/src/194/odoo/odoo/addons/base/models/res_groups_privilege.py:4-14`
  ```python
  class ResGroupsPrivilege(models.Model):
      _name = 'res.groups.privilege'
      _description = "Privilege"
      category_id = fields.Many2one('ir.module.category', string='Category', index=True)
      group_ids = fields.One2many('res.groups', 'privilege_id', string='Groups')
  ```
- `_get_user_type_groups()` /
  `_check_user_disjoint_groups()`:
  `~/src/194/odoo/odoo/addons/base/models/res_groups.py:121-150`
  (constraint on `res.users.group_ids` searches for any user holding two
  disjoint "user type" groups and raises `ValidationError` if found — this
  is now a real DB-level-searchable invariant, not implicit).

**Why it matters**: A stale LLM fixing a bug about "user wrongly has both
portal and internal access" or writing a migration/module that creates a new
group intended to be exclusive with another will look for `category_id`
on `res.groups` (16/17 pattern) and either get a "field does not exist"
error or, worse, silently set `category_id` on the *privilege* record
thinking it affects exclusivity — it does not; exclusivity is governed by
`disjoint_ids`/`implied_ids` graph via `SetDefinitions`, not by category
membership.

**Confidence**: high

---

## 5. `res.users.groups_id` renamed to `group_ids` (+ new `all_group_ids` for transitive closure)

**Title**: `res.users` field rename `groups_id` → `group_ids`

**Claim**: The classic `res.users.groups_id` Many2many field (used
pervasively in 16/17 code, XML, and tests to add/remove a user's groups) was
renamed to `group_ids`, with a new companion computed field `all_group_ids`
representing the transitive closure (previously this closure was typically
obtained ad hoc or via `trans_implied_ids`-style helpers on `res.groups`
rather than a directly queryable field on the user).

**Evidence**:
- Field definitions:
  `~/src/194/odoo/odoo/addons/base/models/res_users.py:249-251`
  ```python
  group_ids = fields.Many2many('res.groups', 'res_groups_users_rel', 'uid', 'gid',
      string='Groups', default=lambda s: s._default_groups(), help="Groups explicitly assigned to the user")
  all_group_ids = fields.Many2many('res.groups', string="Groups and implied groups",
      compute='_compute_all_group_ids', compute_sudo=True, search='_search_all_group_ids')
  ```
- Explicit rename commit in `git log` (repo
  `~/src/194/odoo`, branch `saas-19.4`):
  `4f13b617547a [IMP] base: rename field res.users.groups_id into group_ids & all_group_ids`
  (dated Fri Jan 31 2025 — i.e. this happened well after the 16/17 era a
  stale model would know).
- Constraint and write-path usage consistently on the new name:
  `~/src/194/odoo/odoo/addons/base/models/res_users.py:533`,
  `:548`, `:617` (`if 'group_ids' in vals and any(self._ids):`).
- Confirmed as canonical test idiom too:
  `~/src/194/odoo/odoo/addons/test_orm/tests/test_access.py:27`
  `'group_ids': [Command.set(groups.ids)]`.

**Why it matters**: A stale LLM writing a patch that assigns/reads a user's
groups (e.g. fixing "new users don't get the right group" or "user group
sync" bugs) will likely write `user.groups_id = [(6, 0, ids)]` or
`user.write({'groups_id': [...]})` — either a hard failure (unknown field)
or, if some backward-compat property still exists, subtly wrong behavior.
It should also use `all_group_ids` rather than manually walking
`implied_ids` when it needs the transitive set of a user's groups.

**Confidence**: high

---

## 6. `bypass_search_access` field attribute replaces ad hoc "skip access check on comodel" overrides

**Title**: `bypass_search_access=True` on relational fields

**Claim**: Relational fields (`Many2one`, `One2many`, `Many2many`) now carry
a first-class attribute `bypass_search_access` that tells the ORM to skip
access-right filtering when resolving/searching the comodel through that
field (this is also what `_inherits` delegation fields set automatically).
In 16/17, this kind of "don't apply access rules when following this
relation" behavior existed only in scattered, model-specific overrides (or
via `sudo()` calls buried in compute methods), not as a declared field
attribute inspected by domain/`any`/`any!` resolution.

**Evidence**:
- Attribute declaration & doc:
  `~/src/194/odoo/odoo/orm/fields_relational.py:39`
  `bypass_search_access: bool = False  # whether access rights are bypassed on the comodel`
  and `:234`, `:902` (docstrings on `Many2one`/`One2many`).
- Automatically set for `_inherits` delegation fields:
  `~/src/194/odoo/odoo/orm/fields_relational.py:262-263`
  ```python
  # self.delegate implies self.bypass_search_access
  self.bypass_search_access = True
  ```
- Used at multiple points in field read/write/search resolution, e.g.
  `~/src/194/odoo/odoo/orm/fields_relational.py:181`, `:413`,
  `:501`, `:691`, `:863`, `:1205`, `:1808`.
- Cross-referenced from the domain layer:
  `~/src/194/odoo/odoo/orm/domains.py:96` (comment: "if
  bypass_search_access is set on the field, see `any!`") and `:997`
  ("inherits implies both Field.delegate=True and Field.bypass_search_access=True").

**Why it matters**: A stale LLM debugging "field X unexpectedly hides/shows
records depending on user" bugs on a delegated/inherited field, or trying to
intentionally bypass access checks on a relational field the way older code
would (custom `search()` overrides with `sudo()`), might not realize the
declarative `bypass_search_access=True` field kwarg is the current idiom —
leading to either reinventing an inconsistent workaround or missing that a
field's current `bypass_search_access=True` is *why* a Sentry AccessError
isn't being raised where the reporter expected one.

**Confidence**: medium

---

## 7. Security data file naming convention: `security/ir.access.csv`, not `security/ir.model.access.csv`

**Title**: Module security CSV file is now `ir.access.csv`

**Claim**: Every module's manifest now declares its permission/rule data
file as `security/ir.access.csv` (single file, single row format per the
unified model from candidate #1), not the old `ir.model.access.csv`. A
module's `security/` folder may also still contain hand-written
`<record model="ir.access">` XML for cases needing dynamic domains or
`active=False` overrides of an inherited access (used instead of overriding
individual CSV rows).

**Evidence**:
- `~/src/194/odoo/odoo/addons/base/security/ir.access.csv` exists;
  no `ir.model.access.csv` exists anywhere under
  `~/src/194/odoo/odoo/addons/base/security/`.
- Migration tool explicitly writes the new filename:
  `~/src/194/odoo/odoo/upgrade_code/19.4-00-ir-access.py:401`
  ```python
  file_name = 'security/ir.access.csv' if module in has_security else 'ir.access.csv'
  ```
- Real modules confirm the convention at scale: 267 manifests in
  `~/src/194/enterprise` reference `ir.access.csv`; 0 reference
  `ir.model.access.csv`.
- Override-via-XML idiom (deactivating one inherited permission without
  touching the base module's CSV):
  `~/src/194/enterprise/timesheet_grid/security/timesheet_security.xml:4-6`
  ```xml
  <record id="hr_timesheet.timesheet_line_rule_user" model="ir.access">
      <field name="active">False</field>
  </record>
  ```

**Why it matters**: This is the most mechanical, easy-to-get-wrong detail
for an agent generating a *file-level* patch (new module, or adding an
access row to fix a permission bug): naming the new/edited file
`ir.model.access.csv` (16/17 habit) means the manifest's `data` list
entry and the actual file the loader expects (`ir.access.csv`) mismatch,
and the access row silently never loads — the Sentry AccessError the patch
was meant to fix would persist even though the "fix" was merged.

**Confidence**: high

---

## 8. `AccessError` messages now carry structured `_get_groups_with_access` info and a `suggested_company` context for multi-company mismatches

**Title**: Richer, structured `AccessError` messages + `suggested_company` context hint

**Claim**: The `AccessError` raised by `ir.access` is no longer just a plain
string. It's built with helper methods that (a) list which groups *would*
grant the missing operation (`_get_groups_with_access`), (b) in debug mode
(`base.group_no_one` + internal user) list the specific failing record
descriptions and which named access record ("Blame the following accesses")
caused the failure, and (c) for record-level failures involving
`company_id`, compute a `suggested_company` and attach it as
`exception.context = {'suggested_company': {...}}` for the frontend to offer
a "switch company" affordance. None of this structured
introspection/context-attachment existed on `AccessError` in 16/17 — it was
a static translated string.

**Evidence**:
- Group-listing in the plain (non-debug) message:
  `~/src/194/odoo/odoo/addons/base/models/ir_access.py:373-385`
  (`_make_model_access_error`, "This operation is allowed for the following
  groups" vs. "No group currently allows this operation").
- Debug-mode detailed failure attribution + `context` attached directly on
  the exception object:
  `~/src/194/odoo/odoo/addons/base/models/ir_access.py:449-531`
  (`_make_record_access_error`), specifically:
  ```python
  elif suggested_company and suggested_company in self.env.user.company_ids:
      context = {'suggested_company': {
          'id': suggested_company.id,
          'display_name': suggested_company.display_name,
      }}
      ...
  exception = AccessError(message)
  if context:
      exception.context = context
  return exception
  ```
- Test assertions locking in the exact structured message shape (group name
  interpolation, "Blame the following accesses:", named access record in the
  message):
  `~/src/194/odoo/odoo/addons/test_orm/tests/test_access.py:220-323`
  (`test_error_message_no_access`, `test_error_message_partial_access`, etc.)

**Why it matters**: A stale LLM triaging a Sentry-captured `AccessError` by
matching on message text (a common Sentry-triage heuristic: grep the
exception message to decide which code path failed) may assume the old
flat, single-sentence message format and either miswrite a message-matching
fix or fail to recognize that recent versions already tell you (in the
message itself, in debug mode) exactly which access record/group is
responsible — meaning the "fix" might need to touch the identified
`ir.access` row directly rather than application code. It also means any
code that catches `AccessError` and inspects `str(e)` for a specific old
phrase will break silently against the new wording.

**Confidence**: medium

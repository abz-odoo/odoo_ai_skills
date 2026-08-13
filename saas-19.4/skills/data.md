# Data-loading mechanics — candidates (Odoo SaaS-19.4)

Scope note: this pass focused on `odoo/odoo/tools/convert.py`, `odoo/odoo/import_xml.rng`,
`odoo/odoo/modules/loading.py`, `odoo/odoo/orm/models.py` (`_load_records`), the `ir.access`
model, and grep sweeps across XML/CSV data files in all four repos, cross-checked against
`git log` in the `odoo/` repo (which is a real git checkout — the other three were not
inspected via git history, only via grep). Coverage of `enterprise/`, `industry/` and
`design-themes/` was limited to targeted greps (confirming patterns found in core), not a
full independent sweep — those repos were not exhaustively searched for repo-specific data
quirks. A subagent was dispatched for a deeper pass over `ir.model.data` internals
(`_load_xmlid`, `_update_xmlids`, uninstall/orphan handling, `_load_records`) and its
confirmed findings are folded into candidate 7 and the `<test>`-tag aside below; its other
observations (e.g. a recent xmlid-cache-invalidation fix, translation-refresh-on-upgrade
behavior) were judged more about ORM/translation internals than XML/CSV data-file authoring
mechanics and were left out to keep this list focused.

---

## 1. `ir.model.access` + `ir.rule` were merged into a single model `ir.access`

**Claim:** The two access-control models a 16/17-era agent knows — `ir.model.access` (CRUD
permissions, loaded from `ir.model.access.csv`) and `ir.rule` (record-level domain rules,
loaded from XML `<record model="ir.rule">`) — no longer exist as separate models in
SaaS-19.4. They were merged into one model, `ir.access`. Neither `ir.model.access` nor
`ir.rule` is defined anywhere in the codebase anymore.

**Evidence:**
- New model: `~/src/194/odoo/odoo/addons/base/models/ir_access.py:64-81` —
  `class IrAccess(models.Model): _name = 'ir.access'` with fields `model_id`, `group_id`,
  `operation` (Selection over `CRUD_SELECTION`, e.g. `'crud'`, `'r'`, `'cru'`...), and `domain`
  (Char) — i.e. one record now carries both what used to be 4 boolean perm columns AND what
  used to be `ir.rule.domain_force`.
- Confirmed zero survivors: `grep -rn "_name = 'ir.model.access'"` and
  `grep -rn "_name = 'ir.rule'"` under `odoo/addons/base/models/` return nothing.
  `grep -rl 'model="ir.rule"'` and `grep -rl 'model="ir.model.access"'` across all four
  repos' `*.xml` return **0 files**.
- CSV filename changed: `ir.model.access.csv` → `ir.access.csv`. `find . -iname
  "ir.model.access.csv"` returns 0 hits; `find . -iname "ir.access.csv"` returns 500 files
  across the four repos, e.g.
  `~/src/194/odoo/addons/hr/security/ir.access.csv`,
  `~/src/194/enterprise/obox/security/ir.access.csv`.
- CSV column header changed from the old
  `id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink` to:
  `id,name,model_id,group_id/id,operation,domain` — confirmed both from real files (e.g.
  `odoo/addons/hr/security/ir.access.csv:1`) and from the official migration script that
  generates these files, `~/src/194/odoo/odoo/upgrade_code/19.4-00-ir-access.py:389`
  (`writer.writerow(["id", "name", "model_id", "group_id/id", "operation", "domain"])`).
- XML record example merging what used to be an `ir.rule` domain directly onto the access
  record: `~/src/194/odoo/addons/hr_timesheet/security/hr_timesheet_security.xml:34-43`:
  ```xml
  <field name="model_id" ref="analytic.model_account_analytic_line"/>
  <field name="group_id" ref="base.group_portal"/>
  <field name="operation">r</field>
  <field name="domain">[('project_id', '!=', False), ...]</field>
  ```
- Semantic shift for "no group" rows: in old `ir.model.access.csv`, an empty `group_id`
  meant "granted to everyone". In `ir.access`, a record with no `group_id` is a
  **restriction** (AND-ed, narrows access for everybody), not a permission — see
  `ir_access.py:128-131` (`_compute_kind`: `'permission' if access.group_id else
  'restriction'`) and `ir_access.py:340-344` (`_get_domain_for`: permissions are OR-ed,
  restrictions are AND-ed).
- Introduced by commit `77c17e7111c1` "`[ADD] core: new model ir.access that merges
  ir.model.access and ir.rule`" (Apr 16 2024), with a dedicated migration script
  `odoo/upgrade_code/19.4-00-ir-access.py` that mechanically rewrites every module's
  `ir.model.access.csv` + `ir.rule` XML into `ir.access.csv`.

**Why it matters:** A stale agent triaging an access-rights Sentry crash (`AccessError`,
"missing ir.model.access", "record rule violation") would almost certainly (a) look for or
try to add a row to a file named `ir.model.access.csv` with `perm_read/perm_write/
perm_create/perm_unlink` boolean columns — which no longer exists as a concept — and
(b) try to add a separate `<record model="ir.rule">` for a domain-based restriction instead
of adding a `domain` value to the same `ir.access` row (or a group-less restriction row).
It could also misread a group-less row as "open to everyone" instead of "a global
restriction", inverting the intended access logic in the patch. The `operation` value is
also order-sensitive (must match a key in `CRUD_SELECTION`, e.g. `'cru'` is valid, `'urc'`
is not) — a plausible source of a `ValidationError` if guessed.

**Confidence:** high

---

## 2. `groups_id` renamed to `group_ids` on `ir.ui.menu` and `ir.ui.view`

**Claim:** The classic field used to restrict a menu or view to certain security groups —
`groups_id` (a Many2many to `res.groups`) — has been renamed to `group_ids`. This affects
every `<menuitem groups="...">` shortcut and every `<template groups="...">` / `<record
model="ir.ui.view">` that sets that field.

**Evidence:**
- `~/src/194/odoo/odoo/addons/base/models/ir_ui_menu.py:29` —
  `group_ids = fields.Many2many('res.groups', 'ir_ui_menu_group_rel', ...)`.
- `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:177` —
  `group_ids = fields.Many2many('res.groups', 'ir_ui_view_group_rel', 'view_id', 'group_id', ...)`.
- The XML loader itself writes to `group_ids`, not `groups_id`, when expanding the
  `<menuitem groups="...">` shortcut:
  `~/src/194/odoo/odoo/tools/convert.py:338-348` — builds
  `Command.link`/`Command.unlink` and sets `values['group_ids'] = groups`. Same pattern for
  `<template groups="...">`: `convert.py:551-554` sets
  `record.append(Field(name="group_ids", eval=...))`.
- Confirmed across the whole tree: `grep -rn 'name="groups_id"' --include=*.xml` → 0 hits;
  `grep -rn 'name="group_ids"' --include=*.xml` → 234 hits, e.g.
  `odoo/addons/hr/views/hr_employee_views.xml:881`
  (`<field name="group_ids" eval="[Command.link(ref('hr.group_hr_user'))]"/>`).

**Why it matters:** A stale agent patching a "menu/view visible to wrong users" bug would
naturally write `<field name="groups_id" eval="[(6,0,[...])]"/>`, which now targets a
field that doesn't exist on `ir.ui.menu`/`ir.ui.view` — Odoo would raise (unknown field)
rather than silently doing nothing, but the agent could easily "fix" this by inventing a
wrong workaround instead of using the real current field name.

**Confidence:** high

---

## 3. `forcecreate="1"` lets a record be *created* under another module's xmlid prefix

**Claim:** Historically (since ~v10), Odoo forbade creating a brand-new record whose
`id="..."` prefix belongs to a module other than the one being loaded (only *updating* an
existing foreign record was allowed). Since a 2024 change, setting `forcecreate="1"` on the
`<record>`/`<template>` node explicitly overrides this and allows creating the record even
though the xmlid's module prefix is foreign.

**Evidence:** `~/src/194/odoo/odoo/tools/convert.py:402-410`:
```python
foreign_record_to_create = False
if xid and xid.partition('.')[0] != self.module:
    record = self.env['ir.model.data']._load_xmlid(xid)
    if not record and not (foreign_record_to_create := nodeattr2bool(rec, 'forcecreate')):
        ...
        raise Exception("Cannot update missing record %r" % xid)
```
and later `convert.py:483-484`: `if foreign_record_to_create: model =
model.with_context(foreign_record_to_create=foreign_record_to_create)`. Introduced by
commit `6a99878c8da1` "`[IMP] base: Allow creating foreign records if explicitely stated`"
(Feb 20 2024), with the explicit rationale (commit message) of letting a localisation
module (e.g. `l10n_xx_hr_payroll`) create demo/base data owned by a common dependency
(`base`/`account`) without needing a bridge module.

**Why it matters:** A stale agent fixing "duplicate demo company" / "cannot find xml_id"
bugs across two sibling modules might assume the only fix is introducing a new bridge
module or duplicating the xmlid under the current module's own prefix — not realizing
`forcecreate="1"` is the sanctioned, minimal-diff way to let module B create a record
whose id lives in module A's namespace.

**Confidence:** medium-high

---

## 4. `auto_sequence="1"` on `<odoo>`/`<data>` auto-assigns the `sequence` field from document order

**Claim:** Setting `auto_sequence="1"` (or `"true"`) on the root `<odoo>`/`<data>` element
makes the loader auto-populate any record's `sequence` field (when the model has one and
the `<record>` doesn't set it explicitly) with an auto-incrementing value (+10 per record),
derived purely from the record's position in the file. This is heavily used in accounting
localization/tax-report modules.

**Evidence:**
- `~/src/194/odoo/odoo/tools/convert.py:628-629` (in `_tag_root`):
  `self._sequences.append(0 if nodeattr2bool(el, 'auto_sequence', False) else None)`.
- `convert.py:662-666` (`next_sequence`): increments by 10 each call.
- `convert.py:475-478` (in `_tag_record`): `if 'sequence' not in res and 'sequence' in
  model._fields: sequence = self.next_sequence(); if sequence: res['sequence'] = sequence`.
- RelaxNG schema explicitly allows the attribute:
  `~/src/194/odoo/odoo/import_xml.rng:290` —
  `<rng:optional><rng:attribute name="auto_sequence" /></rng:optional>`.
- Widely used: `grep -rl "auto_sequence" --include=*.xml .` → 407 files, e.g.
  `odoo/addons/l10n_in/data/account_tax_report_tcs_data.xml:2` (`<odoo auto_sequence="1">`),
  `odoo/addons/l10n_es/data/mod115.xml:3` (`<data auto_sequence="1">`).

**Why it matters:** For a Sentry bug about tax-report lines or similar ordered records
appearing in the wrong order, a stale agent would likely try to fix it by adding/editing
explicit `<field name="sequence" eval="..."/>` values. But under `auto_sequence`, the
*position of the `<record>` tag in the file* is what actually determines the sequence —
reordering the XML nodes (or inserting a new one in the middle) silently changes every
subsequent auto-assigned sequence number, which is very likely the actual root cause /
actual fix needed, not a `sequence` field edit.

**Confidence:** medium-high

---

## 5. `type="bytes"` is replacing `type="base64"` for binary field values in data files

**Claim:** For loading binary field values from a file (`<field type="base64" file="..."/>`),
a newer `type="bytes"` is now supported and is the intended replacement — `base64` triggers a
`DeprecationWarning` (not a hard error yet in 19.4) telling authors to switch. Existing usage
is still overwhelmingly on `type="base64"`.

**Evidence:**
- `~/src/194/odoo/odoo/tools/convert.py:145-148`:
  ```python
  if t == 'base64':
      warnings.warn("Since 20.0, use type=bytes instead of type=base64", DeprecationWarning)
      with file_open(node_file, 'rb', env=env) as f:
          return BinaryBytes(f.read()).to_base64()
  ```
- RelaxNG schema lists both as valid: `~/src/194/odoo/odoo/import_xml.rng:74-79`.
- Introduced by commit `425f0374392c` "`[IMP] upgrade_code: use type="bytes" in data
  files`" (Mar 17 2026 in this repo's history) — "Replace the 'base64' to avoid double
  encoding when loading data files" — alongside a companion `fields.Binary` rework
  (`41fe2ebdb9cc` "`[IMP] core, *: fields.Binary return BinaryValue`", Nov 11 2025) that
  changes Binary field cache/record representation to a lazy `BinaryValue` buffer object
  instead of a base64 string.
- Adoption is still tiny: `grep -rln 'type="bytes"' --include=*.xml .` → only 2 files (both
  in `odoo/addons/pos_restaurant/data/scenarios/`), vs. `grep -rln 'type="base64"'` → 914
  files, e.g. `enterprise/social_demo/data/social_demo.xml:86`.

**Why it matters:** This is a forward-looking change, not yet a broadly enforced one — so
it's low risk to keep writing `type="base64"` in 19.4. The risk is the *inverse*: a stale
agent that has never seen `type="bytes"` at all might, upon spotting it in one of the two
files that use it (or in an error/deprecation log), assume it's a typo or unsupported
attribute and "fix" it back to `base64`, or misunderstand a related `BinaryValue`-related
Sentry traceback (e.g. an addon comparing a Binary field value against a raw base64 string)
as unrelated to data loading.

**Confidence:** medium

---

## 6. `active="True"/"False"` on `<template>`/`<asset>` root is ignored on update once the record exists

**Claim:** For `<template>` (→ `ir.ui.view`) and `<asset>` (→ `ir.asset`), an `active`
attribute on the tag's root element only sets the field's value the *first time* the record
is created. On a subsequent module update, if the record already exists, the attribute is
silently ignored — specifically to avoid re-activating/re-deactivating a view or asset that
was toggled at runtime (e.g. by a website editor or a cron).

**Evidence:**
- `~/src/194/odoo/odoo/tools/convert.py:540-547` (`_tag_template`):
  ```python
  # If the "active" value is set on the root node (instead of an inner
  # <field>), it is treated as the value for the "active" field but only
  # when *not updating*. This allows to update the record in a more recent
  # version without changing its active state (compatibility).
  if (active := attrib.pop('active', None)) in ("True", "False"):
      view_id = self.id_get(tpl_id, raise_if_not_found=False)
      if self.mode != "update" or not view_id:
          record.append(Field(name='active', eval=active))
  ```
- Same pattern for `<asset>`: `convert.py:596-604`.
- The `<asset>` tag's special handling of `active` was added specifically to fix a bug
  where module updates were resetting *every* `ir.asset.active` to its XML-declared value,
  wiping out snippets/pages that relied on assets a user had toggled — commit
  `16e1544557b4` "`[FIX] tools, *: ignore the active field when updating assets`" (Dec 21
  2022), which explicitly introduces the `<asset>` shortcut tag as "an alias of `<record
  model=\"ir.asset\">` with the additional feature that it avoids taking the `active` field
  into account during updates for existing records, just like `<template>`."

**Why it matters:** For a Sentry bug like "snippet/view X is inactive after upgrading the
module even though the XML says active=True" (or the reverse — "we set active=False in the
XML but it's still active"), a stale agent would likely conclude the `active` attribute
isn't being applied at all and go hunting for an unrelated cause, instead of recognizing
this is expected compatibility behavior — the actual current state lives in the DB and was
presumably toggled at runtime (or by a previous version's data), and the fix must go through
something else (e.g. a `<function>` call, a data migration script, or an inner `<field
name="active">` instead of the root attribute, which does NOT get this special treatment).

**Confidence:** medium

---

## 7. The `install_xmlid` load-context key is gone; use a model field named `xmlid` instead

**Claim:** Older Odoo versions exposed the xml_id currently being loaded via
`self.env.context.get('install_xmlid')` inside overridden `create`/`write`/computed methods
during data loading (alongside `install_module`, `install_mode`, `install_filename`, which
still exist). This context key has been removed. If a model needs to know its own xmlid at
load time, the current mechanism is different: define a field literally named `xmlid` on
the model, and the loader auto-populates it for you — no context lookup involved.

**Evidence:**
- Removal: `~/src/194/odoo/odoo/tools/convert.py:367-372` — the
  `model.with_context(install_mode=True, install_module=self.module,
  install_filename=self.xml_filename)` call no longer includes `install_xmlid=rec_id`
  (confirmed via `git show 2bec4a904ed0 -- odoo/tools/convert.py`, which diffs out exactly
  that line).
- Replacement mechanism, same file: `convert.py:479-480`:
  ```python
  if 'xmlid' not in res and 'xmlid' in model._fields:
      res['xmlid'] = xid
  ```
- Commit `2bec4a904ed0` "`[IMP] core: remove xmlid from the context`" (2025-08-09):
  "Remove the xmlid from the context when loading data. We don't need it and we can handle
  it the same way we handle sequences: just add it as a value if the record has it." The
  same commit updates `ir_ui_view.py`'s error-context dict from
  `self.env.context.get('install_xmlid')` to `self.xml_id` directly.
- Confirmed no lingering usage anywhere in the four repos:
  `grep -rl "install_xmlid" --include=*.py .` → 0 files.

**Why it matters:** A stale agent patching a bug in an overridden `create()`/`write()` that
needs to know "what xmlid is this record being loaded under" would write
`self.env.context.get('install_xmlid')`, which now silently returns `None` — a subtle,
non-crashing logic bug (the code runs, just always takes the "no xmlid" branch), which is
exactly the hard-to-spot kind of regression a triage agent could introduce while otherwise
believing its patch is correct and modeled on "existing conventions" it half-remembers.

**Confidence:** medium

---

### Aside: the legacy `<test>` XML tag is fully dead, not just rare

While investigating, a subagent confirmed the old `<test>` data tag (used in very old Odoo
versions for embedding test/log strings in data files) is not merely rarely used — it has
**no handler at all** in the current loader. `odoo/tools/convert.py`'s tag dispatch table
(`_tags = {...}`, `convert.py:676-685`) registers only `record`, `delete`, `function`,
`menuitem`, `template`, `asset`, and the root tags (`odoo`/`data`/`openerp`) — there is no
`_tag_test` entry. Grepping all four repos for a genuine `<test` data tag returns zero real
hits (the only matches were unrelated `<testIndicator>` QWeb elements in Intrastat export
templates). If a stale agent's training data includes ancient `<test>`-tag lore, it is not
just outdated style — the tag is inert in this codebase.

---

## 8. Creating a record with a foreign-module-prefixed xmlid can hard-fail during "Import Module"

**Claim:** When data is loaded under the `import_file` context (used by the "Import Module"
/ module-upload feature, e.g. Studio-exported modules), creating a new record whose xmlid's
dot-prefix happens to match the name of an *already-installed* module raises a hard
`UserError` telling the author to use a non-colliding prefix (suggested: `__import__`),
rather than silently letting the record collide with — or later get deleted alongside —
that real module's own data.

**Evidence:** `~/src/194/odoo/odoo/orm/models.py:4575-4584` (inside
`_load_records`):
```python
if self.env.context.get('import_file'):
    existing_modules = self.env['ir.module.module'].sudo().search([]).mapped('name')
    for data in to_create:
        xml_id = data.get('xml_id')
        if xml_id and not data.get('noupdate'):
            module_name, sep, record_id = xml_id.partition('.')
            if sep and module_name in existing_modules:
                raise UserError(_(
                    "The record %(xml_id)s has the module prefix %(module_name)s. "
                    "... Use either no prefix and no dot or a prefix that isn't an "
                    "existing module. For example, __import__, resulting in the "
                    "external id __import__.%(record_id)s.", ...))
```

**Why it matters:** This is a narrow but concrete trap: a Sentry crash surfacing during a
module-import/Studio-export workflow with a `UserError` about a "module prefix" is not a
generic xmlid bug — it's this specific, deliberate guard. A stale agent unfamiliar with it
might try to "fix" the crash by changing xmlid-parsing logic generically, when the correct
fix is simply renaming the offending id to avoid colliding with a real installed module
name (or using the suggested `__import__` prefix).

**Confidence:** low-medium

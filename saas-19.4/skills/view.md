# View-arch candidates — SaaS-19.4 (discovery pass)

Scope note: this pass focused depth-first on the highest-signal areas (list/tree
rename, attrs/states removal, chatter tag, kanban card templates, xpath-attribute
inheritance, column_invisible). It did NOT do a systematic sweep of every view
sub-area (e.g. calendar/gantt/pivot/graph-specific arch changes, search view
`<searchpanel>` internals beyond what's cited, or activity view arch) — those
were skipped for time, not because they're known to be unchanged. Treat this as
a curated top-8, not full coverage of "views".

All evidence gathered by grep + direct file reads against:
- `~/src/194/odoo/odoo/` (core framework)
- `~/src/194/odoo/addons/` (community addons)
- `~/src/194/enterprise/`
- `~/src/194/industry/`
- `~/src/194/design-themes/`

---

## 1. `<tree>` is gone — root/list-view tag is `<list>`, and using `<tree>` is a hard validation error, not just deprecated style

**Claim:** In SaaS-19.4 the list-view architecture root tag is exclusively `<list>`.
`<tree>` is not merely "discouraged", it is absent from every shipped view in all
four repos, and the view validator explicitly raises if a view's root tag doesn't
match its declared `type` (so a `type=list` view with root `<tree>` fails to load).

**Evidence:**
- Grep counts across all 4 repos, `--include=*.xml`:
  - `<tree ` / `<tree>` tag occurrences: **0** (community addons, core framework `odoo/odoo/`, enterprise, industry, design-themes — all zero).
  - `<list ` / `<list>` tag occurrences: 653 (community addons), 631 (enterprise), 80 (industry), 0 (design-themes, no data-driven views there).
- `ir.ui.view` field declaration only lists `'list'` as a type, no `'tree'`:
  `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:150`
  ```python
  type = fields.Selection([('list', 'List'),
                           ('form', 'Form'), ...
  ```
- Root-tag validation raises `ValidationError` if the arch root tag doesn't equal
  `view_type`: `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:1817-1821`
  ```python
  view_type = view_type or self.type
  if node.tag != view_type:
      self._raise_view_error(_(
          'The root node of a %(view_type)s view should be a <%(view_type)s>, not a <%(tag)s>',
          view_type=view_type, tag=node.tag,
      ), node)
  ```

**Why it matters:** A stale-trained LLM patching a Sentry crash in a list view will
very likely emit `<tree>` (or `<xpath expr="//tree" ...>`) when adding/inheriting a
column or fixing a KeyError on a field. This won't silently degrade — it throws a
hard `ValidationError` on module install/upgrade ("root node ... should be a
<list>, not a <tree>"), so the "fix" would itself crash view loading. Also: any
xpath the agent writes to target the list view (e.g. `//tree`, `//tree/field`)
must use `//list` instead, or the xpath will silently match nothing (inheritance
that finds no node also raises an error at load time).

**Confidence:** high

---

## 2. `attrs="..."` / `states="..."` are not just deprecated — they raise a hard ValidationError on view load, and direct attribute expressions are 100% of live usage

**Claim:** `attrs=` and `states=` modifiers (dict/tuple domain syntax like
`attrs="{'invisible': [('state','=','done')]}"`) are entirely gone from real view
arch in this codebase. Every conditional attribute (`invisible`, `readonly`,
`required`, `column_invisible`) is written as a direct Python-ish boolean
expression string. The framework enforces this at install time, not just via lint.

**Evidence:**
- Explicit validator, `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:498-504`:
  ```python
  if combined_arch.xpath('//*[@attrs]') or combined_arch.xpath('//*[@states]'):
      view_name = f'{view.name} ({view.xml_id})' if view.xml_id else view.name
      err = ValidationError(_('Since 17.0, the "attrs" and "states" attributes are no longer used.\nView: %(name)s in %(file)s',
          name=view_name, file=view.arch_fs
      ))
  ```
- Grep for `attrs=` on real `<field>`/`<button>`/`<tree|list>` view-arch nodes
  returns **zero** hits in `*/views/*.xml` across all repos; the only remaining
  `attrs=` matches (54 in community addons, 24 in enterprise, 0 in industry/
  design-themes) are all OWL/JS component templates using `attrs` as an ordinary
  prop name (unrelated concept), e.g.
  `~/src/194/odoo/addons/sale_management/static/src/fields/sale_order_line_field/sale_order_line_field.xml:8`
  (`attrs="{ 'class': ... }"` — an OWL component attribute, not view-arch).
- `states=` occurrences in view XML: **0** everywhere checked.
- Real direct-expression examples confirming the dominant/only pattern, e.g.
  `~/src/194/odoo/addons/sale/views/sale_order_views.xml:301,319,348,364,380,393,401`:
  ```xml
  invisible="invoice_status != 'no' or state != 'sale'"
  invisible="state != 'draft' or invoice_count &gt;= 1"
  invisible="not locked"
  invisible="state not in ['draft', 'sent', 'sale'] or not id or locked"
  ```
- Also relevant: a residual deprecation warning (not error) for using JS-style
  `true`/`false` literals instead of Python `True`/`False` in these expressions:
  `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:3499`
  (`"Using Javascript syntax 'true, 'false' in expressions is deprecated, found %s"`).

**Why it matters:** A stale LLM asked to conditionally hide/disable a field for a
Sentry-reported KeyError/AccessError will likely emit
`attrs="{'invisible': [('state', '=', 'done')]}"`. This is not silently ignored —
it raises `ValidationError: Since 17.0, the "attrs" and "states" attributes are no
longer used` at module install/update, breaking the whole module load, not just
the one view. The correct fix is a bare Python-expression string on `invisible=`/
`readonly=`/`required=` directly on the node.

**Confidence:** high

---

## 3. Kanban card templates are named `t-name="card"`, never `t-name="kanban-box"` — and can live in a separate reusable `<card>` view type

**Claim:** The kanban view's per-record template is declared as
`<t t-name="card">` inside `<templates>`. The old `t-name="kanban-box"` name is
completely retired — not a single occurrence exists in any of the four repos.
Additionally, SaaS-19.4 introduces a standalone `ir.ui.view` type `'card'` whose
arch can be defined once and referenced from a `<kanban card_id="%(xml_id)d">` —
decoupling the card template from the kanban view record entirely.

**Evidence:**
- `grep -rlo kanban-box` across `odoo/addons`, `odoo/odoo`, `enterprise`,
  `industry`, `design-themes` (`*.xml`) → **0 files, all five repos.**
- `t-name="card"` usage count: 169 occurrences in community addons alone
  (`grep -rho 't-name="[a-z-]*"' */views/*.xml | sort | uniq -c`), vs. 0 for
  `kanban-box`.
- Real inline example: `~/src/194/odoo/addons/crm/views/crm_lead_views.xml:366-368`
  ```xml
  <templates>
      <t t-name="card">
          <field name="name" class="fw-bold fs-5"/>
  ```
- Standalone `card` view type in the `type` Selection field:
  `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:150-158`
  (`('card', "Card")` alongside `list`, `form`, `kanban`, etc.)
- Real decoupled example — a separate `ir.ui.view` record of type `card`
  referenced by id from the kanban view:
  `~/src/194/odoo/addons/maintenance/views/maintenance_views.xml:139-181`
  ```xml
  <record id="hr_equipment_request_view_card" model="ir.ui.view">
      <field name="arch" type="xml">
          <card>
              <templates>
                  <t t-name="card"> ... </t>
              </templates>
          </card>
      </field>
  </record>
  <record id="hr_equipment_request_view_kanban" model="ir.ui.view">
      <field name="arch" type="xml">
          <kanban card_id="%(hr_equipment_request_view_card)d" ... />
      </field>
  </record>
  ```
  Same `card_id=` pattern also seen in `mrp/views/mrp_production_views.xml:671`,
  `project/views/project_project_views.xml:584`, `project/views/project_task_views.xml:765`,
  `stock_picking_batch/views/stock_picking_batch_views.xml:232`.
- JS side confirms `card` template parsing is shared/required infrastructure, not
  convention: kanban delegates to the same parser used by the standalone card
  view — `~/src/194/odoo/addons/web/static/src/views/kanban/kanban_arch_parser.js:4-9`
  (`import { CardArchParser } from "@web/views/card/card_arch_parser"`), and
  `~/src/194/odoo/addons/web/static/src/views/card/card_arch_parser.js:6,17-18`
  defines `CARD_ATTRIBUTE = "card"` and indexes templates by their `t-name`.

**Why it matters:** Fixing a Sentry crash/blank-kanban-card bug, a stale LLM will
almost certainly write `<t t-name="kanban-box">` — this silently produces an
empty/default card (no template found under the name the renderer looks up), a
much harder bug to spot than a hard error, and could easily be mis-diagnosed as
"the kanban view isn't picking up my field" when actually the whole template is
being ignored. Additionally, if the kanban view being patched uses `card_id=`
(no inline `<templates>` at all), a stale LLM might try to inline a `<templates>`
block directly under `<kanban>` and never realize the actual template lives in a
separate `ir.ui.view` record of type `card` — patching the wrong file/view entirely.

**Confidence:** high

---

## 4. `<chatter/>` is a self-closing tag compiled by a form_compiler plugin, not the old `<div class="oe_chatter">` + explicit `message_follower_ids`/`message_ids`/`activity_ids` field block

**Claim:** The chatter is now a single opaque `<chatter/>` tag (optionally with
boolean-ish reload-control attributes) placed directly in the form view, compiled
client-side into the Chatter OWL component. The old pattern of manually listing
`<div class="oe_chatter"><field name="message_follower_ids" widget="mail_followers"/>...`
inside the form is gone from real view files.

**Evidence:**
- Real usage, `~/src/194/odoo/addons/mail/views/res_partner_views.xml:18`
  and `~/src/194/odoo/addons/mail/views/res_users_views.xml:59`:
  ```xml
  <chatter/>
  <chatter reload_on_follower="True"/>
  ```
  Also widely used in enterprise, e.g.
  `~/src/194/enterprise/l10n_ch_hr_payroll/views/l10n_ch_declaration_views.xml:51`
  (`<chatter reload_on_attachment="True"/>`) and
  `~/src/194/enterprise/whatsapp/views/whatsapp_template_views.xml:150`.
- Client-side compiler that turns `<chatter>` into the Chatter component and reads
  its reload-control attributes:
  `~/src/194/odoo/addons/mail/static/src/chatter/web/form_compiler.js:6-24,45-48`
  ```js
  function compileChatter(node, params) {
      ...
      hasParentReloadOnActivityChanged: Boolean(node.getAttribute("reload_on_activity")),
      hasParentReloadOnAttachmentsChanged: Boolean(node.getAttribute("reload_on_attachment")),
      hasParentReloadOnFollowersUpdate: Boolean(node.getAttribute("reload_on_follower")),
      hasParentReloadOnMessagePosted: Boolean(node.getAttribute("reload_on_post")),
      isAttachmentBoxVisibleInitially: Boolean(node.getAttribute("open_attachments")),
      ...
  }
  registry.category("form_compilers").add("chatter_compiler", {
      selector: "chatter",
      fn: compileChatter,
  });
  ```

**Why it matters:** If a Sentry crash is e.g. an `AttributeError`/`ValueError` from
manually adding `message_follower_ids`/`message_ids` fields into a form (assuming
old-style manual chatter wiring), a stale LLM might "fix" it by re-adding a raw
`<div class="oe_chatter">` block with explicit mail-thread fields instead of using
`<chatter/>` — this may work superficially, but diverges from the framework's real reload
lifecycle (`reload_on_follower`/`reload_on_attachment`/`reload_on_post`/
`reload_on_activity`, and `open_attachments` for the attachment box), meaning
targeted fixes about "chatter not refreshing after X" need to touch these
attributes on `<chatter/>`, not custom field wiring.

**Confidence:** high

---

## 5. `invisible=` vs `column_invisible=` are two distinct mechanisms in `<list>` views — using `invisible` where `column_invisible` is needed is a real, current footgun

**Claim:** Inside `<list>` (former `<tree>`) views, a `<field>`'s `invisible=`
attribute and `column_invisible=` attribute are evaluated differently and are not
interchangeable: the framework itself auto-generates `column_invisible` (rather
than `invisible`) specifically when the arch root tag is `list`, for fields it
injects to satisfy expression dependencies. `column_invisible` hides the entire
column (evaluated once, not per-row); `invisible` (still valid on list fields too)
hides the cell content per-record/row.

**Evidence:**
- Framework auto-injection explicitly branches on root tag:
  `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:1567`
  ```python
  attrs = {
      'name': name,
      'invisible' if root.tag != 'list' else 'column_invisible': 'True',
      'readonly': str(readonly),
      ...
  }
  ```
- Widespread real usage of `column_invisible` specifically (215 files in
  community addons alone), e.g.
  `~/src/194/odoo/addons/sale/views/sale_order_views.xml:130-131,143`:
  ```xml
  <field name="message_needaction" column_invisible="True"/>
  <field name="currency_id" column_invisible="True"/>
  <field name="company_id" groups="!base.group_multi_company" column_invisible="True"/>
  ```
- JS renderer treats them as separate concepts with separate evaluation contexts:
  `~/src/194/odoo/addons/web/static/src/views/list/list_renderer.js:353,1180-1181`
  ```js
  if (this.evalColumnInvisible(col.column_invisible)) { ... }
  evalColumnInvisible(columnInvisible) {
      return evaluateBooleanExpr(columnInvisible, this.props.list.evalContext);
  }
  ```
  (evaluated against the list's own evalContext — not the row/record — confirming
  it is a column-level, not row-level, condition), and
  `~/src/194/odoo/addons/web/static/src/views/list/list_arch_parser.js:99-101,110,184`
  parses `column_invisible` as its own attribute distinct from `invisible`.

**Why it matters:** Fixing a Sentry bug like "field X always shows in the list even
though I marked it invisible", a stale LLM will add `invisible="1"` (or a
condition) to the `<field>` inside the `<list>` view. That may not remove the
column header / may behave inconsistently with optional-column toggling, because
`column_invisible` is the mechanism actually meant for "never show this column"
(the framework's own auto-added technical fields like `message_needaction`,
`currency_id`, `display_type` universally use `column_invisible`, not
`invisible`, for exactly this purpose). Conversely, using `column_invisible` on a
field meant to conditionally show/hide per-row content is also wrong.

**Confidence:** high

---

## 6. xpath inheritance now edits `invisible`/`readonly`/etc. directly via `<attribute name="invisible">expr</attribute>` — not `<attribute name="attrs">{...}</attribute>`

**Claim:** Since `attrs=` doesn't exist (see candidate #2), `position="attributes"`
xpath patches that used to rewrite a single `attrs` dict now instead target each
modifier attribute (`invisible`, `readonly`, `required`, `column_invisible`,
`class`, etc.) individually and directly.

**Evidence:**
- `position="attributes"` remains extremely common (1111 occurrences in community
  addons alone via `grep -rho 'position="[a-z_]*"' */views/*.xml | sort | uniq -c`),
  confirming this is still the standard inheritance idiom for modifying an
  existing node's attributes — but its *contents* have changed.
- Real example rewriting `invisible` directly via xpath inheritance:
  `~/src/194/odoo/addons/account_payment/views/payment_provider_views.xml:12-14`
  ```xml
  <group name="provider_config_others" position="attributes">
      <attribute name="invisible">False</attribute>
  </group>
  ```
- Other real `position="attributes"` targets confirm per-attribute (not
  per-`attrs`-dict) usage:
  `~/src/194/odoo/addons/sale/views/sale_order_views.xml:105-107,214-216,227-229,233-238`
  ```xml
  <kanban position="attributes"><attribute name="js_class">sale_file_upload_kanban</attribute></kanban>
  <list position="attributes"><attribute name="string">Quotations</attribute></list>
  <field name="state" position="attributes"><attribute name="optional">show</attribute></field>
  <field name="invoice_status" position="attributes"><attribute name="optional">hide</attribute></field>
  ```
- No occurrences of `<attribute name="attrs">` or `<attribute name="states">`
  found in any of the sampled real view files (consistent with candidate #2's
  finding that `attrs=`/`states=` are gone entirely, including inside xpath
  patches).

**Why it matters:** A stale LLM patching a Sentry bug via inheritance (e.g. "make
this button invisible for this edited module") will often emit
`<field name="x" position="attributes"><attribute name="attrs">{'invisible': [...]}</attribute></field>` —
this both fails candidate #2's validator (raises on any `attrs=`/`states=` found
anywhere in the combined arch, including ones introduced only via inheritance)
and, separately, silently leaves the original `invisible=`/`readonly=` expression
on the target node completely untouched (an `<attribute name="attrs">` with no
matching original attribute does not override `invisible=`). The correct fix
targets the modifier attribute by its real name (`invisible`, `readonly`,
`required`, `column_invisible`) directly.

**Confidence:** high

---

## 7. `<list>` inline sub-views (one2many editable lists) reuse the *form* view's postprocessing/validation, not tree-specific logic — `editable=` is still `top`/`bottom` only, validated in Python for inline views

**Claim:** In the arch-processing code, `<list>` nodes are validated and
postprocessed by literally delegating to the form-view handlers
(`_postprocess_tag_form`/`_validate_tag_form`), and inline list views embedded in
a form (as `<field name="line_ids"><list editable="...">`) get an extra Python-side
check for `editable` because such inline lists bypass RNG schema validation.

**Evidence:** `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:1759-1761`:
```python
def _postprocess_tag_list(self, node, name_manager, node_info):
    # reuse form view post-processing
    self._postprocess_tag_form(node, name_manager, node_info)
```
and around line 1892-1900 (`_validate_tag_list`):
```python
def _validate_tag_list(self, node, name_manager, node_info):
    # reuse form view validation
    self._validate_tag_form(node, name_manager, node_info)
    if not node_info['validate']:
        return
    # inline list views inside form views aren't rng validated, so we must validate the
    # editable attribute in python
    editable_attr = node.get("editable")
    if editable_attr and editable_attr not in ["top", "bottom"]:
```

**Why it matters:** Less about surface XML syntax and more about where to look
when a Sentry crash traces into view validation for an inline one2many list
(`editable="..."` typo, or a field-group-visibility bug inside a one2many list
column) — the actual validating/postprocessing code path for those inline
`<list>` nodes is the *form* tag's handlers, not a separate list-specific code
path. A stale LLM (or one unfamiliar with this codebase) hunting for
"`_validate_tag_list`" logic expecting list-specific field/group validation rules
would misdiagnose where a bug in inline-editable-list validation actually lives.
Also concretely: `editable=` only accepts `"top"` or `"bottom"` — any other value
(e.g. a boolean-ish `"true"`) is invalid and raises here, not silently ignored.

**Confidence:** medium (this is more of an internals/debugging nuance than a
pure view-XML-authoring nuance, so its direct relevance to "what a patch should
look like" is a bit more indirect than candidates 1-6, but it's directly useful
for triaging where a related Sentry traceback frame comes from).

---

## 8. Root-tag/type mismatches, not just `attrs`, are validated per-node against the *view's own concrete tag* — writing a raw JS-boolean `true`/`false` in an expression still loads today but logs a deprecation warning, so it is NOT proof the arch is broken

**Claim:** Distinct from candidate #2 (hard error for `attrs=`/`states=`), there is
a separate, *softer* legacy-syntax check: using the JS literals `true`/`false`
(instead of Python's `True`/`False`) inside an `invisible=`/`readonly=` boolean
expression string does not raise — it's caught and just logged as a deprecation
warning, continuing to evaluate. This matters for triage: seeing this warning in
logs near a Sentry error does not mean the expression itself is the crash cause.

**Evidence:** `~/src/194/odoo/odoo/addons/base/models/ir_ui_view.py:3496-3500`
```python
info = self.available_fields[name].get('info')
if info is None:
    if name in ['false', 'true']:
        _logger.warning("Using Javascript syntax 'true, 'false' in expressions is deprecated, found %s", name)
        continue
```
(part of the `used_fields` consistency check — `false`/`true` are treated as
lowercase identifiers referencing a "field" named `false`/`true`, specifically
special-cased here instead of raising the generic "unknown field used in
expression" error that other bogus identifiers would trigger.)

**Why it matters:** If a Sentry-adjacent server log shows this deprecation
warning, a stale/pattern-matching LLM might conclude "the view arch is broken,
attrs/states must be present" (per candidate #2's hard error) and go rewrite
unrelated code, when actually the arch loaded fine — the only issue is
`true`/`false` vs `True`/`False` capitalization in one expression, a trivial
fix, and not the actual root cause of whatever crash prompted the investigation.
Conflating "logs a warning" with "view failed to load" would send the fix in the
wrong direction entirely.

**Confidence:** medium


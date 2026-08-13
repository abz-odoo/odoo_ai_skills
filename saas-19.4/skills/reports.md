# Reports (QWeb PDF/HTML) — candidate nuances for Odoo SaaS-19.4

**Coverage note**: This pass focused on `odoo/odoo/addons/base/models/ir_actions_report.py`,
`report_paperformat.py`, `report_layout.py`, the two PDF-engine addons
(`base_report_wkhtmltox`, `base_report_paper_muncher`), `web/views/report_templates.xml`,
`web/models/base_document_layout.py`, `web/data/report_layout.xml`, and `t-lang` in
`ir_qweb.py`. It did not systematically sample enterprise/industry/design-themes templates
beyond a spot check, nor per-module `ir_actions_report.py` overrides in `account`/`sale`/`stock`.
Git history was available and used to corroborate several findings.

---

## 1. PDF rendering is now a pluggable multi-engine architecture, not hardcoded wkhtmltopdf

**Claim**: The base model (`odoo/odoo/addons/base/models/ir_actions_report.py`) only defines an
engine-agnostic contract (`_get_pdf_engine`, `_run_pdf_engine`, `_run_pdf_engine_without_processing`,
`get_pdf_engine_state`) that all raise `NotImplementedError` by default. Actual engines are
separate addons overriding these hooks: `base_report_wkhtmltox` and the new
`base_report_paper_muncher`. Git confirms deliberate refactor: `d3b4294a56a5 [REF] base:
modularize the reporting engines`.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_actions_report.py:769-833` — hooks raise `NotImplementedError`
  ("If we reach here it means no module handled the pdf rendering with the given engine_name.").
- `odoo/addons/base_report_wkhtmltox/models/ir_actions_report.py:127-133` — adds `report_type`
  selection value `qweb-pdf-wkhtmltopdf` via `selection_add`, overrides hooks only
  `if engine_name == 'wkhtmltopdf'`.
- `odoo/addons/base_report_paper_muncher/models/ir_actions_report.py:24-28` — adds
  `qweb-pdf-paper-muncher` the same way.
- Git log on that file shows `d3b4294a56a5 [REF] base: modularize the reporting engines` as top
  structural commit.

**Why it matters**: A stale LLM will assume `ir_actions_report.py` directly shells out to
wkhtmltopdf, or assume one engine system-wide. Which engine runs is data-driven per report
(`report_type`) and/or `report.pdf_engine_default`; patches must target the correct engine addon,
not the base model.

**Confidence**: high

---

## 2. Paper Muncher — a brand-new in-house PDF engine, not a wkhtmltopdf wrapper

**Claim**: 19.4 ships a second, in-house PDF engine ("Paper Muncher") running as a subprocess
speaking a custom HTTP-like protocol (via `h11`) over stdin/stdout pipes — not CLI flags + temp
files like wkhtmltopdf. It fetches documents via synthetic `GET /paper-muncher/<idx>.html` and
returns the PDF via `PUT /paper-muncher/output.pdf`. For other requests it re-enters Odoo's own
WSGI stack (`_handle_fallback` calls `root(environ, start_response)`), running "as the public
user" per code comment.

**Evidence**:
- `odoo/addons/base_report_paper_muncher/__manifest__.py:1-16` — links to
  `https://odoo.github.io/paper-muncher/`.
- `odoo/addons/base_report_paper_muncher/paper_muncher.py:1-30` — imports `h11`, defines
  `PaperMuncherServer.serve()` using `selectors`.
- `paper_muncher.py:139-141` (`GET_DOCUMENT_RE`) and `_handle_get_document`/`_handle_put`/
  `_handle_fallback` (~178-280) show the protocol.
- `paper_muncher.py:41` — `FALLBACK_BIN_PATH = '/opt/paper-muncher/bin/paper-muncher'`.
- `odoo/addons/base_report_paper_muncher/models/ir_actions_report.py:95-96` — undocumented env
  var: `if os.getenv('ODOO_PAPER_MUNCHER_FEATURE') == '1': extra_args += ['--feature', '*=on']`.
- Git: `e9a488fb11a1 [ADD] base_report_paper_muncher: integrate paper-muncher into odoo`, plus
  `180915b81068 [FIX] paper-muncher: fix type inconsistency for report_ref` and
  `45b532d7cdaf [FIX] paper-muncher: fix header/footer splitting` — actively iterated, young
  subsystem.

**Why it matters**: A stale LLM diagnosing "PDF looks wrong / generation hangs" will reflexively
suspect wkhtmltopdf, when the report may use `qweb-pdf-paper-muncher` and the failure is in
`PaperMuncherServer` protocol handling, `SERVE_TIMEOUT = 15*60`, or
`assert body.startswith(b'%PDF-')`. Header/footer handling also differs fundamentally:
paper-muncher builds one full HTML doc per page (`make_multi_docs_html`, string-based),
wkhtmltopdf writes real temp files.

**Confidence**: high

---

## 3. Paper format margins are silently ignored by the Paper Muncher engine

**Claim**: `report.paperformat` fields `margin_top/bottom/left/right`, `header_spacing`,
`header_line`, `disable_shrinking`, `dpi` are consumed fully by the wkhtmltopdf engine
(`_build_wkhtmltopdf_args`), but the Paper Muncher engine **always** passes `--margins none` and
only ever forwards `--paper`/`--width`/`--height`, `--orientation`, `--scale` (from a `scale`
override, default 72 — not the paperformat's `dpi` field). Margins/header-spacing/header-line/
disable-shrinking have no equivalent in the paper-muncher path.

**Evidence**:
- `odoo/addons/base_report_paper_muncher/models/ir_actions_report.py:90-108`:
```python
extra_args = [
    '--scale', f'{scale}dpi',
    '--margins', 'none',
]
...
if paperformat and paperformat.format:
    if paperformat.format != 'custom':
        extra_args += ['--paper', paperformat.format]
    elif paperformat.page_height and paperformat.page_width:
        extra_args += ['--width', f'{paperformat.page_width}mm']
        extra_args += ['--height', f'{paperformat.page_height}mm']
```
No margin/header field referenced anywhere in this file.
- Contrast `odoo/addons/base_report_wkhtmltox/models/ir_actions_report.py:264-311`
  (`_build_wkhtmltopdf_args`) which maps every one of those fields to `--margin-top/bottom/left/
  right`, `--header-spacing`, `--header-line`, `--disable-smart-shrinking`.

**Why it matters**: If a bug report says "margins wrong for this customer's paperformat," a stale
LLM will look at `report.paperformat` computation and assume a bug there. If that report resolves
to paper-muncher, margins were never meant to reach the engine — fix belongs in the report's own
CSS/`@page` rules or in extending `_run_paper_muncher`, not paperformat margin logic.

**Confidence**: high

---

## 4. Silent HTML fallback when no PDF engine is available/configured

**Claim**: When no PDF engine resolves (`engine_name == 'html'`), Odoo does **not** raise — it
silently renders as HTML and returns it in place of the PDF, with debug mode forced off. "PDF"
report failures can show up as garbled output or HTML content served where `%PDF-` bytes were
expected, rather than as exceptions.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_actions_report.py:474-489` (`_render_qweb_pdf_prepare_streams`):
```python
engine_name = self._get_pdf_engine(report_sudo)
if engine_name == 'html':
    # PDF generation is not setup
    ...
    return self.with_context(**additional_context)._render_qweb_html(report_ref, all_res_ids_wo_stream, data=data)[0]
```
- `ir_actions_report.py:769-781` (`_get_pdf_engine`):
```python
engine_name = (
    report_type.removeprefix('qweb-pdf-')
    if report_type.startswith('qweb-pdf-')
    else self.env['ir.config_parameter'].sudo().get_str('report.pdf_engine_default') or 'html'
    )
return engine_name if engine_name and engine_name != 'html' else default_engine
```
A plain `qweb-pdf` report_type with no config parameter set falls back to `'html'`.

**Why it matters**: A stale LLM chasing a "corrupted/blank PDF attachment" error may assume a
wkhtmltopdf crash or `_merge_pdfs` bug, when the real cause is that the resolved engine was
`'html'` all along — the "PDF" was never converted, and downstream attachment logic (expecting
real PDF bytes for splitting) may misbehave on HTML content instead.

**Confidence**: medium — branch is unambiguous; how often real deployments hit this state wasn't
tested.

---

## 5. `t-lang` is a same-node alias for `t-options-lang`, valid only alongside `t-call`

**Claim**: `t-lang` is a compile-time QWeb directive, rewritten to `t-options-lang`, and it
**raises a `SyntaxError` at compile time if used without `t-call` on the same element**. It can't
be placed on a parent to affect nested calls.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_qweb.py:2734-2738`:
```python
def _compile_directive_lang(self, el, compile_context, level):
    if 't-call' not in el.attrib:
        raise SyntaxError("t-lang is an alias of t-options-lang but only available on the same node of t-call")
    el.attrib['t-options-lang'] = el.attrib.pop('t-lang')
    return self._compile_node(el, compile_context, level)
```
- Doc block `ir_qweb.py:257-267` documents the restriction and semantics explicitly.
- Real usage, e.g. `odoo/addons/stock_fleet/report/report_picking_cmr.xml:103` —
  `<t t-call="stock_fleet.report_cmr_translated_label" t-lang="copy_lang_code"/>`, always
  co-located with `t-call`.

**Why it matters**: A Sentry `SyntaxError` mentioning `t-lang` might tempt a stale LLM to move it
onto a wrapping `t-foreach`/parent node, which just relocates the same compile error rather than
fixing multi-language rendering. Fix must keep `t-lang` and `t-call` on the same element and
address the language-expression value.

**Confidence**: high

---

## 6. Wkhtmltopdf still exists and has its own quirks — don't assume it's replaced or that it's the only engine

**Claim**: `base_report_wkhtmltox` is `auto_install: True` and does real work: binary discovery,
version parsing, DPI-zoom compensation gated on version `>= 0.12.2`, and a workaround splitting
large HTML tables into ≤500-row chunks because wkhtmltopdf has *exponential* processing time on
big tables. It coexists with Paper Muncher rather than being replaced.

**Evidence**:
- `odoo/addons/base_report_wkhtmltox/__manifest__.py:1-14` — `'auto_install': True` (vs.
  `base_report_paper_muncher`, not auto_install).
- `base_report_wkhtmltox/models/ir_actions_report.py:27-46` (`_split_table`) — 500-row workaround,
  citing internal ticket `opw-1689673` at line 442.
- `ir_actions_report.py:58-95` (`_wkhtml()`) — `state='upgrade'` if `<0.12.0`;
  `dpi_zoom_ratio=True` only if `>=0.12.2`; `state='workers'` if `config['workers']==1`.
- `ir_actions_report.py:461-474` — return code `-11` specifically decoded as OOM/subprocess-limit.

**Why it matters**: The inverse risk of #1/#2: an LLM that just learned "Odoo 19 uses Paper
Muncher" might wrongly blame paper-muncher for every PDF bug and ignore wkhtmltopdf-specific
evidence (e.g. stack trace with `wkhtmltopdf` in argv, or a report stuck on >500-row tables). Fix
must be scoped to whichever engine the failing report actually uses.

**Confidence**: high

---

## 7. External layout catalog has expanded well beyond the classic 4 (standard/boxed/bold/striped)

**Claim**: `report.layout` offers at least 7 layouts in 19.4: `standard`, `bubble`, `wave`,
`folder`, `center`, `dual`, `lines`. Separately, *table* styling is an independent axis via
company field `report_tables_id` (6 choices: `light`, `boxed`, `bold`, `striped`, `bubble`,
`column`) applied as CSS classes inside `external_layout_body`, decoupled from which
`external_layout_*` page-chrome template is used.

**Evidence**:
- `odoo/addons/web/data/report_layout.xml:8,14,20,26,33,39,45` — `view_id` refs to
  `web.external_layout_standard/bubble/wave/folder/center/dual/lines`.
- `odoo/odoo/addons/base/models/res_company.py:123-130`:
```python
report_tables_id = fields.Selection([
    ('light', 'Light'), ('boxed', 'Boxed'), ('bold', 'Bold'),
    ('striped', 'Striped'), ('bubble', 'Bubble'), ('column', 'Column'),
], string='Table Design', default='light')
```
- `odoo/addons/web/views/report_templates.xml:438-448` — `external_layout_body` computes table
  CSS classes from `company.report_tables_id` independently of the page layout.
- `report_templates.xml:466-490` (`external_layout_standard`) vs `:492-523`
  (`external_layout_wave`) — genuinely different DOM/SVG structure per layout.

**Why it matters**: A stale LLM fixing "logo/company info misplaced in header" might edit
`external_layout_standard` assuming it's the only/default template, or conflate table styling
with page layout since they used to be closer concepts. Actual active template must be read from
`company.external_report_layout_id.key` (default fallback at `report_templates.xml:861`), not
assumed.

**Confidence**: medium — template inventory solidly cited; whether all 7 are new-to-19.x
specifically (vs. some pre-existing in 17) wasn't diff-verified against an older checkout.

---

## 8. PDF-to-record splitting relies on PDF outline structure, with a specific fallback and error UX

**Claim**: Multi-record PDF printing tries, in order: (1) single record with no cached stream →
use whole PDF; (2) page count equals record count → split 1 page = 1 record; (3) otherwise split
via **PDF outline (`/Outlines`) entries** derived from `<h?>` tags, falling back to a fully serial
per-record re-render if outline count mismatches or the first outline isn't at page 0. Separately,
PDF merge failures raise `RedirectWarning` pointing at an `act_window` listing the exact
problematic record(s), not a bare `UserError`.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_actions_report.py:536-604` — full outline-splitting logic
  (`has_valid_outlines`, `has_same_number_of_outlines`, `has_top_level_heading`, recursive
  per-record fallback).
- `ir_actions_report.py:519-525` — explicit `UserError` if `report.attachment` is set but template
  lacks `data-oe-model`/`data-oe-id` on the `article` div.
- `ir_actions_report.py:686-718` — `custom_handle_merge_pdfs_error` collects `error_record_ids`,
  raises `RedirectWarning` with an `act_window` action filtered to those ids.

**Why it matters**: A stale LLM debugging "wrong record's data on someone else's printed invoice
in bulk-print" needs to know splitting is heuristic (outline-based) and can silently mis-split if
the template's heading structure is missing/duplicated/reordered — the bug may be in the *report
template's heading structure*, not the merge/split Python. It also shouldn't "fix" a merge failure
by suppressing exceptions, since callers rely on catching `RedirectWarning` specifically to show
users the problematic records.

**Confidence**: medium — logic precisely cited; whether this exact 3-tier strategy is new-to-19.4
vs. inherited from 17 unchanged wasn't diff-verified.

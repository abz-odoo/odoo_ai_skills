# ir.actions.* candidates — Odoo SaaS-19.4

Scope note: this pass focused on `odoo/odoo/addons/base/models/ir_actions.py`,
`ir_actions_report.py`, `ir_cron.py`, `ir_embedded_actions.py` and their tests
in `odoo/odoo/addons/base/tests/`, cross-checked with `git log` on those files
(the `194/odoo` repo is a real git checkout on branch `saas-19.4`). I did not
do a systematic sweep of `enterprise/`, `industry/`, or `design-themes/` for
ir.actions usage beyond a couple of spot checks implied by the git log
messages (e.g. `enterprise#85875` referenced in the `_notify_progress`
deprecation commit) — if deeper cross-repo confirmation is wanted, a follow-up
pass should grep those repos for `_commit_progress`, `_notify_progress`,
`state == 'webhook'`, and `ir.embedded.actions` usages. 8 candidates below,
capped as instructed.

Also worth flagging up front: **the `base` module moved**. It is no longer at
`addons/base/` (top-level) — in this checkout it lives at
`odoo/odoo/addons/base/`. A stale agent hardcoding `addons/base/models/...`
paths will fail to find these files at all.

---

## 1. `ir.actions.server` gained a real `webhook` state (new action type)

**Claim**: Server actions now support a `state == 'webhook'` type that POSTs a
JSON payload to an external URL. This is not a community/OCA bolt-on — it's a
built-in `state` selection value in base, with its own fields, sample-payload
preview, and a dedicated runner. In Odoo 16/17, `ir.actions.server.state` only
had `code`, `object_write`, `object_create`, `multi` (plus mail-related states
added by other modules) — there was no webhook concept in base at all.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_actions.py:591-607` — `state` selection
  includes `('webhook', 'Send Webhook Notification')`.
- `odoo/odoo/addons/base/models/ir_actions.py:683-689` — new fields:
  `webhook_url`, `webhook_field_ids` (m2m to `ir.model.fields`, via relation
  table `ir_act_server_webhook_field_rel`), `webhook_sample_payload` (computed
  preview).
- `odoo/odoo/addons/base/models/ir_actions.py:1041-1083` —
  `_run_action_webhook(self, eval_context=None)`: builds a JSON payload from
  `webhook_field_ids`, sends via `requests`, and does so as a "send and
  forget" postcommit call — see comment at line ~1072 ("'send and forget'
  strategy, and avoid locking the user if the webhook...").
- `odoo/odoo/addons/base/tests/test_ir_actions.py:585-622` —
  `test_90_webhook`: note the explicit comment
  `self.env.cr.postcommit.run()  # webhooks run in postcommit` (lines 618,
  622) — the HTTP call is deferred until after the transaction commits, not
  fired synchronously inline with `run()`.
- git log: `bc87ff7600cd [FIX] base: prevent webhook sample payload
  serialization crash`, `5d576e4c7544 [FIX] base: send webhook calls after
  cursor commit` — both on `ir_actions.py`.

**Why it matters**: A Sentry crash inside a server action that fires an
outbound HTTP call, or a crash/hang related to `requests` inside `ir.actions
.server`, might be mis-triaged by a stale-knowledge LLM as "someone must have
written custom code doing `requests.post(...)` inside a code-type server
action" — leading it to patch the user's Python code — when the actual
mechanism is the built-in webhook state, which runs post-commit (so DB state
is committed, any raised exception inside `_run_action_webhook` won't roll
back business logic) and has its own field-level access-restriction warning
(see candidate 5). A patch that tries to make the webhook call "transactional"
or synchronous would fight the intended architecture.

**Confidence**: high

---

## 2. `ir.embedded.actions` — a whole new model, not a variant of menu items

**Claim**: SaaS-19.4 introduces `ir.embedded.actions`, a standalone model
(`odoo/odoo/addons/base/models/ir_embedded_actions.py`, 112 lines) with no
16/17 analogue. It lets an `ir.actions.act_window` have one or more
"embedded actions" attached (e.g. buttons/tabs shown inside a view, tied to
`parent_action_id` + `parent_res_model`/`parent_res_id`), which can point
either to a full `ir.actions.actions` record (`action_id`) OR to a bare
Python method name (`python_method`) that returns an action dict — mutually
exclusive by a SQL CHECK constraint.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_embedded_actions.py:8-41` — model
  definition; note the two hard constraints:
  - `_check_only_one_action_defined` (lines 31-37): CHECK that exactly one of
    `action_id`/`python_method` is set.
  - `_check_python_method_requires_name` (lines 38-41): CHECK that
    `python_method` implies `name` is set.
- `odoo/odoo/addons/base/models/ir_embedded_actions.py:77-96` —
  `_compute_is_visible`: visibility is computed from `active_id` in context,
  `groups_ids` intersected with `self.env.user.all_group_ids`, and whether
  `python_method` actually exists on the target model
  (`hasattr(self.env[parent_res_model], record.python_method)`).
- git log on this file shows it is actively evolving: `f983703dfa3c [IMP]
  web, base: Add embedded actions`, `12c1da9bc27e [IMP] base: allow use of
  server actions on ir.embedded.actions`, `68918a457ebf [FIX] web,base: makes
  sure python_method and action_id is not both set`.
- `odoo/odoo/addons/base/tests/test_ir_embedded_actions.py` has dedicated
  tests: `test_cannot_delete_default_embedded_action`,
  `test_domain_on_embedded_action`, `test_groups_on_embedded_action`,
  `test_create_embedded_action_with_action_and_python_method` (lines 63-166).

**Why it matters**: If a Sentry crash traceback mentions `ir.embedded.actions`
or `_compute_is_visible`/`python_method`, a stale LLM has zero prior model for
this and may try to "fix" it by treating it like a regular `ir.actions
.act_window` misconfiguration (e.g. editing menu/action XML) rather than the
actual embedded-action record (its domain, groups_ids, or the
`hasattr(..., python_method)` guard). It could also miss that `action_id` and
`python_method` are mutually exclusive by DB constraint — a naive patch that
sets both fields at once would violate the CHECK constraint and crash with an
IntegrityError that looks unrelated to the original bug.

**Confidence**: high

---

## 3. `ir.cron` scheduled actions now run through an explicit batching/retry
state machine (`_commit_progress` + `CompletionStatus`), not a single call

**Claim**: Cron execution is no longer "call the server action once, then
reschedule by interval." Each job run is wrapped by `_process_job`/`_run_job`
and produces a `CompletionStatus` (`FULLY_DONE`, `PARTIALLY_DONE`, `FAILED` —
a `enum.StrEnum`). Long-running cron functions are expected to call
`self.env['ir.cron']._commit_progress(processed=..., remaining=...)`
periodically; this commits the current transaction, logs progress into a new
`ir.cron.progress` model, and returns the remaining wall-clock time budget
(seconds) for the current run — the intended pattern is a `while` loop that
keeps working until `_commit_progress` returns a falsy/zero-or-negative
budget, at which point the job self-reschedules ASAP as `PARTIALLY_DONE`
rather than being killed/timed out.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_cron.py:63-66` —
  `class CompletionStatus(enum.StrEnum): FULLY_DONE = 'fully done'; \
  PARTIALLY_DONE = 'partially done'; FAILED = 'failed'`.
- `odoo/odoo/addons/base/models/ir_cron.py:400-453` — `_process_job` docstring
  explains the three outcomes and their scheduling consequences (rescheduled
  later vs. ASAP vs. deactivated-if-repeatedly-failing).
- `odoo/odoo/addons/base/models/ir_cron.py:869-911` — `_commit_progress`
  definition: signature `(self, processed: int = 0, *, remaining:
  int | None = None, deactivate: bool = False) -> float`; commits the cursor
  (`self.env.cr.commit()`), writes to `ir.cron.progress`, and returns
  `max(ctx.get('cron_end_time', float('inf')) - time.monotonic(), 0)`.
- `odoo/odoo/addons/base/tests/test_ir_cron.py:227-300` (`test_cron_process_job`)
  and `:673-712` (`test_cron_commit_progress`) show canonical usage, e.g. line
  270: `while self.env['ir.cron']._commit_progress(remaining=state['remaining']):`
  followed by incremental `_commit_progress(1)` calls per item.
- New supporting model `ir.cron.progress`
  (`odoo/odoo/addons/base/models/ir_cron.py:936-955`) with fields
  `remaining`, `done`, `deactivate`, `timed_out_counter`, auto-vacuumed weekly.

**Why it matters**: A Sentry crash/timeout in a cron job (e.g. "cron job X
keeps failing/timing out") is exactly the scenario where a stale LLM will
reach for 16/17-era mental models: "cron just calls the method once per
`nextcall`, so make the loop finish faster or increase `numbercall`/interval."
It will not know that (a) partial progress is expected and normal
(`PARTIALLY_DONE` just means "reschedule ASAP, don't treat as an error"), (b)
the correct fix for a long-running batch job is almost always to add
`_commit_progress()` calls inside the processing loop rather than trying to
process everything in one shot, and (c) repeated `FAILED` status only
deactivates the cron after a threshold (see candidate 7) — so "the cron got
disabled" is a symptom, not something to patch by re-enabling it once.

**Confidence**: high

---

## 4. `_notify_progress` is deprecated in favor of `_commit_progress` — same-era code may use either

**Claim**: There are two progress-reporting APIs on `ir.cron` right now:
the old `_notify_progress(*, done, remaining, deactivate=False)` — now
decorated `@deprecated("Since 19.0, use _commit_progress")` and emitting a
`DeprecationWarning` at call time — and the new `_commit_progress(processed,
*, remaining=None, deactivate=False)`. They are NOT drop-in replacements:
`_notify_progress` takes an absolute `done` count, while `_commit_progress`
takes an incremental `processed` count added to the running total.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_cron.py:849-868` —
  ```
  @deprecated("Since 19.0, use _commit_progress")
  def _notify_progress(self, *, done: int, remaining: int, deactivate: bool = False):
  ```
- `odoo/odoo/addons/base/models/ir_cron.py:872-911` — `_commit_progress`
  signature takes `processed: int = 0` (incremental), and computes
  `done = progress.done + processed` (line ~903) unless the caller passes an
  absolute `remaining`.
- `odoo/odoo/addons/base/models/ir_cron.py:24` —
  `from odoo.tools.func import deprecated` (the decorator implementation).
- git log: `1209d62f1506 [IMP] base: deprecate _notify_progress` — commit
  message: "The goal is now to use `_commit_progress` which have a simpler
  API." References `enterprise#85875`, implying enterprise cron jobs were
  migrated too — worth checking `enterprise/` for lingering `_notify_progress`
  calls before assuming a Sentry stack trace using it is "normal."

**Why it matters**: If a stale LLM sees `_notify_progress` in a stack trace
or existing code, it may assume that's simply "the" progress API (since it
never learned `_commit_progress` existed) and write a fix using
`_notify_progress` semantics — passing an absolute `done` where
`_commit_progress`'s `processed` argument is expected, silently double- or
under-counting progress, or miss the `DeprecationWarning` entirely if warnings
aren't surfaced in the Sentry report.

**Confidence**: high

---

## 5. Server actions can carry a blocking `warning` field — `run()` raises `ServerActionWithWarningsError` if unaddressed

**Claim**: `ir.actions.server` has a computed `warning` field
(`recursive=True`) aggregating multiple validation problems — e.g.
mismatched models/groups between a "Multi Actions" parent and its children,
JSON-typed fields used in an update path, sequence-evaluation misuse, or
group-restricted fields included in a webhook payload. If `self.warning` is
truthy, calling `.run()` raises a dedicated `ServerActionWithWarningsError`
(a `UserError` subclass) *before* attempting to execute the action at all —
this is a pre-flight gate, not an execution-time failure.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_actions.py:618` —
  `warning = fields.Text(string='Warning', compute='_compute_warning', recursive=True)`.
- `odoo/odoo/addons/base/models/ir_actions.py:541` — `class
  ServerActionWithWarningsError(UserError):`.
- `odoo/odoo/addons/base/models/ir_actions.py:1184-1186`:
  ```
  if self.warning:
      raise ServerActionWithWarningsError(_("Server action %(action_name)s has one or more warnings, address them first.", action_name=self.name))
  ```
  (inside `_run`, called from `run()`).
- `odoo/odoo/addons/base/models/ir_actions.py:724-772` —
  `_get_warning_messages()` enumerates the concrete conditions (child model
  mismatch, child group mismatch, JSON field in update path, sequence misuse
  on non-char field, group-restricted webhook fields).
- Also new in the same area: `ir.actions.server.history`
  (`odoo/odoo/addons/base/models/ir_actions.py:696-722`) — every time `code`
  changes on a server action (create or write), a history row is inserted, and
  `show_code_history` is computed by checking if any history entry differs
  from the current code. git log: `07d0216047d8 [IMP] base: add server
  actions code history`.

**Why it matters**: A Sentry-reported `ServerActionWithWarningsError` (or a
`UserError` whose message reads "has one or more warnings, address them
first") is not a bug in the action's `code`/logic at all — it's the framework
refusing to run because of a structural misconfiguration (e.g. a multi-action
whose children target different models). A stale LLM would not recognize this
exception class and might dig into the Python `code` field logic looking for
a bug, when the actual fix is to correct the model/group mismatch or field
list flagged by `_get_warning_messages()`. It also would not know about
`ir.actions.server.history`, so if asked "did this code just change," it
should check that model rather than relying on `ir.actions.server.write_date`
or external revision control alone.

**Confidence**: high

---

## 6. `check_access(operation)` replaces `check_access_rights` + `check_access_rule` — used directly in server-action execution

**Claim**: The two-step 16/17 access API (`check_access_rights('write')` for
model-level ACL, then `check_access_rule('write')` for record rules) has been
unified into a single `check_access(operation)` method (and a companion
boolean `has_access(operation)`), and the old names are gone from
`orm/models.py` — not deprecated aliases, just absent. `ir.actions.server`'s
own permission gate for running an action calls the new unified method.

**Evidence**:
- `odoo/odoo/orm/models.py:3377-3396` —
  ```
  @api.private  # use has_access
  @typing.final
  def check_access(self, operation: str) -> None:
  ```
  Docstring: "Verify that the current user is allowed to perform `operation`
  on all the records in `self`." No `check_access_rights`/`check_access_rule`
  definitions exist anywhere in this file (`grep` for both names returns
  nothing besides `check_access`).
- `odoo/odoo/addons/base/models/ir_actions.py:1227` (inside
  `_can_execute_action_on_records`): `self.env[model_name].check_access("write")`.
- `odoo/odoo/addons/base/models/ir_actions.py:1238`: `records.check_access('write')`.
- `odoo/odoo/addons/base/models/ir_actions.py:995`: `self.check_access('write')`.

**Why it matters**: If a Sentry `AccessError` originates from server-action
execution (`_can_execute_action_on_records`), a stale LLM patching the
permission logic might try to call `check_access_rights(...)` /
`check_access_rule(...)` (methods that no longer exist in 19.4 — this would
raise `AttributeError`, not fix anything) or might misunderstand the error
message format, since `check_access` now builds a combined "inaccessible
records" message via `_make_access_error_message` using `_filtered_access`
and `_access_domain` (new private helpers), not the old two-stage rights/rule
error strings.

**Confidence**: high

---

## 7. Cron triggers support coalescing windows, and the minimum per-job time budget was bumped 10s → 120s

**Claim**: `ir.cron._trigger()` gained a keyword-only `coalesce: int = 0`
parameter (minutes). When set, multiple trigger calls within the same
coalescing window get rounded up to the same wakeup timestamp, so bursty
`_trigger()` calls collapse into one execution instead of firing the cron
repeatedly. Separately, the hardcoded minimum wall-clock budget granted to a
single cron job run (`MIN_TIME_PER_JOB`) was raised from 10 seconds to 120
seconds, specifically because some jobs (example cited: fetchmail) need setup
time / remote connections before doing any work.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_cron.py:750` —
  `def _trigger(self, at: datetime | Iterable[datetime] | None = None, *, coalesce: int = 0):`
- `odoo/odoo/addons/base/models/ir_cron.py` (in the same method body) —
  coalescing math:
  ```
  if coalesce:
      factor = coalesce * 60
      at_list = [
          datetime.fromtimestamp(math.ceil(dt.timestamp() / factor) * factor)
          for dt in at_list
      ]
  ```
  (from `git show 9bed32543947`, "[ADD] base: basic coalescing to cron
  triggers", closes odoo/odoo#249492).
- `odoo/odoo/addons/base/models/ir_cron.py:37` —
  `MIN_TIME_PER_JOB = 120  # seconds` (was `10`). git show `a4ae2ed89587`,
  "[FIX] base: allocated time for cron jobs bumped to 120s": "The existing
  scheduler has a hardcoded value MIN_RUNS_PER_JOB after which it tell the
  cron job to stop. That value is set to 10s, which is must too low when the
  cron has some setup to do or remote connections to establish."
- `odoo/odoo/addons/base/tests/test_ir_cron.py:745-783` — dedicated
  coalescing tests: `test_cron_trigger_coalesce_instant`,
  `test_cron_trigger_coalesce_explicit`, `test_cron_trigger_coalesce_same_window`,
  `test_cron_trigger_coalesce_different_windows`.

**Why it matters**: For a Sentry report showing a cron firing far more often
than its configured interval (looks like "trigger storm" or reentrancy bug),
a stale LLM won't know `_trigger(coalesce=N)` exists as the idiomatic fix
(reduce wakeup frequency without changing the underlying record-driven
trigger logic) — it might instead add manual debouncing/locking code. And for
a job that appears to timeout consistently around ~10s of work, the LLM might
assume that's still the platform floor and misdiagnose "the job is being
killed too early" as unfixable at the application level, not realizing 19.4
already grants 120s minimum and the real timeout ceiling lies elsewhere
(`MAX_FAIL_TIME`, worker `--limit-time-cpu`, etc.).

**Confidence**: high

---

## 8. `ir.actions.report` PDF rendering is now a pluggable multi-engine dispatch, not hardcoded wkhtmltopdf

**Claim**: PDF report generation goes through `_get_pdf_engine()` /
`_run_pdf_engine(engine_name, ...)` indirection rather than calling a
wkhtmltopdf-specific function directly. The engine name is resolved from
either the report's own `report_type` (e.g. `qweb-pdf-<engine>`) or a system
parameter `report.pdf_engine_default`, defaulting to `wkhtmltopdf` only as a
fallback. A separate installable module, `base_report_paper_muncher`
(present in this checkout at `odoo/addons/base_report_paper_muncher/`),
supplies an alternative engine ("paper muncher") that overrides
`ir_actions_report.py` to plug into this same dispatch — implying
wkhtmltopdf is being displaced as *the* engine and becoming *an* engine.

**Evidence**:
- `odoo/odoo/addons/base/models/ir_actions_report.py:769-780` —
  ```
  def _get_pdf_engine(self, report=None, default_engine='wkhtmltopdf') -> str:
      """Resolve the PDF engine name from report settings and db's fallback.
      ...
      """
      report_type = (getattr(report, 'report_type', '') or '').lower()
      engine_name = (
          report_type.removeprefix('qweb-pdf-')
          if report_type.startswith('qweb-pdf-')
          else self.env['ir.config_parameter'].sudo().get_str('report.pdf_engine_default') or 'html'
      )
      return engine_name if engine_name and engine_name != 'html' else default_engine
  ```
- `odoo/odoo/addons/base/models/ir_actions_report.py:794` — `def
  _run_pdf_engine(self, engine_name: str, html: str, report_ref: ..., ...)`
  (dispatches by `engine_name`).
- `odoo/odoo/addons/base_report_paper_muncher/models/ir_actions_report.py` and
  `odoo/odoo/addons/base_report_paper_muncher/paper_muncher.py` — a whole
  second module overriding the same report model to add the paper-muncher
  engine.
- git log: `d3b4294a56a5 [REF] base: modularize the reporting engines` on
  `ir_actions_report.py` — the commit that introduced this indirection,
  preceded by many wkhtmltopdf-specific fixes (`b85e1aae6734 [REM] *: legacy
  PyPDF API uses`, `191f25d36ba7 [REF] base: make it possible to patch
  wkhtmltopdf calls`, etc.) that suggest active churn in this exact area.

**Why it matters**: A Sentry crash inside PDF report generation (garbled
PDF, merge failure, missing binary, timeout) might stem from whichever
engine is actually configured via `report.pdf_engine_default` — not
necessarily wkhtmltopdf. A stale LLM patching "the wkhtmltopdf call" (e.g.
editing `_run_wkhtmltopdf`-style code, tweaking command-line flags, or
adding workarounds for wkhtmltopdf-specific bugs referenced in old GitHub
issues) could be fixing the wrong engine entirely if the deployment has
`base_report_paper_muncher` installed and configured as default, or could
break the wkhtmltopdf path while leaving the actual configured engine
(paper muncher) untouched.

**Confidence**: medium (the dispatch mechanism and paper_muncher module are
solidly confirmed; how commonly paper_muncher is actually the *default*
engine in a given SaaS-19.4 deployment vs. still being wkhtmltopdf was not
verified — the code defaults to wkhtmltopdf unless the config parameter is
set).

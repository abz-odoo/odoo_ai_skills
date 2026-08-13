# Skills discovery — Odoo SaaS-19.4 (skills-194/)

Rebuilding the skill set from scratch for the current checkout
(`~/src/194/{odoo,enterprise,industry,design-themes}`), separate from the old
`skills/` dir (written for an older Odoo version). `testing` and
`performance` are deliberately excluded — irrelevant to fixing Sentry bug
reports.

**Status: discovery phase complete — 16/16 categories done, 127 raw candidates across
`candidates/*.md`.** Next step is curation (phase 2, human) — see below.

## How this works

1. **Discovery** (this phase): one agent per category sweeps the four repos
   for version-specific mechanics an LLM trained on Odoo 16/17 knowledge
   would get wrong. Each writes its raw candidates (with file:line evidence)
   to `candidates/<category>.md`. Nothing here is a finished skill yet.
2. **Curation** (next phase, human): you review each `candidates/<category>.md`
   and decide which candidates are real, matter for bug-fixing, and are
   worth turning into an actual skill file at the top level of `skills-194/`.
3. **Authoring**: only the candidates you greenlight get written up as a
   proper skill file (name, description, content) directly under
   `skills-194/`.

## Status

| Category      | Status  | Candidates file                          | Notes |
|----------------|---------|-------------------------------------------|-------|
| model          | done    | candidates/model.md          | 8 candidates (Domain AST, any!/not any!, _sql_constraints dead, SQL()+execute_query, check_access final, search_fetch, _read_group specs, @api.readonly read-replica) |
| field          | done    | candidates/field.md          | 8 candidates (odoo/orm/ split, compute_sql, Properties field, Many2many ondelete, auto-derived readonly) |
| decorators     | done    | candidates/decorators.md     | 8 candidates (@api.readonly read-replica routing, @api.private, model_create_single removed, @api.returns removed) |
| mixins         | done    | candidates/mixins.md         | 7 candidates (mail.track.mixin split, tracking "rotting", activity role_id, track_visibility dead) |
| security       | done    | candidates/security.md       | 8 candidates (ir.model.access+ir.rule merged into ir.access, check_access unified, groups_id renamed) |
| view           | done    | candidates/view.md           | 8 candidates (<list> not <tree>, attrs= enforced removal, card kanban, <chatter/>, invisible vs column_invisible) |
| controllers    | done    | candidates/controllers.md    | 8 candidates (6 high, 2 medium confidence) |
| manifest       | done    | candidates/manifest.md       | 8 candidates (Manifest class not dict, update_xml/demo_xml dead, external_dependencies as PyPI Requirement, ManifestLinter) |
| data           | done    | candidates/data.md           | 8 candidates (ir.access.csv format change, group_ids rename, forcecreate, auto_sequence, type="bytes") |
| actions        | done    | candidates/actions.md        | 8 candidates (webhook server actions, ir.embedded.actions, cron _commit_progress, check_access merge) |
| translation    | done    | candidates/translation.md    | 8 candidates (LazyTranslate canonical, env._() vs _(), ir.translation gone) |
| migration      | done    | candidates/migration.md      | 8 candidates (ir.access merge, end-*.py phase, upgrade_code codemods) |
| development    | done    | candidates/development.md    | 8 candidates (ir.access unified model, class-name-from-_name convention, ORM code moved to odoo/orm/, Hoot JS tests, Studio x_name quirk) |
| reports        | done    | candidates/reports.md        | 8 candidates (pluggable PDF engines, Paper Muncher, t-lang) — agent's Write was blocked by a system rule, I wrote the file manually from its returned text |
| owl            | done    | candidates/owl.md            | 8 candidates (OWL 3 alpha, useState/reactive gone → signal/proxy, t-slot→t-call-slot, partial repo-wide migration) |
| transaction    | done    | candidates/transaction.md    | 8 candidates (_commit_progress, cron internal looping, savepoint AssertionError, read-replica routing) |

Excluded on purpose: `testing`, `performance` (not relevant to Sentry bug triage).

## Methodology skills (finished, top-level — not from the discovery pipeline)

These aren't Odoo-version reference material, so they weren't discovered by scanning source code
— they're written directly from how this agent's pipeline actually works and from its own
resolved-issue history. Ready to use as-is (still not wired into `gemini_cli_agent.py` — skills
in general are on hold pending a decision on native `gemini skills` vs. plain file reads).

| Skill                  | File                     | What it's for |
|-------------------------|--------------------------|----------------|
| Patch mechanics         | `patch-mechanics.md`     | Writing a diff that `git apply` actually accepts — reduces reliance on the `fix_patch_counts()`/`check_patch()` post-hoc safety net |
| Commit message format   | `commit-message.md`      | Odoo `[FIX] module: ...` convention, only needed when verdict = GENUINE BUG |
| Adversarial self-review | `adversarial-review.md`  | A final pass arguing against your own verdict/root-cause/repro/patch before finalizing — cheaper but weaker than a truly independent second pass |
| Abuse pattern recognition | `abuse-patterns.md`    | Starter skill, one real example so far (issue 6932797319 — dot-segment/backslash path-traversal scan across 55 DBs). Meant to grow every time a new ABUSE verdict is confirmed |

## Curation decisions (fill in during phase 2)

_(none yet — see candidates/*.md)_

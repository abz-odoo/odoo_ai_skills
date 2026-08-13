# Abuse/attack pattern recognition — this SaaS platform specifically

**Status: starter skill, one real example.** Unlike the reference skills discovered by scanning
Odoo source, this one has to be built from your own resolved Sentry history — code doesn't tell
you what a real scan looks like, only past triage does. Right now `results/` only has one issue
genuinely resolved as ABUSE/ATTACK ATTEMPT. Add a new dated entry below every time another one is
confirmed, so this grows into an actual corpus instead of staying a single anecdote.

## Confirmed case: issue 6932797319 (dot-segment / encoded-backslash path traversal probe)

**Signal that made this unambiguous**: 3221 events across **55 different customer databases**
and **17 different servers**, all hitting the exact same malformed URL shape. A real user's bug
is tied to a specific tenant's data/configuration; indiscriminate fan-out across dozens of
unrelated databases with byte-for-byte the same crafted input is a scanner signature, not a
coincidence of many customers hitting the same legitimate bug.

**The payload**: `/%5C%5Cd261ab72wzzsk0.cloudfront.net/../css`
- `%5C%5C` decodes to `\\` — a backslash-backslash prefix, a classic trick to make a path look
  like a protocol-relative URL pointing at an attacker-controlled host
  (`d261ab72wzzsk0.cloudfront.net`), probing for open-redirect / SSRF-style URL-handling bugs.
- `/../` — a dot-segment, probing for path traversal.
- Crashed in `odoo.tools.urls.urljoin` via `Website._get_canonical_url` /
  `_is_canonical_url` (`ValueError: Dot segments are not allowed`).

**Why a real user couldn't produce this**: browsers normalize dot-segments and this backslash
trick client-side before the request ever leaves the browser. The only ways to send the raw,
unnormalized bytes are a manual crafted request or an automated tool — never a click/navigation
in the actual Odoo web client.

## Generalizable signals (apply even without a matching prior case)

These aren't mined from this project's history yet, but they're the same reasoning applied
forward — treat them as a starting checklist, not a substitute for checking the actual evidence
in each new event:

- **Multi-tenant fan-out**: the same crash, with the same or near-identical request shape, across
  many unrelated databases/servers in the issue's own "Affected DBs"/"Affected servers" stats is
  strong evidence of automated scanning regardless of what the payload looks like — check this
  stat before reading the stacktrace, not after.
- **Byte sequences a browser wouldn't send unprompted**: encoded traversal (`%2e%2e`, `%5c`),
  null bytes, protocol-relative tricks, mismatched/duplicated encoding layers.
- **Paths targeting things that don't exist in Odoo's own URL space**: probes for
  `/etc/passwd`, `wp-admin`, `.env`, `.git/config`, PHP/ASP file extensions — Odoo doesn't serve
  any of these, so a request for them is definitionally not a normal-usage bug.
- **Direct calls to internal RPC endpoints with malformed/missing parameters** that no rendered
  page in the web client would ever construct that way (see the core triage prompt's Category 1
  for the general rule — this file is about payload-shape recognition specifically, not the
  overall category boundary).

## How to extend this file

When a future issue gets a confirmed ABUSE/ATTACK ATTEMPT verdict with a distinctive payload
shape (not just "another 404 scan"), add a new `## Confirmed case: issue <id> (...)` section above
following the same structure: the fan-out/statistics signal, the payload, and why a legitimate
user couldn't have produced it. Don't add a case you're not fully confident about — a wrong entry
here actively teaches the wrong lesson to every future triage.

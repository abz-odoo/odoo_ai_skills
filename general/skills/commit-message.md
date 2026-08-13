# Commit message format (Odoo convention)

Only relevant once the triage verdict is GENUINE BUG and a patch has been produced — skip this
entirely for ABUSE/ATTACK ATTEMPT, EXPECTED EXCEPTION, USER CUSTOMIZATION FAULT, or OUT OF SCOPE
verdicts, since no commit is being written for those.

## Structure

```
[TAG] module: short description
<blank line>
<body>
<blank line>
Steps to reproduce:
1. ...
<blank line>
sentry-<ISSUE_ID>
```

## Line 1 — tag + summary

- Valid tags: `FIX`, `IMP`, `ADD`, `REF`, `REV`, `MERGE`, `CLA`, `MOV`, `REL`, `TEST`/`TESTS`. For
  a Sentry bug fix this is virtually always `FIX`.
- Format: `[FIX] module: short description` — imperative mood, max 72 characters total.
- `module` is the technical module name (the addon directory name), not a human-readable label —
  e.g. `account`, `web_editor`, `payment_authorize`, not "Accounting" or "Website Editor".
- Real examples:
  - `[FIX] web_editor: don't fail on whitespace-only html field`
  - `[FIX] payment_authorize: fix bill address computation`

## Body — pick Structure A or B, don't mix

**Structure A — Issue / Cause / Solution** (default, use this unless Before/After reads more
naturally for the specific bug):
```
Issue:
<one or two sentences: what the user-visible symptom is>

Cause:
<the technical root cause — name the actual mechanism, not just "a value was None">

Solution:
<what the patch does and why that's the right fix, not just a guard>
```

**Structure B — Before / After** (use when the change is best understood as a before/after
behavior contrast, e.g. a computation that returned the wrong value rather than crashing):
```
Before:
<what happened>

After:
<what happens now>
```

## Steps to reproduce

Reuse the same reproduction steps written for section 2 of the investigation output — numbered,
UI-only, no RPC calls, no shell, no SQL. Don't write a second, different version here; the commit
message's repro steps and the investigation's repro steps must be the same steps.

## Trailer

- Last line, after a blank line: `sentry-<ISSUE_ID>` using the exact issue ID from the user
  message — not a placeholder, not reformatted.
- If this trailer is ever missing from a generated message, the caller appends it automatically,
  but write it yourself — don't rely on that fallback.

## Full example

```
[FIX] account: prevent crash when invoice line has no account

Issue:
Saving a draft invoice containing a product whose default income
account has been archived raises an AttributeError.

Cause:
_get_default_account() returns an empty recordset when the account
is archived, and the caller accesses .id on it without guarding.

Solution:
Add a fallback to the company's default account when the product
account resolves to an empty recordset.

Steps to reproduce:
1. Go to Accounting -> Configuration -> Chart of Accounts
2. Find the income account set on product 'All / Saleable / Service' (e.g. 400000 Sales)
3. Open it and click Archive -> confirm
4. Go to Accounting -> Customers -> Invoices -> New
5. Set Customer to any partner
6. In the invoice lines, add the product 'All / Saleable / Service'
7. Click Save  <- crash: AttributeError on the archived account recordset

sentry-7332943074
```

## Anti-examples (never do this)

- `[FIX] fixed bug in invoice` — no module name, not imperative, no useful description.
- Mixing Structure A and B (`Issue:` followed by `After:`).
- Steps to reproduce that differ from the ones in the investigation output.
- Omitting the blank line before the `sentry-<id>` trailer (breaks the convention other tooling
  may parse this against).

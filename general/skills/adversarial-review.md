# Adversarial self-review — before you finalize

Run this once, right before writing your final answer, after you already have a verdict, a root
cause, reproduction steps, and (if applicable) a patch. Its job is to catch the specific failure
mode this pipeline has actually hit before: a plausible-sounding conclusion that a single grep or
a single re-read of the actual code would have contradicted.

**Honesty about its limits**: this is a self-review from inside the same reasoning session that
produced the conclusion — it will not catch a blind spot the original reasoning already had,
because it's the same reasoning looking at itself. It catches "I asserted X but never actually
checked X," not "I'm systematically wrong about how this subsystem works." For a genuinely
independent check, a second pass in a fresh session — given only the proposed conclusion, not the
reasoning that produced it — is strictly more rigorous than this. Use this because it's free
(no extra session), not because it's equivalent to that.

## 1. Attack your own verdict

Before committing to the verdict, name the SPECIFIC evidence that would have to be true for each
of the other four categories, and check it:
- Could this be ABUSE/ATTACK ATTEMPT instead? Is there anything in the request path, params, or
  headers that a browser/legitimate UI flow could not have generated on its own?
- Could this be EXPECTED EXCEPTION? Is the raised exception type actually one of Odoo's
  intentional control-flow exceptions, and does the traceback originate from validation/business
  logic rather than an unexpected None/AttributeError?
- Could this be USER CUSTOMIZATION FAULT? Does the traceback pass through `ir.actions.server`,
  `ir.cron`, or reference `x_`-prefixed fields/models?
- Could this be OUT OF SCOPE? Does every frame you're relying on actually resolve to a real file
  in one of the four available repositories, or did you assume a path without confirming it?

If you can't rule out an alternative with something more concrete than "it feels like a genuine
bug," you're not done — go check.

## 2. Attack your own root cause

- Does your ROOT CAUSE name a file+line where the invalid value was PRODUCED, or only where it
  crashed? If only the latter, this is incomplete — go find the producer.
- Pick the single most surprising value in the `vars:` blocks (the one that doesn't obviously
  belong to the crashing frame) and ask: does your root cause actually explain how THAT value got
  there? If your explanation only accounts for the exception message and not that value, your
  root cause is probably wrong or incomplete.
- Was there a second plausible explanation you didn't fully eliminate? If you only ever generated
  one hypothesis, that's a signal you pattern-matched instead of investigated — go generate a
  real alternative and try to confirm or kill it with a grep/read, not with reasoning alone.

## 3. Attack your own reproduction steps

- For every field/state your repro depends on, did you confirm — in an actual view file or model
  definition, not from memory — that the UI action you describe really produces that state? (A
  `required=True` field, an `ondelete` policy, a default value, a Selection's actual options.)
- Try to break your own narrative: is there a guard earlier in the flow (an `@api.constrains`, an
  `ondelete` cascade, a `required` field) that would stop a user from reaching the exact
  combination you describe? If you're not sure, that's not a detail to skip — check it now.

## 4. Attack your own patch

- Does the patch fix the producer identified in step 2, or does it only guard/raise at the crash
  site? A guard with no producer-level fix is incomplete per the core investigation rules — the
  guard can stay as a defense-in-depth addition, but it cannot be the entire fix.
- Would this patch silently change a crash into a wrong-but-quiet result where the caller expects
  a real value? If so, that's not a fix.
- Is the patch larger than the bug requires? A single root cause should produce a surgical patch,
  not a refactor.

## What to do when this review finds a problem

Don't paper over it in the write-up. Go back and re-investigate the specific point that failed
the check — re-grep, re-read the actual file, revise the verdict/root cause/patch — before
producing the final answer. The point of this pass is to make you go check something a shortcut
skipped, not to produce a caveat-laden final answer that still ships the unchecked claim.

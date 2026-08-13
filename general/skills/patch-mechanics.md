# Patch mechanics — writing a diff that actually applies

Why this exists: `gemini_cli_agent.py` already runs every generated diff through
`fix_patch_counts()` (recomputes `@@` hunk headers) and `check_patch()` (`git apply --check`)
before trusting it. Those exist because models routinely get the mechanical details of a diff
wrong even when the actual code fix is correct — this skill's job is to reduce how often that
safety net has to catch something, not replace it.

## The rules, in order of how often they're violated

1. **Context and removed lines must be copied verbatim from the file you actually read** —
   not retyped from memory, not "cleaned up" indentation, not paraphrased. A single character of
   drift (a tab vs. spaces, trailing whitespace, a different quote style) makes the hunk fail to
   match at apply time even though the header counts are correct. If you read the file with a
   tool, copy the exact bytes of every unchanged and removed line from that tool output.

2. **Hunk header counts are arithmetic, not estimates**:
   `@@ -oldStart,oldCount +newStart,newCount @@`
   - `oldCount` = number of context lines (no prefix) + number of removed lines (`-`) in that hunk.
   - `newCount` = number of context lines (no prefix) + number of added lines (`+`) in that hunk.
   - Count them after writing the hunk body, don't guess them before. This is the single most
     common failure mode this pipeline has seen (it's exactly what `fix_patch_counts()` corrects
     mechanically) — do the arithmetic yourself instead of relying on that fallback.

3. **One `--- a/<path>` / `+++ b/<path>` pair per file, never mixed** — if the fix spans two
   files, that's two separate header pairs, each followed by its own `@@` hunk(s). Never let a
   hunk from file B appear under file A's header.

4. **Include enough surrounding context to survive minor drift, but no more than needed** —
   typically 3 lines of unchanged context above and below the change is enough for `git apply` to
   locate the hunk. Don't pad hunks with unrelated lines "just in case," and never fold in an
   unrelated whitespace/formatting change alongside the real fix — that's exactly the kind of
   noise `git apply --check` and a human reviewer both have to untangle from the actual fix.

5. **Before finalizing, re-derive the hunk from what you read, and check it against what you
   wrote** — for every `-` line, confirm it exists at that position in the file content you
   actually read this session. For every context line, same check. This is a final mechanical
   check, distinct from the adversarial content review in `adversarial-review.md` — this one is
   purely "does this diff match the bytes of the file," not "is this the right fix."

## What NOT to do

- Don't invent line numbers for the `@@` header without counting.
- Don't "fix" unrelated formatting in the same hunk as the real change.
- Don't split one logical change across multiple disconnected hunks in the same file if a single
  hunk with full context would do — it makes the patch harder to review and more fragile to
  apply.

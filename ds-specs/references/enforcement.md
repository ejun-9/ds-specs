# Enforcement — from editor to deploy

An audit nobody runs is documentation. Enforcement is what makes it a system.

Five gates, escalating. Autofix is allowed at gates 1–3 and **forbidden** at gates 4–5.

| Gate | When | Behavior | Autofix |
|---|---|---|---|
| 1. Session | LLM starts work | `CLAUDE.md` points at specs + tokens | prevention |
| 2. Local | On demand | `npm run audit:tokens -- --fix` | yes, exact matches only |
| 3. Pre-commit | `git commit` | Blocks commit, offers `--fix` | prompted |
| 4. CI / PR | Push, pull request | Fails the check, blocks merge | **never** |
| 5. Deploy | Release pipeline | Fails the build, blocks deploy | **never** |

## Why autofix stops at gate 3

The token mapping is inference, not fact.

- **Exact match** — `#1868DB` and `--color-link` is `#1868DB`. Safe. Autofixable.
- **Near miss** — `#1868DA`, one digit off. Is it a typo or a deliberate one-off? A script cannot know. Report it, flag it as a near miss, never rewrite it.
- **Structural** — a raw `16px` in `padding`. Probably `--space-200`. "Probably" is not good enough to run unattended against a release branch.

Autofixing in CI also means either committing to a branch unreviewed, or fixing inside the runner and deploying artifacts that don't match the repo. And a pipeline that silently cleans up teaches the team nothing — violations accumulate forever because nobody ever sees one.

**Rule: autofix requires a human at a terminal who can look at the diff.**

## Gate 2 — the `--fix` flag

Add to the audit script. Exact matches only, with a mandatory summary:

```js
const FIX = process.argv.includes('--fix');

function attemptFix(file, violations) {
  const exact = violations.filter(v => v.matchType === 'exact');
  const skipped = violations.filter(v => v.matchType !== 'exact');
  if (!FIX || !exact.length) return { fixed: 0, skipped: violations.length };

  let src = fs.readFileSync(file, 'utf8');
  for (const v of exact) src = src.split(v.raw).join(`var(${v.token})`);
  fs.writeFileSync(file, src);
  return { fixed: exact.length, skipped: skipped.length };
}
```

Output must state what it refused to touch:

```
=== Autofix ===
Fixed:   14 exact matches across 6 files
Skipped:  3 near misses — review by hand:
  src/Nav.css L42  #1868DA  (closest: --color-link #1868DB, off by 1)
  src/Card.css L18 15px     (closest: --space-200 16px)
  src/Modal.css L7 z-index: 9001  (no token — add one)

Review the diff before committing.
```

A near miss is a **higher-severity finding than an exact match**, not a lower one. An exact match is laziness; a near miss is either a bug or an undocumented decision.

## Gate 3 — pre-commit

```bash
#!/bin/sh
# .husky/pre-commit
npm run audit:tokens || {
  echo ""
  echo "Design system audit failed. Fix before committing:"
  echo "  npm run audit:tokens -- --fix    # exact matches"
  echo "  npm run audit:tokens             # see what's left"
  echo ""
  echo "Legitimate exception? Add above the line:"
  echo "  /* audit-ok: <reason> */"
  exit 1
}
```

## Gate 4 — CI, blocks the merge

```yaml
# .github/workflows/design-system.yml
name: Design System
on: [pull_request, push]

jobs:
  tokens:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - name: Token audit
        run: npm run audit:tokens
      - name: Spec coverage
        run: npm run audit:specs
```

Make it a **required status check** in branch protection. A check that can be merged past is a suggestion.

Second script, `audit:specs`, catches the other drift direction: a component that exists in `src/components/` with no spec file. Warning for a week after the file appears, error after that — new components get a grace period, abandoned ones don't.

## Gate 5 — deploy

The last gate exists because CI can be bypassed: hotfix branches, manual dispatch, someone with admin rights merging past a red check at 2am.

```yaml
deploy:
  needs: [tokens]
  if: github.ref == 'refs/heads/main'
  steps:
    - name: Design system gate
      run: |
        npm run audit:tokens || {
          echo "::error::Deploy blocked — hardcoded values in the design system."
          echo "::error::Run: npm run audit:tokens -- --fix"
          exit 1
        }
```

Failure message must name the fix command. A red X with no next step gets the gate deleted within a month.

### Break-glass

Real incidents happen. Provide an override that is loud rather than convenient:

```yaml
env:
  DS_AUDIT_OVERRIDE: ${{ inputs.override_audit }}
```

When set, the gate passes but **must**: print every violation to the deploy log, post the count to the team channel, and open an issue titled `Design system debt from <sha>` assigned to whoever triggered it.

An override with no paper trail becomes the default path. An override that files an issue with your name on it gets used only when it's actually warranted.

## The documentation half

The gates get generated as a spec file, not just as config. Config explains *what* runs; the spec explains *why*, and it's the file a new contributor or an LLM reads.

Write `specs/foundations/enforcement.md`:

```markdown
# Enforcement

## Principle
Every visual value flows through a token. This is enforced automatically,
not by review, because review misses it — a reviewer sees `padding: 12px`
and reads it as reasonable, because individually it is.

## Gates
| Gate | Trigger | Blocks |
|---|---|---|
| Pre-commit | `git commit` | The commit |
| CI | Pull request | The merge |
| Deploy | Push to main | The release |

## What fails a build
- Hardcoded color in any style file or arbitrary Tailwind color
- Raw spacing in padding, margin, or gap
- Hardcoded border-radius or font-size
- Raw z-index integer
- Component in src/ with no spec file (after its grace period)

## What warns but passes
- Transition durations and box-shadow values
- Anything carrying `/* audit-ok: <reason> */`

## Fixing a failure
1. `npm run audit:tokens -- --fix` — resolves exact matches
2. Review the diff. Autofix never touches near misses.
3. Anything left is a real decision: add a token, or add an
   `audit-ok` comment stating why this value is exempt.

## Adding a token
Never add a token just to silence the audit. If the value is genuinely
new, add it to tokens.css AND specs/tokens/token-reference.md AND
push it to Figma so both sides stay joined. A token that exists in
code but not Figma will surface in the next reconcile.

## Override
Deploy can be forced with DS_AUDIT_OVERRIDE. It logs every violation,
notifies the team, and opens an issue in your name. Use it for
incidents, not for deadlines.
```

Include the same gate table in `CLAUDE.md` so the LLM knows a violation will be caught before it writes one.

## Rollout on an existing codebase

A repo with 400 violations cannot turn on gate 5 today. Ratchet instead:

1. Run the audit, record the count, commit it as `.audit-baseline`
2. CI fails only if the count **increases** — new code is clean, old code is grandfathered
3. Fix a category at a time (colors first — highest visibility, easiest to match exactly)
4. Lower the baseline as you go; never raise it
5. When the baseline reaches zero, delete it and switch to hard-fail

Turning on hard-fail against 400 violations on day one gets the gate disabled by Thursday. The ratchet ships.

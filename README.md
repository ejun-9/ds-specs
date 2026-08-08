# ds-specs

A Claude skill that builds design systems an LLM can actually follow — and keeps them in sync with Figma.

An LLM makes 200–300 visual micro-decisions per coding session. Each one is individually reasonable. Together they aren't a design system, they're 200 guesses. Next session, zero memory, 200 *different* guesses. By session ten your prototype looks like three products built by three teams.

`ds-specs` turns every visual decision into a lookup instead of a guess.

Built on [Hardik Pandya's method for LLM-readable design systems](https://hvpandya.com/llm-design-systems) — extended with a two-way Figma bridge and deployment enforcement.

> **Source article:** https://hvpandya.com/llm-design-systems
> **Built with:** Claude Opus 5 (high reasoning effort)

---

## What it does

**Three tiers, each referencing only the tier above it.**

```
specs/
├── foundations/   color, typography, spacing, radius, elevation, motion…
├── tokens/        the master variable map
├── atoms/         ─┐
├── molecules/      ├─ components, 8-section template each
├── organisms/     ─┘
└── patterns/      layout and composition rules
```

**Three layers of token indirection.** Components never touch the upstream layer.

```css
--ds-text: #292A2E;                        /* upstream library */
--color-text: var(--ds-text, #292A2E);     /* your alias */
color: var(--color-text);                  /* components — alias only */
```

**Five enforcement gates.** Session start → local autofix → pre-commit → CI → deploy. Self-healing is allowed only locally, only on exact matches, only with a human reading the diff. CI and deploy block and never rewrite your code.

**A two-way Figma bridge.** Figma variables carry a `codeSyntax` field. Set it to the CSS custom property and the Figma variable and your token become one record seen from two sides — so sync is exact instead of fuzzy name-matching.

---

## Three modes

| Say | Mode | What happens |
|---|---|---|
| "create a design system," "tokenize this," "my prototype is drifting" | **Build** | Spec tree, token layer, audit script, enforcement gates |
| "push my specs to Figma," "generate Figma docs pages from my specs" | **Push** | Emits a build plan, hands off to `figma-generate-library` |
| "sync from Figma," "does Figma match my code" | **Reconcile** | Six-category diff report. Changes nothing. |

**Neither side is canonical.** Reconcile reports and stops. A mismatch is ambiguous — only a human knows whether the designer moved or the developer did.

---

## Install

**Claude Code / Cowork**

```bash
git clone https://github.com/ejun-9/ds-specs.git /tmp/ds-specs-repo
cp -r /tmp/ds-specs-repo/ds-specs ~/.claude/skills/
```

**claude.ai**

Download [`ds-specs.skill`](../../releases/latest) from Releases and upload it in Settings → Capabilities → Skills.

**Keep it current** — add to your Claude settings or project instructions:

```bash
git clone --depth=1 https://github.com/ejun-9/ds-specs.git /tmp/ds-specs-latest \
  && cp -r /tmp/ds-specs-latest/ds-specs ~/.claude/skills/
```

---

## Using it

Name it explicitly the first few times, until you see where it fires on its own:

> use ds-specs to set up a design system for this project

> use ds-specs to reconcile my Figma file against the code

After that it triggers on its own for phrases like the ones in the table above, and for implicit cases — styling work in a repo that already has a `specs/` tree or a `tokens.css`.

**Under-triggering is the normal failure mode.** Short requests ("make this button blue") often won't fire it, even though that's exactly when a token should be enforced. If you find it consistently missing a phrase you use, add that phrase to the `description` field in `SKILL.md` — that field is what activates the skill; the tables in the body only route between modes once it's already loaded.

---

## Figma modes need two more skills

Push and Reconcile call the Figma Plugin API through Figma's own skills rather than reimplementing it:

- `figma-use` — Plugin API syntax
- `figma-generate-library` — variable collections, component sets, variant grids

Both ship with Figma's MCP server. `ds-specs` owns the spec layer, the token layer, and the diff; it never writes to Figma directly. Two skills mutating Figma independently diverge within weeks.

---

## Design decisions worth knowing

**Autofix stops at pre-commit.** The token mapping is inference. An exact hex match is safe. A near miss — `#1868DA` against a `#1868DB` token — is either a typo or a deliberate one-off, and no script can tell. Near misses are reported at *higher* severity than exact matches and never auto-rewritten.

**Only spec what exists.** A small accurate spec layer beats a comprehensive stale one. The skill scales the tree to project size and refuses to generate specs for components you haven't built.

**Ratchet, don't hard-fail.** On an existing codebase with hundreds of violations, the gates start from a recorded baseline and fail only when the count *increases*. Turning on hard-fail against 400 violations gets the gate deleted by Thursday.

**Break-glass leaves a paper trail.** Deploy can be forced, but it logs every violation, notifies the team, and opens an issue in the name of whoever triggered it.

---

## Repo contents

```
ds-specs/
├── SKILL.md                        workflow, mode routing, the seven steps
└── references/
    ├── tokens-css.md               three-layer indirection, theming, naming
    ├── tokens-tailwind.md          @theme, closing the palette, what "hardcoded" means
    ├── spec-templates.md           templates for all three tiers
    ├── audit-scripts.md            working audit implementations, near-miss detection
    ├── enforcement.md              five gates, CI and deploy config, ratcheted rollout
    └── figma-bridge.md             codeSyntax join key, mapping table, reconcile report
```

---

## Credit

The core method — three-tier specs, closed token sets, the audit script, and the framing of LLM failure modes — comes from Hardik Pandya's article:

**https://hvpandya.com/llm-design-systems**

Read it first. It explains the reasoning this skill only encodes. What's added here is the two-way Figma bridge, the five enforcement gates, stack detection for Tailwind, and near-miss classification in the audit.

This skill was authored with **Claude Opus 5** (high reasoning effort).

## License

MIT. See [LICENSE](LICENSE).

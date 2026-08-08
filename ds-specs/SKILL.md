---
name: ds-specs
description: Build and maintain an LLM-readable design system that is also a two-way bridge to Figma — a three-tier spec tree (foundations/tokens → components → patterns), a closed token layer with three-layer indirection, a CI-ready audit script that blocks hardcoded values, and a reconcile report that diffs Figma variables, component sets, variants, and bindings against the specs. Use this skill whenever the user says "create a design system," "set up a design system," "make my design system LLM-readable," "tokenize this," "write specs for my components," "my prototype is drifting," or "the AI keeps picking different colors." ALSO use it for any syncing between Figma and code — "push my specs to Figma," "sync from Figma," "does Figma match my code," "my Figma and code have drifted apart," "generate Figma docs pages from my specs," or any request involving Figma variables, variant sets, component properties, or bound variables alongside a codebase. Works for greenfield and retrofit, Tailwind and plain CSS — detect which and branch. Load `figma-use` and `figma-generate-library` alongside this skill for any Figma operation.
---

# LLM-readable design systems

Based on Hardik Pandya's method (hvpandya.com/llm-design-systems).

## The problem this solves

An LLM makes 200–300 visual micro-decisions per coding session. Each one is individually reasonable. Together they aren't a design system — they're 200 guesses. Next session, zero memory, 200 *different* guesses. By session ten the prototype looks like three products built by three teams.

Three LLM failure modes, three countermeasures:

| Failure | Countermeasure |
|---|---|
| Fabricates plausible values (`padding: 12px`) | **Closed token layer** — pick from a finite named set |
| No cross-session memory | **Spec files** — read at the start of every session |
| Silent drift, upstream and within-session | **Audit script** — exit code 1, runs in CI |

The goal: every visual decision becomes a *lookup*, never a guess. Human taste goes in once. The LLM follows it mechanically after that.

## Pick the mode first

Three entry points. Route before anything else.

| User says | Mode | Go to |
|---|---|---|
| "create a design system," "tokenize this," "my prototype is drifting" | **Build** | Step 0 below |
| "push my specs to Figma," "generate Figma docs pages from my specs" | **Push** | `references/figma-bridge.md` → Direction 1 |
| "sync from Figma," "does Figma match my code," "my Figma and code have drifted apart" | **Reconcile** | `references/figma-bridge.md` → Direction 2 |

Phrases are illustrative, not literal — match on intent. Also trigger on requests that *imply* this work without naming it: adding a component to a project that has a `specs/` tree, picking a color or spacing value when tokens exist, or any styling task in a repo with `tokens.css`. Under-triggering is the common failure; a needless load costs little, a missed one costs a session of drift.

Push and Reconcile both require `figma-use` and `figma-generate-library` loaded alongside this skill. ds-specs owns the spec and token layers; it never calls the Figma Plugin API itself.

**Neither side is canonical.** Reconcile reports a diff and stops. It does not edit specs, does not edit Figma, does not edit tokens.css. A mismatch is ambiguous — only a human knows whether the designer moved or the developer did.

## Step 0: Detect the situation

Before doing anything, determine two axes. Do not ask the user if the codebase answers it.

**Axis 1 — Stack.** Check for `tailwind.config.*`, a `@import "tailwindcss"` / `@tailwind` directive, or heavy `className=` utility usage.
- Tailwind detected → read `references/tokens-tailwind.md`
- Plain CSS / SCSS / CSS Modules / styled-components → read `references/tokens-css.md`
- Both present → tokens live in the Tailwind theme; the CSS audit still runs on any hand-written CSS files

**Axis 2 — Greenfield or retrofit.** Count existing style declarations.
- Meaningful existing styles → **retrofit**: run all seven steps in order, starting with the audit
- Little or no existing code → **greenfield**: skip Step 1, derive tokens from stated taste or an existing reference (a Figma file, a screenshot, a named design system), and write specs *ahead of* the components

State which branch you're taking in one line, then proceed. Don't narrate the detection at length.

## Step 0.5: Scale the scope

Match the spec tree to the project. Over-speccing a five-component prototype produces stale files nobody reads.

| Size | Signal | Spec tree |
|---|---|---|
| **Small** | < 10 components, single-page, throwaway prototype | `foundations/` (color, spacing, typography, radius only) + `tokens/token-reference.md` + one `components/` folder, flat. Audit script still required. |
| **Medium** | 10–30 components, multi-screen product | Full foundations, `tokens/` split by category, `atoms/` + `molecules/`, and `patterns/` for any layout used more than twice |
| **Large** | 30+ components, shipping product, multiple contributors | The full three-tier tree below, plus drift detection and a design review checklist |

Announce the tier you picked and why in one sentence. If the user disagrees they'll say so.

**Only ever spec components that actually exist.** A small accurate spec layer beats a comprehensive stale one. Never generate a spec for a component the project doesn't have "for completeness."

## The hierarchy (non-negotiable)

Three tiers. **Each tier may only reference the tier above it.** That constraint is the entire mechanism — it's what makes lookups terminate instead of circling.

```
your-project/
├── specs/
│   ├── foundations/          # Tier 1 — visual primitives
│   │   ├── color.md
│   │   ├── typography.md
│   │   ├── spacing.md
│   │   ├── radius.md
│   │   ├── elevation.md
│   │   ├── motion.md
│   │   ├── z-index.md
│   │   ├── iconography.md
│   │   ├── accessibility.md
│   │   ├── breakpoints.md
│   │   ├── grid.md
│   │   ├── borders.md
│   │   └── opacity.md
│   ├── tokens/               # Tier 1 — the variable reference
│   │   ├── token-reference.md      # master map: every variable, value, when to use
│   │   ├── color-tokens.md
│   │   ├── spacing-tokens.md
│   │   ├── typography-tokens.md
│   │   ├── elevation-tokens.md
│   │   └── motion-tokens.md
│   ├── atoms/                # Tier 2 — single-purpose elements
│   ├── molecules/            # Tier 2 — composed components
│   ├── organisms/            # Tier 2 — product-specific assemblies
│   └── patterns/             # Tier 3 — layout & composition rules
├── tokens.css                # every variable, three-layer indirection
├── scripts/token-audit.js    # catches hardcoded values, CI-ready
└── CLAUDE.md                 # read at the start of every session
```

What each tier is for:

1. **Foundations + tokens** — define raw values and give them names. What blues exist. What spacing steps exist. What `--color-link` resolves to.
2. **Components** — atoms use tokens. Molecules compose atoms. Organisms assemble molecules. A button knows its padding token, color token, radius token.
3. **Patterns** — how to arrange components on a page. Three-column rules. Spacing between sections. Panel expand/collapse behavior.

The payoff: when the LLM builds a form, it reads `patterns/form-layout.md` for spacing, `molecules/form.md` for structure, `atoms/input.md` for the field, `tokens/spacing-tokens.md` for exact values. Every decision is a lookup.

## The seven steps

### Step 1 — Audit (retrofit only)

Scan every style file. List every hardcoded visual value: hex and rgb/rgba colors, pixel spacing, raw font sizes, font weights, border radii, z-index values, box shadows, transition durations. Group by category. Count totals. Report which files have the most.

That count is the baseline. Flag the worst files explicitly — that's where the most guessing happened, and those are the files to review by hand.

For Tailwind, "hardcoded" means arbitrary values (`p-[13px]`, `text-[#2563EB]`, `w-[347px]`) and off-scale utilities. See `references/tokens-tailwind.md`.

### Step 2 — Token layer

Three layers of indirection, always. Components never touch layer 1.

```css
--ds-text: #292A2E;                     /* L1: upstream design system */
--color-text: var(--ds-text, #292A2E);  /* L2: project alias + fallback */
color: var(--color-text);               /* L3: components — alias only */
```

The alias layer is the protection. Upstream renames a token → you change one line. Dark mode, high contrast, any future theme → resolves through the chain with zero component changes.

Cover at minimum: colors (text, background, link, border, interactive states), spacing (8+ steps on a consistent base grid), typography (families, sizes, weights, line heights), radius, elevation, z-index, motion.

Full patterns and the Tailwind equivalent: `references/tokens-css.md` and `references/tokens-tailwind.md`.

### Step 3 — Spec files

Write the tree from Step 0.5. Every component spec follows the same 8-section template:

1. Metadata (name, category, status)
2. Overview — **when to use and when *not* to use**
3. Anatomy — the parts
4. Tokens used — which variables it references
5. Props/API
6. States — default, hover, active, focus, disabled, error
7. Code example
8. Cross-references — "Uses" and "Used by"

Section 2 is the one that matters most and the one most likely to be written lazily. Source code already shows what was built; specs exist to say how to build the next thing. "When not to use" is knowledge that lives only in a designer's head until someone writes it down.

Templates for foundations, components, and patterns: `references/spec-templates.md`.

Write foundations first and surface them for review before generating component specs. Color and spacing govern everything downstream.

### Step 4 — Audit script

Create `scripts/token-audit.js` (or `.sh`) that:
- scans style files for hardcoded values
- suggests the correct token for each violation
- prints file, line number, violation, suggestion
- returns exit code 1 if any errors found
- distinguishes **errors** (hardcoded colors, spacing) from **warnings** (raw durations, uncommon one-offs)

```
Token Audit
Scanning 28 CSS file(s)...

src/components/Nav.css
  x L42: Hardcoded color #1868DB, use var(--color-link)
  x L78: Raw spacing 12px in padding, use var(--space-150)
  ! L96: Raw duration 0.2s, consider using --motion-* token

=== Summary ===
Files scanned:      28
Files with issues:  0
Errors:             0
Warnings:           0
```

Implementations for both stacks: `references/audit-scripts.md`.

The script needs a `--fix` flag that resolves **exact matches only** and prints what it refused to touch. Never autofix a near miss.

For medium and large projects, also write a design review checklist covering hardcoded values, state coverage, component anatomy, spacing, typography, motion, accessibility, and documented divergences.

### Step 5 — Replace every hardcoded value

Go through every style file. Every `color:`, `background:`, `padding:`, `margin:`, `gap:`, `border-radius:`, `font-size:`, `font-weight:`, `box-shadow:`, `z-index:`, and `transition:` flows through a token. No raw values remain.

Run the audit at the end and confirm zero errors. Report the before/after count.

### Step 6 — Enforcement gates

Read `references/enforcement.md` and generate all of it. This step is not optional and not "wire up CI later" — an audit nobody runs is documentation.

Produce: the `--fix` flag, a pre-commit hook, a CI workflow set as a required status check, a deploy gate with a break-glass override that files an issue, and **`specs/foundations/enforcement.md`** — the spec file explaining why the gates exist and how to clear a failure.

Generate the spec file, not just the config. Config says what runs; the spec says why, and it's what a new contributor or a fresh LLM session actually reads.

On an existing codebase with many violations, ratchet from a baseline count instead of turning on hard-fail immediately. Hard-fail against 400 violations gets the gate deleted by Thursday.

### Step 7 — Session instructions + drift detection

Add to `CLAUDE.md` (or `.cursorrules`, or whatever the project uses):

> Before writing or modifying any UI code, read the relevant spec file in `specs/`. Use only tokens from `tokens.css`. Run `scripts/token-audit.js` before committing. Zero errors required.

For large projects, pin upstream design system package versions and add a sync routine that flags which spec files may need updating when the library ships changes. **Non-blocking** — it reports, it never auto-edits specs.

## Maintenance rules to hand the user

- New component? Write the spec before or while building it, not after.
- Design system update? The sync routine flags it. Update the affected specs.
- New pattern? If someone corrects the same thing in two PR reviews, it's a pattern. Write it down.
- Audit fails? Fix the hardcoded value, not the audit script.

## What not to do

- Don't generate specs for components that don't exist yet
- Don't let a component reference a Layer 1 (`--ds-*`) token directly
- Don't let a spec reference a tier *below* it (a foundation must never mention a specific button)
- Don't soften the audit script to make it pass
- Don't autofix in CI or at deploy — exact matches only, locally, with a human reading the diff
- Don't ship an audit without gates, or gates without `specs/foundations/enforcement.md`
- Don't skip the "when not to use" section — it's the part source code can't provide
- Don't produce the full 64-file tree for a 6-component prototype
- Don't let reconcile auto-resolve a mismatch, in either direction
- Don't push tokens to Figma without setting `codeSyntax` — that silently breaks every future reconcile
- Don't reimplement the Figma write path; hand off to `figma-generate-library`

## Reference files

- `references/tokens-css.md` — three-layer indirection in plain CSS, theming, fallback strategy
- `references/tokens-tailwind.md` — the Tailwind path: `@theme`, config tokens, what "hardcoded" means with utilities
- `references/spec-templates.md` — fill-in templates for foundation, component, and pattern specs
- `references/audit-scripts.md` — working audit script implementations for both stacks
- `references/enforcement.md` — the five gates, autofix boundaries, CI and deploy config, ratcheted rollout
- `references/figma-bridge.md` — the two-way bridge: codeSyntax join key, spec↔Figma mapping table, push plan, reconcile report format

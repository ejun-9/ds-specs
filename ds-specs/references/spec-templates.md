# Spec templates

Three templates, one per tier. Keep them literal — consistency across files is what makes them cheap for an LLM to read at session start.

---

## Tier 1 — Foundation spec

`specs/foundations/<name>.md`

```markdown
# <Foundation name>

## Principle
One paragraph. The rule this foundation encodes and the reasoning behind it.
Example: "All spacing derives from a 4px base grid. Values outside the grid
signal either a missing token or a mistake."

## Scale
| Token | Value | Use for |
|---|---|---|
| `--space-050` | 4px | Icon-to-label gaps, tight inline spacing |
| `--space-100` | 8px | Default gap inside a component |
| `--space-200` | 16px | Padding inside cards and panels |

## Rules
- Bulleted, testable statements.
- "Never use a value not on this scale."
- "Vertical rhythm between sections is --space-600 minimum."

## Do / Don't
| Do | Don't |
|---|---|
| `gap: var(--space-100)` | `gap: 10px` |

## Related
Links to the tokens file and any pattern that depends on this foundation.
```

Foundations must never name a specific component. `spacing.md` describing "the button uses 12px padding" is a tier violation — that belongs in `atoms/button.md`.

---

## Tier 2 — Component spec

`specs/atoms/<name>.md` (or `molecules/`, `organisms/`)

Two parts: a machine-readable `figma:` frontmatter block that generates and diffs against the Figma component set, then the 8-section prose body that Claude reads at session start and that becomes the Figma documentation page.

**Two body templates.** Projects with Storybook use the short template (end of this section): props, variants and working code already live in the stories, so the spec skips them. Projects without Storybook use the full 8-section template, because the spec is the only place an agent can find props and examples.

**Declare states once, in `variantAxes`.** Section 6 describes what each state *looks like* and *means*; the axis list is the single source. If the two disagree, the reconcile report flags it — but don't let it get there.

````markdown
---
component: Button
category: atom
status: stable
source: src/components/Button.tsx
storybook: atoms-button     # Storybook projects only: the story's ID
figma:
  page: "Components / Button"
  variantAxes:
    appearance: [primary, subtle, danger]
    state: [default, hover, pressed, focus, disabled]
  # 3 x 5 = 15 combinations. Adding size: [small, medium] would make 30 —
  # over the limit. Ship size as a second component set instead.
  properties:
    - { name: label,          type: TEXT,           default: "Button" }
    - { name: hasLeadingIcon, type: BOOLEAN,        default: false }
    - { name: leadingIcon,    type: INSTANCE_SWAP,  default: "Icon/Plus" }
  bindings:
    fill:         --color-interactive
    textFill:     --color-text-inverse
    paddingX:     --space-200
    paddingY:     --space-100
    cornerRadius: --radius-100
  autolayout:
    direction: HORIZONTAL
    gap: --space-050
    alignItems: CENTER
---
````

Rules for the block:
- Every value in `bindings` must be a **Layer 2** token (`--color-*`), never Layer 1 (`--ds-*`) and never a raw value. Same rule as CSS.
- Multiply the `variantAxes` lengths. Above 24, split `state` out and document it as interaction behavior instead — a 30-variant set is unmaintainable in Figma.
- `properties` names get an `#id:id` suffix when Figma creates them. Match on the part before `#`.
- Omit the whole `figma:` block for components that intentionally don't exist in Figma. Reconcile will skip rather than report them missing.

Then the prose body:

```markdown
# <Component name>

## 1. Metadata
- **Name:** Button
- **Category:** Atom
- **Status:** Stable | Beta | Deprecated
- **Source:** `src/components/Button.tsx`

## 2. Overview
**What it is.** One or two sentences.

**When to use.**
- Primary action on a page or in a dialog
- Any action that mutates state

**When NOT to use.**
- Navigation between pages — use Link
- Toggling a persistent setting — use Toggle
- More than one primary button in the same view

## 3. Anatomy
1. Container — sets padding, radius, background
2. Leading icon (optional)
3. Label — required, sentence case
4. Trailing icon (optional)

## 4. Tokens used
| Property | Token |
|---|---|
| Background (primary) | `--color-interactive` |
| Text | `--color-text-inverse` |
| Padding | `--space-100` `--space-200` |
| Radius | `--radius-100` |
| Transition | `--motion-fast` |

## 5. Props / API
| Prop | Type | Default | Notes |
|---|---|---|---|
| `appearance` | `primary \| subtle \| danger` | `subtle` | |
| `isDisabled` | `boolean` | `false` | |

## 6. States
| State | Behavior |
|---|---|
| Default | |
| Hover | `--color-interactive-hovered`, no size change |
| Active / pressed | |
| Focus | 2px `--color-focus-ring`, offset 2px, never removed |
| Disabled | `--color-interactive-disabled`, `cursor: not-allowed` |
| Loading | Spinner replaces leading icon, label persists, width locked |
| Error | If applicable |

## 7. Code example
Real, copyable, from this project. Not pseudocode.

## 8. Cross-references
- **Uses:** Icon, Spinner
- **Used by:** Form, ModalDialog, PageHeader
- **See also:** `patterns/form-layout.md`
```

Section 2's "when NOT to use" and section 6's full state coverage are the two most-skipped and most-valuable parts. Source code shows what exists; these describe how to choose.

### Short template (Storybook projects)

Same frontmatter, including `storybook:`. Six sections, none of them a copy of the code:

```markdown
# <Component name>

## 1. Metadata
- **Name:** Button
- **Category:** Atom
- **Status:** Stable | Beta | Deprecated
- **Source:** `src/components/Button.tsx`
- **Storybook:** `atoms-button` (props, variants and story code, read over Storybook's MCP)

## 2. Overview
**What it is.** One or two sentences.

**When to use.**
- Primary action on a page or in a dialog

**When NOT to use.**
- Navigation between pages: use Link

## 3. Anatomy
## 4. Tokens used
## 5. States
## 6. Cross-references
```

Sections 3 to 6 follow the full template. Leave out Props / API and Code example: Storybook builds both from the source, so a hand-written copy can only drift. Keep anything a story can't show, like "all native `<input>` props pass through" or a rule for picking between variants, in Overview or States.

After editing a short spec, run the spec sync from `references/spec-sync.md`. It copies the Overview into the component's doc comment, so an agent gets the "when NOT to use" in the same MCP answer as the code.

---

## Tier 3 — Pattern spec

`specs/patterns/<name>.md`

```markdown
# <Pattern name>

## Problem
What layout or composition question this answers.
"How do form fields, labels, help text, and actions arrange vertically."

## Structure
The arrangement, with spacing tokens between every element.
A small ASCII or nested-list diagram helps here.

## Spacing rules
| Between | Token |
|---|---|
| Label and field | `--space-050` |
| Field and help text | `--space-050` |
| Field group and next group | `--space-300` |
| Last field and action row | `--space-400` |

## Responsive behavior
| Breakpoint | Behavior |
|---|---|
| < 768px | Single column, full-width fields, actions stack |
| >= 768px | Two column, actions right-aligned |

## Components used
Links to the Tier 2 specs this pattern composes.

## Edge cases
- Validation errors: message appears below field, container grows, no layout shift above
- Long labels: wrap, never truncate
- Empty state: what renders when there's nothing

## Anti-patterns
- Don't nest this pattern inside itself
- Don't override the spacing tokens locally
```

---

## Token reference

`specs/tokens/token-reference.md` — the master map. One row per variable, no exceptions.

```markdown
| Token | Resolves to | Layer 1 source | Use for |
|---|---|---|---|
| `--color-link` | `#1868DB` | `--ds-link` | Inline text links, nav items |
| `--space-200` | `16px` | `--ds-space-200` | Card and panel padding |
```

This file is what the audit script reads to generate its "use X instead" suggestions. Keep it exact — a wrong row produces a wrong suggestion, which is worse than no suggestion.

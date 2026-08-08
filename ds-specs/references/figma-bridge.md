# The Figma bridge

Specs are a **shared format**, not a generated artifact. Neither Figma nor the repo is canonical. Every sync reports a diff; a human resolves it.

## The join key

Figma variables carry a `codeSyntax` field. Set it to the CSS custom property and the Figma variable and the CSS token become **the same record seen from two sides**:

```javascript
v.setVariableCodeSyntax('WEB', 'var(--color-link)');
```

This is the entire bridge. Without it you are matching on names, and names diverge (`Color/Text/Link` vs `--color-link` vs `text-link`). With it, matching is exact and reconcile is mechanical.

**Always store the original CSS variable name verbatim.** Never derive it from the Figma variable name. If the codebase says `--sds-color-background-brand-default`, that exact string goes in `codeSyntax`.

Any variable with no `codeSyntax` is unmappable and gets reported as an orphan.

## Delegation — do not reimplement the write path

ds-specs owns the spec layer, the CSS/Tailwind token layer, and the audit. It does **not** call the Figma Plugin API directly.

| Task | Owner |
|---|---|
| Spec tree, tokens.css, audit script | ds-specs |
| Emitting a build plan from specs | ds-specs |
| Creating collections, variables, components, variants | `figma-generate-library` + `figma-use` |
| Reading Figma state back | `figma-generate-library` discovery scripts |
| Diffing and reporting | ds-specs |

Load `figma-use` and `figma-generate-library` alongside this skill for any Figma operation. Two skills mutating Figma independently will diverge within weeks.

## Mapping table

| Spec concept | Figma construct | Code construct |
|---|---|---|
| Token (Layer 1, `--ds-*`) | Primitive variable, `scopes: []` | `:root` custom property |
| Token (Layer 2, alias) | Semantic variable, aliased to primitive, scoped | `var(--ds-x, fallback)` |
| Theme | Variable collection **mode** | `[data-theme]` selector |
| Foundation spec | Documentation page + swatch/specimen frames | `specs/foundations/*.md` |
| Component spec | Component set | Component file |
| `variantAxes` | Variant properties (grid axes) | Props with enum values |
| `properties` | `addComponentProperty` — TEXT / BOOLEAN / INSTANCE_SWAP | Props |
| `bindings` | `setBoundVariable` on fills, padding, radius | `var(--token)` in CSS |
| `anatomy` | Auto-layout node structure | JSX structure |
| Prose sections (when-not-to-use, edge cases) | Documentation page text | Spec markdown only |

**Scope rules that matter for round-tripping:** primitives get `scopes: []` so designers never see them in pickers — the picker should only offer semantic tokens, exactly mirroring the rule that components never reference Layer 1. Semantic variables get narrow scopes (`FRAME_FILL`, `SHAPE_FILL`, `GAP`, `CORNER_RADIUS`). `ALL_SCOPES` on a semantic variable is a finding, not a default. BOOLEAN variables support no scopes at all.

## Direction 1: spec → Figma

1. Read the spec tree. Parse the `figma:` frontmatter block from every component spec.
2. Emit a **build plan** and show it before any mutation:
   - collections and modes to create
   - primitives, then semantics with alias targets, scopes, and `codeSyntax`
   - per component: page name, variant axis matrix, property definitions, bindings
   - documentation pages derived from prose sections
3. Hand the plan to `figma-generate-library`. Its phase structure, checklists, and validation apply unchanged. Tokens before components, always.
4. Documentation pages come from the prose: Overview → page description, Anatomy → numbered callouts, When-not-to-use → a usage panel, Cross-references → links.

Variant axis count is a trap. Three axes of 3 × 2 × 5 is 30 components in one set. Warn above 24 and suggest splitting `state` out as a documented interaction rather than a variant axis.

## Direction 2: Figma → spec

Read local variable collections and component sets, then diff. Never auto-edit a spec file.

```javascript
// Variables, with the join key
const colls = await figma.variables.getLocalVariableCollectionsAsync();
const allVars = await figma.variables.getLocalVariablesAsync();
return {
  collections: colls.map(c => ({ name: c.name, modes: c.modes, defaultModeId: c.defaultModeId })),
  variables: allVars.map(v => ({
    name: v.name,
    resolvedType: v.resolvedType,
    scopes: v.scopes,
    codeSyntax: v.codeSyntax,          // <- match on this
    valuesByMode: v.valuesByMode
  }))
};
```

```javascript
// Component sets and their properties
const sets = figma.currentPage.findAllWithCriteria({ types: ['COMPONENT_SET'] });
return sets.map(cs => ({
  name: cs.name,
  variantCount: cs.children.length,
  properties: Object.entries(cs.componentPropertyDefinitions).map(([key, def]) => ({
    key, type: def.type, defaultValue: def.defaultValue, variantOptions: def.variantOptions
  }))
}));
```

Note: `addComponentProperty` returns a key with an `#id:id` suffix for TEXT/BOOLEAN/INSTANCE_SWAP. Match on the portion before `#`, not the raw key.

## The reconcile report

The primary output of this skill in bidirectional mode. Six categories, always in this order:

```
Design System Reconcile
Figma: "Lune DS" · Repo: specs/ + tokens.css

=== VALUE MISMATCH (3) ===
  --color-link          Figma #1868DB   Code #2563EB
  --space-300           Figma 24px      Code 20px
  --radius-200          Figma 8px       Code 6px

=== IN FIGMA, NOT IN CODE (2) ===
  --color-surface-sunken   #F4F5F7   no matching CSS token
  Component set "Toast"    5 variants · no spec file

=== IN CODE, NOT IN FIGMA (1) ===
  --color-warning-subtle   #FFF4E5   no Figma variable

=== VARIANT DRIFT (1) ===
  Button.state   Figma [default, hover, pressed, disabled]
                 Spec  [default, hover, pressed, focus, disabled]
                 -> "focus" missing in Figma (accessibility risk)

=== UNMAPPABLE (2) ===
  Color/Brand/Blue-500   no codeSyntax set — cannot match
  Color/Legacy/Accent    no codeSyntax set — cannot match

=== BINDING GAPS (4) ===
  Button / primary       fill is a hardcoded paint, not a bound variable
  Card                   cornerRadius hardcoded 8 — bind to --radius-200

Summary: 13 findings across 6 categories. No files modified.
```

Rules for the report:
- **Nothing is written.** Not the specs, not Figma, not tokens.css. Report only.
- Order matters — value mismatches first because they're the ones causing visible drift right now.
- Every finding names both sides and the exact values. "These differ" is useless.
- Missing focus states get flagged as an accessibility risk explicitly, not listed neutrally.
- After the report, offer three resolutions per finding: take Figma's value, take code's value, or leave it and record the divergence in the spec's Notes section.

**Recording a divergence is a legitimate outcome.** Sometimes Figma is intentionally ahead of code. An undocumented divergence is drift; a documented one is a roadmap item. If a finding recurs across three reconciles unresolved, say so — that's a decision someone keeps deferring.

## Round-trip integrity check

After any push or pull, verify:
- every semantic variable has `codeSyntax` set
- every `codeSyntax` value resolves to a real token in tokens.css
- no primitive has non-empty scopes (except alpha overlays → `["EFFECT_COLOR"]`)
- variant axis values in Figma match the spec's `variantAxes` exactly
- `npm run audit:tokens` still exits 0

A push that leaves `codeSyntax` unset has broken the bridge for every future reconcile. Treat it as a failed push, not a partial success.

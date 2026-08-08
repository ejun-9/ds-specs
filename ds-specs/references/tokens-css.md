# Token layer — plain CSS / SCSS / CSS Modules

## The three layers

```css
/* ---------- Layer 1: upstream design system ---------- */
/* Prefixed. Mirrors whatever library you're on (Atlaskit, Radix, MUI,
   shadcn, or primitives you derived yourself in greenfield). */
:root {
  --ds-text:         #292A2E;
  --ds-text-subtle:  #6B6E76;
  --ds-surface:      #FFFFFF;
  --ds-link:         #1868DB;
  --ds-border:       #E1E3E6;
  --ds-space-100:    8px;
  --ds-space-200:    16px;
  --ds-radius-100:   4px;
  --ds-radius-200:   8px;
}

/* ---------- Layer 2: project aliases ---------- */
/* Every alias references Layer 1 with the raw value as fallback.
   This layer is the contract components code against. */
:root {
  --color-text:         var(--ds-text, #292A2E);
  --color-text-subtle:  var(--ds-text-subtle, #6B6E76);
  --color-surface:      var(--ds-surface, #FFFFFF);
  --color-link:         var(--ds-link, #1868DB);
  --color-border:       var(--ds-border, #E1E3E6);
  --space-100:          var(--ds-space-100, 8px);
  --space-200:          var(--ds-space-200, 16px);
  --radius-100:         var(--ds-radius-100, 4px);
  --radius-200:         var(--ds-radius-200, 8px);
}

/* ---------- Layer 3: components ---------- */
/* Alias only. Never --ds-*. Never a raw value. */
.card {
  color: var(--color-text);
  background: var(--color-surface);
  padding: var(--space-200);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-200);
}
```

## Why the fallback matters

`var(--ds-text, #292A2E)` means the project still renders correctly if the upstream package fails to load, is being swapped, or hasn't been installed yet. It also makes the alias file self-documenting — you can read the intended value without opening the library.

## Theming

Themes override **Layer 1 only**. Components change zero lines.

```css
[data-theme="dark"] {
  --ds-text:    #E6E7E9;
  --ds-surface: #1D1F23;
  --ds-border:  #3B3E45;
}

@media (prefers-contrast: more) {
  :root { --ds-border: #000000; }
}
```

If you find yourself writing a dark-mode override inside a component file, the token layer has a gap. Fix the layer, not the component.

## Naming

Semantic over literal. `--color-link`, not `--color-blue-600`. The name should survive a rebrand.

Numeric scales use consistent steps so the LLM can reason about relative size:

```
--space-050:  4px
--space-100:  8px
--space-150: 12px
--space-200: 16px
--space-300: 24px
--space-400: 32px
--space-600: 48px
--space-800: 64px
```

Interactive states get their own tokens rather than inline `filter` or `opacity` tweaks:

```css
--color-interactive:          var(--ds-link, #1868DB);
--color-interactive-hovered:  var(--ds-link-hover, #1558C0);
--color-interactive-pressed:  var(--ds-link-press, #123F91);
--color-interactive-disabled: var(--ds-disabled, #B7BAC0);
```

## Minimum coverage

Don't ship a token layer missing these categories — every gap is a place the LLM will fabricate.

- **Color** — text (default, subtle, inverse), background/surface (default, raised, sunken), link, border, and interactive states (default, hovered, pressed, disabled, selected), plus semantic status (success, warning, danger, info)
- **Spacing** — at least 8 steps on one base grid (4px or 8px)
- **Typography** — families, sizes (caption → display), weights, line heights
- **Radius** — subtle through fully rounded (`--radius-full: 9999px`)
- **Elevation** — shadow steps, named by intent (`--elevation-raised`, `--elevation-overlay`)
- **Z-index** — named layers (`--z-dropdown`, `--z-modal`, `--z-tooltip`, `--z-toast`), never raw integers
- **Motion** — durations and easing curves

## styled-components / emotion

Same three layers. Layer 1 and 2 stay in a global stylesheet as CSS custom properties — do **not** move them into a JS theme object. CSS variables survive the theme switch without re-render and the audit script can still scan them. Components interpolate `var(--color-text)` as normal.

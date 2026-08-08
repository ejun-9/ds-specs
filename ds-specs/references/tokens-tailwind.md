# Token layer — Tailwind

Tailwind changes what "hardcoded" means. The article's model assumes hand-written CSS with `var(--token)`. In a Tailwind codebase, values live in `className` strings, so a CSS-scanning audit finds nothing while the project drifts freely.

Translate the model like this.

## What a violation is in Tailwind

| Violation | Example | Why it's a violation |
|---|---|---|
| Arbitrary value | `p-[13px]`, `text-[#2563EB]`, `w-[347px]` | Bypasses the scale entirely — this is the direct equivalent of a raw hex |
| Off-scale utility | `p-[0.8rem]`, `gap-[7px]` | Same as above, wearing a rem costume |
| Raw palette color | `text-blue-600`, `bg-gray-100` | Not semantic. Which blue is "the link blue"? Nobody knows next session. |
| Inline style | `style={{ padding: 16 }}` | Escapes Tailwind and the audit both |
| Hand-written CSS with raw values | any `.css` file with `#hex` or `px` | Audit these with the plain-CSS rules |

Arbitrary values aren't *banned* forever — some one-offs are legitimate. They're **warnings** on first appearance and **errors** the second time the same value appears, because a repeated arbitrary value is a missing token.

## The three layers in Tailwind v4

Layer 1 and 2 still live in CSS. Layer 3 becomes utility classes.

```css
/* app.css */
@import "tailwindcss";

/* ---------- Layer 1: upstream ---------- */
:root {
  --ds-text:      #292A2E;
  --ds-link:      #1868DB;
  --ds-surface:   #FFFFFF;
  --ds-border:    #E1E3E6;
}

/* ---------- Layer 2: project aliases, exposed to Tailwind ---------- */
@theme {
  --color-text:      var(--ds-text, #292A2E);
  --color-link:      var(--ds-link, #1868DB);
  --color-surface:   var(--ds-surface, #FFFFFF);
  --color-border:    var(--ds-border, #E1E3E6);

  --spacing-100:      8px;
  --spacing-200:     16px;
  --spacing-300:     24px;

  --radius-100:       4px;
  --radius-200:       8px;

  --text-caption:    12px;
  --text-body:       14px;
  --text-heading:    20px;

  --shadow-raised:   0 1px 2px rgb(9 30 66 / 0.12);
  --shadow-overlay:  0 8px 16px rgb(9 30 66 / 0.16);
}
```

`@theme` generates the utilities: `text-text`, `bg-surface`, `text-link`, `p-200`, `rounded-200`, `shadow-raised`.

```jsx
/* ---------- Layer 3: components ---------- */
<div className="bg-surface text-text p-200 rounded-200 border border-border">
```

## Tailwind v3

Same idea via config, referencing the CSS variables so the alias layer still exists:

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        text:    'var(--color-text)',
        link:    'var(--color-link)',
        surface: 'var(--color-surface)',
        border:  'var(--color-border)',
      },
      spacing: { 100: 'var(--space-100)', 200: 'var(--space-200)' },
      borderRadius: { 100: 'var(--radius-100)', 200: 'var(--radius-200)' },
    },
  },
}
```

Don't put hex values directly in the config. That collapses layers 1 and 2 and you lose theming.

## Closing the set

The single highest-value move in a Tailwind project: **remove the default palette** so the LLM can't reach for `blue-600`.

In v4, `@theme` replaces rather than extends when you redefine a namespace. In v3, define `theme.colors` (not `theme.extend.colors`) so the defaults drop. This is the closed-token-set principle — if `text-blue-600` doesn't compile, it can't drift.

Expect friction for a session or two. It's worth it.

## Theming

Same as plain CSS — override Layer 1 only.

```css
[data-theme="dark"] {
  --ds-text:    #E6E7E9;
  --ds-surface: #1D1F23;
}
```

Utilities resolve through the chain automatically. Avoid `dark:` variants scattered across components; they're the Tailwind version of a hardcoded dark-mode fix.

## Spec files still apply, unchanged

Tailwind gives you a token layer. It does **not** give you memory. The `specs/` tree matters exactly as much here — arguably more, because a `className` string carries no indication of which component pattern it belongs to. The "Tokens used" section of each component spec lists utility classes instead of `var()` references.

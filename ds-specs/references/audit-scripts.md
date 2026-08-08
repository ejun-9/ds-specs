# Audit scripts

The audit is what makes this a system rather than documentation. Documentation says what exists; the audit blocks what shouldn't.

Non-negotiables:
- exit code 1 on any error, so CI fails
- every violation prints file, line, what's wrong, and the token to use instead
- errors and warnings are distinct categories
- suggestions come from `specs/tokens/token-reference.md`, not from guesses

## Output format

```
Token Audit
Scanning 28 file(s)...

src/components/Nav.css
  x L42: Hardcoded color #1868DB, use var(--color-link)
  x L78: Raw spacing 12px in padding, use var(--space-150)
  ! L96: Raw duration 0.2s, consider using --motion-* token

=== Summary ===
Files scanned:      28
Files with issues:  1
Errors:             2
Warnings:           1
```

## Severity

**Errors** (exit 1) — hardcoded colors, hardcoded spacing in padding/margin/gap, hardcoded border-radius, hardcoded font-size, raw z-index integers.

**Warnings** (exit 0) — transition durations, box-shadow values, line-height, any value with an adjacent `/* audit-ok: reason */` comment.

The escape hatch is deliberate and deliberately annoying: an inline comment stating *why*. If a file accumulates several, that's a missing token, not a stubborn edge case.

## Plain CSS — `scripts/token-audit.js`

```js
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');

const ROOT = process.argv[2] || 'src';
const IGNORE = /node_modules|dist|build|tokens\.css/;

// Map raw values → tokens. Generate from specs/tokens/token-reference.md.
const TOKENS = {
  color:   { '#1868db': '--color-link', '#292a2e': '--color-text' },
  spacing: { '4px': '--space-050', '8px': '--space-100', '16px': '--space-200' },
  radius:  { '4px': '--radius-100', '8px': '--radius-200' },
};

const RULES = [
  { name: 'color',   severity: 'error',
    re: /(?<!var\()#[0-9a-fA-F]{3,8}\b|rgba?\([^)]*\)/g,
    lookup: v => TOKENS.color[v.toLowerCase()] },
  { name: 'spacing', severity: 'error',
    re: /(?:padding|margin|gap|inset)[^:]*:\s*([^;]+);/g,
    lookup: v => TOKENS.spacing[v.trim()] },
  { name: 'radius',  severity: 'error',
    re: /border-radius:\s*([^;]+);/g,
    lookup: v => TOKENS.radius[v.trim()] },
  { name: 'z-index', severity: 'error', re: /z-index:\s*(\d+)/g },
  { name: 'duration',severity: 'warning', re: /\b\d*\.?\d+m?s\b/g },
];

let errors = 0, warnings = 0, scanned = 0, dirty = 0;

function walk(dir) {
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, entry.name);
    if (IGNORE.test(p)) continue;
    if (entry.isDirectory()) walk(p);
    else if (/\.(css|scss)$/.test(entry.name)) check(p);
  }
}

function check(file) {
  scanned++;
  const lines = fs.readFileSync(file, 'utf8').split('\n');
  const hits = [];

  lines.forEach((line, i) => {
    if (line.includes('audit-ok')) return;
    for (const rule of RULES) {
      rule.re.lastIndex = 0;
      let m;
      while ((m = rule.re.exec(line))) {
        const raw = (m[1] || m[0]).trim();
        if (raw.includes('var(')) continue;
        const token = rule.lookup ? rule.lookup(raw) : null;
        const fix = token
          ? `use var(${token})`
          : `no matching token — add one to tokens.css`;
        hits.push({ line: i + 1, severity: rule.severity, raw, rule: rule.name, fix });
        rule.severity === 'error' ? errors++ : warnings++;
      }
    }
  });

  if (hits.length) {
    dirty++;
    console.log(`\n${file}`);
    for (const h of hits) {
      const mark = h.severity === 'error' ? 'x' : '!';
      console.log(`  ${mark} L${h.line}: ${h.rule} "${h.raw}", ${h.fix}`);
    }
  }
}

console.log('Token Audit');
walk(ROOT);
console.log(`\n=== Summary ===
Files scanned:      ${scanned}
Files with issues:  ${dirty}
Errors:             ${errors}
Warnings:           ${warnings}`);
process.exit(errors > 0 ? 1 : 0);
```

Populate `TOKENS` from the token reference when you generate the file. An audit with an empty map still catches violations but gives useless suggestions.

## Tailwind — `scripts/token-audit.js`

Scans JSX/TSX for arbitrary values and non-semantic palette classes.

```js
const RULES = [
  { name: 'arbitrary-value', severity: 'error',
    re: /\b[a-z-]+-\[[^\]]+\]/g,
    msg: v => `arbitrary value ${v} — add a token to @theme` },

  { name: 'raw-palette', severity: 'error',
    re: /\b(?:text|bg|border|ring|fill)-(?:slate|gray|zinc|neutral|stone|red|orange|amber|yellow|lime|green|emerald|teal|cyan|sky|blue|indigo|violet|purple|fuchsia|pink|rose)-\d{2,3}\b/g,
    msg: v => `${v} is not semantic — use a named token like text-link` },

  { name: 'inline-style', severity: 'error',
    re: /style=\{\{/g,
    msg: () => 'inline style escapes the token layer' },

  { name: 'dark-variant', severity: 'warning',
    re: /\bdark:/g,
    msg: () => 'dark: variant in a component — theme via Layer 1 instead' },
];
```

Same walker, same summary, same exit code. File pattern `/\.(jsx|tsx)$/`. Also run the CSS rules above over any hand-written stylesheets in the project.

**Repeat detection:** track arbitrary values across files. First occurrence is a warning; the same value appearing twice is an error, because a repeated arbitrary value is a token you haven't named yet.

## Near-miss detection

A value one step off a token is the highest-value finding and the one autofix must never touch.

```js
function nearest(hex, tokenMap) {
  const dist = (a, b) => {
    const p = h => [1,3,5].map(i => parseInt(h.substr(i,2),16));
    const [r1,g1,b1] = p(a), [r2,g2,b2] = p(b);
    return Math.abs(r1-r2) + Math.abs(g1-g2) + Math.abs(b1-b2);
  };
  let best = null;
  for (const [val, token] of Object.entries(tokenMap)) {
    if (!/^#[0-9a-f]{6}$/i.test(val)) continue;
    const d = dist(hex, val);
    if (d > 0 && d <= 12 && (!best || d < best.d)) best = { token, val, d };
  }
  return best;
}
```

Classify every violation as `exact` (autofixable) or `near-miss` / `unmatched` (never autofixable). Report near misses with the closest token and the distance:

```
  x L42: color "#1868DA" — near miss, closest --color-link (#1868DB). Typo or intentional? Not autofixed.
```

## Wiring it up

```json
{
  "scripts": {
    "audit:tokens": "node scripts/token-audit.js src",
    "precommit": "npm run audit:tokens"
  }
}
```

CI:

```yaml
- run: npm run audit:tokens
```

## Drift detection (large projects)

Pin upstream design system versions. On update, diff the package's token export against `specs/tokens/token-reference.md` and print which spec files reference changed tokens.

Report only. Never auto-edit a spec — a human decides whether an upstream change should propagate.

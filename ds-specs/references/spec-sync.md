# Spec sync (Storybook projects)

Use this when the project has Storybook and its component specs follow the short template in `references/spec-templates.md`. Skip it otherwise.

Storybook's MCP server (the `@storybook/addon-mcp` addon) answers an agent's "how do I use Button?" with the component's props, its stories as code, and the doc comment above the component. The spec sync writes the spec's Overview into that doc comment. One source, the spec, and the agent gets the "when NOT to use" in the same answer as the code.

The same script, with `--check`, is the drift check. It fails when:

- a spec's `source:` file doesn't exist
- a component has no story next to it
- the spec's `storybook:` ID doesn't match the story's ID
- the doc comment above the component is out of date with the spec

Run it in gate 3 (pre-commit) and gate 4 (CI) next to the token audit. It never edits a spec, only the generated comment, and only without `--check`.

## Script

`scripts/spec-sync.mjs`. Set `SPEC_DIRS` to the spec tree's component folders.

```js
#!/usr/bin/env node
// node scripts/spec-sync.mjs          write each component's doc comment from its spec
// node scripts/spec-sync.mjs --check  fail on drift (pre-commit and CI)
import fs from 'node:fs'
import path from 'node:path'

const SPEC_DIRS = ['specs/atoms', 'specs/molecules', 'specs/organisms', 'specs/components']
const CHECK = process.argv.includes('--check')
const MARKER = 'Generated from'

const frontmatter = (md) => {
  const block = md.match(/^---\n([\s\S]*?)\n---/)?.[1] ?? ''
  const field = (key) => block.match(new RegExp(`^${key}:\\s*([^#\\n]+)`, 'm'))?.[1].trim()
  return { component: field('component'), source: field('source'), storybook: field('storybook') }
}

// "## 2. Overview" and "## Overview" both match "Overview".
const section = (md, heading) => {
  const lines = md.split('\n')
  const start = lines.findIndex((l) => /^## /.test(l) && l.replace(/^##\s*(\d+\.\s*)?/, '').trim().toLowerCase() === heading.toLowerCase())
  if (start === -1) return ''
  const rest = lines.slice(start + 1)
  const end = rest.findIndex((l) => l.startsWith('## '))
  return rest.slice(0, end === -1 ? undefined : end).filter((l) => l.trim() !== '---').join('\n').trim()
}

const buildComment = (md, specPath) => {
  const text = [section(md, 'Overview'), `${MARKER} ${specPath} by scripts/spec-sync.mjs. Edit the spec, not this comment.`]
    .filter(Boolean).join('\n\n').replace(/\*\//g, '*\\/')
  return ['/**', ...text.split('\n').map((l) => (l ? ` * ${l}` : ' *')), ' */'].join('\n')
}

// Storybook builds a story ID from the story's title: "Atoms/Button" becomes "atoms-button".
const storyIdFor = (storySource) => {
  const title = storySource.match(/title:\s*['"`]([^'"`]+)['"`]/)?.[1] ?? ''
  return title.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')
}

const problems = []
let written = 0
const specs = SPEC_DIRS.filter((d) => fs.existsSync(d))
  .flatMap((d) => fs.readdirSync(d).filter((f) => f.endsWith('.md') && !f.startsWith('_')).map((f) => path.join(d, f)))

for (const specPath of specs) {
  const md = fs.readFileSync(specPath, 'utf8')
  const { component, source, storybook } = frontmatter(md)
  if (!component || !source) { problems.push(`${specPath}: frontmatter needs component: and source:`); continue }
  if (!fs.existsSync(source)) { problems.push(`${specPath}: source ${source} does not exist`); continue }

  const ext = path.extname(source)
  const storyPath = source.replace(new RegExp(`\\${ext}$`), `.stories${ext}`)
  if (!fs.existsSync(storyPath)) problems.push(`${component}: no story at ${storyPath}`)
  else {
    const id = storyIdFor(fs.readFileSync(storyPath, 'utf8'))
    if (!storybook) problems.push(`${specPath}: add storybook: ${id} to the frontmatter`)
    else if (storybook !== id) problems.push(`${specPath}: storybook: ${storybook}, but the story's ID is ${id}`)
  }

  // Replace the doc comment directly above the component's export, or add one.
  // The comment group can't cross a closing */, so it never swallows code above it.
  const code = fs.readFileSync(source, 'utf8')
  const exportRe = new RegExp(`(\\/\\*\\*(?:(?!\\*\\/)[\\s\\S])*\\*\\/\\n)?(export (?:default )?(?:function|const) ${component}\\b)`)
  const match = code.match(exportRe)
  if (!match) { problems.push(`${component}: no "export function ${component}" or "export const ${component}" in ${source}`); continue }
  const comment = buildComment(md, specPath)
  if ((match[1] ?? '').trim() === comment) continue
  if (CHECK) problems.push(`${component}: doc comment is out of date with ${specPath} (run node scripts/spec-sync.mjs)`)
  else { fs.writeFileSync(source, code.replace(exportRe, `${comment}\n$2`)); written++ }
}

if (CHECK) {
  if (problems.length) { console.error(`x Spec drift:\n${problems.map((p) => `  ${p}`).join('\n')}`); process.exit(1) }
  console.log(`✓ ${specs.length} spec(s) in sync with their components and stories.`)
} else {
  console.log(`Wrote ${written} doc comment(s).`)
  if (problems.length) console.log(`Drift to fix:\n${problems.map((p) => `  ${p}`).join('\n')}`)
}
```

## Wiring

- `package.json`: `"specs:sync": "node scripts/spec-sync.mjs"` and `"audit:specs": "node scripts/spec-sync.mjs --check"`.
- Pre-commit and CI: run `audit:specs` right after the token audit. Same rules as the audit: CI checks, it never writes.
- Session instructions: "After editing a component spec, run `npm run specs:sync`. Never hand-edit a comment that says Generated from."

## What it doesn't check

Props. On the short template the spec doesn't list them, and Storybook reads them from the source, so they can't drift. If a project keeps the full template, the token audit and a reviewer still cover section 5.

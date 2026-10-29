# Changelog

## 0.2.0

- Detects Storybook (a `.storybook/` folder or `@storybook/*` in `package.json`).
  On Storybook projects, component specs use a short template: no Props or Code
  example sections, since the stories hold both, and a `storybook:` ID in the
  frontmatter and metadata.
- New `references/spec-sync.md`: a script that writes each spec's Overview into
  the component's doc comment, which Storybook's MCP returns with the code. With
  `--check` it fails on a missing story, a wrong story ID or an outdated comment.
  Runs at pre-commit and in CI.
- Projects without Storybook keep the full 8-section template.

## 0.1.1

- Shorten the SKILL.md description to 1,000 characters so claude.ai accepts the
  upload (its limit is 1,024). Every trigger phrase is kept.

## 0.1.0

Initial release. Authored with Claude Opus 5 (high reasoning effort), based on
Hardik Pandya's method: https://hvpandya.com/llm-design-systems

- Three-tier spec tree: foundations/tokens, components, patterns
- Three-layer token indirection for plain CSS and Tailwind
- Stack detection (Tailwind vs CSS) and greenfield vs retrofit branching
- Scope scaling to project size
- Audit script with exact-match and near-miss classification
- Five enforcement gates with ratcheted rollout for legacy codebases
- Two-way Figma bridge via the `codeSyntax` join key
- Reconcile report — six categories, never auto-resolves

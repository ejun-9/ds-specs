# Push this to GitHub

Two options. Both create a **public** repo under your account.

## Option A — GitHub CLI (one command)

If you have `gh` installed and authenticated:

```bash
cd ds-specs-repo
gh repo create ds-specs --public --source=. --remote=origin --push \
  --description "A Claude skill that builds design systems an LLM can follow, with a two-way Figma bridge"
```

Done. The git history is already committed.

## Option B — Manual

1. Create an empty public repo named `ds-specs` at https://github.com/new
   (no README, no .gitignore, no license — this repo already has them)

2. Then:

```bash
cd ds-specs-repo
git remote add origin https://github.com/YOUR-USERNAME/ds-specs.git
git push -u origin main
```

## After pushing

**Fix the placeholder.** The README has `YOUR-USERNAME` in the install commands:

```bash
sed -i '' 's/YOUR-USERNAME/your-actual-username/g' README.md
git commit -am "Fix install URLs"
git push
```

**Attach the .skill file to a release** so claude.ai users can install without cloning:

```bash
gh release create v0.1.0 ../ds-specs.skill \
  --title "v0.1.0" --notes "Initial release"
```

The README's claude.ai install link points at `releases/latest`, so it only works once a release exists.

**Add topics** for discoverability: `claude`, `claude-skill`, `design-systems`, `design-tokens`, `figma`, `ai-design`

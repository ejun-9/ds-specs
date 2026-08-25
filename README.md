# ds-specs

**A Claude Skill that builds a design system an LLM can actually follow — and keeps it in
sync with Figma.**

An LLM makes 200–300 visual micro-decisions per coding session. Each one is individually
reasonable. Together they aren't a design system, they're 200 guesses. Next session, zero
memory, 200 *different* guesses. By session ten your prototype looks like three products
built by three teams.

`ds-specs` turns every visual decision into a lookup instead of a guess. Human taste goes in
once; the model follows it mechanically after that.

I wrote about building it here: [ds-specs: a Claude Skill for LLM-first design systems, with
two-way Figma sync](https://eunji.design/thoughts/ds-specs).

Built on [Hardik Pandya's method for LLM-readable design
systems](https://hvpandya.com/llm-design-systems) — extended with a two-way Figma bridge and
deployment enforcement. Authored with Claude Opus 5.

---

## Is this for you?

Use it if you are building with an AI coding agent and any of these sound familiar:

- The prototype drifts — same component, three different blues, across three sessions.
- The agent invents values. `padding: 13px`. `#1868DA` when your token is `#1868DB`.
- Your Figma file and your code have quietly stopped agreeing, and nobody knows which moved.
- You have a design system, but it's written for humans, so the agent doesn't read it.

Works greenfield or retrofit, Tailwind or plain CSS — it detects which and branches.

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

**Five enforcement gates.** Session start → local autofix → pre-commit → CI → deploy.
Self-healing is allowed only locally, only on exact matches, only with a human reading the
diff. CI and deploy block and never rewrite your code.

**A two-way Figma bridge.** Figma variables carry a `codeSyntax` field. Set it to the CSS
custom property and the Figma variable and your token become one record seen from two sides
— so sync is exact instead of fuzzy name-matching.

### The three failure modes it targets

| Failure | Countermeasure |
|---|---|
| Fabricates plausible values (`padding: 12px`) | **Closed token layer** — pick from a finite named set |
| No cross-session memory | **Spec files** — read at the start of every session |
| Silent drift, upstream and within-session | **Audit script** — exit code 1, runs in CI |

---

## Three modes

| Say | Mode | What happens |
|---|---|---|
| "create a design system," "tokenize this," "my prototype is drifting" | **Build** | Spec tree, token layer, audit script, enforcement gates |
| "push my specs to Figma," "generate Figma docs pages from my specs" | **Push** | Emits a build plan, hands off to `figma-generate-library` |
| "sync from Figma," "does Figma match my code" | **Reconcile** | Six-category diff report. Changes nothing. |

**Neither side is canonical.** Reconcile reports and stops. A mismatch is ambiguous — only a
human knows whether the designer moved or the developer did.

---

## Install

Two ways, depending on where you use Claude. Pick one.

### Option A — claude.ai (no terminal needed)

1. Go to the [latest release](../../releases/latest) on this repo.
2. Under **Assets**, download `ds-specs.skill`.
3. In claude.ai, go to **Settings → Capabilities → Skills**.
4. Click **Upload skill** and select the file you just downloaded.
5. That's it — Claude pulls in `ds-specs` automatically when a conversation matches, or you
   can name it explicitly (see [Using it](#using-it)).

To update later: repeat with the newest release — the upload replaces the old version.

### Option B — Claude Code / Cowork (terminal)

Requires `git` (macOS and Linux already have it; check with `git --version`).

```bash
git clone https://github.com/ejun-9/ds-specs.git /tmp/ds-specs-repo
cp -r /tmp/ds-specs-repo/ds-specs ~/.claude/skills/
```

Then restart Claude Code, or start a new session, so it picks up the skill.

**Verify it worked** — in a session, type:

```
use ds-specs to set up a design system for this project
```

If Claude references the spec tree, the token layer, or the three modes above, you're set.

**Keep it current** — skills don't auto-update:

```bash
git clone --depth=1 https://github.com/ejun-9/ds-specs.git /tmp/ds-specs-latest \
  && rm -rf ~/.claude/skills/ds-specs \
  && cp -r /tmp/ds-specs-latest/ds-specs ~/.claude/skills/
```

---

## Using it

Name it explicitly the first few times, until you see where it fires on its own:

> use ds-specs to set up a design system for this project

> use ds-specs to reconcile my Figma file against the code

After that it triggers by itself on phrases like the ones in the mode table, and on implicit
cases — styling work in a repo that already has a `specs/` tree or a `tokens.css`.

**Under-triggering is the normal failure mode.** Short requests ("make this button blue")
often won't fire it, even though that's exactly when a token should be enforced. If it keeps
missing a phrase you use, add that phrase to the `description` field in `SKILL.md` — that
field is what activates the skill. The tables in the body only route between modes once it's
already loaded.

---

## Figma modes need two more skills

Push and Reconcile call the Figma Plugin API through Figma's own skills rather than
reimplementing it:

- `figma-use` — Plugin API syntax
- `figma-generate-library` — variable collections, component sets, variant grids

Both ship with Figma's MCP server. `ds-specs` owns the spec layer, the token layer, and the
diff; it never writes to Figma directly. Two skills mutating Figma independently diverge
within weeks.

---

## Design decisions worth knowing

**Autofix stops at pre-commit.** The token mapping is inference. An exact hex match is safe.
A near miss — `#1868DA` against a `#1868DB` token — is either a typo or a deliberate one-off,
and no script can tell. Near misses are reported at *higher* severity than exact matches and
never auto-rewritten.

**Only spec what exists.** A small accurate spec layer beats a comprehensive stale one. The
skill scales the tree to project size and refuses to generate specs for components you
haven't built.

**Ratchet, don't hard-fail.** On an existing codebase with hundreds of violations, the gates
start from a recorded baseline and fail only when the count *increases*. Turning on hard-fail
against 400 violations gets the gate deleted by Thursday.

**Break-glass leaves a paper trail.** Deploy can be forced, but it logs every violation,
notifies the team, and opens an issue in the name of whoever triggered it.

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

The core method — three-tier specs, closed token sets, the audit script, and the framing of
LLM failure modes — comes from Hardik Pandya:

**https://hvpandya.com/llm-design-systems**

Read it first. It explains the reasoning this skill only encodes. What's added here is the
two-way Figma bridge, the five enforcement gates, stack detection for Tailwind, and near-miss
classification in the audit.

## Related

- [figma-comments-mcp](https://github.com/ejun-9/figma-comments-mcp) — an MCP server that
  lets Claude read your Figma comments and Dev Mode annotations.
- [More of what I'm figuring out](https://eunji.design/thoughts) about designing with agents.

## License

MIT. See [LICENSE](LICENSE).

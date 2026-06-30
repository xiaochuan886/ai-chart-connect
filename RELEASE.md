# Release notes & publishing guide

This skill is distributed as an [agent skill](https://skills.sh) installable via
`npx skills add <owner>/ai-chart-connect`. It is **not** an npm package — the
`skills` CLI pulls the `SKILL.md` + `references/` folder straight from this
GitHub repo via the GitHub Trees API. So "publishing" = "push to GitHub main".

## Versioning

The skill carries a semantic version in `SKILL.md` frontmatter (`version: x.y.z`).

- **patch** (x.y.**Z**): doc fixes, accuracy corrections, typos, example tweaks
- **minor** (**Y**.z): new reference docs, new capability sections, additive
- **major** (**X**.0): breaking contract changes (API endpoint renames, required
  config-key changes, renderer-shell protocol changes)

Bump the version in `SKILL.md` frontmatter on every meaningful change so users
can tell which version they have installed.

## How users install / update

```bash
# Install (first time)
npx skills add <owner>/ai-chart-connect

# Update to the latest
npx skills update ai-chart-connect
```

The CLI installs into `~/.agents/skills/ai-chart-connect/` and symlinks into the
detected agent's skill directory (Claude Code → `~/.claude/skills/`, etc.).

## Publishing checklist (per release)

1. [ ] All doc changes are accurate against the current workbench source
       (`src/layout/`, `src/workspace/schema.ts`, `server/workspaceRoutes.ts`).
       The reference docs describe the **runtime** contract — drift here is the
       #1 source of agent trial-and-error, so re-verify before tagging.
2. [ ] Bump `version:` in `SKILL.md` frontmatter.
3. [ ] Update the changelog below.
4. [ ] Commit on `main` (the CLI reads the default branch).
5. [ ] (Optional) Tag: `git tag v1.5.1 && git push --tags` so the version is
       pinable.

## Repo layout (do not rename)

```
ai-chart-connect/
├── SKILL.md          ← entry point; frontmatter name/description/version
├── references/       ← progressive disclosure docs; agent loads on demand
│   ├── api-reference.md
│   ├── apply-data.md
│   ├── build-report.md
│   ├── create-template.md
│   ├── layout-shell.md
│   └── refine-template.md
├── README.md         ← human-facing intro (install / usage / prereqs)
├── LICENSE
├── RELEASE.md        ← this file
└── .gitignore
```

The `skills` CLI recognizes `SKILL.md` at the repo root (highest-priority
prefix) and pulls its sibling `references/` folder with it. Do **not** nest
the skill under a subdirectory — root placement is what makes
`npx skills add <owner>/ai-chart-connect` resolve without a path argument.

---

## Changelog

### 1.5.0 — 2026-06-30

First public release. Accuracy-audited against workbench source:

- **layout-shell.md**: `yNice` truth (档位跳档 258→500, no double-nicing),
  axis-gutter sizes, `estimateTextWidth`/`wrapText`/`computeChartLayout`
  exports, palette status fields (positive/negative/neutral/text), tick-value
  consistency.
- **create-template.md**: required config keys (showTitle/backgroundColor/
  showLegend) now flagged as hard-required, `yNice` decision guidance, new
  hard rule ⑦ ("use layout tool functions, don't hand-compute").
- **refine-template.md**: business-notes strict-validation timing, purpose
  gate applies to save too.
- **api-reference.md**: health service name corrected to `vizkit`.
- **apply-data.md / build-report.md**: verified accurate.

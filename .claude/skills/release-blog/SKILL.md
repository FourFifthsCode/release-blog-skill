---
name: release-blog
description: Generate a release blog post draft from GitHub commits. Analyzes commits not yet in a release tag, associates them with PRs and linked issues, categorizes and ranks by blog-worthiness, fetches deep PR/issue context, and drafts a structured blog post. Use when asked to "write a release blog post", "generate release notes blog", or "draft a blog post for a release".
---

# Release Blog Post Generator

End-to-end pipeline: commits → PRs → categories/rankings → blog post draft.

**IMPORTANT:**

All tool subcommands write to stdout by default; use `-o <file>` to persist.

## Workflow

### Resume Check (evaluate before Step 1)

Check whether `release-blog/research.json` exists in the current working directory (use `ls release-blog/research.json 2>/dev/null`).

- If the user says "pick up at step 5", "use existing research.json", "skip to drafting", or similar **OR** the file exists: ask "Found `release-blog/research.json` — skip straight to drafting the blog post? (yes to skip, no to run the full pipeline and overwrite existing files)"
- If confirmed: skip to Step 5, reading `release-blog/research.json` and inferring `{version}` from `git describe --tags --abbrev=0`.
- If not confirmed (or file absent): run the full pipeline from Step 1 (this will overwrite any existing intermediate files).

### Step 1 — Gather parameters

Detect from git context where possible:
- `--repo`: inferred from git remotes — prefers `upstream` over `origin` (pass `--remote <name>` to override)
- `--tag`: inferred from `git describe --tags --abbrev=0` (or ask user to confirm)
- `--output-dir`: default is `./release-blog/` — create the folder first with `mkdir -p release-blog`

All intermediate files (`commits.json`, `prs_detailed.md`, `prs_categorized.md`, `research.json`) and the final `blog_post_{version}.md` are written into this folder.

**Fork/upstream workflows:** If the local repo is a fork (e.g. working against `argoproj/argo-cd` via an `upstream` remote), the tool auto-detects `upstream` as the canonical remote. Pass `--remote upstream` explicitly if needed.

### Step 2 — Run the pipeline (3 fast commands)

Set `SKILL_DIR` to the directory containing this SKILL.md file (the directory you loaded it from — e.g. `~/.claude/skills/release-blog` or `{project}/.claude/skills/release-blog`). Use that path for all `release-pipeline` invocations so commands work regardless of where Claude is running from.

```bash
SKILL_DIR=<directory containing this SKILL.md>
mkdir -p release-blog
uv run $SKILL_DIR/tools/release-pipeline commits   --repo {owner/repo} --tag {vX.Y.Z} [--remote {remote}] -o release-blog/commits.json
uv run $SKILL_DIR/tools/release-pipeline fetch-prs --repo {owner/repo} --input release-blog/commits.json -o release-blog/prs_detailed.md
uv run $SKILL_DIR/tools/release-pipeline categorize --input release-blog/prs_detailed.md -o release-blog/prs_categorized.md
```

- `commits`: extracts PR numbers directly from git log subjects (`(#NNNNN)` pattern, ~97% hit rate); falls back to GitHub API only for the remainder; **batch-validates all PR numbers against the target repo** and automatically excludes any that belong to fork branches or deleted PRs — no manual filtering needed
- `fetch-prs`: GraphQL batches of 50 — ~10 API calls for 500 PRs; handles partial GraphQL responses gracefully (logs warnings, continues with valid data)
- `categorize`: pure Python, no API calls

### Step 3 — Editorial selection

Read `release-blog/prs_categorized.md` and present the Summary table to the user. Then:

**Blog Highlights** — include all (these are manually curated by maintainers).
- Exception: if a Blog Highlight is a bare dependency bump with no user-facing description, offer to skip it (the maintainer likely tagged it in anticipation of writing the description themselves).
- pull in other features if related into single grouping

**New Features** — propose top 10 by score.
- Flag any entry that is clearly not user-facing (dead code removal, pure refactors, internal plumbing with no config surface). Propose a swap from rank 11+ with a more compelling subject.
- Look for narrative groupings (e.g., multiple ApplicationSet UI features, multiple Hydrator fixes) — group them in the blog post even if individually ranked.

**Notable Bug Fixes** — propose top 5 by score.
- Prefer fixes that affect common workflows (auto-sync, rollback, multi-tenancy, namespace handling) over CI/tooling fixes.

**Documentation** — scan for anything that exposes/documents a significant previously-hidden feature (e.g., undocumented env vars, previously-missing upgrade guidance). These can appear in the blog post as a brief mention.

Ask the user to confirm or adjust the selection before proceeding to research.

### Step 4 — Deep research

Once selection is confirmed:

```bash
uv run $SKILL_DIR/tools/release-pipeline research \
  --repo {owner/repo} \
  --prs {comma-separated PR numbers} \
  -o release-blog/research.json
```

Uses 2 batched GraphQL queries (all PR bodies + all issue bodies) plus sequential REST calls for comments. Typically ~35 API calls for 17 PRs.

### Step 5 — Draft the blog post
The next_version should be the next minor semantic version from the `{version}` tag. 
Read `release-blog/research.json` and write `release-blog/blog_post_{next_version}.md`.

**Structure:**
```
# Argo CD {next_version} Release Candidate
---
[Intro: 2 paragraphs]
## [Featured Highlight title]   ← one section per Blog Highlight PR
## [Feature group or individual feature]  ← New Features (grouped by theme)
## Notable Bug Fixes             ← brief prose + bullet list for top 5
## Thank You / Conclusion
```

**Per-feature formula** (keep concise, group like features together):
1. **Problem** — one sentence describing the pain point, grounded in the issue description
2. **Solution** — one or two sentences on what was built
3. **How to use** — config snippet, CLI example, or annotation — whatever is actionable, make it concise
4. **Feature Group** - close each feature group by thanking contributor(s) for feature use contributor github handle with link to their github profile


**Style:**
- Tone: enthusiastic and community-oriented but not too over the top ("The community is excited to announce...") but not ("this feature is amazing or significant..")
- H2 per section, H3 for sub-features within a grouped section
- Include YAML/bash snippets for any feature with a config surface
- No trailing summary paragraph per section — end each cleanly
- Dependency Updates, Maintenance, Other Bug Fixes, and Documentation (unless notable) are **not** included in the blog post
- No non-user facing techinical details
- Don't link to PRs

**Intro paragraph formula:**
> "The Argo CD community is thrilled to announce Argo CD vX.Y! This release delivers [N] new features and [N] bug fixes, with highlights including [2–3 themes]. We encourage the community to test the release candidate and share feedback before the final release."

**Conclusion** - add this to the bottom of the article
```
## Where Can I Get the New Release?
For installation instructions and the full changelog, check out the [release notes](https://github.com/argoproj/argo-cd/releases).

We’d love to hear your feedback! Find us on the [#argo-cd](https://cloud-native.slack.com/archives/C01TSERG0KZ) channel in [CNCF Slack](https://slack.cncf.io/) to share your experience, report issues, or just say hi.

---

## Thank You

A huge thanks to all Argo Community contributors and users for their contributions, feedback, and help in testing the release!
```

## Categorization Rules

Classification order — **first match wins**:

| Priority | Category | Rule |
|----------|----------|------|
| 1 | Blog Highlights | Any label matching `for-release-blog*` |
| 2 | New Features | Title prefix `feat:` or `feat(*):` |
| 3 | Notable Bug Fixes | `fix:` prefix AND (linked issues OR `major` label OR any `cherry-pick/*` label) |
| 4 | Other Bug Fixes | Remaining `fix:` prefixes |
| 5 | Documentation | `docs:` prefix or `[Bot] docs:` |
| 6 | Dependency Updates | `chore(deps):` prefix OR `dependencies` label |
| 7 | Maintenance | Everything else (`chore:`, `refactor:`, `ci:`, `test:`) |

## Scoring (within category, descending)

| Signal | Points |
|--------|--------|
| `for-release-blog*` label | +100 |
| `major` label | +50 |
| Has linked issues | +20 |
| Any `cherry-pick/*` | -10 |
| `component:*` label | +10 |
| `type:scalability` +10 |
| `type:tech-debt` | -5 |

Tiebreaker: ascending PR number.

## Common Heuristics

- **Hydrator PRs** often cluster — group them as "Source Hydrator Improvements" rather than listing individually
- **ApplicationSet UI features** (list page, tree view, slide-out, watch API) form a coherent story — group them
- **Score-0 features** are not necessarily unimportant — scan their titles for user-facing value before excluding
- **CI/tooling fixes** (prefix `fix(ci):`) almost never belong in blog posts regardless of score
- **`chore(deps):` PRs** in Blog Highlights indicate a dependency with notable new capabilities — ask the user if they know what's new, or check the dependency's GitHub releases

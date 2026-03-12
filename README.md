# release-blog skill for Claude Code

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that automates drafting release blog posts from GitHub commit history. Given a repo and a release tag, it walks the commit log, associates commits with PRs and linked issues, scores and categorizes changes by blog-worthiness, and drafts a structured, publication-ready blog post.

## Installation

Copy the `.claude/` directory from this repo into your project root (or your home directory for global availability):

```bash
cp -r .claude/ /path/to/your/project/
```

Claude Code will automatically discover skills placed under `.claude/skills/`.

## Usage

Trigger the skill with natural language in Claude Code:

- "Write a release blog post for v2.14"
- "Generate release notes blog for argoproj/argo-cd v2.13.0"
- `/release-blog`

Claude will infer the repo and tag from your git context where possible, then run the pipeline interactively.

## How It Works

The skill runs a 5-step pipeline:

| Step | Command | What it does |
|------|---------|--------------|
| 1 | `commits` | Extracts commits not yet in the release tag; parses PR numbers from commit subjects (~97% hit rate); batch-validates against GitHub; excludes fork/deleted PRs automatically |
| 2 | `fetch-prs` | Fetches PR metadata via GraphQL in batches of 50 |
| 3 | `categorize` | Classifies PRs (Blog Highlights, New Features, Bug Fixes, Docs, etc.) and scores by blog-worthiness |
| 4 | `research` | Deep-fetches PR bodies, linked issues, and comments for selected PRs |
| 5 | _(draft)_ | Claude writes the structured blog post (named for the next minor version) |

After step 3, Claude presents a summary table with proposed selections — Blog Highlights (all), top 10 New Features, top 5 Notable Bug Fixes — and asks you to confirm or adjust before fetching full context and writing the post. Related PRs are grouped into narrative sections (e.g., multiple ApplicationSet UI features or Hydrator fixes).

If `release-blog/research.json` already exists in your working directory, the skill will offer to skip straight to drafting without re-running the pipeline.

### Fork / upstream workflows

If your local repo is a fork, pass `--remote upstream` to the `commits` command so the tool uses the canonical upstream remote rather than `origin`.

### Pipeline tool

The underlying CLI tool lives at `.claude/skills/release-blog/tools/release-pipeline`. It requires no dependencies beyond the Python standard library and the GitHub CLI. All intermediate files are written into a `release-blog/` subdirectory.

```bash
uv run ./tools/release-pipeline commits   --repo owner/repo --tag vX.Y.Z -o release-blog/commits.json
uv run ./tools/release-pipeline fetch-prs --repo owner/repo --input release-blog/commits.json -o release-blog/prs_detailed.md
uv run ./tools/release-pipeline categorize --input release-blog/prs_detailed.md -o release-blog/prs_categorized.md
uv run ./tools/release-pipeline research  --repo owner/repo --prs 1234,5678 -o release-blog/research.json
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [GitHub CLI](https://cli.github.com/) (`gh`), authenticated via `gh auth login`
- Python 3.11+
- [`uv`](https://github.com/astral-sh/uv) (used as a script runner; no virtualenv setup needed)

## License

MIT — see [LICENSE](LICENSE).

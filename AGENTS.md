# Agent guide

This repository holds the MCP Builders program rules and skill submissions. It does not contain the Pipefy MCP server, CLI, or install docs. For setup, product context, and toolkit skills, use the [Pipefy AI Toolkit](https://github.com/pipefy/ai-toolkit).

## Read first

- [CONTRIBUTING.md](CONTRIBUTING.md) — how people submit skills, contact info, folder rules, and review outcomes
- [docs/writing-a-skill.md](docs/writing-a-skill.md) — skill content requirements
- [docs/judging.md](docs/judging.md) — rubric and acceptance criteria
- [template/](template/) — `SKILL.md` and `EVIDENCE.md` starters

## Submission layout

```
submissions/<github-handle>/<skill-name>/
├── SKILL.md
├── EVIDENCE.md
└── assets/          # optional screenshots referenced from EVIDENCE.md
```

- `<github-handle>` is the submitter's GitHub username, lowercase.
- `<skill-name>` is kebab-case and must match the `name` field in the `SKILL.md` frontmatter.

## Validation

Every pull request that touches `submissions/` runs the validator in CI (`.github/workflows/validate-submission.yml`).

Run the same check locally:

```bash
python3 scripts/validate_submission.py submissions/<github-handle>/<skill-name>
```

## Hard rules

- Never commit tokens, API keys, real organization or pipe IDs, customer names, or personal data.
- Keep contact email in the issue or PR description only — do not add it to submission files.
- Do not duplicate install or MCP setup docs here; link to [pipefy/ai-toolkit](https://github.com/pipefy/ai-toolkit) instead.

## Converting issues to pull requests

When turning a [skill submission issue](.github/ISSUE_TEMPLATE/submit-skill.yml) into a PR:

1. Create the folder at `submissions/<issue-author>/<skill-name>/`.
2. Commit `SKILL.md` and `EVIDENCE.md` with the issue author as the commit author.
3. Use the contact email from the issue for follow-up; do not write it into the repo.

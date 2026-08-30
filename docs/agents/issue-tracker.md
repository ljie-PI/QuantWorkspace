# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues. Use the `gh` CLI for repository issue operations.

## Conventions

- Create, read, list, comment on, label, and close issues with the corresponding `gh issue` commands.
- Infer `ljie-PI/QuantWorkspace` from the current Git remote.
- Read issue comments and labels when a skill needs the complete ticket context.

## Pull requests as a triage surface

**PRs as a request surface: no.**

GitHub shares one number space across issues and PRs. Resolve an ambiguous `#42` by checking the PR first, then the issue.

## Skill semantics

- "Publish to the issue tracker" means create a GitHub issue.
- "Fetch the relevant ticket" means read the GitHub issue and its comments.
- Wayfinder maps and child tickets use GitHub issues, sub-issues/dependencies when available, and task-list fallbacks otherwise.

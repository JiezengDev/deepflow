# CI/CD Tag Trigger Rule

This repository's packaging workflows are configured so that:

- Tag-triggered packaging/release jobs run **only** when the tagged commit is contained in `origin/develop-local`.
- If a tag points to a commit outside `develop-local`, the packaging job is skipped.

Current workflows with this rule:

- `.github/workflows/cli-build.yml`
- `.github/workflows/docker-compose-build.yml`

Implementation detail:

- A `check_develop_local_tag` job validates tag source with:
  - `git fetch origin develop-local`
  - `git merge-base --is-ancestor <tag_commit> origin/develop-local`
- If `origin/develop-local` is unavailable, tag packaging is skipped (safe default).

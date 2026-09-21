# thermal-label `.github`

Org-level metadata, community files, and contributor guides for
**[thermal-label](https://github.com/thermal-label)**.

## What's here

| Path | What |
|---|---|
| [`profile/README.md`](./profile/README.md) | The public org profile shown on github.com/thermal-label |
| [`profile/architecture.md`](./profile/architecture.md) | Architecture diagram (Mermaid) |
| [`.github/CODE_OF_CONDUCT.md`](./.github/CODE_OF_CONDUCT.md) | Org-wide code of conduct |
| [`.github/SECURITY.md`](./.github/SECURITY.md) | Security policy + private-disclosure flow |
| [`.github/PULL_REQUEST_TEMPLATE.md`](./.github/PULL_REQUEST_TEMPLATE.md) | Default PR template (inherited by every repo) |
| [`.github/ISSUE_TEMPLATE/`](./.github/ISSUE_TEMPLATE) | Default issue templates: bug, feature, question, hardware verification, new device |
| [`.github/FUNDING.yml`](./.github/FUNDING.yml) | GitHub Sponsors + Ko-fi |
| [`CONTRIBUTING/`](./CONTRIBUTING) | Contributor guides — index, adding a driver (stub), release process, docs conventions |
| [`plans/`](./plans) | Cross-repo plans archive (`implemented/`, `backlog/`) |

## How org defaults work

GitHub uses files in this `.github` repo as fallback for any repository in the
**thermal-label** org that doesn't define its own copy. So:

- A repo without a `CODE_OF_CONDUCT.md` shows the one from this repo.
- A repo without `.github/ISSUE_TEMPLATE/` shows the templates from this repo.
- A repo without `.github/PULL_REQUEST_TEMPLATE.md` shows the one from this repo.

Per-repo overrides win when present. Drivers and the CLI deliberately do not
override — they inherit from here so templates stay in sync.

## Questions and ideas

Everything goes through Issues; Discussions is not enabled. Bug reports,
feature requests, and questions go to the repo that owns the package (the
**Question** template needs nothing but the question). Anything org-wide
goes on this repo.

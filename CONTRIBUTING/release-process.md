# Release process

> **Status:** captures the current manual flow. Tooling (changesets or similar)
> is an open decision tracked in the docs plan; this guide is rewritten when
> that lands.

## Versioning

Pre-1.0. Breaking changes are allowed in minor bumps; document them in the
package's `DECISIONS.md` and call them out in release notes.

No automated changesets workflow yet. Each release is a manual `pnpm version`
in the affected packages.

## Per-package publish flow

For each package being released:

1. Land all PRs to `main`. Make sure CI is green.
2. Update `CHANGELOG.md` (when a repo has one — many don't yet).
3. `pnpm version <patch|minor|major>` in the package directory.
4. `pnpm publish --access public` — npm public scope.
5. `git push --follow-tags origin main`.

## Cross-package ordering

The dependency arrows in this ecosystem flow:

```
contracts ← transport ← <driver>-core ← <driver>-node / <driver>-web ← cli
```

When a release crosses repo boundaries:

1. **`contracts`** first. Bump and publish before anyone consumes a new symbol.
2. **`transport`** next, if its peer-dep on contracts changed.
3. **Driver `core`**, then **`node`** + **`web`** for that driver.
4. **`cli`** last, picking up new driver versions in its `dependencies`.

## Tagging

Per-package tags use the convention `<package-name>-v<version>`, e.g.
`brother-ql-core-v0.3.0`. The docs site uses these tags to pin which version
of `docs/` to pull (see [docs-conventions.md](./docs-conventions.md)).

## Triggering a docs rebuild

After a successful publish, the source repo's release workflow dispatches a
`repository_dispatch` event to `thermal-label.github.io` with the repo name
and tag. The docs site rebuilds against the new content.

If the dispatch fails, you can manually trigger from the docs site's
**Actions** tab using `workflow_dispatch`. The docs build itself also has a
nightly fallback for content drift on `main` (off by default — flip on if
needed).

## Pre-push hook

Each repo includes a `pre-push` git hook that runs `pnpm docs:api` (typedoc)
to ensure `docs/api/` is up-to-date before changes land on the remote. The
hook is wired up automatically on `pnpm install` via the `prepare` script —
no manual setup needed. If the hook fails, fix the cause and re-stage; do not
bypass with `--no-verify` unless the failure is genuinely unrelated to your
push.

## Unpublishing

Avoid. If a published version must come down, use `npm deprecate` with a clear
message pointing at the replacement. Outright `npm unpublish` breaks any
downstream lockfile and is a last resort.

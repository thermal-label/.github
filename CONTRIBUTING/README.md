# Contributing to thermal-label

Thanks for your interest in helping out. This folder is the index — pick the
guide that matches what you want to do.

## Where things live

| Concern | Where |
|---|---|
| Bug reports for a specific package | The owning repository's **Issues** tab |
| Questions and ideas | The closest repository's **Issues** tab (Question template); org-wide topics on [`.github`](https://github.com/thermal-label/.github/issues) |
| Hardware verification reports | The driver repo's Issues tab (template provided) |
| Security vulnerabilities | [`SECURITY.md`](../.github/SECURITY.md) — private advisory flow |
| User-facing documentation | The owning repo's `docs/` folder. The org docs site at [thermal-label.github.io](https://thermal-label.github.io) pulls those at build time. |
| Plans / decisions / progress logs | The owning repo's root (`PROGRESS.md`, `DECISIONS.md`) and `plans/` folder |

## Repository map

| Repo | Role |
|---|---|
| [`contracts`](https://github.com/thermal-label/contracts) | Shared TypeScript interfaces — `Transport`, `PrinterAdapter`, `PrinterDiscovery`, `MediaDescriptor`, `PrinterStatus`, structured errors. Single import surface. |
| [`transport`](https://github.com/thermal-label/transport) | Byte-channel implementations: `UsbTransport`, `TcpTransport` (Node), `WebUsbTransport`, `WebBluetoothTransport`, `WebSerialTransport` (browser). |
| [`brother-ql`](https://github.com/thermal-label/brother-ql) | Brother QL series driver — `core` / `node` / `web` packages. |
| [`labelmanager`](https://github.com/thermal-label/labelmanager) | DYMO LabelManager (D1 / WebHID) driver. |
| [`labelwriter`](https://github.com/thermal-label/labelwriter) | DYMO LabelWriter driver. |
| [`cli`](https://github.com/thermal-label/cli) | `thermal-label-cli` — unified CLI that aggregates every installed driver. |
| [`thermal-label.github.io`](https://github.com/thermal-label/thermal-label.github.io) | Public docs site (VitePress). Consumes per-repo `docs/` markdown at build time. |
| [`.github`](https://github.com/thermal-label/.github) | Org profile, default community files (this repo). |

## Guides in this folder

- [**Adding a driver**](./adding-a-driver.md) — how to wire a new printer family
  into the layered architecture.
- [**Verifying hardware**](./verifying-hardware.md) — what to run on your
  printer and how to file a verification report.
- [**Hardware-status schema**](./hardware-status-schema.md) — canonical
  schema for the per-driver `docs/hardware-status.yaml` files.
- [**Maintainer runbook**](./maintainer-runbook.md) — operational guide
  for processing verification reports, adding devices, and keeping the
  docs site in sync.
- [**Release process**](./release-process.md) — how packages get bumped and
  published.
- [**Docs conventions**](./docs-conventions.md) — what each repo's `docs/`
  folder must look like so the docs site can pull it cleanly.

## Quick contributor checklist

Before opening a PR:

- Run `pnpm lint`, `pnpm typecheck`, and `pnpm test` in the touched packages.
- Don't commit `.claude/`, scratch screenshots, or local sanity scripts —
  every repo gitignores `.claude/`.
- For public-API changes: drop a short note in the repo's `DECISIONS.md` so
  future-you remembers the why.
- Pre-1.0: breaking changes are allowed, just call them out in the PR body.

## Code of Conduct

By contributing you agree to abide by the
[Code of Conduct](../.github/CODE_OF_CONDUCT.md).

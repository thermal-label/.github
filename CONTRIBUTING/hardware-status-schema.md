# `hardware-status.yaml` schema reference

This document is the single source of truth for the `hardware-status.yaml`
files shipped in each driver repo. Each driver carries its own validator
script (`scripts/validate-hardware-status.mjs`) that enforces the rules
below; if the schema changes, every driver's validator updates with it.

> **TL;DR for contributors filing a verification:** you don't edit this
> file. Open a [Hardware verification issue](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/hardware_verification.yml)
> on the relevant driver repo. A maintainer translates the issue into a
> YAML edit. See [verifying-hardware.md](./verifying-hardware.md).

> **TL;DR for maintainers:** when an accepted verification arrives, append
> a row to `reports[]`, recompute the rolled-up `status` /
> `lastVerified` / `packageVersion`, and run
> `pnpm validate:hardware-status` before pushing.

---

## File location

`docs/hardware-status.yaml` at the root of the driver repo. The existing
`pull-driver-docs.mjs` in `thermal-label.github.io` copies the entire
`docs/` tree, so the file rides along with no puller changes.

VitePress treats `.yaml` files as static assets, not routes — they're
not rendered as pages and don't show up in the sidebar.

## Top-level fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `schemaVersion` | integer | yes | Currently `1`. Bump on incompatible changes. |
| `driver` | string | yes | Matches the driver repo name (`brother-ql`, `labelmanager`, …). |
| `devices` | list | yes | One entry per device that has at least one verification report. Devices in `DEVICES` not listed here render as **untested** on the docs site. |

## `devices[]` entry

| Field | Type | Required | Notes |
|---|---|---|---|
| `pid` | integer (hex) | yes | Matches a `pid` value in the driver core's `DEVICES` registry. |
| `name` | string | yes | Display name. **Cached** from `DEVICES[byPid].name`; the validator enforces equality (write-through cache, see [I4](../plans/implemented/driver-and-hardware-ecosystem-DECISIONS.md#i4)). |
| `status` | enum | yes | `verified` \| `partial` \| `broken` \| `untested`. |
| `transports` | mapping | no | Per-transport status. Keys ∈ `usb`, `tcp`, `webusb`, `web-bluetooth`, `web-serial`, `serial`. Values ∈ same status enum. Only transports declared in `DEVICES[byPid].transports` may appear. Omit a key for "n/a". |
| `lastVerified` | ISO date | yes | `YYYY-MM-DD`. Must be ≥ the latest `reports[].date`. |
| `packageVersion` | semver | yes | The `@thermal-label/<driver>-node` (or `-web`) version exercised. Pre-release tags allowed (e.g. `0.3.0-beta.1`). |
| `quirks` | markdown | no | Editorial — written by maintainers, not derived from reports. Renders as a prominent callout. See "`quirks` vs `notes`" below. |
| `notes` | markdown | no | Rolled-up summary of the reports. Renders as a footnote / tooltip. |
| `reports` | list | yes (may be empty) | Verification report rows. |

## `reports[]` entry

| Field | Type | Required | Notes |
|---|---|---|---|
| `issue` | integer | yes | GitHub issue number on the driver repo. Unique within the file. |
| `reporter` | string | yes | GitHub `@handle` (with `@`). For self-verified maintainer reports, see `selfVerified` below. |
| `date` | ISO date | yes | When the verification ran. |
| `result` | enum | yes | One of the four status values. |
| `os` | enum | no | `Linux` \| `macOS` \| `Windows`. |
| `notes` | string | no | Free-form report notes from the issue. |
| `selfVerified` | boolean | no | `true` when the reporter is a maintainer verifying on their own bench. Defaults to `false`. See [I7](../plans/implemented/driver-and-hardware-ecosystem-DECISIONS.md#i7). |

## Status semantics

| Value | Meaning |
|---|---|
| `verified` | At least one accepted report shows the device working end-to-end on at least one transport. |
| `partial` | Some transports work, others don't, or specific capabilities (two-color, auto-cut, etc.) fail. |
| `broken` | Reproducible failure on the most recent published version. |
| `untested` | Device is in `DEVICES` but no report has been filed. Computed at docs-build time — **don't write this status into the YAML**. Devices with no entry default to untested. |

## `quirks` vs `notes`

Two distinct fields. Keep them apart.

| Field | Who writes it | When it changes | Renders as |
|---|---|---|---|
| `quirks` | Maintainer, editorial | Only when the device's underlying behaviour changes (firmware, hardware revision, new mode discovered) | A prominent callout on the docs site — users see it before reading the table row |
| `notes` | Rolled up from verification reports + maintainer summary | Each time a new report lands | A footnote / tooltip on the table row |

Use `quirks` for: hybrid devices with multiple USB interfaces; devices
that boot into a different mode by default and need out-of-band setup
(`usb_modeswitch`); models sharing VID/PID with another model;
firmware-version-dependent behaviour; hardware-locked media; transport
oddities; anything a buyer should know **before** committing.

Use `notes` for: per-feature confirmations ("two-colour DK-22251
confirmed"), transport coverage gaps ("TCP not exercised by reporter"),
follow-up issue refs.

## Update flow

1. Verification issue lands on a driver repo with the `hardware` label.
2. Maintainer reviews. If accepted: open a PR.
3. PR appends to `reports[]`, updates rolled-up `status` /
   `lastVerified` / `packageVersion`, runs `pnpm validate:hardware-status`.
4. Merge → `repository_dispatch` to docs site → `/hardware/` page rebuilds.
5. Close issue with `verified` (or `partial` / `broken`) label.

## Validator rules

The validator script (`scripts/validate-hardware-status.mjs` in each
driver repo) enforces:

1. `schemaVersion === 1`.
2. `driver` matches the driver core's `family` field.
3. Each `devices[i].pid` exists in `DEVICES` (matched by `.pid` value).
4. `devices[i].name === DEVICES[byPid].name`.
5. `devices[i].status` ∈ status enum (excluding `untested` — that's
   never written, only computed).
6. Every `transports` key is in `DEVICES[byPid].transports`.
7. Every `transports` value ∈ status enum.
8. `lastVerified` parses as `YYYY-MM-DD`.
9. `packageVersion` parses as semver (`major.minor.patch[-prerelease]`).
10. `reports[].issue` is unique across the entire file.
11. `lastVerified >= max(reports[].date)`.
12. No duplicate `pid` entries across `devices[]`.

A clean run prints `OK — N devices, M reports`. A failure prints the
offending row and exits non-zero.

## Example

```yaml
schemaVersion: 1
driver: brother-ql

devices:
  - pid: 0x20a7
    name: QL-820NWB
    status: verified
    transports:
      usb: verified
      tcp: verified
      webusb: verified
    lastVerified: 2026-04-15
    packageVersion: '0.2.1'
    notes: |
      Two-color printing on DK-22251 confirmed. Auto-cut works on USB
      and TCP.
    reports:
      - issue: 42
        reporter: '@example-user'
        date: 2026-04-15
        result: verified
        os: Linux
        notes: 'TCP via wireless; udev rule applied.'

  - pid: 0x20a8
    name: QL-1100
    status: verified
    transports:
      usb: verified
    lastVerified: 2026-04-20
    packageVersion: '0.2.1'
    quirks: |
      Boots in **mass-storage mode** by default. Linux users typically
      apply a `usb_modeswitch` rule to make the device come up in label
      mode directly. The driver detects mass-storage mode via
      `isMassStorageMode()` and surfaces a clear error.
    reports:
      - issue: 11
        reporter: '@maintainer'
        date: 2026-04-20
        result: verified
        os: Linux
        selfVerified: true
        notes: 'Bench verification on usb_modeswitch-applied unit.'
```

## Schema versioning

`schemaVersion: 1` is current. When a future change is incompatible
(new required field, renamed key, changed semantics), bump the version
and update each driver's validator simultaneously. Keep one version of
the validator code per repo — drivers never serve two schema versions.

# Plan — driver authoring + hardware coverage system

Date: 2026-04-27 (rev 2026-04-27 — expanded from original "adding a driver" stub)
Status: backlog

This plan covers two intertwined pieces of the **thermal-label** ecosystem:

1. **The contributor guide for adding a new driver** (the original spin-off
   from `THERMAL_LABEL_DOCS_PLAN.md` §6.6).
2. **A unified hardware coverage system** — single source of truth for
   *what devices are supported*, *what state they're in (verified,
   partial, broken)*, and *how community members report results*.

These belong in one plan because driver authors care about both: a new
driver lands with a small DEVICES registry and zero verification reports,
and grows over time as the community tests it. The same data shapes and
workflows serve "I just added the LabelManager 360D" and "I just verified
my QL-820NWB on a fresh checkout".

---

## Decisions locked in

Captured from the planning conversation (2026-04-27):

| # | Decision | Why |
|---|---|---|
| D1 | **Validation state lives in `hardware-status.yaml` per driver repo.** | Keeps device truth next to device code. Maintainers update it from verification issues; the docs site pulls it. |
| D2 | **Docs site grows a top-level `/hardware/` page** aggregating every driver's devices. | Single URL listing every supported device with family, status, last-verified date. Per-driver `/<repo>/hardware` stays as the deep view. |
| D3 | **Submission stays low-tech for v1.** | Issue template + thorough guide + maintainer process. CLI helper and GitHub Action automation are logged as future improvements (see §10). |

---

## Goals

1. **Lower the bar to add a driver.** A new contributor with a printer and
   curiosity should be able to follow one walkthrough from "I have a
   device" to "my driver is on npm".
2. **Make hardware coverage legible.** Anyone evaluating thermal-label for
   a project can see at a glance which printers are tested, on which
   transports, and how recently — across all drivers in one place.
3. **Make it easy to be a verifier.** A volunteer with a printer should
   need to read one guide, run a small set of commands, and click one issue
   template button — everything else handled by maintainers.
4. **Single source of truth per concern.**
   - Code-level device support → the driver's `core/src/devices.ts`
     (`DEVICES` export).
   - Verification state → the driver's `hardware-status.yaml`.
   - User-facing list → generated on the docs site from the two above.

---

## Scope

**In scope:**

- Writing the full `CONTRIBUTING/adding-a-driver.md` guide (replacing the
  current stub).
- Defining the `hardware-status.yaml` schema and seeding the file in each
  driver repo.
- Building the unified `/hardware/` page on the docs site.
- Per-driver `/<repo>/hardware` page enhancements (status badges, transport
  matrix).
- Writing `CONTRIBUTING/verifying-hardware.md` (the volunteer's guide) and
  per-driver `docs/verification-checklist.md`.
- The maintainer flow: verification issue → triage → `hardware-status.yaml`
  PR.

**Out of scope (logged for later):**

- `thermal-label verify` CLI subcommand.
- GitHub Action that auto-PRs `hardware-status.yaml` from verification
  issues.
- Historical chart of verification activity over time.
- Cross-driver "compatibility matrix" content (transport × OS × printer)
  beyond what falls out of the unified page.

---

## Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│  per driver repo (brother-ql, labelmanager, labelwriter, …)       │
│                                                                   │
│   packages/core/src/devices.ts       ← supported devices (code)   │
│   packages/core/src/media.ts         ← supported media (code)     │
│   docs/hardware-status.yaml          ← verified state (data)      │
│   docs/verification-checklist.md     ← what to test (markdown)    │
│   docs/hardware.md                   ← human reference (markdown) │
└───────────────────────────────────────────────────────────────────┘
                              │
              pulled at build time by
              scripts/pull-driver-docs.mjs
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  thermal-label.github.io                                          │
│                                                                   │
│   docs/hardware/index.md             ← unified page (generated)   │
│   docs/<repo>/hardware.md            ← per-driver (pulled +       │
│                                        injected status table)     │
│   scripts/build-hardware-page.mjs    ← merges DEVICES (npm) +     │
│                                        hardware-status.yaml       │
│                                        per repo into one page     │
└───────────────────────────────────────────────────────────────────┘
                              ▲
                              │
              issue gets filed via
              .github/ISSUE_TEMPLATE/hardware_verification.yml
                              │
┌───────────────────────────────────────────────────────────────────┐
│  thermal-label/.github                                            │
│                                                                   │
│   .github/ISSUE_TEMPLATE/hardware_verification.yml  ← form        │
│   CONTRIBUTING/verifying-hardware.md                ← guide       │
│   CONTRIBUTING/adding-a-driver.md                   ← driver guide│
└───────────────────────────────────────────────────────────────────┘
```

---

## §A — `hardware-status.yaml` schema

Lives at `docs/hardware-status.yaml` in each driver repo (so the existing
puller picks it up alongside the markdown).

```yaml
# hardware-status.yaml — community-verified status per device.
#
# This file is the source of truth for verification state. The DEVICES
# registry in @thermal-label/<driver>-core is the source of truth for
# *expected* / *supported* devices; this file tracks *reality*.
#
# Devices appear here once at least one verification report has been
# filed. Devices in DEVICES that aren't listed here render as "untested"
# on the docs site.
#
# Schema version 1 — bump when the shape changes incompatibly.
schemaVersion: 1
driver: brother-ql

devices:
  - pid: 0x209d                         # required, matches DEVICES[].pid
    name: QL-820NWB                     # display name (cached from DEVICES for offline lookup)
    status: verified                    # verified | partial | broken | untested
    transports:                         # per-transport status; omit a key for "n/a"
      usb: verified
      tcp: verified
      webusb: verified
    lastVerified: 2026-04-15            # ISO date
    packageVersion: '0.2.0'             # of @thermal-label/<driver>-node at time of report
    notes: |                            # optional, verification observations; markdown OK
      Two-color printing on DK-22251 confirmed. Auto-cut works on USB and TCP.
    reports:                            # one entry per accepted verification issue
      - issue: 42
        reporter: '@user1'              # GitHub handle, with '@'
        date: 2026-04-15
        result: verified
        os: Linux
        notes: 'TCP via wireless; udev rule applied.'

  - pid: 0x2044
    name: QL-710W
    status: partial
    transports:
      usb: verified
      tcp: untested
    lastVerified: 2026-03-10
    packageVersion: '0.2.0'
    notes: |
      USB confirmed. TCP path not exercised by the reporter.
    reports:
      - issue: 38
        reporter: '@user2'
        date: 2026-03-10
        result: partial
        os: macOS

  # Example of a "Frankenstein" device that needs editorial context. The
  # `quirks` field is written by maintainers (not derived from reports)
  # and renders as a prominent callout on the docs site so users know to
  # read it before buying or wiring up the device.
  - pid: 0x1001
    name: LabelManager PnP
    status: verified
    transports:
      usb: verified
    lastVerified: 2026-04-20
    packageVersion: '0.2.0'
    quirks: |
      Boots in **mass-storage mode** by default — Windows / macOS auto-mount
      a virtual CD with the vendor's installer, which steals the USB
      interface. The driver detects this and exposes `isMassStorageMode()`;
      Linux users typically apply a `usb_modeswitch` rule shipped in the
      driver repo's `udev/` folder so the device boots straight into label
      mode. See `docs/hardware.md#mass-storage-mode` for details.
    notes: |
      Verified end-to-end on Linux after `usb_modeswitch` was applied.
```

### Status semantics

| Value | Meaning |
|---|---|
| `verified` | At least one accepted report shows the device working end-to-end on at least one transport. |
| `partial` | Some transports work, others don't, or specific capabilities (e.g. two-color, auto-cut) fail. |
| `broken` | Reproducible failure on the most recent published version. |
| `untested` | In `DEVICES` (driver claims support) but no verification reports filed. |

### Transport status

Each transport key takes the same status values **plus** the implicit
"absent key = not applicable for this device" (e.g. a USB-only printer
omits `tcp`).

### `quirks` vs `notes`

Two distinct freeform fields at the device level. They serve different
purposes — keep them apart so the docs site can render them differently.

| Field | Who writes it | When it changes | Renders as |
|---|---|---|---|
| `quirks` | Maintainer, editorial | Only when the device's underlying behaviour changes (firmware, hardware revision, new mode discovered) | A prominent callout on the docs site — users see it before reading the table row |
| `notes` | Rolled up from verification reports + maintainer summary | Each time a new report lands and the rolled-up state is updated | A footnote / tooltip on the table row |

Use `quirks` for things like:

- Hybrid devices that present multiple USB interfaces (e.g. label
  printer + tape printer in one chassis, or printer + virtual CD-ROM)
- Devices that boot into a different mode by default and need
  out-of-band setup (`usb_modeswitch`, kernel module unload, …)
- Models that share VID/PID with another model but behave differently
- Firmware-version-dependent behaviour ("only 1.5+ supports auto-cut")
- Hardware-locked media (e.g. NFC-locked DYMO 550-series)
- Transport-specific oddities ("WebUSB pairing requires unplug-replug
  after first connect on Windows")
- Anything a buyer would want to know *before* committing to the device

Use `notes` for things like:

- "Confirmed two-color printing on DK-22251"
- "TCP not exercised by reporter; community contributions welcome"
- "Auto-cut works inconsistently on Windows — issue #57"

### Update flow

1. Verification issue lands.
2. Maintainer reviews; if accepted, opens a PR that adds an entry to
   `reports[]` and updates the rolled-up `status` / `lastVerified` /
   `packageVersion` fields.
3. PR merges → `repository_dispatch` fires → docs site rebuilds → new
   status visible on `/hardware/`.

### Validation

A small Node script (lives in each driver repo at
`scripts/validate-hardware-status.mjs`, or — better — once in a shared
place pulled in via the pre-push hook) validates:

- Every `pid` exists in `DEVICES`.
- `status` matches one of the four enum values; same for transport keys.
- `reports[].issue` is unique within the file.
- `lastVerified` >= the latest `reports[].date`.
- `packageVersion` parses as semver.

The pre-push hook (already in place per the docs-site cleanup) gains a
second check: if `hardware-status.yaml` changed, run the validator.

---

## §B — Device + media registry sync

Each driver's `*-core` package already exports `DEVICES` and `MEDIA` as
the source of truth for code-level support. The docs site already depends
on these packages (it imports them for the LiveDemo components).

**For the unified `/hardware/` page** the docs site can `import { DEVICES }
from '@thermal-label/<driver>-core'` at build time and merge with each
repo's pulled `hardware-status.yaml`.

**Caveat:** the npm-installed version may lag behind the source repo's
latest device additions. Two options:

- **(recommended) Pin the docs site's deps to the latest published version
  of each driver.** Bump on every driver release. The
  `repository_dispatch` payload from the driver's release workflow already
  pins to the released tag — extend it to also bump the docs site's
  package.json + lockfile via a small in-CI step (or just have the
  maintainer bump manually as part of the release dance).

- **Read from source.** The puller could parse `packages/core/src/devices.ts`
  via a TS-aware tool. Avoids the bump dance but adds AST parsing
  complexity. Skip until the manual bump becomes annoying.

---

## §C — Unified `/hardware/` page

New page at `docs/hardware/index.md` on the docs site. **Generated** by a
new build script `scripts/build-hardware-page.mjs` that:

1. Imports `DEVICES` from each `*-core` package in the docs site's deps.
2. Reads each pulled `docs/<repo>/hardware-status.yaml`.
3. Merges into one table:

| Family | Model | PID | Transports | Status | Last verified | Package version | Reports |
|---|---|---|---|---|---|---|---|
| Brother QL | QL-820NWB | 0x209d | USB, TCP, WebUSB | ✓ verified | 2026-04-15 | 0.2.0 | [#42] |
| Brother QL | QL-710W | 0x2044 | USB ✓ · TCP ?  | ⚠ partial | 2026-03-10 | 0.2.0 | [#38] |
| Brother QL | QL-820NWBc | 0x2098 | — | · untested | — | — | — |
| LabelWriter | LW 450 | 0x0020 | USB | ✓ verified | 2026-02-01 | 0.2.0 | [#11] |
| LabelManager | LM PnP | 0x1002 | USB | ✓ verified | 2026-04-20 | 0.2.0 | [#15] |

4. Writes `docs/hardware/index.md` with the table + filters explainer.

The page also gets:

- **Filter chips** (client-side) for family, status, transport.
- A **"verify your device"** call-to-action linking to the verification
  guide and the issue template.
- **Per-row links** to the per-driver `/<repo>/hardware` deep page.
- **A "quirks" indicator** on devices that have a `quirks:` entry — small
  badge in the row, expanding (or linking) to the full quirk text. Quirky
  devices also surface in a separate "Read these first" section above the
  main table for buyers who skim.

### Per-driver `/<repo>/hardware` page enhancements

Existing per-driver hardware markdown stays. The build script also
generates a small fragment that gets prepended (via VitePress markdown
include or by injection) showing **just that driver's** status table —
identical schema, narrower scope.

---

## §D — Verification guide for volunteers

Lives at `CONTRIBUTING/verifying-hardware.md` in the `.github` repo.

Outline:

1. **Why verify** — short context, what the maintainers do with the
   results.
2. **What you need** — the printer, a USB cable (or network), Node 24+,
   `thermal-label-cli`, the relevant `*-node` driver.
3. **Identify your device** — find VID/PID via `lsusb` (Linux/macOS) or
   Device Manager (Windows). Confirm it's on the family's `DEVICES` list
   (link to `/hardware/`).
4. **The standard sequence** (per family — drivers ship a
   `docs/verification-checklist.md` with the exact commands and expected
   outputs). Generally:
   - `thermal-label list` → device appears
   - `thermal-label status` → returns within timeout
   - `thermal-label print text "test"` (with appropriate `--media`
     for that family)
   - `thermal-label print image small.png`
   - For network-capable devices: same with `--host`
   - For browser support: open the live demo at `/demo/<family>`, pair,
     print
5. **Capability-specific tests** — two-color (Brother QL DK-22251), auto-cut,
   tape-width matrix (LabelManager), NFC-locked media (LabelWriter 550), …
6. **What "verified" / "partial" / "broken" mean** — link to §A status
   semantics.
7. **Submitting your report** — link to the issue template, with a
   screenshot/walkthrough of the form.
8. **What happens after you submit** — triage timeline, who updates the
   status file, when it appears on the docs site.

### Per-driver `docs/verification-checklist.md`

A short, family-specific checklist. New file in each driver repo.
Skeleton:

```markdown
# Verification checklist — Brother QL

These are the commands every Brother QL verification should run.
Capture the output (terminal + a photo of the printed label) and paste
into the verification issue.

## 1. Device is detected
\```bash
thermal-label list
\```
Expected: your printer appears with the right model name and PID.

## 2. Status is readable
\```bash
thermal-label status
\```
Expected: `Status: Ready`, no errors, detected media populated.

## 3. Print a text label
\```bash
thermal-label print text "verify $(date +%Y-%m-%d)" --media 259
\```
Expected: a sharp, readable label exits the printer.

## 4. (TCP-capable models) Print over network
\```bash
thermal-label print text "tcp test" --media 259 --host 192.0.2.42
\```
Expected: same as USB.

## 5. (QL-800 series) Two-color
\```bash
thermal-label print image two-color-test.png --media 251
\```
Expected: black + red render correctly on DK-22251.

## 6. (Browser) WebUSB demo
Open https://thermal-label.github.io/demo/brother-ql, pair, print.
Expected: same label as step 3.
```

---

## §E — Driver authoring guide

This is the original "adding a driver" content, expanded. Lives at
`CONTRIBUTING/adding-a-driver.md` (replacing the current stub).

Outline:

1. **What "thermal-label driver" means** — the layered architecture, where
   your code fits, what you get for free, what you have to provide.
2. **Pick a printer** — what kind of device works well here (USB raster,
   raw TCP, WebUSB-friendly), what doesn't.
3. **Bootstrap the repo** — directory layout, `pnpm-workspace.yaml`,
   `package.json` per workspace package, tsconfig conventions, the
   `.githooks/pre-push` + `.gitattributes` setup.
4. **The `core` package — protocol layer**
   - Implement `encodeJob`, `parseStatus`, your `MediaDescriptor` subclass
   - Worked example: encoding a 1bpp raster job
   - Where to source byte-level documentation (manufacturer SDK, USB
     captures, reverse engineering)
5. **The `node` package — adapter + discovery**
   - Implement `PrinterAdapter` using `UsbTransport`
   - Implement and export `discovery: PrinterDiscovery` (singleton)
   - TCP discovery patterns
6. **The `web` package — browser adapter**
   - Implement using the appropriate browser transport
   - `requestPrinter()` pattern for browser pairing
   - Why no `discovery` in browser packages
7. **Status mapping conventions**
8. **Media + orientation** — `pickRotation()`, `DEFAULT_MEDIA`
9. **Testing**
   - Unit tests for `core` with byte fixtures
   - Mock-transport integration tests for `node`
10. **Hardware coverage from day one**
    - Seed `docs/hardware-status.yaml` with your bench-tested device(s)
    - Write `docs/verification-checklist.md` for your family — this is
      what volunteers will run
11. **Docs**
    - The required `docs/index.md`, `docs/getting-started.md`,
      `docs/hardware.md`, `docs/verification-checklist.md`
    - Wiring `pnpm docs:api` (typedoc)
    - Linking pattern: relative inside the repo, `/<other-repo>/...` across
      repos
12. **Publish + integrate**
    - Version order (core → node → web)
    - `repository_dispatch` to trigger docs rebuild
    - The CLI auto-discovers your driver — no CLI change required
13. **What to do on day 2**
    - File a hardware verification on your own repo (yes, against your
      own driver — bench results count)
    - Open Discussions on `.github` to surface the new driver
    - Maintainer ergonomics: PROGRESS.md, DECISIONS.md, plans/

---

## §F — Maintainer flow (verification issue → status update)

Documented inside `CONTRIBUTING/adding-a-driver.md` §maintainer-bits and
optionally as a tiny `CONTRIBUTING/maintainer-runbook.md`. The flow:

1. **Triage** — verification issue appears with the `hardware` label.
   Maintainer checks: device is on `DEVICES`? Reporter ran the
   checklist? Output looks plausible?
2. **PR `hardware-status.yaml`** — append to `reports[]`, update rolled-up
   `status` / `lastVerified` / `packageVersion`. Use the issue number in
   the PR title (`Verify QL-820NWB on 0.2.0 (#42)`).
3. **Reply on the issue** — link to the PR, thank the reporter.
4. **Merge** — the `repository_dispatch` to the docs site fires, the
   `/hardware/` page rebuilds, the entry shows up.
5. **Close the issue** — keep `hardware` + `verified` labels for the
   archive view.

---

## Implementation phases

Each phase is independently shippable.

### Phase 1 — schema + seed

1. Define `hardware-status.yaml` schema (this plan, §A).
2. Add the validator script.
3. Seed `hardware-status.yaml` in each driver repo with whatever the
   maintainer can attest to from their own bench. Empty `reports[]` for
   devices that haven't been formally verified is fine.

### Phase 2 — docs site unified page

1. Bump the docs site's `*-core` deps to latest published versions.
2. Write `scripts/build-hardware-page.mjs` (merges `DEVICES` + each repo's
   pulled `hardware-status.yaml` → markdown table).
3. Wire into `docs:build` after `docs:pull`.
4. Add `/hardware/` to the top-level nav.
5. Inject the per-driver status fragment into each `/<repo>/hardware`
   page.

### Phase 3 — verification guide

1. Write `CONTRIBUTING/verifying-hardware.md`.
2. Write per-driver `docs/verification-checklist.md` files.
3. Link from the existing `hardware_verification.yml` issue template's
   intro markdown ("read this guide first").
4. Link from the docs site's per-driver hardware page.

### Phase 4 — driver authoring guide

1. Replace `CONTRIBUTING/adding-a-driver.md` stub with the full guide
   from §E.
2. Cross-link to the verification material so a new driver author knows
   to seed `hardware-status.yaml` + write a `verification-checklist.md`
   on day one.

### Phase 5 — maintainer runbook

1. Write the verification-issue → PR → merge flow as either a section in
   the driver authoring guide or a standalone `CONTRIBUTING/maintainer-runbook.md`.

---

## Future improvements (logged, not in v1 scope)

These were deferred from the planning conversation. Re-evaluate once the
v1 system has been running for a few months.

### F1 — `thermal-label verify` CLI command

A subcommand that runs the canned sequence per driver (the same as the
checklist) and prints a markdown blob ready to paste into the issue:

```bash
$ thermal-label verify --printer brother-ql
Running checklist for Brother QL...
✓ list — QL-820NWB detected
✓ status — Ready
✓ print text — submitted (please confirm label printed)
…

--- paste the following into your verification issue ---

## Verification report

- Family: brother-ql
- Model: QL-820NWB (PID 0x209d)
- Transports tested: USB, TCP
…
```

Or, even better, opens a prefilled GitHub issue URL in the user's
browser via `?body=<encoded>`.

### F2 — GitHub Action that auto-PRs `hardware-status.yaml`

A workflow on the driver repo that triggers when an issue with the
`hardware` label is closed (or labelled `accepted`). Parses the issue
body via the structured fields the YAML template produces, opens a PR
adding the entry. Maintainer reviews and merges.

### F3 — Historical activity chart

A second page on the docs site showing verification activity over time
per driver — useful for spotting drivers that haven't seen recent
verification.

### F4 — Cross-driver compatibility matrix

A bigger view: transport × OS × printer family. Rare combinations may
warrant their own callout.

---

## Open questions

1. **Does `hardware-status.yaml` belong at `docs/hardware-status.yaml`
   (pulled by the existing puller) or at the repo root?** Repo-root
   would mean a separate pull pass; `docs/` is zero-config. **Lean
   toward `docs/`.**

2. **Should `untested` devices appear on the unified page?** Yes — the
   point is a complete picture. They render as a separate visual state
   ("Listed in DEVICES, no community report yet") with a CTA to verify.

3. **What package version do we record for "untested" devices?** Latest
   published. Computed at docs-site build time, not stored.

4. **What do we do with verification reports against unpublished or
   pre-release versions?** Accept them but mark the entry's
   `packageVersion` clearly (e.g. `0.3.0-beta.1`). The unified page
   shows the latest stable verification.

5. **How do we handle a device that gets re-verified after a regression
   fix?** Append a new `reports[]` entry; the rolled-up `status` reflects
   the latest. Old reports stay for history.

6. **Cap on `reports[]` length per device?** No cap initially. If the
   files get unwieldy, archive older reports to a per-device subfile.

7. **Localization of the verification guide / checklists?** English-only
   for v1.

---

## Definition of done

- `CONTRIBUTING/adding-a-driver.md` is the full guide (no longer a stub).
- `CONTRIBUTING/verifying-hardware.md` exists and is linked from the
  issue template.
- Each driver repo has a `docs/hardware-status.yaml` (even if minimal)
  and a `docs/verification-checklist.md`.
- The docs site has a `/hardware/` page generated from registries +
  status files; per-driver pages show family-specific status.
- The maintainer flow for accepting a verification issue is documented.
- This plan moves to `plans/implemented/`.
- F1–F4 stay in `plans/backlog/` as their own (much smaller) plans if
  pursued.

---

## Effort estimate

- Phase 1 (schema + seed): 1 sitting
- Phase 2 (unified page): 2 sittings (script + integration + styling)
- Phase 3 (verification guide): 1-2 sittings (depends on per-driver
  checklist depth)
- Phase 4 (driver authoring guide): 2-3 sittings of focused writing
- Phase 5 (maintainer runbook): 0.5 sitting

Total: **~7–9 sittings of focused work**, plus review.

The driver authoring guide is the largest single piece. The hardware
coverage system is a collection of small pieces that compose into
something useful as soon as Phase 2 lands.

---

## Sources to mine when writing

- `thermal-label/contracts/src/` — interface surface a driver implements
- `thermal-label/transport/src/` — what the driver gets for free
- The three driver repos' `DECISIONS.md` files — they encode the
  conventions a new driver should follow
- `plans/implemented/driver-retrofit.md` (in this `.github` repo) —
  captures the shared playbook the existing three drivers were built
  against
- The existing driver READMEs and `docs/getting-started.md` files —
  implicit examples

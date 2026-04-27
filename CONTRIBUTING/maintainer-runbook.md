# Maintainer runbook

Operational guide for the people who maintain a thermal-label driver
repo. Covers the recurring flows that aren't obvious from reading the
code: triaging hardware reports, adding devices, processing
verification issues, and keeping the docs site in sync.

> If you're authoring a *new* driver, read
> [adding-a-driver.md](./adding-a-driver.md) first. This runbook
> assumes the driver already exists.

---

## Triage: hardware verification issues

When a [Hardware verification](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/hardware_verification.yml)
issue lands on your driver repo with the `hardware` label:

### 1. Sanity-check

- Is the device on `DEVICES`? If not, redirect the reporter to the
  [New device support](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/new_device.yml)
  template — verification only applies to devices the driver claims
  to support.
- Did the reporter follow the family checklist
  (`docs/verification-checklist.md`)? Output that's missing the
  `list` / `status` / `print text` / `print image` steps is hard to
  act on. Ask politely for the missing pieces.
- Does the package version they tested match a real published release
  (or a clearly-named pre-release)?

### 2. Decide the result

Reporter's classification (`works` / `partial` / `broken`) is a
suggestion. Translate to the schema's enum:

| Reporter says | YAML `status` |
|---|---|
| Works perfectly | `verified` |
| Partially works | `partial` |
| Does not work | `broken` |

Use your judgment on edge cases. If TCP works but USB times out,
that's `partial` even if the reporter said "works perfectly" (it
doesn't on the only USB transport).

### 3. Open the YAML PR

```bash
git checkout -b verify-<model>-<short>
# edit docs/hardware-status.yaml
pnpm validate:hardware-status   # gate-check
git commit -m "Verify <model> on <pkg-version> (#<issue>)"
git push origin verify-<model>-<short>
gh pr create --fill
```

What goes in the YAML edit:

- **Append** a new entry to `reports[]` for the device's row. Don't
  modify or delete existing reports — they're history.
- **Recompute** the rolled-up `status`, `lastVerified`, and
  `packageVersion` from the latest accepted reports.
- Update `notes:` if the reporter surfaced something the rolled-up
  summary should mention (transport coverage gaps, capability
  failures).
- Leave `quirks:` alone unless the report reveals a new editorial
  caveat (firmware variants, hidden modes) — that's a maintainer
  decision, not driven by reports.

If this is the device's **first** report (entry doesn't exist yet),
add the full device entry. Use the schema doc's example as a
template.

### 4. Reply on the issue

Link the PR. Thank the reporter. Don't close the issue yet — keep it
open until the PR merges so the docs site rebuild reflects reality
before we declare it done.

### 5. Merge → close

After the PR merges:

- The driver repo's release workflow fires `repository_dispatch` to
  the docs site → `/hardware/` rebuilds → the new status appears.
- Close the issue with `verified` / `partial` / `broken` label
  attached (in addition to the existing `hardware` label).

### 6. Edge cases

**Reporter wants their handle removed.** Edit the YAML report's
`reporter` to `@anonymous-N` (next available `N`). Keep the row.
Mention in the PR that this was a privacy request.

**Re-verification of a device after a regression fix.** Append a new
`reports[]` row. The rolled-up `status` reflects the latest report.
Older reports stay for history.

**Reports against pre-release versions.** Accept them. Mark
`packageVersion` clearly (e.g. `0.3.0-beta.1`). The unified page
shows the latest stable verification — pre-releases are visible in
the expanded reports list.

**`reports[]` array growing very long.** No cap right now. If it ever
becomes unwieldy, archive older reports to a per-device subfile —
that's a future improvement, not v1 scope.

## Adding a device to an existing driver

Adding to `DEVICES` is a code change in the driver repo. The
verification system implications:

1. The new device immediately appears as **untested** on the unified
   `/hardware/` page (after the docs site bumps your `*-core` dep —
   see below).
2. You can pre-seed a verification entry in `hardware-status.yaml`
   if you've bench-tested it. Use `selfVerified: true`.
3. The new device shows up in CLI `list` immediately after users
   upgrade `*-node`.

**Visibility lag:** until the docs site bumps its pinned
`@thermal-label/<driver>-core` to your new release, the new device
is invisible on `/hardware/`. This is by design (per
[I3](../plans/implemented/driver-and-hardware-ecosystem-DECISIONS.md#i3))
— the docs site reads DEVICES from npm, not from source. Bump the
docs site as part of your release dance.

### Release flow with docs-site bump

1. `pnpm changeset` — describe the change.
2. `pnpm changeset version` — bump versions.
3. `pnpm -r build && pnpm -r publish --access public`.
4. `repository_dispatch` to `thermal-label/thermal-label.github.io`
   (existing release workflow handles this).
5. **Manually:** open a PR on the docs site bumping
   `@thermal-label/<driver>-core` in `package.json` to the new
   version. The unified hardware page picks up your new device on
   the next docs build.

If step 5 becomes annoying enough to automate, that's the right
moment to revisit decision I3.

## Editorial: writing `quirks`

The `quirks` field on a device's YAML entry is a prominent callout
on the unified page. Use it for things buyers should know **before**
committing to the device:

- Hybrid devices (label printer + tape printer in one chassis,
  printer + virtual CD-ROM)
- Devices that boot into a different mode by default (mass-storage,
  HID-vs-printer-class) and need out-of-band setup
- Models sharing VID/PID with another model but behaving differently
- Firmware-version-dependent behaviour (`only firmware ≥ 1.5
  supports auto-cut`)
- Hardware-locked media (NFC-locked DYMO 550-series)
- Transport-specific oddities (`WebUSB pairing requires unplug-
  replug after first connect on Windows`)

Don't use `quirks` for per-report observations — those go in the
report's `notes` field. The distinction:

- **`quirks`** answers "what's permanently true about this device?"
- **`notes`** answers "what did this round of testing find?"

A good `quirks` entry stays valid across firmware revisions and
package versions. If you find yourself rewriting it every release,
the content probably belongs in the driver's protocol code or in
`docs/hardware.md` instead.

## Keeping the docs site in sync

The docs site (`thermal-label.github.io`) pulls each driver's `docs/`
folder verbatim. You don't push to the docs site directly — you push
to your driver repo, the dispatch fires, the site rebuilds.

What lives in your driver's `docs/` that the docs site cares about:

- `index.md`, `getting-started.md`, `<everything>.md` — pulled as-is.
- `hardware-status.yaml` — read by the build script to render the
  unified table. Validated on push by your pre-push hook.
- `_status-fragment.md` (generated, not committed) — the docs site's
  build script writes it after the pull, then your `docs/hardware.md`
  picks it up via `<!--@include: ./_status-fragment.md-->`.

When you add a new doc page in your driver's `docs/`, also bump the
docs site's `docs/.vitepress/config.ts` sidebar to surface it. Open
a PR on the docs site for that.

## When you can't act on a verification report

Sometimes a report describes a problem you can't fix without the
hardware. Options:

- **Open a follow-up issue** in your driver repo titled clearly
  ("QL-1100 over TCP times out per #42"), label it `bug` and link
  the verification issue. Mark the device's status as `partial` in
  the YAML — that surfaces the gap on the unified page.
- **Ask for hardware donation** if the device is rare. The
  `HARDWARE.md` of each driver typically has a "donation /
  contribution" section with an issue template link.
- **Accept the gap.** If a single transport on a single device
  fails and the reporter doesn't have time to dig deeper, that's
  fine — `partial` with a clear `notes` block is honest data.

## Cadence

There's no SLA. A reasonable cadence:

- **Triage** within 3-7 days of a verification issue landing.
- **PR** within 14 days for a clean accepted report. Longer is fine
  if the report needs back-and-forth.
- **Bump the docs site's deps** within a release cycle of your
  driver release. Stale deps don't break anything but make the
  unified page look out of date.

If you're going to be away, set the repo's auto-reply on Issues or
add a banner to the README. Verification reports trickling in
without acknowledgement burns volunteer goodwill.

---

## Quick links

- [Verification schema](./hardware-status-schema.md)
- [Verifying hardware (volunteer guide)](./verifying-hardware.md)
- [Adding a driver](./adding-a-driver.md)
- [Release process](./release-process.md)
- [Docs conventions](./docs-conventions.md)

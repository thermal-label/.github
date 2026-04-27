# Adding a driver

This is the path for adding a new printer family to the **thermal-label**
ecosystem. It is _not_ for adding a new device to an existing driver —
for that, file a [New device support](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/new_device.yml)
issue on the relevant driver repo.

> The three reference drivers are the canonical worked examples:
> [`brother-ql`](https://github.com/thermal-label/brother-ql),
> [`labelmanager`](https://github.com/thermal-label/labelmanager),
> [`labelwriter`](https://github.com/thermal-label/labelwriter).
> When this guide says "see X" it usually means "open one of those
> repos and read X."

---

## 1. What "thermal-label driver" means

A driver lives in three layered packages, all in one monorepo:

```
<your-driver>/
  packages/
    core/    Pure protocol layer. No I/O. Encodes job bytes, parses
             status, exports DEVICES + MEDIA registries. Browser-safe.
    node/    Node integration. Composes core + @thermal-label/transport/node.
             Implements PrinterAdapter + PrinterDiscovery.
    web/     Browser integration. Composes core + @thermal-label/transport/web.
             Implements PrinterAdapter + a requestPrinter() for pairing.
  docs/      Markdown picked up by thermal-label.github.io at build time.
```

Published as `@thermal-label/<driver>-{core,node,web}`. The org rules:

- **`core`** depends only on `@thermal-label/contracts` and your
  protocol logic. No imports from `node:*`, no DOM globals.
- **`node`** exports a class implementing
  [`PrinterAdapter`](https://github.com/thermal-label/contracts/blob/main/src/adapter.ts)
  and a singleton `discovery: PrinterDiscovery` so `thermal-label-cli`
  can auto-load it.
- **`web`** exports a similar adapter using `WebUsbTransport` /
  `WebHidTransport` / `WebBluetoothTransport` from
  `@thermal-label/transport/web`. WebUSB pairing is
  `navigator.usb.requestDevice()`; the driver exports its own
  `requestPrinter()` helper but does **not** implement
  `PrinterDiscovery` for the browser.
- The adapter returns a `PrinterStatus` with the standard fields
  (`ready`, `mediaLoaded`, `errors[]`, optional `detectedMedia`).
  Driver-specific error reasons go into `errors[].code` — strings,
  not enums.
- Media is described via `MediaDescriptor` (or a driver-specific
  subclass). Use `pickRotation()` from
  `@thermal-label/contracts/orientation` to map bitmap orientation
  to the printer's natural feed direction.

## 2. Pick a printer

Drivers in this ecosystem do well when:

- The device speaks a documented (or reverse-engineerable) raster
  protocol over USB Printer Class, USB HID, raw TCP, Bluetooth Serial,
  or WebUSB.
- You have at least one physical unit on your bench. **Don't start a
  driver you can't test.** The verification system the org runs
  assumes day-1 reports come from a maintainer's bench.
- The protocol fits the contracts package's mental model: an image →
  bytes encoder, a status parser, a media table.

Drivers that struggle:

- Devices that need a vendor SDK runtime (closed binary blobs we
  can't ship).
- Devices that only speak a print-driver-mediated rendering API
  (CUPS / IPP / PCL) — those belong in a different layer than this
  ecosystem.
- Devices with no programmatic interface at all (Bluetooth-only
  consumer label printers that talk to a vendor app and nothing else).

## 3. Bootstrap the repo

Use one of the three existing drivers as a template. The fastest path:

```bash
git clone https://github.com/thermal-label/brother-ql my-driver
cd my-driver
git remote remove origin
# wipe brother-ql-specific code; keep the scaffolding
```

You're keeping these files essentially as-is:

- `pnpm-workspace.yaml`
- `tsconfig.base.json`
- `eslint.config.js`
- `.gitattributes`, `.gitignore`
- `.githooks/pre-push`
- `package.json` scripts (rename, but keep the script set)

You're rewriting:

- `packages/core/src/*` — your protocol
- `packages/node/src/*` — your node adapter
- `packages/web/src/*` — your web adapter
- All docs

Pin yourself to the same dev-dep versions the existing drivers ship —
they're the org-standard `@mbtech-nl/eslint-config`,
`@mbtech-nl/prettier-config`, `@mbtech-nl/tsconfig`, and
`@mbtech-nl/bitmap` packages. No reinvention.

### A note on `@thermal-label/contracts`

The contracts package pins the interface surface that your driver
implements. Track its `^0.x.0` major in your dependencies and don't
fork or shadow types — the whole point of the ecosystem is that
`PrinterAdapter` means the same thing in every driver.

If something feels missing from contracts (a status field your printer
needs to expose, a transport class shape that's awkward), open an
issue on `thermal-label/contracts`. The right move is usually to grow
contracts, not to invent an extension.

## 4. The `core` package — protocol layer

Required exports:

```ts
// packages/core/src/index.ts
export { DEVICES, findDevice }    from './devices.js';
export { MEDIA, DEFAULT_MEDIA }   from './media.js';
export { encodeJob }              from './protocol.js';
export { parseStatus, STATUS_REQUEST } from './status.js';
export { createPreviewOffline }   from './preview.js';
export { ROTATE_DIRECTION }       from './orientation.js';
export { pickRotation }           from '@thermal-label/contracts';
// + your driver-specific types
```

### `DEVICES` shape

```ts
export const DEVICES = {
  YOUR_MODEL_A: {
    name: 'Your Model A',           // human display name
    family: '<your-driver>',        // driver family identifier
    transports: ['usb', 'webusb', 'tcp'],  // declared transports
    vid: 0x1234, pid: 0x5678,       // USB IDs
    // ...family-specific fields (head dots, capabilities, network, ...)
  },
} as const satisfies Record<string, YourDeviceDescriptor>;
```

Source the byte-level details from:

- The manufacturer's developer SDK / programming reference if one exists.
- USB packet captures via `usbmon` (Linux) / Wireshark on `usbpcap`
  (Windows) of the vendor app driving the device.
- Existing reverse-engineering work — check the linked `related-orgs`
  page on the docs site.

Encode `encodeJob` as a **pure function** taking `RawImageData` + your
media descriptor + `PrintOptions` and returning `Uint8Array`. No I/O,
no globals, no `node:*` imports. This is the contract that lets your
package run in a browser bundle.

### `MEDIA` registry

A `Record<id, MediaDescriptor>` with one entry per media SKU. The
descriptor carries dimensions, margins, and (for multi-colour media)
the `palette` — a list of named ink colours the driver classifies
pixels into. Default media goes in `DEFAULT_MEDIA` for ergonomics.

### Status parsing

`parseStatus(bytes: Uint8Array): PrinterStatus` — also pure.
Map vendor status bytes to the standard status fields:

- `ready: boolean` — true when the printer is willing to accept a job
- `mediaLoaded: boolean` — physical media present
- `errors: PrinterError[]` — `{ code: string, message: string }[]`
- `detectedMedia?: MediaDescriptor` — when the printer reports it
- Plus driver-specific fields via a subclass of `PrinterStatus`

### Tests for `core`

Capture real byte fixtures from a USB-mon trace on a working print
job, commit them as `__tests__/fixtures/*.bin`, and assert
`encodeJob(...)` produces the same bytes for the same input. For
`parseStatus`, check the canonical status responses (ready / no media
/ cover open / error states).

The protocol-layer test suite is your primary regression net. Aim
for >80% line coverage on `protocol.ts` and `status.ts`.

## 5. The `node` package — adapter + discovery

Your `node` package implements a class extending `PrinterAdapter` and
a singleton `discovery: PrinterDiscovery`:

```ts
// packages/node/src/index.ts
import { UsbTransport, TcpTransport } from '@thermal-label/transport/node';
import { discoverAll, matchDevice } from '@thermal-label/transport';
import { DEVICES, encodeJob, parseStatus } from '@thermal-label/<driver>-core';
import type { PrinterAdapter, PrinterDiscovery } from '@thermal-label/contracts';

export class YourPrinter implements PrinterAdapter {
  // ...
}

export const discovery: PrinterDiscovery = {
  async listPrinters() {
    const usb = await UsbTransport.list();
    return usb
      .filter(d => matchDevice(d, Object.values(DEVICES)))
      .map(d => ({ /* DiscoveredPrinter shape */ }));
  },
  async openPrinter(handle, options) { /* ... */ },
};
```

The CLI auto-loads a driver via the `discovery` named export — no
CLI change required.

### TCP discovery

If your printer family supports TCP, the convention is to **not**
auto-discover by network scan (too noisy). Instead, the CLI's
`--host <ip>` flag opens a TCP transport directly. Your adapter
`openPrinter({ host })` should accept a host and skip USB discovery.

The transport package ships `TcpTransport` in `@thermal-label/transport/node`.

### Tests for `node`

Use `MockTransport` (see how the existing drivers wire it) — a
fake transport that records writes and replays canned responses.
Assert that `printer.print(...)` calls `transport.write(...)` with
the bytes `encodeJob` produces, that `printer.getStatus(...)` reads
the right number of bytes and parses correctly, and that error
states surface as `errors[]` entries.

Real-hardware integration tests live behind an env flag
(`<DRIVER>_INTEGRATION=1 pnpm test`). They're optional in CI but
required before you ship.

## 6. The `web` package — browser adapter

```ts
// packages/web/src/index.ts
import { WebUsbTransport } from '@thermal-label/transport/web';
import { buildUsbFilters } from '@thermal-label/transport';
import { DEVICES, encodeJob, parseStatus } from '@thermal-label/<driver>-core';

export async function requestPrinter(): Promise<YourWebPrinter> {
  const filters = buildUsbFilters(Object.values(DEVICES));
  const device = await navigator.usb.requestDevice({ filters });
  const transport = await WebUsbTransport.open(device);
  return new YourWebPrinter(transport);
}

export class YourWebPrinter implements PrinterAdapter { /* ... */ }
```

There is **no `discovery` export** in web packages — browsers don't
let JS enumerate USB devices without prior user pairing. The
`requestPrinter()` helper is the standard entry point.

For non-USB transports, swap in `WebHidTransport`,
`WebBluetoothTransport`, or `WebSerialTransport` from
`@thermal-label/transport/web`.

## 7. Status mapping conventions

The `errors[].code` strings are an organic convention — each driver
adds the codes its devices emit. Prefer existing codes when they
fit:

- `no_media` — no roll/cassette installed
- `cover_open` — cover/door is open
- `cutter_jam` — auto-cutter is jammed
- `media_end` — end of roll detected
- `paper_out` — paper out (or NFC-locked roll on LabelWriter 550)
- `over_temp` — print head too hot
- `comm_error` — internal comms failure

For driver-specific errors, prefix with the family name
(`brother-ql:editor_lite_active`, `labelwriter:nfc_invalid`) so
consumers can distinguish.

## 8. Media + orientation

Use `pickRotation()` from `@thermal-label/contracts` to decide
whether to rotate a `RawImageData` before encoding. The function
takes the input dimensions and the media's `defaultOrientation` and
returns one of `0 | 90 | 180 | 270`. `print()` accepts `options.rotate
= 'auto' | 0 | 90 | 180 | 270` — `'auto'` calls `pickRotation`,
explicit values bypass it.

Multi-colour media (e.g. Brother DK-22251 black+red) carry a
`palette` array on their `MediaDescriptor`. The `print()` code path
calls `renderMultiPlaneImage()` from `@mbtech-nl/bitmap` with that
palette and hands the resulting planes to `encodeJob`. The encoder
takes one plane per ink and emits the vendor's plane-switch
commands.

## 9. Testing

Required:

- `pnpm test` passes locally and in CI.
- `pnpm typecheck` passes.
- `pnpm lint` passes (org ESLint config — don't relax rules).
- `core` has byte-fixture tests for `encodeJob` + `parseStatus`.
- `node` has mock-transport tests for the adapter.
- `web` has at least smoke tests for the adapter (mock the WebUSB
  device with a stub).

Optional but expected for shipping:

- Real-hardware integration tests behind an env flag.
- Coverage budget — see existing drivers for examples.

## 10. Hardware coverage from day one

Before you publish, two files must exist:

### `docs/hardware-status.yaml`

Seed the community-verified status with whatever you've personally
tested on your bench. Mark reports as `selfVerified: true`. The
schema is canonical at
[CONTRIBUTING/hardware-status-schema.md](./hardware-status-schema.md).

```yaml
schemaVersion: 1
driver: <your-driver>

devices:
  - pid: 0x5678
    name: 'Your Model A'
    status: verified
    transports:
      usb: verified
    lastVerified: 2026-04-27
    packageVersion: '0.1.0'
    reports:
      - issue: 0  # placeholder until first real verification issue
        reporter: '@your-handle'
        date: 2026-04-27
        result: verified
        os: Linux
        selfVerified: true
        notes: 'Bench verification at driver release.'
```

Devices in `DEVICES` you haven't bench-tested are simply not listed
here — they render as `untested` on the unified `/hardware/` page.

### `docs/verification-checklist.md`

A short, family-specific sequence volunteers can run. Skeleton in
the existing drivers; copy and adapt for your protocol's quirks.
Cover at minimum: `list`, `status`, `print text`, `print image`,
TCP if applicable, browser demo if applicable, and family-specific
capability tests (multi-colour media, cutter, NFC checks, etc.).

### `scripts/validate-hardware-status.mjs`

Copy the validator from any existing driver. Change the
`EXPECTED_DRIVER` constant. Wire it into:

- `package.json`: `"validate:hardware-status": "node scripts/validate-hardware-status.mjs"`
- `.githooks/pre-push`: an additional check after the `docs:api`
  one. Existing drivers show the pattern.

The validator imports `DEVICES` from your built `core/dist/index.js`,
so run `pnpm -r build` before validating during development.

## 11. Docs

Required files in `docs/`:

| File | Purpose |
|---|---|
| `index.md` | Driver landing page on the site |
| `getting-started.md` | npm install + a minimal print example |
| `core.md` | Protocol-layer overview |
| `node.md` | Node usage |
| `web.md` | Browser usage |
| `hardware.md` | Static reference: VID/PID, head geometry, family quirks |
| `hardware-status.yaml` | Verification state (Phase 1 seed) |
| `verification-checklist.md` | Volunteer sequence (Phase 3 seed) |
| `api/` | Generated by typedoc (`pnpm docs:api`) |

The docs site at [thermal-label.github.io](https://thermal-label.github.io)
auto-pulls your `docs/` folder via
[`scripts/pull-driver-docs.mjs`](https://github.com/thermal-label/thermal-label.github.io/blob/main/scripts/pull-driver-docs.mjs).
You don't run any docs publishing yourself — your `docs/` directory
**is** the published source.

### Linking conventions

- Inside your repo, use relative links: `[Hardware](./hardware)`.
- Across repos (rare in driver docs), use absolute paths starting at
  the docs site root: `[contracts](/contracts/)`.
- For external links to GitHub source, use full URLs with `main`
  branch: `https://github.com/thermal-label/contracts/blob/main/src/adapter.ts`.

### Per-driver hardware status fragment

The docs site's build script generates `docs/<driver>/_status-fragment.md`
after the pull and includes it into your `docs/hardware.md` via the
VitePress include directive at the bottom of the file:

```markdown
<!--@include: ./_status-fragment.md-->
```

Add this directive to your `hardware.md`. The fragment is a no-op
when built outside the docs site, so per-driver `pnpm docs:api`
runs aren't affected.

## 12. Publish + integrate

Version order: `core` → `node` → `web`. The `node` and `web`
packages depend on `core`; bump them after `core` is published.

```bash
pnpm changeset                 # describe the change
pnpm changeset version         # bump versions
pnpm -r build                  # rebuild all
pnpm -r publish --access public
```

The org uses Changesets — see `release-process.md` for the full
flow.

### `repository_dispatch` to the docs site

When you publish, fire a `repository_dispatch` event at
`thermal-label/thermal-label.github.io` so the docs site rebuilds
with your latest pulled docs. The release workflow in the existing
drivers shows the exact YAML.

### Bumping the docs site's deps

The docs site pins `@thermal-label/<driver>-core` versions in its
`package.json` so the unified `/hardware/` page can read DEVICES
directly. After your release, open a PR on the docs site bumping
your `*-core` to the new version. (Per the
[plan decisions](../plans/backlog/driver-and-hardware-ecosystem-DECISIONS.md#i3),
this is manual for now.)

For a brand-new driver, you'll also extend the docs site's:

- `package.json` deps to include your `*-core`
- `scripts/pull-driver-docs.mjs` `REPOS` list
- `scripts/build-hardware-page.mjs` `DRIVERS` list (with display name)
- `docs/.vitepress/config.ts` nav + sidebar entries
- (optionally) a `docs/.vitepress/components/LiveDemo/<Driver>Demo.vue`
  component and a `docs/demo/<driver>.md` page

## 13. Day 2

Once your driver is on npm and the docs site shows it:

1. **File a hardware verification on your own repo.** Yes, against
   your own driver — bench results count and become the seed data
   for the unified page. Mark it `selfVerified: true` in the YAML.
2. **Open a Discussion on `thermal-label/.github`** announcing the
   driver. Surfaces it to anyone watching the org.
3. **Set up the maintainer ergonomics**: `PROGRESS.md`,
   `DECISIONS.md`, `plans/backlog/`, `plans/implemented/`. The
   existing drivers are the template.
4. **Ask for verification reports.** People with the hardware are
   out there; they can't volunteer if they don't know. The org's
   social channels are the easiest way.

## 14. Maintainer ergonomics — what the org expects

These aren't rules but they make running a driver vastly easier:

- `PROGRESS.md` — a checklist of remaining work (per major step).
  Existing drivers show the pattern; treat it as your todo list
  while bootstrapping.
- `DECISIONS.md` — a numbered list of design decisions with their
  rationale. New maintainers (including future-you) read this to
  understand why something is the way it is.
- `plans/backlog/` and `plans/implemented/` — markdown plan files
  for each non-trivial change. Move from `backlog/` to `implemented/`
  when shipped.
- A `pre-push` hook that regenerates `docs/api/` (typedoc) and
  runs `validate:hardware-status` so contributors can't forget.

---

## Reference summary

When in doubt, read the source:

- **Interface contracts:** [`thermal-label/contracts/src/`](https://github.com/thermal-label/contracts/tree/main/src)
- **Transport classes:** [`thermal-label/transport/src/`](https://github.com/thermal-label/transport/tree/main/src)
- **A worked driver:** [`thermal-label/brother-ql`](https://github.com/thermal-label/brother-ql)
  is the most thoroughly documented; the other two are more compact
  and good for spotting "the minimum a driver needs."
- **Verification system:** [`hardware-status-schema.md`](./hardware-status-schema.md),
  [`verifying-hardware.md`](./verifying-hardware.md), and the
  [unified hardware page](https://thermal-label.github.io/hardware/).
- **Docs site:** [`thermal-label/thermal-label.github.io`](https://github.com/thermal-label/thermal-label.github.io)
  — the puller, the build scripts, and the VitePress config.
- **Release flow:** [`release-process.md`](./release-process.md).

Welcome aboard.

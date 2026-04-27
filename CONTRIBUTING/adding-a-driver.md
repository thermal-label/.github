# Adding a driver

> **Status:** stub — the full guide is being drafted under the broader
> [driver authoring + hardware coverage plan](../plans/backlog/driver-and-hardware-ecosystem.md).
> That plan covers writing this guide *plus* the unified hardware-status
> system, the verification flow, and the per-driver
> `verification-checklist.md` files. What's below is the bare-minimum
> orientation. Expect this file to grow.

This is the path for adding a new printer family to the **thermal-label**
ecosystem. It is _not_ for adding a new device to an existing driver — for that,
file a [New device support](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/new_device.yml)
issue on the relevant driver repo.

## The shape of a driver

Every driver follows the same layered split — see existing drivers
[`brother-ql`](https://github.com/thermal-label/brother-ql),
[`labelmanager`](https://github.com/thermal-label/labelmanager),
[`labelwriter`](https://github.com/thermal-label/labelwriter) for working examples.

```
<your-driver>/
  packages/
    core/    Pure protocol layer. No I/O. Encodes job bytes, parses status.
    node/    Node integration. Composes core + @thermal-label/transport/node.
    web/     Browser integration. Composes core + @thermal-label/transport/web.
  docs/      Markdown picked up by thermal-label.github.io at build time.
```

Published as `@thermal-label/<driver>-core`, `-node`, `-web`.

## Conformance contract

The driver is "thermal-label-shaped" when:

1. **`core`** depends only on `@thermal-label/contracts` and your protocol logic.
   No imports from `node:*`, no DOM globals.
2. **`node`** exports a class implementing
   [`PrinterAdapter`](https://github.com/thermal-label/contracts/blob/main/src/adapter.ts)
   and a singleton `discovery: PrinterDiscovery` so `thermal-label-cli` can
   auto-load it by convention.
3. **`web`** exports a similar adapter using `WebUsbTransport` / `WebHID` /
   `WebBluetoothTransport` from `@thermal-label/transport/web`. WebUSB pairing is
   `navigator.usb.requestDevice()`; the driver should not implement
   `PrinterDiscovery` for the browser.
4. The adapter returns a `PrinterStatus` with the standard fields
   (`ready`, `mediaLoaded`, `errors[]`, optional `detectedMedia`). Driver-specific
   error reasons go into `errors[].code` — strings, not enums.
5. Media is described via `MediaDescriptor` (or a driver-specific subclass).
   Use `pickRotation()` from `@thermal-label/contracts/orientation` to map
   bitmap orientation to the printer's natural feed direction.

## Minimal `core` package skeleton

```ts
// packages/core/src/index.ts
import type { MediaDescriptor, RawImageData, PrintOptions } from '@thermal-label/contracts';

export interface FooMedia extends MediaDescriptor { /* … */ }
export const DEVICES = [/* { vid, pid, name, capabilities } */];
export const MEDIA: readonly FooMedia[] = [/* … */];

export function encodeJob(image: RawImageData, media: FooMedia, opts?: PrintOptions): Uint8Array {
  // pure function — no I/O, no globals
}

export function parseStatus(bytes: Uint8Array): PrinterStatus {
  // pure function
}
```

## Minimal `node` adapter skeleton

```ts
// packages/node/src/index.ts
import { UsbTransport } from '@thermal-label/transport/node';
import type { PrinterAdapter, PrinterDiscovery } from '@thermal-label/contracts';
import { DEVICES, encodeJob, parseStatus } from '@thermal-label/foo-core';

export class FooPrinter implements PrinterAdapter { /* … */ }

export const discovery: PrinterDiscovery = {
  async listPrinters() { /* enumerate via UsbTransport.list() filtered by DEVICES */ },
  async openPrinter(handle) { /* … */ },
};
```

## Testing strategy

- **`core`** — unit-test `encodeJob`, `parseStatus`, media lookup with byte
  fixtures captured from a real printer (commit fixtures, not blobs).
- **`node`** — integration tests against a `MockTransport` that records writes
  and replays canned responses. Real-hardware tests live behind an env flag.
- **Hardware verification** — once the device works on your bench, file a
  Hardware verification issue on your repo with the model, PID, and which
  capabilities you exercised. This becomes the public compatibility record.

## Publishing

See [release-process.md](./release-process.md) for the publish flow and version
ordering across `core` → `node` → `web`. The CLI auto-discovers your driver
via the `discovery` named export — no CLI change required.

## What lives where after release

- npm: `@thermal-label/<driver>-{core,node,web}`
- Docs: `https://thermal-label.github.io/<driver>/` (auto-pulled from your repo's `docs/`)
- Issues: your repo's Issues tab. Org defaults supply the templates.

## Next steps

The full guide-to-be will expand each section into a concrete walkthrough,
with a worked example driver. If you want to start a driver before that lands,
read the existing three driver repos in parallel — the file layout and
naming repeat.

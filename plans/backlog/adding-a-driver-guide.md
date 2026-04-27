# Plan — write the full "Adding a driver" guide

Date: 2026-04-27
Status: backlog — spin-off from `THERMAL_LABEL_DOCS_PLAN.md` §6.6

The bare-minimum stub at
[`CONTRIBUTING/adding-a-driver.md`](../../CONTRIBUTING/adding-a-driver.md) is in
place. This plan covers writing the real, walkthrough-style guide that takes a
new contributor from "I have a printer and curiosity" to "my driver is on npm".

---

## Why this is its own plan

Drafting the full guide is non-trivial — it's not just shuffling existing files.
It needs:

- A worked example, ideally a fictional or simple real driver, written end to end
- Concrete byte-level examples of `encodeJob` / `parseStatus` from at least one
  family
- Testing patterns documented from real fixtures in the existing drivers
- A publishing walkthrough that matches whatever release tooling we settle on
- Cross-links to `contracts` source so the conformance contract is not rephrased

Bundling that into the docs cleanup pass would have stalled the cleanup. Spun off.

---

## Sources to mine

The three existing driver repos already encode most of what the guide should
say. Read in this order before writing:

1. [`thermal-label/contracts/src/`](https://github.com/thermal-label/contracts/tree/main/src)
   — the interface surface a driver implements. Especially `adapter.ts`,
   `discovery.ts`, `status.ts`, `media.ts`, `orientation.ts`.
2. [`thermal-label/transport/src/`](https://github.com/thermal-label/transport/tree/main/src)
   — what the driver gets for free (USB, TCP, WebUSB, Web Bluetooth, Web Serial).
3. The three `DECISIONS.md` files in `brother-ql/`, `labelmanager/`,
   `labelwriter/` — they encode the conventions a new driver should follow
   (status mapping, `discovery` named export, `core/node/web` split, etc.).
4. The retrofit plan at `plans/implemented/driver-retrofit.md` — captures the
   shared playbook the existing three drivers were written against.

The existing driver READMEs and `docs/getting-started.md` files double as
implicit examples — link to them.

---

## Outline

1. **What "thermal-label driver" means** — the layered architecture, where
   your code fits, what you get for free, what you have to provide.
2. **Pick a printer** — what kind of device works well here (USB raster, raw
   TCP, WebUSB-friendly), what doesn't (proprietary protocol blobs, Mac-only
   drivers, networked-print-server-only printers).
3. **Bootstrap the repo** — directory layout, `pnpm-workspace.yaml`,
   `package.json` per workspace package, tsconfig conventions.
4. **The `core` package — protocol layer**
   - Implement `encodeJob`, `parseStatus`, `MediaDescriptor` for your family
   - Where to source byte-level documentation (manufacturer SDK, USB captures,
     reverse engineering)
   - Worked example: encoding a 1bpp raster job
5. **The `node` package — adapter + discovery**
   - Implement `PrinterAdapter` using `UsbTransport` from
     `@thermal-label/transport/node`
   - Implement and export `discovery: PrinterDiscovery` (singleton)
   - TCP discovery patterns (mDNS, opportunistic scan, "explicit only")
6. **The `web` package — browser adapter**
   - Implement using `WebUsbTransport` (or `WebHID`, `WebBluetoothTransport`,
     `WebSerialTransport`)
   - `requestPrinter()` pattern for browser pairing
   - Why no `discovery` in browser packages (user-gesture pairing only)
7. **Status mapping conventions**
   - The standard `PrinterStatus` shape
   - Driver-specific `errors[].code` strings
   - When to populate `detectedMedia` vs leave undefined
8. **Media + orientation**
   - `MediaDescriptor` shape; what to put in your driver-specific subclass
   - `pickRotation()` — how the contract picks rotation given media + image
   - The `DEFAULT_MEDIA` convention for previews
9. **Testing**
   - Unit tests for `core` with byte fixtures
   - Mock-transport integration tests for `node`
   - The hardware-verification issue template as the public compatibility record
10. **Docs**
    - The required `docs/index.md`, `docs/getting-started.md`, `docs/hardware.md`
    - Wiring `pnpm docs:api` (typedoc)
    - Linking pattern: relative inside the repo, `/<other-repo>/...` across repos
11. **Publish + integrate**
    - Version order (core → node → web)
    - `repository_dispatch` to trigger docs rebuild
    - The CLI auto-discovers your driver — no CLI change required
12. **What to do on day 2**
    - File a hardware verification on your own repo
    - Open Discussions on `.github` to surface the new driver
    - Maintainer ergonomics: PROGRESS.md, DECISIONS.md, plans/

---

## Effort estimate

~2 sittings of focused writing once the docs site is live and the existing
drivers are stable. The guide should be reviewed by one other contributor
before being promoted out of "stub" status in the `CONTRIBUTING/` folder.

---

## Definition of done

- `CONTRIBUTING/adding-a-driver.md` is the full guide (no longer a stub).
- It links to live, working code in the existing driver repos for every step.
- This plan moves to `plans/implemented/`.

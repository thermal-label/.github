# Driver Retrofit — Combined Amendment

> Retrofit all three existing driver packages to adopt `@thermal-label/contracts`
> and `@thermal-label/transport`. One agent, three repos, consistent decisions.
>
> **Order: labelmanager → labelwriter → brother-ql.** Simplest first, most
> complex last. Each driver builds on lessons from the previous one.
>
> **What changes per driver:**
> - Remove the per-driver CLI package (unpublish from npm)
> - Replace per-driver transport code with `@thermal-label/transport`
> - Implement `PrinterAdapter` and `PrinterDiscovery` from `@thermal-label/contracts`
> - Implement `createPreview()` on the adapter
> - Export `createPreviewOffline()` from the core package
> - Export a `discovery` instance for the unified CLI to find
> - Update `print()` to accept `RawImageData` + optional `MediaDescriptor`
> - Bump to 0.2.0

---

## 1. Shared Decisions (apply to all three drivers)

These decisions are made once and applied consistently across all three
driver repos. Document in each repo's DECISIONS.md with a reference back
to this plan.

### 1.1 Transport Swap

Each driver currently has its own `UsbTransport` (and `TcpTransport` for
brother-ql). Replace with imports from `@thermal-label/transport`:

```typescript
// Before (per-driver, duplicated)
import { UsbTransport } from './transport.js';

// After (shared)
import { UsbTransport } from '@thermal-label/transport/node';
```

For web packages:

```typescript
// Before
import { WebUsbTransport } from './transport.js';

// After
import { WebUsbTransport } from '@thermal-label/transport/web';
```

Delete the per-driver transport source files after verifying the shared
transport works identically.

### 1.2 PrinterAdapter Implementation

Each driver's main printer class implements `PrinterAdapter` from contracts:

```typescript
import type { PrinterAdapter, RawImageData, MediaDescriptor, PrintOptions,
  PrinterStatus, PreviewResult, PreviewOptions } from '@thermal-label/contracts';

export class LabelManagerPrinter implements PrinterAdapter {
  readonly family = 'labelmanager';
  readonly model: string;
  readonly connected: boolean;
  readonly device?: DeviceDescriptor;

  async print(image: RawImageData, media?: MediaDescriptor, options?: PrintOptions): Promise<void>;
  async createPreview(image: RawImageData, options?: PreviewOptions): Promise<PreviewResult>;
  async getStatus(): Promise<PrinterStatus>;
  async close(): Promise<void>;
}
```

### 1.3 print() Signature Change

**Before:** each driver accepts `LabelBitmap` or driver-specific types.
**After:** accepts `RawImageData` (full RGBA). The driver does RGBA → 1bpp
internally using `@mbtech-nl/bitmap`:

```typescript
import { renderImage } from '@mbtech-nl/bitmap';

async print(image: RawImageData, media?: MediaDescriptor, options?: PrintOptions): Promise<void> {
  const resolvedMedia = media ?? this.detectedMedia;
  if (!resolvedMedia) throw new MediaNotSpecifiedError();

  const bitmap = renderImage(image, { dither: true });
  // ... encode and send using existing protocol code
}
```

The protocol encoding functions (`buildPrinterStream`, `encodeLabel`,
`encodeJob`) continue to accept `LabelBitmap` internally — only the
public `print()` entry point changes.

### 1.4 PrinterStatus Update

Each driver's status parsing updates to return `PrinterStatus` from
contracts:

```typescript
// Before (driver-specific shape)
return { ready: true, tapeInserted: true, errors: ['no tape'] };

// After (contracts shape)
return {
  ready: true,
  mediaLoaded: true,
  detectedMedia: matchedMedia,  // or undefined if can't detect
  errors: [{ code: 'no_media', message: 'No tape inserted' }],
  rawBytes: statusBytes,
};
```

Error codes per driver:

**LabelManager:**
- `not_ready` — printer busy
- `no_media` — no tape inserted
- `low_media` — tape supply low

**LabelWriter:**
- `not_ready` — printer busy
- `no_media` — no labels loaded
- `label_too_long` — label exceeded maximum length
- `paper_jam` — paper jam detected

**Brother QL:**
- `no_media` — no roll installed
- `cover_open` — cover is open
- `cutter_jam` — cutter jammed
- `media_end` — end of roll
- `wrong_media` — loaded media doesn't match specified media
- `system_error` — internal error (with raw code)

### 1.5 PrinterDiscovery Implementation

Each driver's node package exports a `discovery` instance:

```typescript
import type { PrinterDiscovery, DiscoveredPrinter, OpenOptions }
  from '@thermal-label/contracts';

class LabelManagerDiscovery implements PrinterDiscovery {
  readonly family = 'labelmanager';

  async listPrinters(): Promise<DiscoveredPrinter[]> {
    // Enumerate USB devices, match against device registry
    // Return DiscoveredPrinter for each match
  }

  async openPrinter(options?: OpenOptions): Promise<LabelManagerPrinter> {
    // Open by VID/PID, serial, or first available
  }
}

// Named export — the unified CLI looks for this
export const discovery = new LabelManagerDiscovery();
```

**Convention:** the export name is `discovery` (lowercase, named export).
The unified CLI's `loadDrivers()` looks for `mod.discovery`. All three
drivers use the same export convention.

### 1.6 DeviceDescriptor Extension

Each driver's device registry extends `DeviceDescriptor` from contracts:

```typescript
import type { DeviceDescriptor } from '@thermal-label/contracts';

// Before (standalone type)
interface LabelManagerDevice {
  name: string;
  vid: number;
  pid: number;
  printableWidth: number;
}

// After (extends contracts base)
interface LabelManagerDevice extends DeviceDescriptor {
  family: 'labelmanager';
  printableWidth: number;
}
```

### 1.7 MediaDescriptor Extension

Each driver defines its own media type extending `MediaDescriptor`:

```typescript
import type { MediaDescriptor } from '@thermal-label/contracts';

interface LabelManagerMedia extends MediaDescriptor {
  type: 'tape';
  colorCapable: false;
  tapeWidthMm: 6 | 9 | 12 | 19;
  printableDots: number;
  bytesPerLine: number;
}
```

### 1.8 CLI Package Removal

Each driver's monorepo currently has a `packages/cli/` directory. After
the unified `thermal-label-cli` ships:

1. Remove `packages/cli/` from the monorepo
2. Remove the CLI from `pnpm-workspace.yaml`
3. Update root `package.json` scripts if they reference CLI
4. Unpublish from npm: `npm unpublish @thermal-label/<family>-cli --force`

### 1.9 createPreview and createPreviewOffline

Each driver implements both:

- `createPreview()` on the `PrinterAdapter` instance — uses detected
  media as fallback, returns `PreviewResult`
- `createPreviewOffline()` as a static export from the `*-core` package —
  requires explicit media, no printer connection needed

Both share the same internal rendering logic — factor it into a shared
function in core.

### 1.10 Web Package Updates

Each driver's web package (`*-web`) updates to use `WebUsbTransport` from
`@thermal-label/transport/web`. The web adapter implements the same
`PrinterAdapter` interface. Web packages don't implement `PrinterDiscovery`
— browser discovery uses `navigator.usb.requestDevice()` which is a
different flow (user gesture required).

---

## 2. Driver A: `@thermal-label/labelmanager-*`

The simplest driver. Single transport (USB only), single colour, small
device registry, minimal status response.

### 2.1 Scope of Changes

```
packages/core/
  src/types.ts        → extend DeviceDescriptor, add LabelManagerMedia
  src/devices.ts      → update registry entries with family + transports fields
  src/status.ts       → return contracts PrinterStatus shape
  src/preview.ts      → NEW: createPreviewOffline()

packages/node/
  src/transport.ts    → DELETE (use @thermal-label/transport/node)
  src/discovery.ts    → NEW: LabelManagerDiscovery + export discovery
  src/printer.ts      → implement PrinterAdapter, print(RawImageData)

packages/web/
  src/transport.ts    → DELETE (use @thermal-label/transport/web)
  src/printer.ts      → implement PrinterAdapter

packages/cli/          → DELETE entirely
```

### 2.2 createPreview (single-colour — trivial)

```typescript
// packages/core/src/preview.ts
import { renderImage, type RawImageData } from '@mbtech-nl/bitmap';
import type { PreviewResult, MediaDescriptor } from '@thermal-label/contracts';

export function createPreviewOffline(
  image: RawImageData,
  media: LabelManagerMedia,
): PreviewResult {
  const bitmap = renderImage(image, { dither: true });
  return {
    planes: [{ name: 'black', bitmap, displayColor: '#000000' }],
    media,
    assumed: false,
  };
}
```

### 2.3 Media Registry

```typescript
export const MEDIA: Record<string, LabelManagerMedia> = {
  TAPE_6MM:  { id: 'tape-6',  name: '6mm tape',  widthMm: 6,  type: 'tape', colorCapable: false, tapeWidthMm: 6,  printableDots: 32, bytesPerLine: 4 },
  TAPE_9MM:  { id: 'tape-9',  name: '9mm tape',  widthMm: 9,  type: 'tape', colorCapable: false, tapeWidthMm: 9,  printableDots: 48, bytesPerLine: 6 },
  TAPE_12MM: { id: 'tape-12', name: '12mm tape', widthMm: 12, type: 'tape', colorCapable: false, tapeWidthMm: 12, printableDots: 64, bytesPerLine: 8 },
  TAPE_19MM: { id: 'tape-19', name: '19mm tape', widthMm: 19, type: 'tape', colorCapable: false, tapeWidthMm: 19, printableDots: 64, bytesPerLine: 8 },
};
```

No `heightMm` — tape is always continuous.

### 2.4 LabelManager Cannot Detect Media

The LabelManager PnP has no status query that returns tape width.
`getStatus()` returns `detectedMedia: undefined`. The user must always
specify media explicitly. `createPreview()` without media option and
without detected media throws `MediaNotSpecifiedError`.

### 2.5 Step Sequence

```
A1. Add @thermal-label/contracts + @thermal-label/transport as deps
A2. Update core types — DeviceDescriptor extension, LabelManagerMedia
A3. Update core status parser — return contracts PrinterStatus
A4. Add core preview.ts — createPreviewOffline
A5. Update node printer — implement PrinterAdapter, print(RawImageData)
A6. Replace node transport — delete local, import from transport/node
A7. Add node discovery — LabelManagerDiscovery + export discovery
A8. Update web printer — implement PrinterAdapter
A9. Replace web transport — delete local, import from transport/web
A10. Delete packages/cli/
A11. Update pnpm-workspace.yaml, root scripts
A12. Update all tests
A13. Gate: typecheck + lint + test + build across all packages
A14. npm unpublish @thermal-label/labelmanager-cli --force
A15. Bump to 0.2.0, commit + push
```

---

## 3. Driver B: `@thermal-label/labelwriter-*`

Medium complexity. USB only (450 series) or USB + TCP (550/Wireless).
Single colour. Two protocol generations (450 vs 550). The 550 can
detect media, the 450 cannot.

### 3.1 Scope of Changes

Same shape as labelmanager, plus:

- Device registry has more models and two protocol variants
- 550 series `getStatus()` populates `detectedMedia`
- 450 series `getStatus()` returns `detectedMedia: undefined`

```
packages/core/
  src/types.ts        → extend DeviceDescriptor, add LabelWriterMedia
  src/devices.ts      → update registry (all PIDs including 0x0020)
  src/status.ts       → return contracts PrinterStatus, detect media on 550
  src/preview.ts      → NEW: createPreviewOffline()

packages/node/
  src/transport.ts    → DELETE
  src/discovery.ts    → NEW: LabelWriterDiscovery + export discovery
  src/printer.ts      → implement PrinterAdapter, print(RawImageData)

packages/web/
  src/transport.ts    → DELETE
  src/printer.ts      → implement PrinterAdapter

packages/cli/          → DELETE
```

### 3.2 Media Registry

```typescript
export const MEDIA: Record<string, LabelWriterMedia> = {
  ADDRESS_STANDARD: {
    id: 'address-standard',
    name: '89×28mm Address',
    widthMm: 89, heightMm: 28,
    type: 'die-cut', colorCapable: false,
    // ... driver-specific fields
  },
  ADDRESS_LARGE: {
    id: 'address-large',
    name: '89×36mm Large Address',
    widthMm: 89, heightMm: 36,
    type: 'die-cut', colorCapable: false,
  },
  CONTINUOUS: {
    id: 'continuous',
    name: 'Continuous',
    widthMm: 56,
    // heightMm omitted — continuous
    type: 'continuous', colorCapable: false,
  },
  // ... other label sizes
};
```

### 3.3 Media Detection (550 only)

The 550 series returns label dimensions in its 32-byte status response.
The driver matches against the media registry to populate `detectedMedia`:

```typescript
// In status parser
const width = statusBytes[4] | (statusBytes[5] << 8);
const height = statusBytes[6] | (statusBytes[7] << 8);
const detectedMedia = findMediaByDimensions(width, height);

return {
  ready: true,
  mediaLoaded: true,
  detectedMedia,  // matched from registry, or undefined if unknown size
  errors: [],
  rawBytes: statusBytes,
};
```

The 450 series has a 1-byte status response with no media info —
`detectedMedia` is always `undefined`.

### 3.4 Step Sequence

```
B1. Add contracts + transport deps
B2. Update core types — DeviceDescriptor extension, LabelWriterMedia
B3. Update core device registry — ensure PID 0x0020 is primary for LW 450
B4. Update core status parser — contracts shape, 550 media detection
B5. Add core preview.ts
B6. Update node printer — PrinterAdapter, print(RawImageData)
B7. Replace node transport
B8. Add node discovery + export
B9. Update web printer
B10. Replace web transport
B11. Delete packages/cli/
B12. Update workspace config
B13. Update all tests
B14. Gate: typecheck + lint + test + build
B15. npm unpublish @thermal-label/labelwriter-cli --force
B16. Bump to 0.2.0, commit + push
```

---

## 4. Driver C: `@thermal-label/brother-ql-*`

Most complex. USB + TCP transports. Single-colour and two-colour modes.
Media auto-detection from status. DK-22251 two-colour tape handling.
`splitTwoColor` colour separation. Editor Lite mode detection.

### 4.1 Scope of Changes

Same shape as the others, plus:

- `splitTwoColor()` added to core for colour plane separation
- `createPreview()` returns two planes when media is `colorCapable`
- Media registry includes the `colorCapable` flag per media entry
- `getStatus()` reads media width and type from 32-byte response
- TCP transport via `TcpTransport` from `@thermal-label/transport/node`
- BLE config on device descriptors (UUIDs TBD — placeholder for now)

```
packages/core/
  src/types.ts        → extend DeviceDescriptor, add BrotherQLMedia
  src/devices.ts      → update registry with family, transports, bluetooth?
  src/media.ts        → update registry with colorCapable per media
  src/status.ts       → return contracts PrinterStatus with detectedMedia
  src/colour.ts       → NEW: splitTwoColor(), isRedish()
  src/preview.ts      → NEW: createPreviewOffline() — two-colour aware

packages/node/
  src/transport.ts    → DELETE (USB)
  src/tcp-transport.ts → DELETE (TCP)
  src/discovery.ts    → NEW: BrotherQLDiscovery + export discovery
  src/printer.ts      → implement PrinterAdapter, print(RawImageData)
                         two-colour via splitTwoColor when media.colorCapable

packages/web/
  src/transport.ts    → DELETE
  src/printer.ts      → implement PrinterAdapter

packages/cli/          → DELETE
```

### 4.2 splitTwoColor

```typescript
// packages/core/src/colour.ts
import { renderImage, type RawImageData, type LabelBitmap } from '@mbtech-nl/bitmap';

export interface TwoColorResult {
  black: LabelBitmap;
  red: LabelBitmap;
}

export function splitTwoColor(
  image: RawImageData,
  options?: { threshold?: number; dither?: boolean },
): TwoColorResult {
  const { threshold = 128, dither = true } = options ?? {};

  const blackImage = extractNonRedPixels(image);
  const redImage = extractRedPixels(image);

  const black = renderImage(blackImage, { threshold, dither });
  const red = renderImage(redImage, { threshold, dither });

  // Overlap resolution: black wins
  resolveOverlap(black, red);

  return { black, red };
}

/**
 * A pixel is "red-ish" if the red channel dominates.
 * The driver owns this heuristic — it knows what its hardware can reproduce.
 */
function isRedish(r: number, g: number, b: number, a: number): boolean {
  if (a < 128) return false;  // transparent → neither plane
  return r > 180 && g < 100 && b < 100;
}

function extractRedPixels(image: RawImageData): RawImageData {
  // Create a copy where non-red pixels become transparent
  const data = new Uint8ClampedArray(image.data.length);
  for (let i = 0; i < image.data.length; i += 4) {
    if (isRedish(image.data[i], image.data[i+1], image.data[i+2], image.data[i+3])) {
      data[i] = image.data[i];
      data[i+1] = image.data[i+1];
      data[i+2] = image.data[i+2];
      data[i+3] = image.data[i+3];
    }
    // else leave as 0,0,0,0 (transparent)
  }
  return { data, width: image.width, height: image.height };
}

function extractNonRedPixels(image: RawImageData): RawImageData {
  // Inverse of extractRedPixels
  const data = new Uint8ClampedArray(image.data.length);
  for (let i = 0; i < image.data.length; i += 4) {
    if (!isRedish(image.data[i], image.data[i+1], image.data[i+2], image.data[i+3])) {
      data[i] = image.data[i];
      data[i+1] = image.data[i+1];
      data[i+2] = image.data[i+2];
      data[i+3] = image.data[i+3];
    }
  }
  return { data, width: image.width, height: image.height };
}

function resolveOverlap(black: LabelBitmap, red: LabelBitmap): void {
  // Where both planes have a set bit at the same position, black wins
  // Clear the red bit
  for (let i = 0; i < black.data.length; i++) {
    red.data[i] &= ~black.data[i];
  }
}
```

### 4.3 createPreview — Two-Colour Aware

```typescript
// packages/core/src/preview.ts
import { renderImage, type RawImageData } from '@mbtech-nl/bitmap';
import type { PreviewResult, MediaDescriptor } from '@thermal-label/contracts';
import { splitTwoColor } from './colour.js';

export function createPreviewOffline(
  image: RawImageData,
  media: BrotherQLMedia,
): PreviewResult {
  if (media.colorCapable) {
    const { black, red } = splitTwoColor(image);
    return {
      planes: [
        { name: 'black', bitmap: black, displayColor: '#000000' },
        { name: 'red', bitmap: red, displayColor: '#ff0000' },
      ],
      media,
      assumed: false,
    };
  }

  const bitmap = renderImage(image, { dither: true });
  return {
    planes: [{ name: 'black', bitmap, displayColor: '#000000' }],
    media,
    assumed: false,
  };
}
```

### 4.4 print() — Two-Colour Aware

```typescript
async print(image: RawImageData, media?: MediaDescriptor, options?: PrintOptions): Promise<void> {
  const resolvedMedia = media ?? this.detectedMedia;
  if (!resolvedMedia) throw new MediaNotSpecifiedError();

  if (resolvedMedia.colorCapable) {
    const { black, red } = splitTwoColor(image);
    const job = encodeJob([{ bitmap: black, redBitmap: red, media: resolvedMedia }]);
    await this.transport.write(job);
  } else {
    const bitmap = renderImage(image, { dither: true });
    const job = encodeJob([{ bitmap, media: resolvedMedia }]);
    await this.transport.write(job);
  }
}
```

### 4.5 Media Registry Update

```typescript
export const MEDIA: Record<string, BrotherQLMedia> = {
  CONTINUOUS_62MM: {
    id: 259, name: '62mm continuous',
    widthMm: 62, type: 'continuous', colorCapable: false,
    printAreaDots: 696, leftMarginPins: 12, rightMarginPins: 12,
  },
  DK_22251: {
    id: 251, name: 'DK-22251 62mm two-colour',
    widthMm: 62, type: 'continuous', colorCapable: true,  // ← the key flag
    printAreaDots: 696, leftMarginPins: 12, rightMarginPins: 12,
  },
  DIE_CUT_29x90: {
    id: 274, name: '29×90mm',
    widthMm: 29, heightMm: 90, type: 'die-cut', colorCapable: false,
    printAreaDots: 306, leftMarginPins: 6, rightMarginPins: 6,
  },
  // ... all other media entries
};
```

### 4.6 Media Auto-Detection

The 32-byte status response has media width at byte 10 and media type at
byte 11. Match against the media registry:

```typescript
function detectMedia(statusBytes: Uint8Array): BrotherQLMedia | undefined {
  const widthMm = statusBytes[10];
  const mediaType = statusBytes[11]; // 0x0A continuous, 0x0B die-cut

  return Object.values(MEDIA).find(m =>
    m.widthMm === widthMm &&
    (mediaType === 0x0A ? m.type === 'continuous' : m.type === 'die-cut')
  );
}
```

For DK-22251: width = 62, type = continuous. Multiple 62mm continuous
entries exist (259 and 251). The two-colour detection is a separate
firmware flag — the expanded mode bit tells the firmware which media
variant is loaded. If the status indicates two-colour mode, match to
DK-22251 (251). Otherwise match to regular 62mm continuous (259).

### 4.7 BLE Config Placeholder

```typescript
QL_820NWB: {
  name: 'QL-820NWB',
  vid: 0x04F9, pid: 0x20A7,
  family: 'brother-ql',
  transports: ['usb', 'tcp', 'web-bluetooth'],
  bluetooth: {
    serviceUuid: 'TBD',          // discovered via GATT sniffing — not done yet
    txCharacteristicUuid: 'TBD',
    namePrefix: 'QL-820',
  },
  // ...
},
```

The BLE UUIDs are not yet discovered. Leave as placeholder strings.
The web-bluetooth transport path won't work until real UUIDs are filled
in. Document in DECISIONS.md.

### 4.8 Step Sequence

```
C1. Add contracts + transport deps
C2. Update core types — DeviceDescriptor extension, BrotherQLMedia
C3. Update core device registry — add family, transports, bluetooth placeholders
C4. Update core media registry — add colorCapable flag to all entries
C5. Add core colour.ts — splitTwoColor, isRedish, extractRedPixels,
    extractNonRedPixels, resolveOverlap
C6. Update core status parser — contracts shape, media auto-detection
C7. Add core preview.ts — createPreviewOffline (two-colour aware)
C8. Update node printer — PrinterAdapter, print(RawImageData), two-colour
C9. Replace node USB transport
C10. Replace node TCP transport
C11. Add node discovery + export
C12. Update web printer
C13. Replace web transport
C14. Delete packages/cli/
C15. Update workspace config
C16. Update all tests (including two-colour splitTwoColor tests)
C17. Gate: typecheck + lint + test + build
C18. npm unpublish @thermal-label/brother-ql-cli --force
C19. Bump to 0.2.0, commit + push
```

---

## 5. Test Updates

### 5.1 What Changes in Tests

**Transport tests:** remove tests for per-driver transport classes.
The transport is now tested in `@thermal-label/transport`. Driver tests
only verify that the printer class correctly calls `transport.write()`
and `transport.read()` — mock the transport interface.

**Status tests:** update expected output shape to match contracts
`PrinterStatus` — `detectedMedia`, `errors: PrinterError[]`, `rawBytes`.

**Print tests:** update to pass `RawImageData` instead of `LabelBitmap`.
Verify that `renderImage` is called internally (mock `@mbtech-nl/bitmap`).

**Discovery tests:** new tests for `listPrinters()` and `openPrinter()`.
Mock USB device enumeration.

**Preview tests:** new tests for `createPreviewOffline()` and
`createPreview()` on the adapter instance.

**Two-colour tests (brother-ql only):**
- `splitTwoColor` with all-black image → black plane full, red plane empty
- `splitTwoColor` with all-red image → red plane full, black plane empty
- `splitTwoColor` with mixed image → correct separation
- `isRedish` boundary values — (181,99,99) = red, (180,100,100) = not red
- `resolveOverlap` — overlapping bits cleared from red plane
- `createPreviewOffline` with `colorCapable: true` → two planes
- `createPreviewOffline` with `colorCapable: false` → one plane
- `print()` with DK-22251 media → two-colour encoding path used
- `print()` with regular media → single-colour encoding path used

### 5.2 What Stays the Same

**Protocol encoding tests:** the byte-level protocol tests (ESC/SYN for
labelmanager, ESC/raster for labelwriter, raster commands for brother-ql)
do NOT change. These test the internal encoding functions which still
accept `LabelBitmap`. Only the public `print()` entry point changes.

---

## 6. Overall Sequence

```
Phase A: labelmanager retrofit (steps A1–A15)
  Commit per step within the labelmanager repo.
  Gate: all packages typecheck + lint + test + build.
  Publish 0.2.0.

Phase B: labelwriter retrofit (steps B1–B16)
  Apply lessons from Phase A.
  Commit per step within the labelwriter repo.
  Gate: all packages typecheck + lint + test + build.
  Publish 0.2.0.

Phase C: brother-ql retrofit (steps C1–C19)
  Apply lessons from Phase A + B.
  Most complex — two-colour, media detection, TCP.
  Commit per step within the brother-ql repo.
  Gate: all packages typecheck + lint + test + build.
  Publish 0.2.0.
```

After all three are done:
- The unified `thermal-label-cli` finds all three drivers via `discovery` export
- `burnmark-cli` can import any driver and call `printer.print(rgba, media)`
- The label-maker app can call `printer.createPreview(rgba)` for any driver
- Per-driver CLIs are unpublished from npm

---

## 7. Key Constraints

- **One agent, three repos, consistent decisions.** Document shared
  decisions once in the first repo's DECISIONS.md, reference from the
  other two.
- **The `discovery` export convention is critical.** All three drivers
  must export `discovery` as a named export from their `*-node` package.
  The unified CLI depends on this.
- **Protocol encoding is untouched.** The internal encode functions still
  accept `LabelBitmap`. Only the public `print()` entry point changes to
  accept `RawImageData`.
- **BLE UUIDs for Brother QL are TBD.** Leave placeholder strings.
  The web-bluetooth transport path won't work until real UUIDs are
  discovered via GATT sniffing. This is a known future task.
- **`printer.close()` always in finally blocks** — verify this is
  already the case in existing code, fix if not.
- **At 0.x, break freely.** Bump all packages to 0.2.0. No deprecation.
- **Each driver's web package does NOT implement PrinterDiscovery** —
  browser discovery uses `navigator.usb.requestDevice()` which requires
  a user gesture. The web adapter implements `PrinterAdapter` only.
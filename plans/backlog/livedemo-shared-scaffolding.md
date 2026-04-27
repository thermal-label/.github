# Plan — refactor LiveDemo scaffolding

Date: 2026-04-27
Status: backlog — spin-off from `THERMAL_LABEL_DOCS_PLAN.md` Phase 4

The unified docs site at `thermal-label.github.io` now hosts all three
per-driver LiveDemos (`BrotherQLDemo.vue`, `LabelManagerDemo.vue`,
`LabelWriterDemo.vue`) under
`docs/.vitepress/components/LiveDemo/`. They were lifted **as-is** from each
driver's old per-repo VitePress site (Phase 3 deleted those).

The Phase 4 stretch goal — factoring out shared scaffolding (text editor,
preview canvas, status panel, USB pairing button) into reusable components —
was deferred to keep the cleanup landing in scope. This plan covers it.

---

## Why split this off

The three components share concept but differ in execution:

| Concern | brother-ql | labelmanager | labelwriter |
|---|---|---|---|
| File size | 647 lines | 481 lines | 494 lines |
| Tabs | single + two-color | single | single |
| Media picker | continuous-tape dropdown | tape width (6/9/12) | fixed media |
| Preview canvas | one + composite for two-color | tape-shaped wrap | label-shaped wrap |
| Status panel | shared shape | shared shape | adds NFC-lock notice on 550-series |
| USB pairing | shared pattern | shared pattern | shared pattern |
| Print path | rotateBitmap + maybe two-color | renderText → bitmapToRawImage | renderText → bitmapToRawImage |

A naïve "copy-paste each" landed all three, gets the docs site working today,
and lets us factor out the shared parts in a follow-up where we can compare
the three side by side.

The right structure (per the original plan §3.2) is:

```
LiveDemo/
  BrotherQLDemo.vue            thin wrapper composing shared/* with bql specifics
  LabelManagerDemo.vue         thin wrapper
  LabelWriterDemo.vue          thin wrapper
  shared/
    TextEditor.vue             single-line text input + invert toggle
    BitmapPreview.vue          canvas + scaling helper, prop-driven
    DevicePairButton.vue       USB pair / disconnect with state styling
    StatusPanel.vue            state dot + label + status message
    composables/
      useBitmapPreview.ts      shared canvas drawing logic
      useUsbPairing.ts         shared connect/disconnect/error state
```

---

## Steps

1. **Extract `StatusPanel.vue`** — the simplest piece. All three components
   already render a `state-dot` + `state-label` + status message in the same
   shape. Lift the template + styles into one component, parameterize
   `printer`, `printerName`, `isConnecting`, `statusMessage`, `statusType`.
2. **Extract `useUsbPairing` composable** — `connect()` / `disconnect()` /
   error-state machine. The brother-ql demo dynamically imports
   `requestPrinter` from the appropriate `*-web` package; that pattern can
   become a generic `useUsbPairing(importPrinter, getPrinterName)` factory.
3. **Extract `BitmapPreview.vue`** — the per-driver `drawPreview()` /
   `updateSinglePreview()` functions all do "render text → scale to height
   → draw to canvas with PREVIEW_SCALE px size". Parameterize target height
   and ink/background colours.
4. **Extract `TextEditor.vue`** — single text input + label, with optional
   invert toggle. Two driver demos already use this exact shape; bql's
   single tab too.
5. **Refactor each `*Demo.vue`** — recompose using shared components.
   Driver-specific logic (two-color compose for bql, tape-width picker for
   labelmanager, NFC-lock notice for labelwriter) stays in the wrapper.
6. **Browser test** — pair real hardware against each demo. The three
   driver families need separate verification: Brother QL (USB + two-color
   on QL-800), DYMO LabelManager (USB), DYMO LabelWriter (USB,
   plus 550-series NFC lock detection).

---

## Definition of done

- The three `*Demo.vue` files are each ≤ ~150 lines, mostly composition.
- `shared/` has the four components + two composables.
- `pnpm docs:build` passes.
- Each demo verified in browser with a real device.
- This plan moves to `plans/implemented/`.

---

## Out of scope

- Adding new demo capabilities (image upload, density control on bql, etc.).
- Replacing the Vue components with a different framework.
- Server-side rendering of the demos — they stay client-only behind
  `<ClientOnly>`.

---

## Effort estimate

~1-2 sittings. The lift in Phase 4 is mostly mechanical; the verification
step depends on having all three printer families on hand.

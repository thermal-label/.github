# Verifying hardware

So you've got a thermal-label printer and a few minutes — perfect.
This guide walks through what we ask you to test and how to file a
report. Maintainers turn your report into a row in
[`docs/hardware-status.yaml`](./hardware-status-schema.md), which feeds
the unified [`/hardware/`](https://thermal-label.github.io/hardware/)
page.

> Even partial results help. If only USB works on your QL-1100, that's
> still better data than "untested". Skip transports you can't test.

---

## 1. Why this matters

Anyone evaluating thermal-label for a project visits `/hardware/` to
see what's actually been verified. Without volunteer reports, every
device past the maintainer's bench shows as **untested** — which makes
the library look unloved even when the protocol is rock-solid.

Five minutes of your time = a permanent green check next to a model.

## 2. What you need

- Your printer (USB cable, or network cable / WiFi for TCP-capable
  models)
- Node 24+
- `thermal-label-cli` and the relevant `*-node` driver, installed
  globally:

  ```bash
  npm install -g thermal-label-cli @thermal-label/<family>-node
  ```

  Where `<family>` is `brother-ql`, `labelmanager`, or `labelwriter`.

- For the browser test (optional but appreciated): a Chromium-class
  browser with WebUSB enabled and a secure context.

- **Linux:** a udev rule for the vendor's VID. The driver repos ship a
  reference rule under `udev/`. Without it you'll get permission errors
  on the USB endpoint.

## 3. Identify your device

Find the **VID:PID** of your printer:

| OS | Command / location |
|---|---|
| Linux / macOS | `lsusb` (Linux) or `system_profiler SPUSBDataType` (macOS) |
| Windows | Device Manager → Properties → Details → Hardware Ids |

Confirm it's listed in the family's `DEVICES` registry — easiest way is
the [`/hardware/`](https://thermal-label.github.io/hardware/) page.
Search by your PID. If it appears with `· untested`, you're in the
right place. If it doesn't appear at all, the device isn't supported
yet — file a [New device support](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/new_device.yml)
issue instead.

## 4. Run the family checklist

Each driver repo ships a short, family-specific checklist at
`docs/verification-checklist.md`:

- [Brother QL](https://github.com/thermal-label/brother-ql/blob/main/docs/verification-checklist.md)
- [DYMO LabelManager](https://github.com/thermal-label/labelmanager/blob/main/docs/verification-checklist.md)
- [DYMO LabelWriter](https://github.com/thermal-label/labelwriter/blob/main/docs/verification-checklist.md)

Open the right one and follow it top to bottom. Each step has a
command, the expected output, and a note if there's a known gotcha.
Capture the terminal output and a photo of the printed label — both
go in the report.

The standard sequence (each driver's checklist refines this):

1. **`thermal-label list`** — your printer appears with the right
   model name and PID.
2. **`thermal-label status`** — returns within timeout, no errors,
   detected media populated where applicable.
3. **`thermal-label print text "verify $(date +%Y-%m-%d)"`** — a
   readable label exits the printer.
4. **`thermal-label print image small.png`** — a graphics print works.
5. **TCP-capable models:** repeat steps 2–4 with `--host <ip>`.
6. **Browser-capable models:** open the live demo at
   `https://thermal-label.github.io/demo/<family>`, pair, print.

## 5. Capability-specific tests

Some devices have features worth testing separately. The family
checklist lists which apply:

- **Brother QL 800-series:** two-colour printing on DK-22251.
- **Brother QL with auto-cut:** auto-cut between labels in a batch.
- **LabelManager:** every supported tape width (6 / 9 / 12 / 19 mm
  where applicable).
- **LabelWriter 550-series:** NFC-locked media — confirm a non-genuine
  roll is correctly rejected with the right error.
- **All families:** WebUSB pairing flow if you have a Chromium-class
  browser.

## 6. What "verified", "partial", "broken" mean

| Term | When it applies |
|---|---|
| **verified** | Every step you ran in the checklist worked end-to-end on at least one transport. |
| **partial** | Some checklist steps worked, others didn't. Or USB worked but TCP didn't (or vice-versa). Capture which steps failed. |
| **broken** | Critical steps reproducibly fail on the latest published version. The driver is unusable for this device today. |

You don't have to decide — pick what fits and the maintainer will
double-check during triage.

## 7. File the report

Open the
[Hardware verification](https://github.com/thermal-label/.github/blob/main/.github/ISSUE_TEMPLATE/hardware_verification.yml)
issue template **on the driver repo for your printer family**:

- Brother QL → [thermal-label/brother-ql/issues/new](https://github.com/thermal-label/brother-ql/issues/new?template=hardware_verification.yml)
- LabelManager → [thermal-label/labelmanager/issues/new](https://github.com/thermal-label/labelmanager/issues/new?template=hardware_verification.yml)
- LabelWriter → [thermal-label/labelwriter/issues/new](https://github.com/thermal-label/labelwriter/issues/new?template=hardware_verification.yml)

Fill in:

- Result: works / partially / does not work
- Model + PID
- Connection types tested
- OS
- Package version (`@thermal-label/<family>-node@x.y.z`)
- A short narrative — what worked, what didn't, anything weird

Attach a photo of the printed label and the terminal output of the
checklist. The form is short and rejects empty submissions so we don't
miss anything important.

## 8. After you submit

A maintainer will:

1. Acknowledge the issue, usually within a couple of days.
2. Open a PR adding your report to `docs/hardware-status.yaml` —
   appending to `reports[]` and recomputing the rolled-up `status`,
   `lastVerified`, and `packageVersion`.
3. Tag you on the PR; you don't need to do anything unless you spot
   something wrong.
4. Merge → the docs site rebuilds → your verification appears on
   [`/hardware/`](https://thermal-label.github.io/hardware/).
5. Close the issue with the appropriate `verified` / `partial` /
   `broken` label.

Your handle stays as the report's author indefinitely. If you want
that removed for privacy reasons, comment on the issue and we'll
anonymise it in the YAML (the report row stays, but `reporter`
becomes `@anonymous-N`).

## 9. Maintainer self-verification — a note

When a driver first ships, the maintainer often files the seed
verification themselves on hardware they own. Those reports are
marked `selfVerified: true` in the YAML so you can see they came
from the maintainer rather than an independent volunteer. They're
real — bench tests still count — but a community report on the same
device adds independent confirmation.

## 10. Edge cases

**My device has weird quirks (mass-storage mode, firmware variants).**
Mention it in the notes field. The maintainer may turn that into a
`quirks` block on the device's row, which renders as a callout above
the unified table — exactly the place future buyers will look first.

**The device prints but auto-cut / two-colour / TCP fails.**
Report it as `partial` with detail on what works. We'd rather have
"USB works, TCP times out" than nothing.

**The CLI doesn't detect my device.**
Run `lsusb` (Linux/macOS) or check Device Manager (Windows) to confirm
the OS sees it. If it does but `thermal-label list` doesn't, that's a
bug — file a regular bug report instead and link the verification
issue if relevant.

**My printer is on the registry but a different revision (firmware,
hardware) than the bench-tested one.**
Mention the firmware version. Different revisions sometimes behave
differently — that nuance becomes a `quirks` block.

---

Thanks for verifying. Every report makes the project measurably more
trustworthy.

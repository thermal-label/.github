# Implementation Progress — driver-and-hardware-ecosystem

> Companion to [driver-and-hardware-ecosystem.md](driver-and-hardware-ecosystem.md).
> Decisions resolved during implementation are recorded in
> [driver-and-hardware-ecosystem-DECISIONS.md](driver-and-hardware-ecosystem-DECISIONS.md).
>
> Implementation began 2026-04-27 in a single unsupervised session.
> Each phase is gate-checked (build/test/typecheck/lint where they apply)
> before its commit.

---

## Phase 1 — schema + validator + seed

### .github repo
- [x] Schema reference doc at `CONTRIBUTING/hardware-status-schema.md`

### Each driver repo (brother-ql, labelmanager, labelwriter)
- [x] `scripts/validate-hardware-status.mjs` — validator
- [x] `docs/hardware-status.yaml` — seeded with maintainer-attested data
- [x] `package.json` script: `validate:hardware-status`
- [x] `.githooks/pre-push` — runs validator on every push

### Gate (per repo)
- [x] `pnpm install` clean (yaml@^2.8.3 added as devDep)
- [x] `pnpm typecheck` passes
- [x] `pnpm lint` passes
- [x] `pnpm test` passes (brother-ql 31/3 skipped, labelmanager 16/1 skipped, labelwriter 24/2 skipped)
- [x] `node scripts/validate-hardware-status.mjs` passes
- [ ] commit (next)

### Phase 1 notes
- PID collision in labelmanager DEVICES (PnP and PC both at 0x1002) caught
  by validator on first run. Fixed validator to accept any candidate
  matching the YAML name; collision left in DEVICES (existing behaviour).
- `name` field validated as write-through cache against DEVICES per I4.
- Three repos seeded:
  - brother-ql: QL-820NWB self-verified (matches existing HARDWARE.md).
  - labelmanager: LabelManager PnP self-verified (matches existing HARDWARE.md).
  - labelwriter: empty `devices: []` — no formal verification yet.
- Issue numbers placeholder `0` in seed reports — to be replaced when a
  real verification issue lands. Documented in the YAML comments.

---

## Phase 2 — Unified `/hardware/` page on docs site

- [ ] Bump `*-core` deps (no bump needed — already on 0.2.0; deferred until first new device added post-release per D3)
- [ ] `scripts/build-hardware-page.mjs`
- [ ] `docs/.vitepress/components/HardwareTable.vue` (sort/filter/search/URL state/a11y)
- [ ] `docs/hardware/index.md` (page chrome)
- [ ] `docs/hardware/_data.json` generated
- [ ] Wire into `docs:build`
- [ ] Add `/hardware/` to top-level nav
- [ ] Per-driver fragment injected into `/<repo>/hardware`

### Gate (docs site)
- [ ] `npm run docs:build` succeeds
- [ ] Smoke test: facets toggle, sorts work, search by name + PID, URL hash restores
- [ ] commit

---

## Phase 3 — Verification guide + per-driver checklists

### .github repo
- [ ] `CONTRIBUTING/verifying-hardware.md`
- [ ] Update `.github/ISSUE_TEMPLATE/hardware_verification.yml` intro to link guide

### Each driver repo
- [ ] `docs/verification-checklist.md`
- [ ] Link from `docs/hardware.md`

### Gate (per repo)
- [ ] commit

---

## Phase 4 — Full driver authoring guide

- [ ] Replace `CONTRIBUTING/adding-a-driver.md` stub with full guide (§E)
- [ ] Cross-link verification + status material

### Gate
- [ ] Markdown links resolve (manual scan)
- [ ] commit

---

## Phase 5 — Maintainer runbook + finalize

- [ ] `CONTRIBUTING/maintainer-runbook.md`
- [ ] Move plan + progress + decisions from `plans/backlog/` to `plans/implemented/`
- [ ] Final commit

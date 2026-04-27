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

- [x] Deps unchanged (already on `^0.2.0` for all `*-core`; resolved 0.2.1 brother-ql, 0.2.1 labelmanager, 0.2.2 labelwriter from local node_modules)
- [x] `scripts/build-hardware-page.mjs` — merges DEVICES + YAML, writes `_data.json`, `index.md`, per-driver `_status-fragment.md`
- [x] `docs/.vitepress/components/HardwareTable.vue` — sort/multi-facet filter/search/URL state/a11y; no extra deps
- [x] `docs/hardware/index.md` (page chrome) — generated
- [x] `docs/hardware/_data.json` generated, gitignored
- [x] Wired into `docs:build` via new `docs:prep` (chains `docs:pull` → `docs:hardware`)
- [x] `/hardware/` added to top-level nav in `docs/.vitepress/config.ts`
- [x] Per-driver `_status-fragment.md` injected via VitePress `<!--@include-->` directive at end of each driver's `docs/hardware.md`
- [x] `srcExclude: ['**/_*.md']` added so include-only fragments don't leak as routes

### Gate (docs site)
- [x] `npm run docs:build` succeeds (5.7 s)
- [x] Smoke check: `_data.json` contains 36 devices across 3 drivers (19 + 6 + 11), `Hardware coverage` chrome rendered, fragments embedded in `/<driver>/hardware` pages.
- [ ] commit (next)

### Phase 2 notes
- Component is `<ClientOnly>`-wrapped; SSR is intentionally skipped because
  the URL-hash state restoration needs `window`. The static HTML still has
  the chrome + counts; the table itself hydrates client-side.
- Build script reads core packages from `node_modules` — pinned versions
  in `package.json` are the contract per D3.
- Pre-existing chunk-size warning is unrelated (LiveDemo bundles bring it).

---

## Phase 3 — Verification guide + per-driver checklists

### .github repo
- [x] `CONTRIBUTING/verifying-hardware.md`
- [x] Issue template intro now links guide + family checklist

### Each driver repo
- [x] `docs/verification-checklist.md` (family-specific)
- [x] Pointer paragraph from `docs/hardware.md` to local checklist

### Docs site
- [x] Sidebar: each driver gets a `Verification checklist` entry under its package
- [x] Build succeeds with all 3 checklists routed

### Gate (per repo)
- [x] All 3 drivers: typecheck + lint + test + validate:hardware-status pass
- [x] Docs site build clean (one fixed dead link en route)
- [ ] commit (next)

---

## Phase 4 — Full driver authoring guide

- [x] Replaced `CONTRIBUTING/adding-a-driver.md` stub with full guide
- [x] Cross-linked verification + status material (§10 Hardware coverage from day one)

### Phase 4 notes
- Guide leans heavily on existing drivers as worked examples ("see X")
  rather than re-explaining every detail. This keeps the doc maintainable —
  the source of truth stays in the repos that drift.
- The guide explicitly tells driver authors to seed `hardware-status.yaml`
  + `verification-checklist.md` on day one (§10), so the unified
  `/hardware/` page is populated from the moment a driver ships.
- §11 documents the docs-site fragment include directive that drivers
  need at the bottom of `docs/hardware.md`.

### Gate
- [x] Markdown links resolve (visual scan)
- [ ] commit (next)

---

## Phase 5 — Maintainer runbook + finalize

- [x] `CONTRIBUTING/maintainer-runbook.md` — triage flow, YAML edit
      conventions, device-addition flow, quirks editorial guidance,
      docs-site sync, cadence
- [x] CONTRIBUTING/README.md index updated to link the new guides
      (verifying-hardware, hardware-status-schema, maintainer-runbook)
- [x] Plan + progress + decisions moved from `plans/backlog/` to
      `plans/implemented/`
- [x] Final commit

## Final summary

Across 4 repos, 14 commits:

| Repo | Commits |
|---|---|
| `thermal-label/.github`              | 5 (schema doc, decisions, progress, verifying-hardware, adding-a-driver, maintainer-runbook, plan move) |
| `thermal-label/brother-ql`           | 3 (P1 validator+seed, P2 include, P3 checklist) |
| `thermal-label/labelmanager`         | 3 (P1 validator+seed, P2 include, P3 checklist) |
| `thermal-label/labelwriter`          | 3 (P1 validator+seed, P2 include, P3 checklist) |
| `thermal-label/thermal-label.github.io` | 3 (P2 unified page, P2 include exclude, P3 sidebar) |

Gates run clean on every commit boundary:

- All 3 driver repos: `pnpm typecheck`, `pnpm lint`, `pnpm test`,
  `pnpm validate:hardware-status` pass.
- Docs site: `npm run docs:build` succeeds with all routes
  generated (unified page, 3 verification checklists, per-driver
  fragments included).

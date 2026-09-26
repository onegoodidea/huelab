# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

huelab is an open source accessible color system toolkit — a pure TypeScript core library and an interactive client-side webapp. It enables building, auditing, and exporting accessible color systems with real-time feedback.

## Build & Run Commands

```bash
# Install dependencies (npm workspaces)
npm install

# Run all tests (core library)
npm test --workspace=@huelab/core

# Run tests in watch mode
npm run test:watch --workspace=@huelab/core

# Development (webapp)
npm run dev

# Build all packages (order: core → preset → webapp)
npm run build

# Type check all packages
npm run typecheck
```

## Testing

Tests use **Vitest** in `@huelab/core`. Test files live alongside source at `packages/core/src/__tests__/*.test.ts`. Follow TDD: write failing test, implement, verify pass, commit. Run the full suite before committing.

## Architecture

**Monorepo** with npm workspaces. Three packages under `packages/`:

### `@huelab/core` — Pure TypeScript library (zero DOM deps)

All functions are **immutable data in, immutable data out**. No side effects, no file I/O.

- `types.ts` — All shared interfaces (`OklchColor`, `Color`, `Ramp`, `RampStop`, `TokenDefinition`, `Preset`, `ContrastResult`, `AuditReport`, etc.)
- `oklch.ts` — Parsing, conversion (OKLCH ↔ hex ↔ RGB), gamut clamping, hue difference
- `contrast.ts` — WCAG 2.2 ratio + APCA Lc calculation
- `distance.ts` — Perceptual distance (deltaE OK), component decomposition (dL/dC/dH)
- `simulate.ts` — Colorblind simulation (protanopia, deuteranopia, tritanopia)
- `ramp.ts` — Generate OKLCH ramps from base color + parameters
- `overrides.ts` — Per-stop pin/override system (apply, clear, clearAll)
- `tokens.ts` — Semantic token schema definition and helpers
- `resolver.ts` — Resolves token mapping against loaded ramps → concrete OKLCH/hex values
- `audit.ts` — Check fg/bg token pairs against WCAG thresholds → `AuditReport`
- `import-css.ts` — Parse CSS custom properties (`:root` / `.dark` / `@theme` blocks)
- `import-figma.ts` — Parse Figma W3C Design Tokens JSON
- `export-css.ts` — Emit three-layer CSS (primitives, semantic, Tailwind mapping)
- `export-figma.ts` — Emit Figma variable JSON with alias support
- `export-json.ts` — Raw data export (ramp JSON, token mapping JSON)
- `preset.ts` — Preset validation and utilities

### `@huelab/preset-shadcn` — shadcn/ui + Tailwind v4 preset

Depends on `@huelab/core`. Provides 11 stops (50–950), 52 semantic tokens, 15 contrast pairs, and three-layer CSS output format.

### `@huelab/webapp` — React + Vite + Tailwind CSS v4

Client-side only, no backend. Three-panel layout: Ramp Editor (left), Token Map (center), Audit Panel (right), with a ramp overview strip at the bottom. State managed via `useReducer` + context.

## Ramp Generation Algorithm

The core algorithm generates 11-stop OKLCH ramps from a base color:

- **Lightness targets**: Fixed per stop (50=0.985, 100=0.93, 200=0.87, 300=0.80, 400=0.71, 500=0.62, 600=0.53, 700=0.45, 800=0.37, 900=0.27, 950=0.17). Non-linear — wider spacing in 300–700.
- **Chroma curves**: `natural` (bell, peak at 500), `linear`, `flat` (uniform — use for neutrals). Scale factors multiply the base color's chroma at each stop.
- **Base color**: Anchors hue + max chroma. It is NOT preserved verbatim — its lightness is overridden by the stop target. This ensures cross-ramp consistency.
- **Hue shift**: Linear interpolation from `-shift/2` at lightest to `+shift/2` at darkest.
- **Gamut mapping**: Reduce chroma, preserve lightness via culori's `clampChroma('oklch')`.
- **Achromatic inputs** (C < 0.001): Hue forced to 0 to avoid noise.
- **Per-stop overrides**: Any stop's L/C/H can be pinned independently. Overridden stops survive ramp param changes. Clear to snap back to algorithm.

## Key Design Decisions

- **OKLCH is the sole internal color space.** Convert to hex/RGB only at output boundaries.
- **WCAG 2.2 AA is the compliance gate** (4.5:1 normal text, 3:1 large text/UI). APCA Lc is computed and reported alongside but never gates pass/fail.
- **Presets are plain objects** conforming to the `Preset` interface — easy to create, share, and swap.
- **Three-layer CSS**: `@theme` (primitives) → `:root`/`.dark` (semantic tokens) → `@theme inline` (Tailwind mapping).
- **TypeScript ESM throughout.** Target ES2022, module ESNext, bundler resolution. Imports use `.js` extensions.

## Dependencies

- `culori` — OKLCH/sRGB conversion, gamut mapping, colorblind simulation, perceptual distance
- `apca-w3` — APCA Lc contrast calculation
- `@types/culori`, `@types/apca-w3` — Required devDeps (neither library bundles types)
- `vitest` — Test framework (core package)
- `react`, `react-dom` — Webapp UI
- `vite`, `@vitejs/plugin-react` — Webapp build
- `tailwindcss` v4 — Webapp styling

## Gotchas

- **culori `displayable()` fails on `#ffffff`** due to float precision (`l: 1.0000000000000002`). Use RGB 0–255 epsilon check instead — see `REFERENCE.md` for details.
- **`Math.round()` on tiny negatives** produces `-0`. Normalize with `|| 0`.
- **`sRGBtoY()` (apca-w3) expects `[R,G,B]` as 0–255 integers**, NOT culori's 0–1 floats.
- **`APCAcontrast()` returns a signed float**: positive = dark-on-light, negative = light-on-dark. Use `Math.abs()` for threshold checks.
- **Hue proximity ≠ visual similarity.** Two ramps with similar hues can look very different if chroma differs significantly. Always consider chroma and lightness together.

## Workflow

- **Always use a git worktree for feature work.** Use the `using-git-worktrees` skill to create an isolated worktree before starting any implementation. This prevents conflicts when multiple agents or sessions work in parallel. Each worktree needs its own `npm install`.
- Create a feature branch before committing (e.g., `task-1/oklch-module`). PRs for all changes.
- Follow TDD: failing test → minimal implementation → pass → commit.
- The implementation plan at `docs/plans/2026-02-24-huelab-implementation-plan.md` defines task ordering and dependencies. Use the `executing-plans` skill to work through it.

## Changelog & Versioning

- **Every PR merge to `main` is a version bump.** No exceptions.
- **`[Unreleased]` lives on feature branches only.** `main` never has an `[Unreleased]` section — its top entry is always the latest tagged version.
- **Update `CHANGELOG.md` at commit time.** When committing a `feat:` or `fix:` change on a feature branch, add a one-liner to the `[Unreleased]` section in the same commit. Use Keep a Changelog categories: Added, Changed, Fixed, Removed, Deprecated. Include issue/PR links when relevant. Skip entries for `docs:`, `style:`, `chore:`, test-only, or internal refactors.
- **Version at PR completion.** Before merging to `main`:
  1. Rename `[Unreleased]` → `[0.x.x] — YYYY-MM-DD` with the new version number (remove the `[Unreleased]` section entirely)
  2. Bump `version` in the relevant `package.json` files
  3. Update the comparison links at the bottom of `CHANGELOG.md`
  4. Commit as part of the PR
  5. After merge, `git tag v0.x.x` on `main`
- **Starting a new feature branch.** Add an `[Unreleased]` section at the top of `CHANGELOG.md` as part of the first feature/fix commit on the branch.
- **Semver.** Until 1.0, minor versions (0.x.0) may include breaking changes. Patches (0.x.y) for fixes.

## Key References

- `docs/plans/2026-02-24-huelab-oss-design.md` — Approved design (architecture, UI layout, preset system, MVP scope)
- `docs/plans/2026-02-24-huelab-implementation-plan.md` — 21-task phased plan for v0.1
- `REFERENCE.md` — Technical lessons and algorithm details from the prior consulting engagement. **Read before implementing color math modules.**

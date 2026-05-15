# CLAUDE.md

This repository is a fork of [OpenCut-app/OpenCut](https://github.com/OpenCut-app/OpenCut) — the open-source CapCut alternative. This file gives Claude Code the context it needs to be useful here from turn 1.

For the cross-agent architecture summary, also read [AGENTS.md](AGENTS.md). This file complements it with Claude Code-specific conventions, commands, and gotchas.

## Architecture in 30 seconds

- **Rust = source of truth.** All non-UI logic belongs in `rust/`. The migration from TypeScript into Rust is ongoing.
- **`apps/` are UI shells.** Each app (`web/` Next.js, `desktop/` GPUI) owns rendering and platform glue, never business logic.
- **`#[export]` macro bridges Rust ↔ TypeScript.** With `--features wasm` it expands to `#[wasm_bindgen]`; otherwise it's a no-op so the same crate compiles natively for desktop.

## Quick commands

| Task | Command |
|---|---|
| Start web dev server | `bun dev:web` (turbopack, autoshifts to :3001 if :3000 busy) |
| Install dependencies | `bun install` |
| Build WASM locally | `bun run build:wasm` (only if you edit `rust/wasm/`) |
| Watch + rebuild WASM | `bun dev:wasm` |
| Start DB + Redis | `docker compose up -d db redis serverless-redis-http` |
| Full Docker stack | `docker compose up -d` (app on :3100) |

Bun lives at `~/.bun/bin/bun`. PATH is set in `~/.zshrc`.

## Code rules to enforce

These are derived from `docs/` and `notes/`. When writing code in this repo, hold the line on them — don't quietly bypass.

1. **Actions only.** UI buttons, shortcuts, and context menus must go through `invokeAction()`. Never call `editor.xxx()` directly from a component — that bypasses toasts, validation, and keybinding support. See [docs/actions.md](docs/actions.md).
2. **Use `resolveEffectPasses`.** Never read `definition.renderer.passes` directly — the helper handles `buildPasses` vs static `passes` dispatch. See [docs/effects-renderer.md](docs/effects-renderer.md).
3. **Keyframe hooks for animatable fields.** Use `useKeyframedNumberProperty` / `useKeyframedColorProperty` (not `usePropertyDraft`) so keyframe vs static write dispatch is automatic. See [docs/keyframes.md](docs/keyframes.md).
4. **Primitives stay primitive.** A type that doesn't mention clips/tracks/effects/layers belongs in a primitives location, not in a domain folder. Don't import `Transform` from `@/rendering` — it's a 2D primitive that lives there for historical reasons. See [notes/primitives-vs-domains.md](notes/primitives-vs-domains.md).
5. **Logic → Rust, UI → apps/.** Before adding TypeScript business logic, check whether it should be a Rust crate. New features that compute, transform, or render anything should default to Rust with `#[export]`.
6. **Shader changes pair with registry entry.** WGSL shader in `rust/crates/gpu/src/shaders/` + identifier in `rust/crates/gpu/src/shader_registry.rs`. TypeScript only names the shader, never embeds WGSL.
7. **Read components before composing them.** Many components apply their own classes — pass-through `className` may or may not override depending on internal merge logic. Check the source first.

## Upstream sync

Origin is `yutasbl800t-ui/OpenCut` (this fork). Upstream is `OpenCut-app/OpenCut`.

```bash
git fetch upstream
git merge upstream/main          # or rebase, depending on preference
git push origin main
```

The repo has many upstream branches (rewrite, codebase-overhaul, ripple-editing, etc.). Only `main` is the canonical line — others are upstream's WIP.

## Repo gotchas

- **Bun required** — npm/yarn/pnpm will not produce a working install. The `bun.lock` is authoritative.
- **Docker is optional for the web app.** Skipping Docker means auth, projects, and other DB-backed features break, but the timeline and editor UI still render.
- **`opencut-wasm` is a published npm package.** Local Rust builds are unnecessary unless you edit `rust/wasm/`. To switch, see "Local WASM development" in [README.md](README.md).
- **Workspace root warning at dev start** comes from `~/package-lock.json` being detected. Harmless; can be silenced by setting `turbopack.root` in `apps/web/next.config.*`.
- **Default port collision** — Next.js autoshifts 3000 → 3001 if 3000 is taken. Check the boot logs for the real URL.
- **Contributing focus areas** (from upstream): timeline functionality, project management, performance, bug fixes. **Avoid for now**: preview panel enhancements and export — those are being refactored upstream with a new binary rendering approach.

## When in doubt

- Architectural questions → start at [AGENTS.md](AGENTS.md), then the relevant file in [docs/](docs/) or [notes/](notes/).
- Adding an effect → [docs/effects-renderer.md](docs/effects-renderer.md).
- Adding an animatable property → [docs/keyframes.md](docs/keyframes.md).
- Adding a keyboard shortcut or UI command → [docs/actions.md](docs/actions.md).
- Adding a Rust crate → [rust/README.md](rust/README.md).

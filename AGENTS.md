# AGENTS.md

This is a practical reference for working in the Terax codebase. For full architecture, see [TERAX.md](TERAX.md).

## Quick start

```bash
pnpm install
pnpm tauri dev        # dev mode with hot-reload (Vite + Tauri)
pnpm exec tsc --noEmit  # type-check
pnpm test             # vitest
```

Rust commands (run from `src-tauri/`):
```bash
cargo clippy --all-targets --locked -- -D warnings
cargo test --locked
cargo nextest run --locked   # CI uses nextest
```

## Repo map

```
src-tauri/src/   Rust backend (Tauri 2 commands)
  modules/pty/     PTY sessions, shell init, agent detection
  modules/fs/      File tree, search, grep, watch, mutate
  modules/git/     Source control commands
  modules/shell/   One-shot commands, persistent sessions, bg procs
  modules/net/     AI HTTP proxy with SSRF guard
  modules/secrets/ OS keychain (keyring crate)
  modules/workspace/ WSL bridge, workspace auth registry
  module/agent/    Claude Code hooks installer
  lib.rs           Command registration, plugin wiring

src/              React frontend
  modules/          Feature modules (terminal, editor, ai, tabs, ...)
  app/App.tsx       Coordinator - wires everything together
  components/ui/    shadcn primitives (don't hand-edit)
  components/ai-elements/  Vercel AI Elements (don't hand-edit)
  lib/              Shared utilities (cn, platform, fonts, launchDir)
  styles/           globals.css, code-highlight.css, fonts.css, tokens

vite.config.ts    Vite config with manual chunk splitting for AI SDKs
src-tauri/capabilities/default.json  Tauri plugin allowlist
```

## Architecture at a glance

**Two-process model**: Rust owns all OS access. Webview invokes via `invoke()` → Tauri commands. No direct FS/process/shell access from frontend.

**State management**: zustand stores for most global state; React context for theme, AI composer.

**Settings**: `tauri-plugin-store` file `terax-settings.json`. Keys in OS keychain (service: `terax-ai`), never on disk.

**AI sessions**: persisted as `terax-ai-sessions.json` with per-session messages under `messages:<id>` keys.

**Tabs**: tagged union (`terminal | editor | preview | markdown | ai-diff | git-diff | git-history | git-commit-file`). Hidden via CSS (not unmounted) on switch so PTYs stream continuously.

## Conventions

- **Comments**: only `// why`, never `// what`. No filler.
- **Imports**: always `@/...` on frontend, never relative across modules.
- **No emojis, no em-dashes** anywhere.
- **pnpm** only.
- **Cross-platform paths**: use `.split(/[\\/]/)` not `.split("/")`.
- **Frontend canonical path form**: forward-slash always.
- **Rust**: `#[cfg(unix)]` / `#[cfg(windows)]` for platform splits.

## Testing expectations

Core subsystems (PTY spawn, workspace auth, git commands, fs mutation, IPC surface, AI tools, OSC parsing) **require a test** that locks the invariant. Pure logic tests are inexpensive and expected. UI rendering / themes / type-level guarantees do not need tests.

- Frontend: vitest (`pnpm test`)
- Rust: cargo nextest + proptest used in fs/mod.rs
- Tests for security.ts cover path guards, symlink defense, Trojan Source attacks

## Gotchas

- **React 19 strict mode** double-mounts `useEffect` in dev. First PTY spawn gets cleaned up immediately. Normal.
- **Windows `SPAWN_LOCK`** mutex in `pty/session.rs` required around `openpty + spawn_command`. Don't remove.
- **Windows Job Object** (`pty/job.rs`) prevents orphaned process trees. Don't disable.
- **Windows `pty_close`** can block on `ClosePseudoConsole` — drain is threaded (see `pty_close` in pty/mod.rs).
- **macOS settings window**: no `parent()` because `child + always_on_top` puts settings behind main. Lifecycle tied via `on_window_event` instead.
- **AiComposerProvider** must be mounted unconditionally at App root — conditional mount remounts the tree and respawns all PTYs.
- **Tab cwd** arrives from OSC 7 as forward slashes. Anything sending it to a Rust fs command on Windows must normalize.
- **Settings cross-window sync**: uses Tauri events (`terax://prefs-changed`) because settings is a separate webview and `LazyStore.onChange` only fires in-process.
- **Vite dev server** runs on port 1420 (HMR on 1421). Ignores `src-tauri/` changes.
- **Build chunks**: each AI SDK is a separate manual chunk in vite.config.ts. Unused providers don't bloat initial load.

## Adding new things

### New Tauri command
1. Implement in `src-tauri/src/modules/<area>/`
2. Register in `lib.rs` `generate_handler![]`
3. If it accesses FS/network/processes, gate through workspace authorization

### New shadcn/ui component
```bash
pnpm dlx shadcn@latest add <component>
```
Don't hand-edit `src/components/ui/*`.

### New AI provider
Add to `PROVIDERS` in `src/modules/ai/config.ts` plus model entries in `MODELS`, pricing in `MODEL_PRICING`, context limit in `MODEL_CONTEXT_LIMITS`.

### New plugin dependency
1. `Cargo.toml` dep
2. `.plugin(...)` in `lib.rs` `run()`
3. Capability in `src-tauri/capabilities/default.json`

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Codex Plugin for Claude Code** (`@openai/codex-plugin-cc`) — a Claude Code plugin that wraps the [Codex CLI](https://developers.openai.com/codex/cli/) and [Codex app server](https://developers.openai.com/codex/app-server/) to enable code reviews and task delegation from within Claude Code. The plugin is distributed as a Claude Code marketplace plugin (not an npm package).

## Common Commands

```bash
# Install dependencies
npm ci

# Run all tests (Node.js built-in test runner, no Jest/Vitest)
npm test                                    # runs: node --test tests/*.test.mjs

# Run a single test file
node --test tests/commands.test.mjs

# Build (TypeScript type-checking only, no emit)
npm run build                               # runs: tsc -p tsconfig.app-server.json

# Version management
npm run check-version                       # verify all manifests have matching versions
npm run bump-version -- 1.0.6               # update version across all manifest files
```

## Architecture

### Two-Layer Structure

The repo has a **marketplace layer** at the root and a **plugin layer** under `plugins/codex/`:

- **Marketplace root** (`.claude-plugin/marketplace.json`): Defines the plugin marketplace metadata and points to `plugins/codex` as the plugin source.
- **Plugin root** (`plugins/codex/`): Contains the actual plugin logic — commands, hooks, agents, skills, scripts, and schemas.

### Key Directories

```
plugins/codex/
  .claude-plugin/plugin.json    # Plugin manifest (name, version)
  commands/*.md                  # Slash-command definitions (review, rescue, setup, etc.)
  agents/codex-rescue.md        # Subagent definition for task delegation
  hooks/hooks.json              # SessionStart, SessionEnd, Stop hooks
  prompts/                      # Prompt templates (adversarial review, stop gate)
  schemas/                      # JSON Schema for structured Codex output
  scripts/
    codex-companion.mjs          # Main CLI entrypoint — dispatches all subcommands
    app-server-broker.mjs        # Broker process for shared Codex runtime
    session-lifecycle-hook.mjs   # SessionStart/SessionEnd hook handler
    stop-review-gate-hook.mjs    # Stop hook for review gate
    lib/
      app-server.mjs             # JSON-RPC client for Codex app-server (spawn + broker transports)
      codex.mjs                  # High-level Codex operations (review, turn, auth, thread management)
      broker-lifecycle.mjs       # Broker session create/teardown/persistence
      job-control.mjs            # Job status, result resolution, cancellation logic
      tracked-jobs.mjs           # Job lifecycle tracking, progress reporting, log files
      state.mjs                  # Workspace state dir, config, job file I/O
      git.mjs                    # Git operations (diff, status, review target resolution)
      render.mjs                 # Human-readable output rendering for all commands
      args.mjs                   # Argument parsing utilities
      prompts.mjs                # Prompt template loading and interpolation
      workspace.mjs              # Workspace root resolution
  skills/                        # Claude Code skills (codex-cli-runtime, gpt-5-4-prompting, etc.)
scripts/
  bump-version.mjs               # Version bump script (updates all 4 manifest files atomically)
```

### How Commands Work

Each slash command (e.g., `/codex:review`) is defined as a markdown file in `plugins/codex/commands/`. These files use Claude Code's plugin command format with YAML frontmatter specifying:
- `allowed-tools`: Which tools the command can use
- `disable-model-invocation`: Whether Claude should not use model invocation
- `argument-hint`: Expected argument patterns

The command markdown instructs Claude to invoke `codex-companion.mjs` via `Bash`. The companion script dispatches to the appropriate handler based on the subcommand.

### App-Server Communication

The plugin communicates with Codex via two transport modes:
1. **Direct**: Spawns `codex app-server` as a child process, communicates via JSON-RPC over stdio.
2. **Broker**: Connects to a persistent broker process via Unix socket (or named pipe on Windows). The broker is shared across commands to avoid repeated startup costs.

`CodexAppServerClient.connect()` in `app-server.mjs` handles transport selection with automatic fallback from broker to direct when the broker is busy or unavailable.

### Job Tracking

Background tasks (reviews, rescue tasks) are tracked as "jobs" with:
- Job files stored under `<workspace>/.codex/companion/jobs/`
- Progress logs alongside job files
- Session-scoped filtering via `CODEX_COMPANION_SESSION_ID` environment variable
- Phases: queued → starting → reviewing/investigating → finalizing → done/failed/cancelled

### Review Gate (Optional Stop Hook)

When enabled via `/codex:setup --enable-review-gate`, the `Stop` hook runs a Codex review before allowing Claude to stop. This creates a Claude↔Codex loop — enable only when actively monitoring.

## Development Notes

- **Module system**: ESM (`"type": "module"` in package.json). All `.mjs` files.
- **TypeScript**: Used only for type-checking (no emit). The `tsconfig.app-server.json` targets core lib files. Generated types from `codex app-server generate-ts` go to `plugins/codex/.generated/`.
- **Node.js requirement**: >= 18.18.0 (uses `node:test`, `node:fs` promises, etc.)
- **Windows support**: The codebase handles Windows-specific concerns (shell spawning, process tree termination, named pipe broker endpoints, sandbox coercion to `danger-full-access`).
- **Version sync**: All 4 manifest files (`package.json`, `package-lock.json`, `plugin.json`, `marketplace.json`) must have matching versions. Use `npm run check-version` to verify, `npm run bump-version -- <version>` to update.
- **Tests**: Use Node.js built-in `node:test` with `node:assert/strict`. Tests are structural/contract tests that verify command markdown files contain expected patterns, hooks are configured correctly, and library functions behave as expected.

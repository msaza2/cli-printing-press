---
name: printing-press
description: Use the Printing Press from Codex to generate, polish, score, reprint, import, or publish agent-native Go CLIs and MCP servers from API descriptions, websites, HAR captures, OpenAPI specs, or public-library entries. Trigger when the user asks Codex to run Printing Press, print a CLI, create an API CLI, polish or score a generated CLI, publish to the Printing Press library, or adapt the original Claude Code Printing Press workflow for Codex.
---

# Printing Press For Codex

Use this skill as the Codex-native entrypoint for the Printing Press. The upstream
workflow was authored as Claude Code slash-command skills under `skills/`; this
adapter keeps those detailed phase contracts usable while mapping them onto
Codex tools and Codex safety rules.

## Core Rule

Treat the Printing Press binary as the deterministic engine and this skill as
the Codex operating contract. Prefer mechanical binary commands for generation,
verification, scoring, and packaging. Use agent judgment for research quality,
feature selection, semantic review, and deciding which issues are systemic.

## Tool Mapping

When a source Printing Press skill refers to Claude Code tools, use these Codex
equivalents:

| Source tool | Codex equivalent |
| --- | --- |
| `Bash` | `shell_command` |
| `Read` | `shell_command` with `Get-Content`, `rg`, or `rg --files` |
| `Glob` | `shell_command` with `rg --files` or `Get-ChildItem` |
| `Grep` | `shell_command` with `rg` |
| `Write` / `Edit` | `apply_patch` for tracked source edits |
| `WebSearch` / `WebFetch` | `web` search/open, with citations when reporting sourced facts |
| `AskUserQuestion` | Ask a concise plain-text question only when genuinely blocked |
| `Agent` | `spawn_agent` only when the user explicitly allowed subagents or parallel agent work |
| `Skill` | Read the relevant sibling skill file under `../../skills/` as reference, then apply this mapping |

If a source skill says to use a Claude-only command such as `/review`,
`/simplify`, or a Claude plugin command, use Codex's available local review,
diff, tests, and direct edits instead. Do not shell out to another agent unless
the user explicitly asked for that delegation.

## Setup

Before running Printing Press commands:

1. Resolve the repository root with `git rev-parse --show-toplevel` when inside a
   git checkout; otherwise use the current directory.
2. Prefer a local repo binary at `<repo-root>/printing-press` when the checkout
   contains `cmd/printing-press`.
3. Otherwise resolve `printing-press` from `PATH`; on Windows also check
   `$HOME/go/bin/printing-press.exe`.
4. If no binary exists but this repo is checked out and Go is installed, build it
   with `go build -o ./printing-press ./cmd/printing-press`.
5. If the binary still cannot be resolved, tell the user to install it:

```bash
go install github.com/mvanhorn/cli-printing-press/v4/cmd/printing-press@latest
```

Use the resolved absolute binary path for every later command. Do not rely on an
exported `PATH` from a previous shell call.

Initialize the normal Printing Press locations when needed:

```bash
PRESS_HOME="$HOME/printing-press"
PRESS_LIBRARY="$PRESS_HOME/library"
PRESS_MANUSCRIPTS="$PRESS_HOME/manuscripts"
```

On Windows PowerShell, use the equivalent `$env:USERPROFILE\printing-press`
paths and create directories with `New-Item -ItemType Directory -Force`.

## Choosing The Workflow

Use this dispatch table:

| User intent | Source workflow to consult |
| --- | --- |
| Generate or print a new CLI | `../../skills/printing-press/SKILL.md` |
| Polish a generated CLI | `../../skills/printing-press-polish/SKILL.md` |
| Score or compare generated CLIs | `../../skills/printing-press-score/SKILL.md` |
| Publish a CLI to the public library | `../../skills/printing-press-publish/SKILL.md` |
| Reprint an existing library CLI | `../../skills/printing-press-reprint/SKILL.md` |
| Import a public-library CLI locally | `../../skills/printing-press-import/SKILL.md` |
| Review sampled generated output | `../../skills/printing-press-output-review/SKILL.md` |
| Browse catalog entries | Prefer `printing-press catalog list/search/show`; consult `../../skills/printing-press-catalog/SKILL.md` only for legacy behavior |

Read only the source workflow needed for the user's request. If that workflow
references files under its `references/` directory, load only the named reference
at the phase where it becomes relevant.

## Generation Loop

For a new CLI request:

1. Run setup and record the resolved binary path.
2. Determine the input shape: API name, website URL, OpenAPI path/URL, HAR file,
   or existing catalog entry.
3. Use the main source workflow for research, spec discovery, sniffing decisions,
   absorb/transcendence review, generation, build, shipcheck, and optional live
   smoke testing.
4. Keep generated artifacts under the Printing Press runstate/library paths that
   the source workflow defines.
5. After generation, run the highest-value available checks from the source
   workflow before reporting status.

When the source workflow mentions "Codex mode", treat this Codex session as the
active implementation host. Do not invoke `codex exec` from inside Codex unless
the user explicitly asks for nested CLI delegation.

## Editing And Verification

Use `apply_patch` for source edits. Never rewrite unrelated user changes.

Before claiming a result, run the command that proves it:

- Skill metadata change: run the bundled `quick_validate.py` against the Codex
  skill directory when available.
- Plugin manifest change: parse `.codex-plugin/plugin.json` as JSON.
- Go behavior change: run `go test ./...` when Go is available.
- Generated CLI change: run the relevant `printing-press verify`, `dogfood`,
  `scorecard`, or build command from the source workflow.

If a verifier cannot run because a dependency such as Go or GitHub CLI is
missing, report that exact blocker and any lower-level validation that did run.

## Publishing

When the user asks to push repository edits, inspect `git status -sb` and stage
only the intended files. Use the repository commit convention from `AGENTS.md`.
For this repo, skill and plugin packaging changes normally use:

```bash
docs(skills): add codex printing press adapter
```

Push to the configured fork remote unless the user asks for a PR or a different
target.

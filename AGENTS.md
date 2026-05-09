# AGENTS.md

Compact instructions for agents working in this repo.

## Architecture

Two-component system:
- **Go server** (`cmd/server/main.go`): HTTP API that executes filesystem operations inside a sandboxed directory. Gin-based, single binary.
- **Chrome extension** (`extension/`): Content script intercepts `<tool>` tags from AI web pages, renders execution cards, proxies to the local server.

## Development Commands

```bash
# Server
go run cmd/server/main.go                          # default: current dir, port 39527, timeout 60s
go run cmd/server/main.go -dir=/path -port=8080    # custom workspace/port
go build -o openlink cmd/server/main.go            # produce binary

# Extension
cd extension && npm install && npm run build        # outputs to extension/dist/

# Quick server test
curl http://127.0.0.1:39527/health                 # health check
curl -X POST http://127.0.0.1:39527/exec \
  -H "Content-Type: application/json" \
  -d '{"name":"exec_cmd","args":{"command":"ls"}}'
```

No test suite exists in this repo.

## Request Flow (how it works)

AI outputs `<tool>` tags → extension content script detects via MutationObserver → user clicks "执行" → HTTP POST to `/exec` on local server → executor dispatches to tool implementation → sandbox validates path/command → filesystem access.

The `/exec` endpoint expects a JSON body with `name` (tool name) and `args` (key-value map). All endpoints require Bearer token auth from `~/.openlink/token`, except `/health` and `/auth`.

## Server internals

- **Module**: `github.com/afumu/openlink`
- **Entry point**: `cmd/server/main.go` parses flags, loads config (`internal/types/types.go`), wires Gin router (`internal/server/server.go`) with executor.
- **Tools** live in `internal/tool/`. Registered via registry in `executor.New()`: exec_cmd, list_dir, read_file, write_file, edit, glob, grep, web_fetch, question, skill, todo_write.
- **Security**: `SafePath()` validates all file paths stay within RootDir using EvalSymlinks + prefix check. `IsDangerousCommand()` blocks destructive/network/privilege commands.
- **Quirk — system prompt re-injection**: every 20 tool calls the executor re-injects the full init_prompt.txt; between those it appends a short identity reminder.
- **Quirk — edit tab fix**: the server replaces `\t` with `\n\t` in `old_string`/`new_string` of edit requests, because some AI models output tabs instead of newlines.

## Skills system

Skills are SKILL.md files scanned from multiple directories (priority order):
1. `<rootDir>/.skills/`
2. `<rootDir>/.openlink/skills/`
3. `<rootDir>/.agent/skills/` and `.claude/skills/`
4. `~/.openlink/skills/`, `~/.agent/skills/`, `~/.claude/skills/`

Each skill directory contains SKILL.md with frontmatter (`name`, `description`). Skills are injected into the prompt on `/prompt` and listed by `/skills`.

## Extension quirks

- Build output goes to `extension/dist/`. Load this path in Chrome Dev Mode.
- Platform support varies: some sites (DeepSeek, Kimi, Mistral, Perplexity) use an injected script instead of MutationObserver. See `getSiteConfig()` in `extension/src/content/index.ts` for per-platform selectors and fill methods.

## Token / Auth

Token stored at `~/.openlink/token`. All API endpoints (except `/health`, `/auth`) require `Authorization: Bearer <token>` header. The server reads this file on startup.

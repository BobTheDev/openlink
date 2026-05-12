# AGENTS.md

Compact instructions for agents working in this repo.

## Architecture

Two-component system:
- **Go server** (`cmd/server/main.go`): HTTP API that executes filesystem operations inside a sandboxed directory. Gin-based, single binary.
- **Chrome extension** (`extension/`): Content script intercepts `<tool>` tags from AI web pages, renders execution cards, proxies to the local server.

## Requirements

- Go 1.23+ (toolchain 1.24.10)
- Node.js 18+
- Module: `github.com/afumu/openlink`

## Development Commands

```bash
# Server
go run cmd/server/main.go                               # default: current dir, port 39527, timeout 60s
go run cmd/server/main.go -dir=/path -port=8080        # custom workspace/port
go build -o openlink cmd/server/main.go                 # produce binary

# Extension
cd extension && npm install
npm run build                   # production build → extension/dist/
npm run dev                     # watch mode (auto-rebuild on changes)

# Quick server test
curl http://127.0.0.1:39527/health                     # health check (no auth)
curl -X POST http://127.0.0.1:39527/exec \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $(cat ~/.openlink/token)" \
  -d '{"name":"exec_cmd","args":{"command":"ls"}}'
```

## Testing

```bash
go test ./...                           # all tests
go test -run TestExec ./internal/tool/  # specific package
go test -v ./internal/...             # verbose output
```

No CI lint/typecheck pipeline exists. Go fmt is conventional style.

## Request Flow (how it works)

AI outputs `<tool>` tags → extension content script detects via MutationObserver → user clicks "执行" → HTTP POST to `/exec` on local server → executor dispatches to tool implementation → sandbox validates path/command → filesystem access.

The `/exec` endpoint expects a JSON body with `name` (tool name) and `args` (key-value map). All endpoints require Bearer token auth from `~/.openlink/token`, except `/health` and `/auth`.

## Server internals

- **Module**: `github.com/afumu/openlink`
- **Entry point**: `cmd/server/main.go` parses flags, loads config (`internal/types/types.go`), wires Gin router (`internal/server/server.go`) with executor.
- **Tools** live in `internal/tool/`. Registered via registry in `executor.New()`: exec_cmd, list_dir, read_file, write_file, edit, glob, grep, web_fetch, question, skill, todo_write.
- **Security**: `SafePath()` validates all file paths stay within RootDir using EvalSymlinks + prefix check. `IsDangerousCommand()` blocks destructive/network/privilege commands (e.g., rm -rf, curl, sudo).
- **Quirk — system prompt re-injection**: every 20 tool calls the executor re-injects the full init_prompt.txt; between those it appends a short identity reminder.
- **Quirk — edit tab fix**: the server replaces `\t` with `\n\t` in `old_string`/`new_string` of edit requests, because some AI models output tabs instead of newlines.
- **Quirk — output truncation**: tool outputs exceeding 2000 lines or 50 KB are truncated; the full output is saved to `~/.openlink/tool-output/<id>` with a hint pointing to it. Use `read_file` with `offset` to read truncated results.
- **Quirk — question tool**: both server and extension participate. The executor blocks on a Go channel waiting for user response via `/question/<id>` endpoint; the extension renders an interactive popup form in the execution card. When user submits, the extension POSTs to `/question/<id>`, which resolves the channel.
- **Quirk — write_file**: creates parent directories automatically if they don't exist (verified at `internal/tool/writefile.go:46`).

## Platform support (extension)

| Platform | fillMethod | useObserver | Notes |
|----------|-----------|-------------|-------|
| Google AI Studio | value | true | Recommended; writes to System Instructions |
| Google Gemini | execCommand | true | |
| ChatGPT | prosemirror | true | |
| 通义千问 (Qwen) | value | true | |
| DeepSeek | paste | false | Uses injected.js |
| Kimi | execCommand | false | |
| Mistral | execCommand | false | |
| Perplexity | execCommand | false | |

See `getSiteConfig()` in `extension/src/content/index.ts` for full list and per-platform selectors/fill methods.

## API Endpoints (auth required unless noted)

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Health check (no auth) |
| `/auth` | POST | Submit token, returns confirmation (no auth) |
| `/exec` | POST | Execute a tool call — body: `{"name": "...", "args": {...}}` |
| `/prompt` | GET | Get system prompt with skills injected |
| `/skills` | GET | List available skills |
| `/files?q=` | GET | Filename lookup for @ completion (supports glob) |
| `/question/<id>` | POST | Resolve a question tool — body: `{"answer": "..."}` |

## Skills system

Skills are SKILL.md files scanned from multiple directories (priority order):
1. `<rootDir>/.skills/`
2. `<rootDir>/.openlink/skills/`
3. `<rootDir>/.agent/skills/` and `.claude/skills/`
4. `~/.openlink/skills/`, `~/.agent/skills/`, `~/.claude/skills/`

Each skill directory contains SKILL.md with frontmatter (`name`, `description`). Skills are injected into the prompt on `/prompt` and listed by `/skills`.

## Extension quirks

- Built with Vite + TypeScript; build output goes to `extension/dist/`. Load this path in Chrome Dev Mode.
- Platform support varies: some sites (DeepSeek, Kimi, Mistral, Perplexity) use an injected script instead of MutationObserver. See `getSiteConfig()` in `extension/src/content/index.ts` for per-platform selectors and fill methods.
- **Slash commands**: typing `/` in a chat editor triggers skill autocomplete via the server's `/skills` endpoint; selecting a skill replaces the token with `<tool name="skill">...` XML.
- **@ file completion**: typing `@` triggers filename lookup via the server's `/files?q=` endpoint. Cached for 5 seconds per query.
- **Auto-send**: after tool execution, the extension auto-submits the editor after a random delay (default 1–4s). Can be disabled in popup settings. If send button not found, falls back to keyboard Enter event.

## Token / Auth

Token stored at `~/.openlink/token`. All API endpoints (except `/health`, `/auth`) require `Authorization: Bearer <token>` header. The server reads this file on startup.

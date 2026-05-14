# AGENTS.md

## Dev Commands

```bash
# Go server
go run cmd/server/main.go -dir=/your/workspace -port=39527 -timeout=60
go build -o openlink cmd/server/main.go
go test ./...

# Chrome extension (Vite + React, NOT webpack)
cd extension && npm install && npm run build   # outputs to extension/dist/
npm run dev        # watch mode for extension
```

## Project Structure

- `cmd/server/main.go` — Go server entrypoint
- `internal/` — All Go server code
  - `executor/executor.go` — Tool dispatcher (add new tools here)
  - `security/sandbox.go` — SafePath() validation (required for all file ops)
  - `security/auth.go` — Token auth
  - `server/server.go` — Gin HTTP routes
- `extension/src/content/index.ts` — Content script + AI platform configs
- `extension/public/manifest.json` — Extension manifest; update `content_scripts.matches` when adding new AI platforms
- `prompts/init_prompt.txt` — System prompt template

## Adding New Tools

In `internal/executor/executor.go`: add a `case` in `Execute()` and register in `ListTools()`. **Every file path must go through `security.SafePath()` first.**

## Adding New AI Platform

In `extension/src/content/index.ts`, add to `getSiteConfig()`. Also update `content_scripts.matches` and `web_accessible_resources.matches` in `extension/public/manifest.json`.

## Skills

Scanned directories (priority order):
```
<rootDir>/.skills/   <rootDir>/.openlink/skills/   <rootDir>/.agent/skills/   <rootDir>/.claude/skills/
~/.openlink/skills/  ~/.agent/skills/             ~/.claude/skills/
```

Each skill is a subdirectory with `SKILL.md` (frontmatter: `name`, `description`).

## Testing the Server

```bash
curl http://127.0.0.1:39527/health
curl http://127.0.0.1:39527/skills -H "Authorization: Bearer <token>"
curl http://127.0.0.1:39527/files?q=main -H "Authorization: Bearer <token>"
# Token is stored in ~/.openlink/token
```

## Release

Push a git tag — GitHub Actions (`.github/workflows/release.yml`) auto-builds via goreleaser (`.goreleaser.yml`):
```bash
git tag v1.0.0 && git push origin v1.0.0
```
Output: platform binaries + `extension.zip`.

## Extension Installation

After `npm run build`, load `extension/dist/` in Chrome via `chrome://extensions/` (Developer mode → Load unpacked).
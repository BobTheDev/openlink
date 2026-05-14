# AGENTS.md

## Essential Commands

```bash
# Build server binary
go build -o openlink cmd/server/main.go

# Run server (requires -dir flag to specify workspace)
go run cmd/server/main.go -dir=/your/workspace

# Run tests
go test ./...

# Build extension (outputs to extension/dist/)
cd extension && npm install && npm run build

# Release: push git tag to trigger CI
git tag v1.0.0 && git push origin v1.0.0
```

## Architecture

- **Server**: `cmd/server/main.go` - Gin HTTP server on port 39527
- **Extension**: `extension/` - TypeScript/React/Vite, loads from `extension/dist/`
- **Entry point**: `internal/executor/executor.go` dispatches all tools

## Key Constraints

- All file operations must go through `security.SafePath()` validation
- Dangerous commands (rm -rf, sudo, curl, wget) blocked by `IsDangerousCommand()`
- Token auth required for all `/exec` and `/skills` endpoints
- Extension must be rebuilt after changes (`npm run build`)

## Files to Reference

- [CLAUDE.md](CLAUDE.md) - Full development and architecture documentation
- [docs/development.md](docs/development.md) - Detailed setup guide
- [README.md](README.md) - Usage and installation

## Release Process

1. Bump version in `extension/package.json`
2. Create git tag: `git tag v<x.y.z>`
3. Push tag to trigger GitHub Actions
4. CI builds: Go binaries + extension.zip via goreleaser
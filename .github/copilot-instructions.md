# Whispaste — Copilot Instructions

## Project Overview

- Single-binary Windows desktop app, Go + CGO (malgo audio)
- System tray app with WebView2 settings UI
- Records speech, transcribes via OpenAI Whisper API, and pastes text anywhere
- Dependencies: malgo, systray, webview_go, hotkey, golang.org/x/sys/windows

## Build & Test

- Build: `$env:CGO_ENABLED="1"; go build -ldflags="-s -w -H windowsgui" -o whispaste.exe .`
- Debug build: `$env:CGO_ENABLED="1"; go build -o whispaste.exe .` (no `-H windowsgui`)
- Tests: `$env:CGO_ENABLED="1"; go test -v -count=1 ./...`
- Requires MinGW GCC in PATH (CGO dependency)
- Go 1.24+ required (dependency constraint)

## Architecture

- File-per-domain: audio.go, api.go, config.go, hotkey.go, overlay.go, paste.go, tray.go, ui.go, update.go, etc.
- Settings UI: single embedded HTML file (ui_settings.html) with CSS/JS, loaded via WebView2
- Localization: data-i18n attributes + JS translations object (EN/DE)
- Logging: structured file logging in logger.go (logDebug/logInfo/logWarn/logError)
- Config: JSON in %APPDATA%\Whispaste\config.json, thread-safe with sync.RWMutex

## Code Conventions

- Use `sync.RWMutex` for shared state (Config, Updater, etc.)
- Config getters use `mu.RLock()/mu.RUnlock()` pattern
- Wrap errors with `fmt.Errorf("context: %w", err)`
- Use `logInfo()`, `logWarn()`, `logError()` from logger.go — never `log.Printf` directly
- Win32 API calls via `golang.org/x/sys/windows` LazyDLL/NewProc pattern
- Embedded resources via `//go:embed`

## UI Icons

- **Never use emojis as icons** — always use inline SVG icons from Lucide (https://lucide.dev)
- Icons in the WebView settings UI use the `.icon` CSS class with `currentColor` stroke
- For dynamic icon updates in JavaScript, use `element.innerHTML` with SVG markup, never `element.textContent` with emoji

## Testing

**Motto: so wenig Tests wie möglich, so viel wie nötig.**
Goal: maximum stability gain at minimum maintenance cost. Not a goal: coverage metrics, enterprise mocking patterns, exhaustive test suites.

### Priority tiers

| Tier | Scope | Write tests? |
|------|-------|-------------|
| **P0** | Auth, payments, data deletion, irreversible actions | Always |
| **P1** | Core user flows (primary happy + failure path) | Always |
| **P2** | Edge cases, nice-to-have coverage | Skip |

Only write P0 and P1 tests. Skip P2.

- Tests in `package main` (same package) for access to unexported functions
- Use `httptest.NewServer` for HTTP-dependent tests
- Use `t.TempDir()` for file isolation
- No mocking frameworks — keep it simple

## Security

- HTTPS-only for all network requests (API, updates, checksums)
- SHA256 checksum verification for auto-updates
- API key stored in config.json with 0600 permissions, never logged
- openURL binding validates https:// prefix before opening
- No admin rights required

## Release & Distribution

- CI: `.github/workflows/ci.yml` (vet + test + build + secret scan)
- Release: `.github/workflows/release.yml` (tag-triggered, version injection via ldflags, SHA256 checksums)
- Auto-update: GitHub Releases API, disabled when running as MSIX Store package
- MSIX packaging: `msix/AppxManifest.xml` for Microsoft Store distribution

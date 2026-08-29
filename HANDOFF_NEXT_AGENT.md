# x-browser-mcp — Handoff Notes
**Date:** 2026-08-29  **Author:** Solar Pro4  **Session:** desktop

## What we did
- Cloned `github.com/feitangyuan/x-browser-mcp` (Go, rod/cdp-based X/Twitter browser MCP server).
- Ran a full security pass: no outbound call-home, no exfiltration, no telemetry, no token uploads, only localhost HTTP server + CDP to a local Chrome instance + whatever the browser itself does on x.com.
- Confirmed the fork at `github.com/tortugas-agent/x-browser-mcp` is a clean fork (0 unique commits vs upstream), so it's a safe base to extend.

## Verdict
**Safe to use for local personal use.** The only real exposure is the on-disk session-cookie file (`x_session_cookies.json`, mode 0600) — intentional for session reuse, risky only if the machine itself is compromised.

## Existing capabilities (2 tools + 2 status ops)
- `check_login_status` / `start_login` (login lifecycle)
- `search_x` (search X discussions)
- `read_home_timeline` (home timeline)

## Auth model
- Local Chrome session via rod/cdp.
- Profile mode (default): persistent isolated Chrome profile in `./debug-chrome`.
- Cookie mode: saves auth_token+ct0 to `x_session_cookies.json` and reloads.
- Remote-debug mode: attach to an existing Chrome DevTools endpoint.
- Login state detected by DOM selectors + cookie presence; no API keys needed.

## Where the code lives (local checkout)
`/Users/mac-mini/x-browser-mcp/` — clean working tree, `main` branch. Remotes: `origin` = upstream `feitangyuan/x-browser-mcp`, `fork` = `github.com/tortugas-agent/x-browser-mcp`.

## Notes / caveats
- X changes its DOM frequently; the repo already has a CDP API-capture path + DOM fallback path. Extend using the same two-path pattern.
- No auth on the HTTP/MCP port (default `:18110`); anyone/anything on the machine can call it. Bind to 127.0.0.1 if you want stricter local isolation.
- The `rod` library is the guts of browser automation; new features should follow the existing `BrowserSession` / `SearchService` patterns.

## Handoff decision points (pick one)
1. **Stay minimal** — keep only search + home timeline; don't extend. Simplest, lowest maintenance.
2. **Extend read-only** — add user search / profile / user posts timeline. Same read-only, low-risk pattern; useful for richer X signals.
3. **Extend read-write** — add follow/unfollow, followers/following lists, maybe DMs. Higher risk, more complex X UI, more maintenance burden.
4. **Replace/fork differently** — if rod/cdp scraping turns out fragile, consider a different local-X approach later (only if this one proves unreliable in practice).

## If extending (read-only route, preferred)
- Follow the existing tool wiring in `mcp_server.go` + `http_server.go` + `service.go`.
- New data fetching should reuse `newAuthenticatedPage` / `ensureLoggedIn`.
- Keep the two-path pattern: try CDP API capture first, fall back to DOM.
- Add types in `types.go`, parsing in `parser.go`, service methods in `service.go`, tools/endpoints in `mcp_server.go` / `http_server.go`.
- Tests: mirror existing `*_test.go` style.

## Files of interest
`main.go`, `config.go`, `browser_session.go`, `service.go`, `parser.go`, `types.go`, `mcp_server.go`, `http_server.go`, `go.mod`, `README.md`.

## Next action we did not take
We stopped after the security audit + fork review and the proposed extension roadmap. We did **not** implement any new tools. If you want extensions, decide scope first (see decision points), then implement — preferably read-only first.

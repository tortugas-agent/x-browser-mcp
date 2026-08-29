# x-browser-mcp — Handoff for Phase 1 Extensions

**Date:** 2026-08-29  
**Author:** Hermes Agent (Solar Pro4)  
**Repo:** `tortugas-agent/x-browser-mcp` (fork of `feitangyuan/x-browser-mcp`)  
**Goal:** Add three read-only MCP tools: `search_users`, `get_user_profile`, `get_user_posts`

---

## Repo state

- Local copy at `/Users/mac-mini/x-browser-mcp`
- Fork is clean — identical to upstream `feitangyuan/x-browser-mcp:main`
- Remotes: `origin` → upstream, `fork` → `https://github.com/tortugas-agent/x-browser-mcp.git`
- Branch: `main`, clean working tree
- Go 1.24, three direct deps: `go-rod/rod v0.116.2`, `modelcontextprotocol/go-sdk v0.7.0`, `sirupsen/logrus v1.9.3`

---

## What exists today (working)

### HTTP endpoints
```
GET  /health
GET  /api/v1/login/status
POST /api/v1/login/start
GET  /api/v1/home?limit=N
POST /api/v1/search   (body: {"query","mode","limit"})
POST /mcp             (MCP streamable HTTP)
```

### MCP tools
- `check_login_status` — is the X session ready?
- `start_login` — opens a visible browser for manual login
- `read_home_timeline` — reads home timeline
- `search_x` — searches X discussions

### Core patterns (this is what Phase 1 must follow)

**1. Browser lifecycle**

`BrowserSession` (in `browser_session.go`) wraps a `*rod.Browser`. Two modes:
- **Profile mode** (default): uses a persistent Chrome user-data-dir (`./debug-chrome`), launches a real browser process, reuses the profile across requests. Login is captured as persistent profile state.
- **Cookie mode**: saves auth cookies (`auth_token` + `ct0`) to `x_session_cookies.json` (0600 perms), loads them back on each request.
- **Remote debug mode**: attaches to an existing Chrome DevTools endpoint (`--remote-debugging-port`).

**2. How search works (the template to copy)**

`SearchX()` in `service.go`:
1. Validates request
2. Checks in-memory cache (5 min TTL, keyed by `query|mode|limit`)
3. Rate-limits via `waitForTurn` (min interval + rolling budget)
4. Opens an authenticated browser session (`newAuthenticatedPage`)
5. Captures the X internal API response via CDP `Network.ResponseReceived` — listens for responses with URL containing `"SearchTimeline"`, grabs the body via `Network.getResponseBody`
6. Parses the JSON payload with `ParseSearchTimeline` (in `parser.go`)
7. If API capture fails → falls back to DOM scraping (`extractPostsFromDOM`)
8. Builds summary + returns result

**3. DOM scraping pattern**

`readDOMCandidates()` in `service.go`:
- Executes JS in the page context via `page.Eval()`
- Queries `article[data-testid="tweet"]` elements
- Extracts href, text, author handle, name, reply/repost/like counts from `aria-label` and `data-testid` attributes
- Returns `[]DOMPostCandidate` → converted to `[]XPost` in `ConvertDOMCandidatesToPosts()`

**4. Login detection**

`isLoggedIn()` in `service.go`:
- Checks if URL contains `/i/flow/login` → not logged in
- Looks for DOM elements: `SideNav_AccountSwitcher_Button`, `AppTabBar_Profile_Link`, `a[href="/home"]`
- Falls back to checking for `auth_token` + `ct0` cookies present

**5. HTTP server wiring**

In `http_server.go`:
- `AppServer` wraps `SearchService` + `*mcp.Server`
- Each endpoint is a method on `AppServer` that calls into `service`, writes JSON responses
- `loggingMiddleware` logs `METHOD PATH` only
- MCP handler is mounted at `/mcp` and `/mcp/`

**6. MCP tool wiring**

In `mcp_server.go`:
- `InitMCPServer(service)` creates the MCP server and registers tools with `mcp.AddTool()`
- Each tool has a `Name`, `Description`, and a handler
- Tool handlers call service methods, return `textResult()` (text content) or `errorResult()`

---

## Phase 1 tools to add — exact scope

### 1. `search_users`

**MCP tool:** `search_users`  
**HTTP endpoint:** `POST /api/v1/users/search`  
**Request body:**
```json
{"query": "mcp", "limit": 10}
```
**Response:** list of users matching the query — handle, name, bio, follower/following counts, verified status, profile image URL.

**How it works (proposed):**
- Navigate to `https://x.com/search?q=QUERY&src=typed_query`
- Capture the timeline API response via the same CDP interception technique used by `captureTimelinePayload` — but look for a different URL pattern (user search results, not tweet timeline)
- Parse user entries from the JSON
- DOM fallback: query for user-search-result cards on the page

**Key unknowns to discover:**
- What is the X internal API URL for user search results? (need to observe it via CDP or inspect the network)
- What does the JSON structure for user search results look like? (need a sample)
- Does the DOM fallback work for user search? (need to test on a real page)

**Suggested approach:** Start by instrumenting the existing flow — run a user search through the browser, capture all network responses, and identify which one contains user data. Then write the parser.

---

### 2. `get_user_profile`

**MCP tool:** `get_user_profile`  
**HTTP endpoint:** `GET /api/v1/users/profile?handle=X`  
**Response:** user profile data — name, handle, bio, follower/following/tweet counts, join date, verification status, protected flag, profile/banner image URLs, location, website, pronouns.

**How it works (proposed):**
- Navigate to `https://x.com/USERNAME`
- Wait for page load
- Extract profile header data from DOM — the profile page has stable elements:
  - Name, handle, bio in the header area
  - Follower/following/tweet counts (parse from `aria-label` like the existing code does for tweet metrics)
  - Verified badge (look for verified icon or `data-testid`)
  - Join date, location, website, pronouns in the profile metadata section
- No API capture needed for this one — DOM scraping should be sufficient for profile data

**Key unknowns:**
- Exact DOM structure of the current X profile page (it changes frequently)
- Which `data-testid` attributes are reliable for profile fields

**Suggested approach:** Navigate to a real profile, dump the DOM / use CDP to inspect elements, map out stable selectors.

---

### 3. `get_user_posts`

**MCP tool:** `get_user_posts`  
**HTTP endpoint:** `GET /api/v1/users/posts?handle=X&limit=N`  
**Response:** list of posts from a user's profile timeline — same `XPost` structure as search results.

**How it works (proposed):**
- Navigate to `https://x.com/USERNAME`
- Ensure the "Posts" tab is active (the profile has tabs: Posts, Replies, Likes, Media, etc.)
- Use the same scroll-and-collect pattern as `extractHomeTimelinePosts` — scroll the profile timeline, collect `article[data-testid="tweet"]` elements, convert to `XPost`
- Pagination via scroll (same as home timeline)

**Key differences from home timeline:**
- Need to click the "Posts" tab if not already active
- Profile timeline may have different DOM structure for tabs

---

## Files to modify / add

### Modify
- `types.go` — add `UserSearchRequest`, `UserSearchResult`, `UserProfile`, `UserProfileRequest` types
- `parser.go` — add `ParseUserSearchResults()` if API capture is used; add DOM helpers for profile extraction
- `service.go` — add `SearchUsers()`, `GetUserProfile()`, `GetUserPosts()` methods to `SearchService`
- `mcp_server.go` — register three new MCP tools
- `http_server.go` — add three new HTTP handlers + routes
- `config.go` — no changes needed (no new config)

### Add (if needed)
- New test files for the new parsing / service methods (follow the existing `*_test.go` pattern)

---

## Gotchas and pitfalls

1. **X changes its DOM frequently.** The existing code already has fallbacks for this reason. Don't over-engineer selectors — prefer `data-testid` attributes and broad patterns.

2. **User search API may not be captured the same way.** The existing `captureTimelinePayload` specifically listens for URLs containing `"SearchTimeline"`. User search results may come from a different endpoint. You'll need to discover it — either by watching network traffic in the browser or by trying the DOM fallback first.

3. **Profile pages may block headless browsers.** X sometimes shows a "sign up" or "log in" interstitial for headless requests. The existing `isLoggedIn` check should catch this, but be aware.

4. **`rod` library complexity.** The CDP event subscription pattern (`page.Context(ctx).EachEvent()`) is the trickiest part of the existing code. Re-read `captureTimelinePayload` carefully before writing new capture logic.

5. **Rate limiting.** The existing `waitForTurn` enforces a 15-second min interval and a rolling budget of 8 searches per 10 minutes. The new tools should respect the same rate limiter (or have their own). For read-only profile views, the existing rate limiter is probably fine.

6. **Profile-mode vs cookie-mode.** The new tools should work in both modes, like the existing tools do. Profile mode is the default and preferred.

7. **Don't persist anything new.** Phase 1 is read-only. Don't add any follow/unfollow/message-send logic in this phase.

---

## Testing approach

1. Build with `go build -o x-browser-mcp .`
2. Start with a logged-in profile (`./x-browser-mcp -port :18110`, then `curl -X POST http://127.0.0.1:18110/api/v1/login/start`)
3. Test each new endpoint with `curl`
4. Write Go tests for parsing functions (follow `parser_test.go` pattern — unit tests with hardcoded JSON samples)
5. Write integration-style tests for service methods (follow `service_login_test.go` pattern — mock the browser)

---

## Future phases (for later, not this handoff)

- **Phase 2:** follow/unfollow, get_following, get_followers — write operations
- **Phase 3:** messages (read/send DMs), trending topics — complex UI, lower priority

---

## Key file locations (for quick reference)

| File | Purpose |
|---|---|
| `/Users/mac-mini/x-browser-mcp/main.go` | Entry point, flag parsing |
| `/Users/mac-mini/x-browser-mcp/config.go` | Config struct + defaults |
| `/Users/mac-mini/x-browser-mcp/browser_session.go` | Browser launching, CDP connection, cookie save/load |
| `/Users/mac-mini/x-browser-mcp/service.go` | Core logic: login, search, timeline, DOM scraping |
| `/Users/mac-mini/x-browser-mcp/parser.go` | JSON parsing + DOM→post conversion + summaries |
| `/Users/mac-mini/x-browser-mcp/types.go` | All request/response types |
| `/Users/mac-mini/x-browser-mcp/mcp_server.go` | MCP tool registration |
| `/Users/mac-mini/x-browser-mcp/http_server.go` | HTTP handlers + routing |
| `/Users/mac-mini/x-browser-mcp/parser_test.go` | Unit tests for parsers |
| `/Users/mac-mini/x-browser-mcp/service_login_test.go` | Tests for login flow |
| `/Users/mac-mini/x-browser-mcp/browser_session_test.go` | Tests for browser session mgmt |

---

## Quick start for the next agent

```bash
cd /Users/mac-mini/x-browser-mcp

# Build
go build -o x-browser-mcp .

# Start server (requires Chrome/Chromium installed)
./x-browser-mcp -port :18110

# In another terminal: start login
curl -X POST http://127.0.0.1:18110/api/v1/login/start

# Check status
curl http://127.0.0.1:18110/api/v1/login/status

# Existing search (for reference)
curl -X POST http://127.0.0.1:18110/api/v1/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"mcp","mode":"latest","limit":5}'
```

To see what network requests the browser makes during a search:
1. Start the server with `-remote-debug-url http://127.0.0.1:9222` (attach to an existing Chrome with DevTools open)
2. Open Chrome DevTools → Network tab
3. Run a search
4. Observe the X internal API URLs and JSON responses

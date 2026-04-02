# Browser Protocol Reference

This document applies to agents that need browser interaction (Reddit fallback, Google Trends, JS-heavy sites). Only read this if you need to use a browser — most research uses WebSearch/WebFetch and does not need this.

## CRITICAL: Browser Tab/Session Lifecycle

Orphaned Chrome windows/tabs are the #1 operational problem with this skill. Every browser tab or session that opens MUST be closed — no exceptions, even if the agent errors out or times out.

## Chrome (claude-in-chrome) — Preferred

**Opening and navigating:**
```
mcp__claude-in-chrome__tabs_create_mcp → returns tabId
mcp__claude-in-chrome__navigate(tabId, url)
mcp__claude-in-chrome__read_page(tabId)  # get page content
```

**Clicking elements:**
```
mcp__claude-in-chrome__read_page(tabId)  # find coordinates of element
mcp__claude-in-chrome__computer(tabId, action: "click", x, y)
```

**Scrolling:**
```
mcp__claude-in-chrome__javascript_tool(tabId, "window.scrollBy(0, 800)")
mcp__claude-in-chrome__read_page(tabId)
```

**Closing when done:**
```
mcp__claude-in-chrome__javascript_tool(tabId, "window.close()")
```

## Playwright-cli — Fallback

Only use when chrome tools are unavailable.

**Rules:**
- **Never use `--headed`.** Always run headless.
- Each agent MUST use a session name prefixed with the run ID (`-s=${RUN_ID}-reddit`, `-s=${RUN_ID}-trends`, etc.)
- **Close the session as soon as you're done reading.**
- **Each agent closes its own session before returning results.** The last thing an agent does is `playwright-cli -s=SESSION_NAME close`.
- Always `sleep 4` after navigation before taking a snapshot.

## Obstacle Handling (both tools)

After every navigation, handle obstacles:

1. **Cookie / consent banners:** Click "Accept", "Accept all", "I agree", "Got it".
2. **CAPTCHA / "Verify you are human":** Attempt to solve. If unsolvable after 2 attempts, close and skip the URL.
3. **Login walls / paywalls:** Do NOT log in. Note the wall and move on.
4. **Age gates / region selectors:** Click through with any reasonable option.
5. **Newsletter popups / modal overlays:** Click "X", "Close", or "No thanks".
6. **Redirect loops / blank pages:** Wait and retry once. If still empty, close and skip.

## Browser Cleanup (mandatory — parent orchestrator)

After ALL agents for this run have completed, the parent orchestrator MUST run cleanup.

**If using playwright-cli:**
```bash
# Force-close all sessions for this run
for session_name in reddit trends discovery reddit-deep fallback; do
  playwright-cli -s="${RUN_ID}-${session_name}" close 2>/dev/null || true
done

for session in $(ls .playwright-cli/${RUN_ID}-* 2>/dev/null); do
  playwright-cli -s="$(basename "$session")" close 2>/dev/null || true
done

mkdir -p "$RESEARCH_DIR/browser-data" 2>/dev/null
mv .playwright-cli/${RUN_ID}-* "$RESEARCH_DIR/browser-data/" 2>/dev/null || true
rmdir .playwright-cli/ 2>/dev/null || true
```

**If using Chrome:** Tabs should already be closed by each agent. No additional cleanup needed since Chrome manages its own process.

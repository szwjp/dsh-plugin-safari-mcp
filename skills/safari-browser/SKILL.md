---
name: safari-browser
description: Use Safari from DSH on macOS — either the user's real Safari tabs via AppleScript tools (safari_*) when login state matters, or the isolated WebDriver session via native mcp__safari__* tools for structured page work. Covers handle refetching, page reading priority, and known limitations.
---

# Safari browser control (macOS)

There are two independent browser channels. Pick the right one **before** acting:

| | `safari_*` (AppleScript) | `mcp__safari__*` (WebDriver) |
|---|---|---|
| Session | The user's **real Safari** — existing tabs, cookies, logins | **Isolated automation session**; does NOT see the user's tabs and does NOT share their cookies |
| Reading | `safari_page_text` (flattened text) | `get_page_content` / `navigate_to_url` (textTree + element UIDs) |
| Actions | `safari_javascript`, `safari_open`, `macos_applescript` | `page_interactions`, `evaluate_javascript`, `create_tab`, `switch_tab`, `screenshot`, `set_viewport_size`, network/console |
| Use when | task needs the user's accounts/logins, or "read the page I'm looking at" | page structure matters, or you need an isolated/clean session |

**Never assume `mcp__safari__*` can see a tab the user has open, and never assume it is logged in.** If the task depends on the user's login state, use `safari_*` first.

## Rules

1. **Re-list tabs after every navigation.** Tab handles are not stable across cross-origin navigations: a handle like `page-B8ACE3DF…` may become `page-D75CA03A…` after a click/navigate. Always call `list_tabs` again and use the fresh handle; a stale handle makes `close_tab` fail with `Could not find browsing context`.
2. **Prefer `get_page_content` over `evaluate_javascript` for reading.** It returns structured text with element UIDs (`uid=54` …). Pass those UIDs to `page_interactions` via `node: "62"` for precise clicks; fall back to `text:` (find-in-page) only when no UID is available.
3. **Do not trust `active` in `list_tabs`.** It may report `false` for every tab even after a successful `switch_tab`. Confirm the current page with `page_info` instead.
4. **Skip `set_emulated_media`.** It reliably fails with `Timed out awaiting the page target` (Safari 27.2). Use `set_viewport_size` for layout checks instead.
5. **Network capture is currently empty.** `list_network_requests` returns `count: 0` on normal pages, so `get_network_request` is not usable yet; don't claim network evidence you don't have.
6. **Clean up after yourself.** Close tabs you created (`close_tab` with a fresh handle) and leave the user's real Safari tabs untouched. The automation session may leave an empty Safari window; that is harmless.
7. **Login helper when needed:** if the isolated session needs the user's login, do not ask for credentials — ask the user to complete the sign-in in the automation window, or switch the task to `safari_*` on their real browser.
8. First tool call can be slow (cold Safari launch + WebDriver session). Upstream `mcp-client` does not auto-reconnect; if `safaridriver` dies, tool calls keep failing until DSH is reloaded/restarted.

## Prerequisites (once per machine)

- Safari 27+; Safari → Settings → Developer → "Allow remote automation and external agents" (or `sudo safaridriver --enable`).
- The `dsh-plugin-safari-mcp` bundle must be installed in the current DSH profile; tools appear as `mcp__safari__*`.
- For the AppleScript channel, Safari → Settings → Developer → "Allow JavaScript from Apple Events" must be on (plus macOS Automation permission for DSH).

## Quick examples

```text
# isolated structured read (recommended for page work)
mcp__safari__navigate_to_url { url: "https://example.com" }        -> textTree with uid=…
mcp__safari__page_interactions  { interactions: [{ type: "click", node: "62", scrollToVisible: true }] }
mcp__safari__list_tabs                                               -> refetch handles after the click
mcp__safari__close_tab { handle: "<fresh handle>" }

# the user's real session (login state matters)
safari_tabs                    -> list real windows/tabs
safari_page_text               -> read the front page
safari_javascript { script: "return document.title" }
```

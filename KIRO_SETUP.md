# Kiro setup for this fork

How this checkout is wired for Kiro: the Obscura MCP server, git remotes, auto-update, and the agent model. The main README is untouched.

## Layout

- `origin`: your fork, `https://github.com/neweracy/obscura` (public, forks of public repos cannot be private).
- `upstream`: `https://github.com/h4ckf0r0day/obscura`. Pull updates from here, never push to it.
- `.kiro/steering/no-third-party-assets.md`: always-on rule, no downloaded third-party assets.
- `.kiro/scripts/auto-update.sh`: fetch, merge, rebuild.

## 1. The MCP server

Launch command (stdio, from `docs/Use-the-MCP-server.md`): `obscura mcp`. There is no `.mcp.json` in the repo.

Build (Windows, about 18 minutes the first time, V8 compiles from source):

```bash
CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo build --release -p obscura-cli --bins --features render
```

Output: `target/release/obscura.exe` and `obscura-worker.exe` (keep them in the same folder). The stealth feature needs `cmake`, which is not installed here, so this is the render-only build.

Kiro blocks agents from writing `mcp.json`, so paste this yourself.

Workspace `.kiro/settings/mcp.json` (this repo only):

```json
{
  "mcpServers": {
    "obscura": {
      "command": "d:/code/obscura/target/release/obscura.exe",
      "args": ["mcp"],
      "disabled": false,
      "autoApprove": [
        "browser_snapshot",
        "browser_markdown",
        "browser_links",
        "browser_extract",
        "browser_interactive_elements",
        "browser_detect_forms",
        "browser_get_attribute",
        "browser_count",
        "browser_search",
        "browser_network_requests",
        "browser_console_messages",
        "browser_tab_list",
        "browser_wait_for",
        "browser_wait_for_text",
        "browser_screenshot",
        "browser_pdf"
      ]
    }
  }
}
```

For every project, add the same `"obscura": { ... }` block inside `mcpServers` in `~/.kiro/settings/mcp.json`. Do not replace the file, it holds your other servers and secrets.

Why this approve list: the server sends no `readOnlyHint` annotations, so the list is chosen by hand. Left out on purpose so they still prompt:

- `browser_navigate`, `browser_back`, `browser_forward`, `browser_reload`: make network requests.
- `browser_evaluate`, `browser_wait_for_function`: run arbitrary JavaScript.
- `browser_click`, `browser_fill`, `browser_fill_form`, `browser_type`, `browser_press_key`, `browser_select_option`, `browser_scroll`: change page state or submit forms.
- `browser_get_cookies`, `browser_storage_state`: can put session tokens into the chat.
- `browser_set_cookie`, `browser_clear_cookies`, `browser_set_storage_state`, `browser_tab_new`, `browser_tab_switch`, `browser_tab_close`, `browser_close`.

Local addresses (`127.0.0.1`, LAN) are blocked by default. For your own local pages, add `"--allow-private-network"` to `args`.

## 2. Live test

Run against a local fixture page (an input with `value="original"`), driving the real binary over stdio JSON-RPC:

| Step | Tool | Result |
| --- | --- | --- |
| Read | `browser_navigate`, `browser_snapshot` | URL, title, `ref=e1-1 input "Name" ... value="original"` |
| Edit | `browser_fill` on `e1-1` | input value became `edited-by-kiro` |
| Undo | `browser_reload` | input value back to `original` |

Obscura has no undo command. "Undo" here means reload, which restores page state but cannot reverse a submitted form or a server-side change. This test did not go through Kiro's MCP client, because the config above has to be pasted first. After pasting, reconnect from the MCP Server view and repeat the three calls.

## 3. Bedrock model

Credentials reach Bedrock (`list-inference-profiles` and `bedrock-runtime converse` both worked, region `us-east-1`).

Use this model id:

```
us.anthropic.claude-sonnet-4-6
```

Also confirmed working: `us.anthropic.claude-sonnet-4-5-20250929-v1:0`, `us.anthropic.claude-haiku-4-5-20251001-v1:0`.

Newer Claude profiles (`sonnet-5`, `sonnet-5-5`, `opus-4-6`, `haiku-5-5`) are listed but fail with `AccessDeniedException`: the IAM user lacks `aws-marketplace:ViewSubscriptions` and `aws-marketplace:Subscribe`. Add those to the user or role, wait about 5 minutes, and retry to unlock them.

## 4. Web search

The `web-search` MCP server is already in your global `mcp.json` and enabled, so nothing was added. Obscura can also provide `web_search` itself via `OBSCURA_WEB_SEARCH=keenable` in the server `env`. That sends queries to `api.keenable.ai`, so it is off by default.

The asset rule (`.kiro/steering/no-third-party-assets.md`) says: build from built-in tools, read and cite from search, never download outside assets. The repo itself had no such rule, so this file is the rule.

## 5. Updating from upstream

```bash
bash .kiro/scripts/auto-update.sh --dry-run   # fetch only, report
bash .kiro/scripts/auto-update.sh             # detached: fetch, merge, rebuild
```

It fetches `upstream`, then merges only if you are on `main` with a clean tree. A conflicting merge is aborted. It never pushes. Log: `.kiro/logs/auto-update.log` (git-ignored by its own `.gitignore`).

A rebuild cannot replace `obscura.exe` while Kiro has the MCP server running (Windows locks the file). The log then shows a failed build; close the session and run the script again. Build takes about 18 minutes cold.

Global SessionStart hook. Save as `~/.kiro/hooks/obscura-auto-update.json` (Kiro blocks agents from writing there):

```json
{
  "version": "v1",
  "hooks": [
    {
      "name": "Obscura auto-update (global)",
      "trigger": "SessionStart",
      "description": "Fetches upstream (h4ckf0r0day/obscura), merges if safe, rebuilds the MCP binary. Detached, never pushes.",
      "action": {
        "type": "command",
        "command": "\"C:/Program Files/Git/bin/bash.exe\" d:/code/obscura/.kiro/scripts/auto-update.sh",
        "timeout": 30
      }
    }
  ]
}
```

After an update, push it to your fork by hand: `git push origin main`.

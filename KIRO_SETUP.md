# Kiro setup for WordCraft

How to drive WordCraft's MCP server from [Kiro](https://kiro.dev), keep a fork up to date, and keep agents inside the repository's asset and licence rules. This file is separate from the README on purpose. Nothing here changes the app.

## 1. The MCP server

WordCraft speaks MCP over stdio. The launch command is `wordcraft-cli mcp` (see [`docs/mcp.md`](docs/mcp.md)). There is no `.mcp.json` in the repository.

Build and install the CLI:

```sh
cargo install --path apps/wordcraft-cli --locked -j 2
```

A first build compiles about 340 crates and links with LTO, so expect it to take a while and use a lot of memory. If the machine runs out of commit memory (os error 1455 on Windows), add `CARGO_PROFILE_RELEASE_LTO=off CARGO_PROFILE_RELEASE_DEBUG=0` to the environment.

## 2. Register it in Kiro

Kiro reads `.kiro/settings/mcp.json` (one project) or `~/.kiro/settings/mcp.json` (every project). Add this inside `mcpServers`, replacing the paths:

```json
"wordcraft": {
  "command": "<absolute path to wordcraft-cli>",
  "args": [
    "mcp",
    "--author", "Kiro",
    "--automation-read-root", "<folder the agent may read>",
    "--automation-write-root", "<existing folder the agent may write>"
  ],
  "disabled": false,
  "autoApprove": [
    "list_commands",
    "list_documents",
    "get_text",
    "inspect_document",
    "render_page",
    "parity",
    "screenshot",
    "ui_inspect"
  ]
}
```

- `autoApprove` holds only tools the server marks `readOnlyHint: true`. `execute`, `batch`, `type_text`, `insert_table`, `select_text`, `new_document`, `open_document`, `save_document`, `click` and `key` still prompt.
- `--automation-read-root` and `--automation-write-root` confine file access (details in `docs/mcp.md`). Both folders must already exist. The write root can be `plan/kiro-out`, which is gitignored.
- To let the server open and save files anywhere, on every drive, leave both `--automation-*-root` flags out (`"args": ["mcp", "--author", "Kiro"]`). Each flag can be given only once, so confinement cannot cover two drives. Without the flags, `open_document` and `save_document` still prompt, because they are not in `autoApprove`.
- `--author Kiro` labels the agent's tracked changes.
- To drive the running app instead of a headless session, use `"args": ["mcp", "--connect", "127.0.0.1:7981"]` and start the app with `wordcraft --control 7981`. Confinement flags cannot be combined with `--connect`.

### Check it works

Call `list_documents` (read), then `new_document`, `type_text` with `Hello`, `get_text` (edit), then `execute` with `{"command": "edit.undo", "params": {}}` and `get_text` again (undo). The text should go from empty, to `Hello`, back to empty.

## 3. Steering

[`.kiro/steering/assets-and-research.md`](.kiro/steering/assets-and-research.md) is always included. It restates the "Assets" and "Clean room" rules from `AGENTS.md` for agents: compose from built-in commands, never download third-party assets, use web search for reading only, record every asset in `ATTRIBUTION.md`.

## 4. Web research

Add a search server next to `wordcraft` if you want agents to look up public specifications:

```json
"web-search": { "command": "uvx", "args": ["duckduckgo-mcp-server@latest"], "disabled": false, "autoApprove": ["search", "expand_link"] }
```

DuckDuckGo sometimes blocks automated queries and the server then reports no results. The `[browser]` extra of `duckduckgo-mcp-server` can help. Queries must never contain document contents, secrets or personal details.

## 5. Fork and updates

```sh
gh repo fork storytold/wordcraft --clone=false
git remote rename origin upstream
git remote add origin https://github.com/<you>/wordcraft.git
git fetch upstream
```

`upstream` is `storytold/wordcraft` and `origin` is your fork. Contributions to the project still go through a pull request.

### Auto-update

[`.kiro/scripts/auto-update.sh`](.kiro/scripts/auto-update.sh) fetches `upstream`, merges `upstream/main` into `main`, and reinstalls `wordcraft-cli`. It finds the checkout from its own location. It refuses to run when HEAD is not on `main`, the tree is dirty, or a merge or rebase is in progress. It never pushes, never forces, and aborts a conflicting merge. It logs to `.kiro/logs/auto-update.log`, which is never committed.

```sh
bash .kiro/scripts/auto-update.sh --dry-run   # fetch and report only
```

Run it at the start of every Kiro session with a global hook at `~/.kiro/hooks/wordcraft-auto-update.json`:

```json
{
  "version": "v1",
  "hooks": [
    {
      "name": "WordCraft auto-update (global)",
      "trigger": "SessionStart",
      "action": {
        "type": "command",
        "command": "\"<path to Git>/bin/bash.exe\" <path to this checkout>/.kiro/scripts/auto-update.sh",
        "timeout": 30
      }
    }
  ]
}
```

The hook returns at once and the update runs detached. A merged update is not pushed to your fork automatically.

## 6. Using a Bedrock model as the agent

Use an inference-profile id (prefix `us.` or `global.`), not a bare `anthropic.claude-*` id. Newer models cannot be invoked by their bare id. List what your account has enabled with:

```sh
aws bedrock list-inference-profiles --region us-east-1
```

Check access with a real call before relying on it:

```sh
aws bedrock-runtime converse --region us-east-1 --model-id <profile id> \
  --messages '[{"role":"user","content":[{"text":"ping"}]}]'
```

## Secrets

Kiro's user-level `mcp.json` can hold API keys and tokens. Do not commit it, and do not paste it into issues or pull requests.

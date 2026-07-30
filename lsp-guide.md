# LSP & Claude Code Plugin LSP Usage

A practical guide to the Language Server Protocol and how Claude Code plugins turn language servers into in-editor code intelligence. Read it once, then keep it open while wiring up Pyright or TypeScript Language Server.

---

## TL;DR

The Language Server Protocol (LSP) standardises how an editor (the **client**) talks to a long-running analysis process (the **language server**) over **JSON-RPC**, usually through stdio or through a socket when configured. Claude Code plays the role of LSP client; each active plugin that declares an `lspServers` block supplies one or more server definitions. When Claude Code opens a file whose extension matches a registered server, it spawns the binary, negotiates capabilities, and exposes navigation (go to definition, references, hover, symbols, implementations, call hierarchy) plus optional diagnostics through its built-in LSP tool — not a slash command.

| Key fact | Value |
|---|---|
| Transport | stdio (default) or socket |
| Config root | plugin `.lsp.json` or inline in `plugin.json` |
| Tool surface | built-in LSP tool, not `/lsp` |
| Example servers | Pyright + TypeScript Language Server |

---

## On this page

1. [Terminology](#1-terminology)
2. [Architecture & lifecycle](#2-architecture--lifecycle)
3. [How Claude Code acts as an LSP client](#3-how-claude-code-acts-as-an-lsp-client)
4. [Configuration reference](#4-configuration-reference)
5. [TypeScript / JavaScript setup](#5-typescript--javascript-setup)
6. [Python setup](#6-python-setup)
7. [Daily use](#7-daily-use)
8. [What "0 plugin LSP servers" means](#8-what-0-plugin-lsp-servers-means)
9. [Troubleshooting runbook](#9-troubleshooting-runbook)
10. [Security & performance](#10-security--performance)
11. [Sources](#11-sources)

---

## 1. Terminology

Before wiring anything up, lock down these terms. They are reused verbatim throughout the LSP specification and Claude Code's plugin docs.

| Term | Definition |
|---|---|
| **LSP** | The Language Server Protocol: a JSON-RPC 2.0 contract between an editor and a language-specific analysis process. Standardises requests, responses, notifications, and capability negotiation so any client can talk to any server. |
| **Client (editor / host)** | The process that owns the buffer and UI: VS Code, Neovim, Helix, or in our case Claude Code. Owns the document state and decides which server to ask for which feature. |
| **Language server** | A standalone long-lived process that parses code, maintains project state, and answers LSP requests. Examples: `pyright-langserver`, `typescript-language-server`, `gopls`, `rust-analyzer`. |
| **JSON-RPC** | The wire format. Every message is a JSON object. Requests carry an `id` and expect a matching response; notifications carry no `id` and are fire-and-forget. Headers use HTTP-style `Content-Length`. |
| **Capability negotiation** | After `initialize`, both sides exchange `capabilities`: what features they support. The intersection is what is actually available. A server that does not advertise `definitionProvider` will never answer go-to-definition. |
| **Workspace** | The set of folders the server reasons about. Servers see `workspace/didChangeWatchedFiles`, `workspace/symbol`, and cross-file queries. Per-file state is called the *document*. |
| **Diagnostics** | Server-pushed lint and type-check results, sent via `textDocument/publishDiagnostics`. Severity levels: error, warning, information, hint. Claude Code can auto-inject these into context after edits. |
| **Navigation** | The bread-and-butter features: go to definition, find references, hover/type info, document & workspace symbols, go to implementation, call hierarchy. Each is gated by its own capability flag. |

---

## 2. Architecture & lifecycle

An LSP session is a long-lived JSON-RPC conversation between two processes. The client spawns the server, both sides shake hands, then exchange document and request messages until shutdown.

### Topology

**Client (Claude Code)**
- Owns the open buffer
- Tracks user prompts
- Spawns server processes
- Routes LSP requests to tool calls
- Receives server notifications

↕

**Language server**
- Parses source files
- Maintains a project model
- Answers navigation requests
- Publishes diagnostics on change
- No UI; no network access needed

Frames are JSON-RPC messages with `Content-Length` headers, delimited on stdin/stdout (default) or a TCP socket when `transport` is set to `socket`.

### Lifecycle phases

1. **initialize** — Client sends rootPath, capabilities, trace. Server replies with its own capabilities.
2. **initialized** — Client notification. Server may now start expensive work (indexing).
3. **requests & notifications** — didOpen / didChange / didSave feed documents. Server publishes diagnostics.
4. **shutdown** — Client asks server to finish work. Server replies then waits.
5. **exit** — Client closes stdin. Server process terminates.

### Common message kinds

| Direction | Kind | Examples |
|---|---|---|
| Client → Server | Request | `textDocument/definition`, `textDocument/references`, `textDocument/hover`, `textDocument/documentSymbol`, `workspace/symbol`, `textDocument/implementation`, `textDocument/prepareCallHierarchy` |
| Client → Server | Notification | `textDocument/didOpen`, `textDocument/didChange`, `textDocument/didSave`, `workspace/didChangeWatchedFiles`. No reply expected. |
| Server → Client | Notification | `textDocument/publishDiagnostics`, `window/logMessage`, `window/showMessage`. The signal channel back to the UI and to Claude Code's diagnostic buffer. |
| Both | Capability | Both sides advertise what they support; the effective feature set is the intersection. No `definitionProvider` capability means no go-to-definition even if the client asks. |

> **Why this matters for Claude Code**: Claude Code coordinates the configured language servers and routes LSP features through its built-in tool. When a server advertises a feature, Claude Code can ask for it on the user's behalf — no plugin needs to re-implement parsing.

---

## 3. How Claude Code acts as an LSP client

Claude Code is the LSP client. It coordinates the configured language servers and uses their advertised capabilities to provide code intelligence (navigation, diagnostics, symbols).

### Role of the plugin

- **Active plugins** provide server definitions. A plugin that ships only skills or commands contributes nothing to LSP.
- **Extension mapping** in each definition tells Claude Code which file extensions route to which server. First registered extension wins on collision (with a warning).
- **Lifecycle**: Claude Code spawns the configured binary when it opens a matching file, negotiates capabilities, and keeps the process alive for the session.

### What's exposed

| Capability | How it works |
|---|---|
| **Auto diagnostics after edits** | After you edit a tracked file, the server publishes diagnostics. Claude Code can inject them into context automatically so the model sees fresh errors before answering. |
| **Built-in LSP tool** | Definition, references, hover, document symbols, workspace symbols, implementations, and call hierarchy flow through Claude Code's built-in LSP tool — invoked by the model, not by a slash command. |
| **Transparency** | There is no `/lsp` command. Users do not type LSP syntax. They ask Claude Code things like "find references" or "what's the type of `x`?" and the underlying tool handles it. |
| **Lifecycle / Reload** | Plugin or config changes require `/reload-plugins`. Servers restart with the new definition. |

> **Not a slash command**: LSP features are not exposed as `/lsp` or similar slash commands. They are invoked implicitly through Claude Code's built-in LSP tool when the model needs navigation or when the user asks a question that requires it.

---

## 4. Configuration reference

Two equivalent ways to declare LSP servers:

- **Plugin-root file**: `.lsp.json` at the plugin root, side-by-side with `.claude-plugin/plugin.json`.
- **Inline**: an `lspServers` key inside `.claude-plugin/plugin.json`. This is what the official marketplace examples use.

### Top-level shape

```json
{
  "<server-name>": {
    "command": "<required>",
    "args": ["..."],
    "transport": "stdio",
    "env": { "KEY": "value" },
    "initializationOptions": { "...": "..." },
    "settings": { "...": "..." },
    "extensionToLanguage": {
      "<ext>": "<language-id>"
    },
    "workspaceFolder": "<path>",
    "startupTimeout": 10000,
    "shutdownTimeout": 5000,
    "restartOnCrash": true,
    "maxRestarts": 5,
    "diagnostics": true
  }
}
```

### Field reference

| Field | Type | Default | Purpose |
|---|---|---|---|
| `command` *required* | string | — | Executable name or absolute path. Resolved through Claude Code's PATH for the spawned process; must be reachable from that PATH. |
| `args` | string[] | `[]` | Arguments passed to `command`. Typical: `["--stdio"]` for stdio-served servers. |
| `transport` | "stdio" \| "socket" | `stdio` | Wire transport. `socket` requires the server to listen on a TCP port and Claude Code to connect. |
| `env` | object<string,string> | inherits | Extra environment variables injected into the server process. Visible values; treat as disclosed. |
| `initializationOptions` | object | `{}` | Forwarded verbatim to the LSP `initialize` request under `initializationOptions`. |
| `settings` | object | `{}` | Forwarded via `workspace/didChangeConfiguration` after init. Use for server-specific runtime config (e.g. Pyright severities). |
| `extensionToLanguage` *required* | object<string,string> | — | Maps file extensions (with leading dot) to LSP language identifiers. First registered extension wins on collision. |
| `workspaceFolder` | string | project root | Override the root URI sent during `initialize`. Useful for monorepos and WSL mounts. |
| `startupTimeout` | number (ms) | 10000 | How long Claude Code waits for the server to complete `initialize` before declaring it failed. |
| `shutdownTimeout` *(v2.1.205+)* | number (ms) | 5000 | How long Claude Code waits for a clean `shutdown` reply before forcing exit. |
| `restartOnCrash` *(v2.1.205+)* | boolean | `true` | Whether Claude Code should respawn the server after a crash, up to `maxRestarts`. |
| `maxRestarts` | number | 5 | Cap on automatic restarts per session to avoid crash-loop spam. |
| `diagnostics` | boolean | `true` | If `false`, navigation features stay on but Claude Code suppresses auto-injected diagnostics in the model context. |

> **Version note**: `restartOnCrash` and `shutdownTimeout` are supported in Claude Code **v2.1.205** or newer. On older builds, including these fields can cause the entire server definition to be skipped; upgrade, omit them, or rename and re-check with `claude --debug`.

---

## 5. TypeScript / JavaScript setup

The official Anthropic-maintained marketplace plugin `typescript-lsp` wires up `typescript-language-server`. End-to-end, that takes four steps plus one reload.

1. Install the server binary globally. Both packages are needed because the language server shells out to `tsc` for project-wide type checking:
   ```bash
   npm install -g typescript-language-server typescript
   ```
2. Install the marketplace plugin in Claude Code:
   ```
   /plugin install typescript-lsp@claude-plugins-official
   ```
3. Verify the binary is on PATH from a Claude process (not your shell PATH — see the troubleshooting runbook). Ask Claude Code to run:
   ```bash
   command -v typescript-language-server
   ```
4. Reload plugin discovery:
   ```
   /reload-plugins
   ```

### Resulting configuration

The official marketplace entry uses the inline `lspServers` form. Reproduced verbatim from `marketplace.json`:

```json
"lspServers": {
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript",
      ".jsx": "javascriptreact",
      ".mts": "typescript",
      ".cts": "typescript",
      ".mjs": "javascript",
      ".cjs": "javascript"
    }
  }
}
```

*Source: `~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json`, lines 3412–3439.*

> **Smoke test**: Open any `.ts` file in a project Claude Code is operating on, then ask: "find every reference to `myFunction`" or "what's the inferred type of `x` on line 42?" If the LSP tool fires, you have working go-to-definition, references, and hover.

---

## 6. Python setup

The official Anthropic-maintained marketplace plugin `pyright-lsp` wires up Pyright. Pyright is published on both npm and PyPI; pick the install method that matches your environment.

1. Install the server. Three options, in order of recommendation:
   - **pipx (recommended for CLI tools)**: `pipx install pyright`
   - **npm**: `npm install -g pyright`
   - **pip**: `pip install pyright`
2. Install the marketplace plugin:
   ```
   /plugin install pyright-lsp@claude-plugins-official
   ```
3. Verify the binary on PATH:
   ```bash
   which pyright-langserver
   ```
4. Reload plugin discovery:
   ```
   /reload-plugins
   ```

### Resulting configuration

Reproduced verbatim from `marketplace.json`:

```json
"lspServers": {
  "pyright": {
    "command": "pyright-langserver",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".py": "python",
      ".pyi": "python"
    }
  }
}
```

*Source: `~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json`, lines 2653–2674.*

> **pipx vs pip**: `pipx install pyright` keeps the binary isolated in its own venv, which avoids surprise version drift if another Python tool wants a different Pyright. `pip install --user pyright` is the next-best fallback on systems without pipx.

---

## 7. Daily use

Once a server is registered and reloaded, day-to-day interaction is three commands plus a small number of natural-language prompts.

### Slash commands you will actually use

- `/plugin` — opens the panel: **Discover** (browse marketplace), **Installed** (toggle / disable), **Errors** (validation issues for installed plugins).
- `/reload-plugins` — refreshes the plugin registry after any change to `.lsp.json`, `plugin.json`, or installed binaries.
- Auto-injected diagnostics appear after edits via a "Found N new diagnostic issues…" notice. Press `Ctrl+O` to expand that notice in place. Toggle the injection off per-server with `"diagnostics": false` in that server's block (navigation still works).

### Interaction workflow

Prompts to type into Claude Code:

- "Find every reference to `build_payload` in this repo."
- "Inspect the inferred type of `cfg.timeout` on line 38."
- "List the workspace symbols matching `parse*`."
- "Where is `TokenError` implemented? Show call hierarchy."
- "The diagnostics on lines 12–17 are wrong; fix them without changing the public API."
- "Go to the definition of `format_iso`."

### The diagnostics trade-off

Auto-injected diagnostics consume tokens. On a long session with a noisy language server, this can be a meaningful slice of your context budget. Two clean levers:

- **Disable per-server**: set `"diagnostics": false` in that server's block. Navigation still works; only the auto-injection is off.
- **Filter at the server**: configure severities via `settings` (e.g. drop `information` and `hint` for Pyright) so less data crosses the wire in the first place.

---

## 8. What "0 plugin LSP servers" means

The message *0 plugin LSP servers* is informational, not an error. It is the count of LSP server definitions accepted from your active plugins right now. Specifically:

- **Active plugins only**. Disabled, broken, or unparseable plugins do not count, even if their `lspServers` block is well-formed.
- **Definitions, not languages**. One plugin can register multiple servers under one `lspServers` object; the count is definitions, not languages.
- **Definitions, not necessarily running processes**. A server is only spawned when a matching file is opened. Until then, the count is the catalog size.
- **Not built-in tool count**. Claude Code's LSP tool always exists; this number is unrelated to it.

### Expected non-zero checklist

| Symptom | Likely cause | First check |
|---|---|---|
| Count stays 0 after install | Plugin disabled in `/plugin` → Installed | Re-enable, then `/reload-plugins` |
| Count stays 0 after reload | `plugin.json` or `.lsp.json` failed validation | `/plugin` → Errors tab |
| Count drops after a session restart | Plugin source changed (e.g. marketplace refresh) | Reinstall; confirm marketplace `sha` |
| Count nonzero but no features | No matching file opened yet | Open a `.ts` / `.py` and retry |

> **Current dev-harness-kit status**: The local dev-harness-kit plugin currently ships no LSP definition, so *0 plugin LSP servers* is the expected baseline. Adding `lspServers` to either `.lsp.json` or inline in `plugin.json` is what changes the count.

---

## 9. Troubleshooting runbook

Work down this matrix from the cheapest signal to the deepest. Most "LSP doesn't work" reports are 1–2 rows in.

| Symptom | Likely cause | Fix |
|---|---|---|
| Server fails to start, log says "command not found" | Binary not on Claude Code's PATH | Use absolute path in `command`; verify with `claude --debug` |
| `/plugin` → Errors tab shows validation failure | `plugin.json` or `.lsp.json` invalid | Run `claude plugin validate ./my-plugin`; fix quoted fields |
| Plugin installed but never appears in `/plugin` | Plugin disabled, or marketplace refresh pending | Re-enable in Installed; then `/reload-plugins` |
| Wrong server picks up a file | `.lsp.json` placed in wrong root (sub-project instead of plugin root) | Move `.lsp.json` next to `.claude-plugin/plugin.json` |
| Two plugins claim `.py`; only one works | Extension collision; first registered wins + warning | Disable the loser or change one plugin's `extensionToLanguage` |
| Server sees wrong project root in monorepo | Default workspace folder is the repo root | Set `workspaceFolder` per package |
| Init hangs and times out | Server expects large index step during `initialized` | Raise `startupTimeout`; check `initializationOptions` |
| Server crashes and stays down | Crash loop exceeding `maxRestarts` or auto-restart off | Enable `restartOnCrash` (v2.1.205+) and inspect `--debug` logs |
| Memory climbs over a long session | Server holds full project index in memory | Close unused workspaces; pick a lighter server variant |
| Stale "0 plugin LSP servers" after install | Cached plugin catalog, marketplace `sha` mismatch | `/reload-plugins`; reinstall the plugin; restart Claude Code |

### Diagnostic incantations

- `claude --debug` — prints plugin load, plugin validation, and server startup / skip events. Use it to confirm a server definition was accepted and to see why one was skipped.
- `claude plugin validate ./my-plugin` — validates `plugin.json` + `.lsp.json` from the CLI.
- `/plugin validate ./my-plugin` — same check, in-session.
- `/reload-plugins` — restart the plugin registry; cheap and idempotent.

---

## 10. Security & performance

| Concern | Detail |
|---|---|
| **Process origin** | A plugin's `command` launches a real local process under your user. Anything that process can do, you can do. Audit before installing. |
| **Environment exposure** | Server `env` is merged into the spawned process's environment. Avoid passing secrets. Use `initializationOptions` or `settings` for server-specific config. |
| **Marketplace trust** | A marketplace is a list of plugin sources, identified by git URL + `sha`. Treat unknown marketplaces like unknown npm scopes. Pin where possible. |
| **Memory** | Large servers (e.g. Pyright on a monorepo, typescript-language-server on hundreds of files) hold whole-project state. Closing files does not unload the index. |
| **Diagnostic context** | Auto-injected diagnostics enter the model's context. A noisy server on a long session is a real token cost. `"diagnostics": false` or `settings`-level severity filters are the cleanest knobs. |
| **Disabling diagnostics** | Turning `diagnostics` off keeps navigation features intact; you only lose the auto-injection into model context. Useful for large repos or noisy rulesets. |

> **Operational rule**: Schema validity only proves the shape of a plugin manifest — it does not vouch for what the plugin does. Before enabling a plugin, review its source, the exact `command` and its arguments, and the trust placed in the marketplace `sha` and the binary that `command` resolves to. `claude plugin validate` confirms shape, not intent.

---

## 11. Sources

- [Official LSP overview (microsoft.github.io)](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/) — the authoritative introduction to the protocol. For the actual wire-level specification (every request, response, and notification), follow the spec links from that page rather than treating the overview as normative.
- [Discover Plugins — Claude Code docs](https://code.claude.com/docs/en/discover-plugins)
- [Plugins Reference — Claude Code docs](https://code.claude.com/docs/en/plugins-reference)

*The marketplace listings `pyright-lsp` and `typescript-lsp` are local official examples; they are correct for the version pinned in the marketplace `sha` but not normative for the spec.*

---

## Source attribution

- [LSP overview](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/) — Microsoft, normative.
- [Claude Code: Discover Plugins](https://code.claude.com/docs/en/discover-plugins) — Claude Code docs.
- [Claude Code: Plugins Reference](https://code.claude.com/docs/en/plugins-reference) — Claude Code docs.
- Local marketplace entries (verified, pinned by `sha`):
  - `pyright-lsp` — `~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json`, lines 2653–2674.
  - `typescript-lsp` — same file, lines 3412–3439.

*Last reviewed against the marketplace pinned `sha` for `claude-plugins-official`. Regenerate after any plugin upgrade.*

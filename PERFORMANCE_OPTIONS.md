# Zapet Performance Options

This note collects options for making Zapet lighter while staying easy to keep in sync with upstream Fresh. The preference order is:

1. Disable behavior through config or CLI flags.
2. Use build features or packaging choices that do not require app-code forks.
3. Only add a small Zapet-specific shim when the upstream app has no usable switch.

## Top Impact

### 1. Disable Plugins and `init.ts`

Best runtime option:

```sh
zapet --safe
```

Equivalent explicit flags:

```sh
zapet --no-plugins --no-init
```

Build option:

```sh
cargo build -p zapet --no-default-features --features runtime
```

Why this helps:

- Avoids loading the TypeScript plugin runtime.
- Avoids bundled embedded plugin extraction/loading.
- Avoids plugin declaration generation and plugin startup hooks.
- Avoids large bundled plugins such as dashboard, theme editor, devcontainer, audit mode, vi mode, and git UI helpers.

Tradeoff:

- Removes plugin-provided features.
- Removes user `init.ts` customization.
- Some language/LSP integrations may be plugin-backed.

Recommendation:

Use this by default for Zapet unless we explicitly decide Zapet needs Fresh's plugin ecosystem.

### 2. Disable Workspace Restore, Hot Exit, and Recovery

CLI option:

```sh
zapet --no-restore
```

Config option:

```jsonc
{
  "editor": {
    "restore_previous_session": false,
    "hot_exit": false,
    "recovery_enabled": false
  }
}
```

Why this helps:

- Avoids restoring previous tabs, splits, cursor state, and file explorer state.
- Avoids scanning and applying hot-exit recovery data.
- Avoids periodic recovery writes while editing.

Tradeoff:

- Less protection against unsaved work loss.
- Less useful as a long-lived general editor.

Recommendation:

Good default for a focused prompt editor. Keep `recovery_enabled` only if we expect users to draft long prompts inside Zapet.

### 3. Disable Update Checking and Telemetry

CLI option:

```sh
zapet --no-upgrade-check
```

Config option:

```jsonc
{
  "check_for_updates": false
}
```

Why this helps:

- Removes startup/background network work.
- Removes behavior Zapet probably does not need if it is distributed separately.

Tradeoff:

- Users do not get Fresh/Zapet update notices from inside the app.

Recommendation:

Disable by default for Zapet.

## Medium Impact

### 4. Prevent LSP Servers From Auto-Starting

Most default LSP configs already use `auto_start: false`, but user or project config can override this.

Example:

```jsonc
{
  "lsp": {
    "rust": {
      "command": "rust-analyzer",
      "enabled": true,
      "auto_start": false
    }
  },
  "universal_lsp": {}
}
```

Why this helps:

- Keeps external LSP processes from starting automatically.
- Avoids diagnostics/completion traffic unless the user manually starts a server.

Tradeoff:

- No automatic completions, definitions, diagnostics, hover, or inlay hints.

Recommendation:

Keep LSP available but manual. For prompt editing, automatic LSP is usually unnecessary.

### 5. Disable Syntax and Semantic Extras

Config option:

```jsonc
{
  "editor": {
    "syntax_highlighting": false,
    "enable_inlay_hints": false,
    "enable_semantic_tokens_full": false,
    "diagnostics_inline_text": false,
    "completion_popup_auto_show": false
  }
}
```

Why this helps:

- Reduces render-time and async editor work.
- Avoids semantic token requests.
- Avoids automatic completion popup scheduling.

Important limitation:

- Today this does not fully prevent grammar registry setup/background grammar machinery from existing. It mostly reduces usage cost, not all startup cost.

Tradeoff:

- Plain-text editing feel.
- Less IDE behavior.

Recommendation:

Good Zapet default if the app is mainly for writing prompts and Markdown-ish text.

### 6. Disable Mouse Hover and High-Volume Mouse Behavior

Config option:

```jsonc
{
  "editor": {
    "mouse_hover_enabled": false
  }
}
```

Why this helps:

- Avoids LSP hover requests from pointer movement.
- On Windows, this can also reduce mouse tracking volume.

Tradeoff:

- No hover documentation popups.

Recommendation:

Disable by default for Zapet.

### 7. Disable Animations

Config option:

```jsonc
{
  "editor": {
    "animations": false,
    "cursor_jump_animation": false
  }
}
```

Why this helps:

- Reduces extra render scheduling and frame work.
- Better behavior over SSH or slow terminals.

Tradeoff:

- UI feels less polished.

Recommendation:

Disable by default for Zapet.

### 8. Reduce Editor Chrome

Config option:

```jsonc
{
  "editor": {
    "show_menu_bar": false,
    "show_tab_bar": false,
    "show_vertical_scrollbar": false,
    "show_horizontal_scrollbar": false,
    "line_numbers": false,
    "highlight_current_line": false,
    "whitespace_show": false
  },
  "file_explorer": {
    "auto_open_on_last_buffer_close": false,
    "follow_active_buffer": false
  }
}
```

Why this helps:

- Less layout and render work.
- Makes Zapet feel less like a full IDE and more like a prompt editor.

Tradeoff:

- Less discoverability and less file-navigation UI.

Recommendation:

Good default for the Zapet profile.

### 9. Disable System Clipboard Paths That Are Not Needed

Config option:

```jsonc
{
  "clipboard": {
    "use_osc52": true,
    "use_system_clipboard": true
  }
}
```

Possible lighter variants:

```jsonc
{
  "clipboard": {
    "use_osc52": true,
    "use_system_clipboard": false
  }
}
```

or:

```jsonc
{
  "clipboard": {
    "use_osc52": false,
    "use_system_clipboard": true
  }
}
```

Why this helps:

- Can avoid problematic clipboard backends in remote/headless setups.
- Reduces failure paths around display-server clipboard APIs.

Tradeoff:

- Clipboard behavior depends heavily on terminal and OS support.
- For image paste UX, system clipboard access is likely required on local desktop runs.

Recommendation:

Keep system clipboard enabled for local Zapet image paste. Consider OSC 52-only for remote or terminal-only profiles.

## Build-Time Options

These do not require changing application code, but they change the build/package.

### Runtime Without Plugins

```sh
cargo build -p zapet --no-default-features --features runtime
```

Impact:

- Removes `plugins`, `embed-plugins`, and plugin runtime dependencies from the editor binary.
- Keeps core terminal editor functionality.

Risk:

- Any code path that assumes plugin-backed commands exist may need UX review.

### Runtime With Plugins but Without Embedded Plugins

```sh
cargo build -p zapet --no-default-features --features runtime,plugins
```

Impact:

- Keeps plugin runtime available.
- Avoids shipping/loading the large embedded plugin bundle.

Risk:

- User-installed plugins can still load.

Recommendation:

This is a good compromise if we want optional plugins but do not want Fresh's bundled plugin set in Zapet.

## Options That Likely Need a Small Zapet Shim Later

These are high-value, but the current Fresh code appears to initialize them unconditionally enough that config alone is not a complete disable.

### Full Grammar Registry Disable

Current state:

- `syntax_highlighting: false` reduces highlighting usage.
- The editor still builds/defaults grammar registry structures and may start background grammar work.

Potential Zapet shim:

- Add a Zapet startup profile that skips full grammar build and uses an empty/default-minimal grammar registry.

### Full LSP Manager Disable

Current state:

- LSP servers can be kept from auto-starting.
- The `LspManager` itself is still constructed.

Potential Zapet shim:

- Add a Zapet profile where `lsp: None` is allowed from startup.

### Integrated Terminal Disable

Current state:

- Terminal manager is constructed, even if no terminal buffer is opened.

Potential Zapet shim:

- Hide terminal commands and skip terminal manager setup in a Zapet profile.

### File Explorer Disable

Current state:

- The file explorer can be kept hidden and less active.
- Supporting structures still exist.

Potential Zapet shim:

- A prompt-editor profile that does not register file explorer providers or UI actions.

## Recommended Zapet Baseline

Start with this profile:

```jsonc
{
  "check_for_updates": false,
  "editor": {
    "restore_previous_session": false,
    "hot_exit": false,
    "recovery_enabled": false,
    "syntax_highlighting": false,
    "enable_inlay_hints": false,
    "enable_semantic_tokens_full": false,
    "diagnostics_inline_text": false,
    "completion_popup_auto_show": false,
    "mouse_hover_enabled": false,
    "animations": false,
    "cursor_jump_animation": false,
    "show_menu_bar": false,
    "show_tab_bar": false,
    "show_vertical_scrollbar": false,
    "show_horizontal_scrollbar": false,
    "line_numbers": false,
    "highlight_current_line": false,
    "whitespace_show": false
  },
  "file_explorer": {
    "auto_open_on_last_buffer_close": false,
    "follow_active_buffer": false
  }
}
```

Run with:

```sh
zapet --safe --no-restore --no-upgrade-check
```

Best build profile for smallest upstream-friendly Zapet:

```sh
cargo build -p zapet --no-default-features --features runtime
```

## Decision Checklist

- Keep plugins? If yes, use `runtime,plugins`; if no, use only `runtime`.
- Keep embedded Fresh plugins? Usually no for Zapet.
- Keep session restore? Usually no.
- Keep recovery/hot exit? Only if users draft long prompts in Zapet.
- Keep LSP automatic behavior? Usually no.
- Keep syntax highlighting? Maybe yes for Markdown/code-heavy prompts, otherwise no.
- Keep image paste system clipboard access? Yes for local desktop Zapet.

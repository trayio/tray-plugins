---
name: set-workspace
description: Set or change the Tray workspace ID for THIS project — a per-project override of the workspace you configured when installing the plugin. User-invoked only; run /tray-workflows:set-workspace [workspace-id].
disable-model-invocation: true
argument-hint: "[workspace-id]"
---

# Set Tray Workspace (per-project)

Set the Tray workspace the plugin builds in for **this project**, overriding the
workspace entered at install time. Useful when you work across several Tray
workspaces in different repos.

**How it works:** the build-workflow skill pins every workspace-scoped call to
`${user_config.workspace_id}`. That value resolves from `pluginConfigs` in your
settings, and **project settings override the global install value** — precedence is
`.claude/settings.local.json` > `.claude/settings.json` > `~/.claude/settings.json`.
So writing the workspace id into this project's settings makes the plugin build here
without changing your global default.

## Process

1. **Get the workspace ID.** If `$ARGUMENTS` contains a workspace id, use it. Otherwise ask the user — do not guess.

2. **Pick the target file** (default to the first):
   - `.claude/settings.local.json` — personal, gitignored. **Default.**
   - `.claude/settings.json` — shared with the team via git. Use only if every collaborator on this repo should default to this workspace.

   Create the file if it doesn't exist.

3. **Merge — never overwrite.** Read the existing file (if present) and merge in the block below, preserving all other keys. The plugin-id key is **`tray-workflows@tray-plugins`** (the `<plugin>@<marketplace>` form), and the value nests under `.options.workspace_id`:

   ```json
   {
     "pluginConfigs": {
       "tray-workflows@tray-plugins": {
         "options": {
           "workspace_id": "<workspace-id>"
         }
       }
     }
   }
   ```

   If the user's global `~/.claude/settings.json` already uses a different
   `pluginConfigs` key for this plugin, match that key instead.

4. **Always show the result.** Print the exact `pluginConfigs` block you wrote. If the settings write is blocked or fails for any reason, print the block and tell the user to paste it into the chosen file themselves — the command must stay useful even when settings writes are restricted.

5. **Tell the user to reload.** The workspace value is substituted into the build-workflow skill **at skill-load time**, so a change only takes effect after:

   ```
   /reload-plugins
   ```

   (or a new session). Confirm: `Workspace for this project set to <id> in <file>. Run /reload-plugins to apply.`

## Notes

- This is a **per-project override**. Your global install workspace is unchanged; other projects keep using it.
- Authentication is handled via OAuth2. To re-authenticate, uninstall and reinstall the plugin.
- If the new workspace doesn't take effect after reloading, confirm the plugin-id key in the file matches the one in your global `~/.claude/settings.json` `pluginConfigs`.

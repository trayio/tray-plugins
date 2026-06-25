---
name: set-workspace
description: Set or change the Tray workspace ID for THIS project, by recording it in AGENTS.md. Run when the user wants to point Tray workflow builds at a specific workspace, e.g. "use Tray workspace <id> here" or "set my Tray workspace".
---

# Set Tray Workspace (per-project)

Record the Tray workspace that workflow builds target **in this project**. The
`build-workflow` and `research-connector` skills pin every workspace-scoped MCP
call to the **Active Tray Workspace** they read from `AGENTS.md`. Writing the ID
there makes every future build in this repo target it automatically — no
substitution engine required (Codex doesn't have one).

**How it works:** Codex always loads `AGENTS.md` (project root, cascading down to
the working directory, plus the global `~/.codex/AGENTS.md`). A line of the form
`Active Tray Workspace: <uuid>` in that file is the single source of truth the
Tray skills look for. Project `AGENTS.md` wins over the global one, so a value
written here overrides any global default without touching other projects.

## Process

1. **Get the workspace ID.** If the user supplied a workspace id, use it.
   Otherwise ask — do not guess. To find it: open the workspace in the browser
   and copy the UUID from the URL `https://app.tray.io/workspaces/<ID>/...`.

2. **Pick the target file** (default to the first):
   - `./AGENTS.md` — the project root `AGENTS.md`, committed with the repo so every
     collaborator builds against this workspace. **Default.**
   - `~/.codex/AGENTS.md` — your personal global default, applied to every project
     that doesn't set its own. Use this for "my usual workspace".

   Create the file if it doesn't exist.

3. **Merge — never clobber.** Read the file (if present) and update just the Tray
   block, preserving everything else. Maintain a clearly delimited section:

   ```markdown
   <!-- tray-workflows:begin -->
   ## Tray

   Active Tray Workspace: 00000000-0000-0000-0000-000000000000
   <!-- tray-workflows:end -->
   ```

   If an `Active Tray Workspace:` line already exists, replace its value in place.
   If the delimited block exists, replace its contents. Otherwise append the block.

4. **Always show the result.** Print the exact block you wrote. If the file write
   is blocked or fails, print the block and tell the user to paste it into the
   chosen `AGENTS.md` themselves — the skill must stay useful even when writes are
   restricted.

5. **Confirm and note when it applies.** `AGENTS.md` is read at session start, so a
   change takes effect in the **next** Codex session (or after Codex re-reads
   project docs, depending on your version). Confirm:
   `Active Tray Workspace set to <id> in <file>. Start a new Codex session to apply.`

## Notes

- This is a **per-project** value when written to `./AGENTS.md`; your global
  `~/.codex/AGENTS.md` default is unchanged and other projects keep using it.
- Authentication is separate from workspace selection — it's handled by the Tray
  MCP server's OAuth flow (`codex mcp login tray`). To re-authenticate, re-run that.
- If a build still targets the wrong workspace, check for a conflicting
  `Active Tray Workspace:` line higher up the `AGENTS.md` cascade (a nested
  directory `AGENTS.md`, or the global one).

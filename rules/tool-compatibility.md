# Tool Compatibility — Capability → Per-Harness Tool Names

Reference for how to refer to agent tools in skills. This is a **reference** —
skills should describe tool use as **capabilities** (natural phrasing, no tool
name), so they work across every harness without a lookup step. Use this table
when you need to know the exact tool name a given harness exposes.

## Convention

- **Describe capabilities, not tool names.** Write "read the file", "edit the
  file", "ask the user a question", "spawn a subagent" — not "use `view_file`".
  The agent maps the capability to the tool its own harness provides.
- **Subagent spawning is the one exception** where naming variants is clearer,
  because `task` is near-universal: "spawn a subagent (via the `task` /
  `invoke_subagent` tool)".
- Never name a tool that exists in only one harness (e.g. `view_file`,
  `ask_question`, `StrReplace`) — it will be wrong for the others.

## Capability → Tool map

| Capability | opencode | agy (antigravity) | copilot | cursor |
|---|---|---|---|---|
| read file | `read` | `view_file` | `view` | `Read` |
| write file | `write` | `write_to_file` | `apply_patch` | `Write` |
| edit file | `edit` | `replace_file_content` | `apply_patch` | `StrReplace` |
| delete file | (via bash/apply_patch) | — | — | `Delete` |
| list directory | (read dir / glob) | `list_dir` | — | `Glob` |
| find files (glob) | `glob` | `find_by_name` | `glob` | `Glob` |
| search file contents | `grep` | `grep_search` | `rg` | `Grep` |
| run shell command | `bash` | `run_command` | `bash` | `Shell` |
| fetch a URL | `webfetch` | `read_url_content` | `web_fetch` | `WebFetch` |
| search the web | `websearch` | `search_web` | `web_search` | `WebSearch` |
| spawn a subagent | `task` | `invoke_subagent` | `task` | `Task` |
| ask the user | `question` | `ask_question` | `ask_user` | (interactive) |
| track todos | `todowrite` | (none) | `sql` todos tables | `TodoWrite` |
| load a skill | `skill` | (skills) | `skill` | (skills) |

## Notes

- `read` / `write` / `edit` / `grep` / `glob` / `bash` / `webfetch` /
  `websearch` are understood as **capabilities** by every harness even when the
  tool name differs. Prefer capability phrasing for these.
- `ask the user` and `track todos` have no consistent tool name across
  harnesses — always use capability phrasing.
- `spawn a subagent` is the one case where `task` is near-universal
  (opencode, copilot, cursor); only agy uses `invoke_subagent`.

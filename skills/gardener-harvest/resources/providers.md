# Provider Command Mappings

This skill uses the shared forge CLI map:

**`.agents/skills/_shared/runtime/providers.md`**

Detect `github` vs `gitlab` from `git remote get-url origin`, authenticate with
`gh` or `glab`, and use only the commands in that table. Do not hardcode a
single forge CLI in skill logic.

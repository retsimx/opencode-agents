# Provider Command Mappings

This skill uses the shared forge CLI map (including **Gardener-only** rows):

**`.agents/skills/_shared/runtime/providers.md`**

Detect `github` vs `gitlab` from `git remote get-url origin`, verify with a
functional repo view (`gh repo view` / `glab repo view`), resolve the default
branch, and use only commands from that table — especially expected-head
squash-merge, same-repo head checks, close, and comments.

Do not hardcode a single forge CLI or a default branch name in skill logic.

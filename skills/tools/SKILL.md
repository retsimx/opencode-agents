---
name: tools
description: >
  Manage MCP tools with natural language commands — list, enable, disable
  individual tools or tool groups in `.agents/mcp.json`, with support for
  permanent and temporary (session-only) overrides. Use when the user says
  "tools", "MCP", "enable", "disable", "restrict tools", or wants to
  control which MCP server tools are available.
---

# MCP Tool Management (`tools`)

## Scheduling

### Goal
Provide a natural-language interface for managing MCP tool availability: listing current status, enabling/disabling tools and tool groups, and applying changes permanently or for the current session only.

### Intent signature
- User says "show tools", "list MCP tools", "enable memory tools", "disable code edit"
- User wants to restrict or expand available MCP tooling
- User asks about tool configuration or MCP server settings

### When to use
Whenever the user wants to inspect or modify which MCP tools are available. Runs inline (no subagent spawning). Reads and writes `.agents/mcp.json` and optional memory-based overrides.

## Structural Flow

Execute `.agents/workflows/tools.md` which covers the full workflow: showing current status, parsing natural language commands, applying permanent or temporary configuration changes, and handling special cases (unknown tools, server conflicts, empty tool lists).

## Logical Operations

All logic is defined in the workflow file at `.agents/workflows/tools.md`. Load and follow it step-by-step without deviation.

## References

- Workflow: `.agents/workflows/tools.md`
- MCP config: `.agents/mcp.json`

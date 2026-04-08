# MemPalace - AI Memory System

A free, open-source AI memory system that stores conversation history locally without summarization or API calls.

**GitHub**: [milla-jovovich/mempalace](https://github.com/milla-jovovich/mempalace)

## Why Explore This

Store persistent memory vaults for Claude usage via MCP server. Instead of losing context at session end, memories are organized into a searchable "palace" structure.

## Key Features

- **Verbatim storage** - No lossy summaries, full conversation history in ChromaDB
- **Local-only** - Runs entirely on your machine, no cloud dependency
- **MCP integration** - Works with Claude Code, Claude Desktop, Cursor, etc.
- **96.6% R@5** on LongMemEval benchmark with zero API calls

## Palace Architecture

Hierarchical organization inspired by memory palace technique:

| Level | Purpose | Example |
|-------|---------|---------|
| **Wings** | People or projects | `nix-config`, `work` |
| **Rooms** | Topics within wing | `auth`, `networking`, `media-stack` |
| **Halls** | Memory types | facts, events, discoveries, preferences, advice |
| **Closets/Drawers** | Summaries → verbatim files | Pointers to full conversations |
| **Tunnels** | Cross-wing connections | Links related topics across domains |

## Installation

### Claude Code (Recommended)
```bash
claude plugin marketplace add milla-jovovich/mempalace
claude plugin install --scope user mempalace
```

### Via MCP Server
```bash
pip install mempalace
claude mcp add mempalace -- python -m mempalace.mcp_server
```

### Setup Flow
```bash
mempalace init ~/projects/myapp    # Initialize palace
mempalace mine ~/chats/ --mode convos  # Import existing chats
# AI handles search automatically via MCP tools
```

## Cost Comparison

| Approach | 6 months daily use (~19.5M tokens) |
|----------|-----------------------------------|
| Traditional LLM summaries | ~$507/year in API fees |
| MemPalace wake-up mode | ~$0.70/year (~170 critical tokens) |

## Potential Use Cases

- [ ] Persist Claude Code session context across projects
- [ ] Build knowledge base from past conversations
- [ ] Track decisions and rationale over time
- [ ] Cross-project memory (tunnels between wings)

## Questions to Explore

- How does it handle NixOS config conversations?
- Integration with existing [[Home Lab Setup/NixOS/index|NixOS]] workflow?
- Storage requirements for long-term use?
- Can it replace/complement CLAUDE.md session notes?

###### Tags
- #AI
- #MCP
- #Claude
- #ToolsToExplore

---
[[index|← Back to Home]]

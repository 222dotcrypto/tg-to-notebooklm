# Telegram → NotebookLM

Claude Code skill that converts Telegram Desktop chat exports (JSON) into structured text and loads them into Google NotebookLM for analysis, search, and content generation.

## Features

- **Telegram Desktop export** — reads `result.json` from any chat type (personal, group, supergroup, bot)
- **Full message processing** — text, replies, forwarded messages, service messages
- **Media annotations** — photos, files, stickers, voice messages noted as text placeholders
- **Auto-chunking** — large chats split into ~150K character parts at message boundaries
- **Artifact generation** — Briefing Doc summary, audio podcast, slide deck, mind map
- **Downloads** — all artifacts saved locally to `~/tg-to-notebooklm/downloads/`

## Prerequisites

- [NotebookLM MCP server](https://github.com/jxnl/notebooklm-mcp) — configured and authenticated
- Claude Code
- A Telegram Desktop chat export (Settings → Advanced → Export chat history → JSON format)

## Installation

### Git clone (recommended)

```bash
# Global (all projects)
git clone https://github.com/222dotcrypto/tg-to-notebooklm.git ~/.claude/skills/tg-to-notebooklm

# Project-local
git clone https://github.com/222dotcrypto/tg-to-notebooklm.git .claude/skills/tg-to-notebooklm
```

### Manual

Copy `SKILL.md` to `~/.claude/skills/tg-to-notebooklm/`.

## Usage

Trigger the skill in Claude Code by:

- Giving a path to a `ChatExport_*` folder or `result.json` file
- Typing: `"load telegram chat"`, `"convert telegram export"`, `"tg chat to notebooklm"`
- Running: `/tg-to-notebooklm /path/to/ChatExport_2026-02-18`

### What happens

1. Checks NotebookLM auth (auto-login if needed)
2. Parses `result.json` — shows chat name, message count, date range, media stats
3. Converts messages to structured plain text optimized for NotebookLM
4. Splits into chunks if needed (>200K characters)
5. Creates a NotebookLM notebook with all parts as text sources
6. Asks what to generate: summary, podcast, slides, or mind map
7. Downloads generated artifacts locally

## Message Format

```
[2026-02-18 17:26] User1:
Hello!

[2026-02-18 17:28] User2:
Hey, how's it going?
[Photo: photo_1@18-02-2026_17-28-51.jpg]

[2026-02-18 17:30] User1:
> Reply to #12344
Great, check this out
[File: document.pdf]
```

## How It Works

```
ChatExport folder → Parse result.json
                  → Convert messages to structured text
                  → Chunk if >200K chars
                  → NotebookLM MCP (create notebook + add text sources)
                  → Studio (generate artifacts)
                  → Download locally
```

## Supported Message Types

| Type | Handling |
|------|----------|
| Text messages | Full text with author and timestamp |
| Photos | `[Photo: filename]` placeholder |
| Files | `[File: filename]` placeholder |
| Stickers/Video/Voice | `[Type: filename]` placeholder |
| Service messages | `=== Service: action (by actor) ===` |
| Replies | `> Reply to #message_id` prefix |
| Forwarded | Original source preserved |

## File Structure

```
tg-to-notebooklm/
├── SKILL.md        # Skill instructions (agent SOP)
└── README.md
```

## License

MIT

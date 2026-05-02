# topic — Claude Code session launcher

A single-file Python tool that turns your Claude Code memory + session history into a topic router. Pick up where you left off in any project by typing one keyword.

## What it does

Claude Code stores every session as a JSONL transcript and supports a per-project `MEMORY.md` index of curated context files. Over time both grow and become hard to navigate. `topic` indexes both, attributes ad-hoc sessions to their MEMORY.md cluster (or surfaces them standalone), and lets you fzf-pick or keyword-jump back into any topic with the right context preloaded.

```
topic                  picker sorted by recency, fzf to filter
topic <keyword>        resume by topic name or person mentioned
topic --spawn <kw>     resume in a NEW terminal window (current shell stays)
topic new              interactive: create a new MEMORY.md cluster
topic list             tree view of all topic clusters with activity
topic dashboard        regenerate Dashboard.md (Obsidian MOC)
topic graph            regen + open Obsidian on Dashboard
topic --dry <name>     preview the launch prompt without launching
```

When you launch a topic, it spawns `claude` with a preamble that:

1. Reads the relevant memory files to load context.
2. Runs `/claude-mem:mem-search <keyword>` to pull recent session activity for that topic.
3. Gives you a tight 3-5 line status, then asks what you want to work on.

For ad-hoc sessions (no MEMORY.md cluster), it resumes the actual session via `claude --resume <session-id>`.

## Why it exists

If you use Claude Code daily across many topics, products, clients, side projects, research, people, your `~/.claude/projects/<project>/` directory becomes a graveyard of half-remembered conversations. `topic` makes it a navigable index. The fzf picker shows everything sorted by recency. Type any distinctive word (a person's name, a project codename, a feature) and it jumps back in with full context already loaded.

## Requirements

- **Python 3.9+** (uses standard library only)
- **fzf** for the picker (`brew install fzf` or your package manager)
- **Claude Code CLI** installed and on your PATH (or set `TOPIC_CLAUDE_BIN`)
- **claude-mem** plugin installed in Claude Code (for the `/claude-mem:mem-search` preamble step)
- **Obsidian** is optional, `topic dashboard` writes a Dashboard.md any markdown reader can open

## Install

**One-line install** (recommended):

```bash
mkdir -p ~/bin && curl -fsSL https://raw.githubusercontent.com/RS-LA/topic-launcher/main/topic -o ~/bin/topic && chmod +x ~/bin/topic
```

Then make sure `~/bin` is on your `PATH`:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc   # or ~/.bashrc
source ~/.zshrc
topic --help
```

**Manual install** (if you cloned the repo or prefer to inspect first):

```bash
mkdir -p ~/bin
cp topic ~/bin/
chmod +x ~/bin/topic
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
topic --help
```

The first time you run `topic`, it builds an index from your existing JSONL sessions. After that, every invocation only re-scans changed files (mtime-based), so it stays fast.

> **Note on slash commands:** `topic` is a shell command, not a Claude Code slash command (a slash command can't launch a new `claude --resume` session, which is what `topic` does). If you want a `/topic` shortcut inside Claude Code anyway, see the [Optional `/topic` slash command](#optional-topic-slash-command) section below.

## Optional: `/topic` slash command

If you use Claude Code, drop a tiny slash command into your config so `/topic` works inside any session:

```bash
mkdir -p ~/.claude/commands
curl -fsSL https://raw.githubusercontent.com/RS-LA/topic-launcher/main/extras/topic.md -o ~/.claude/commands/topic.md
```

Then in any Claude Code session:

- `/topic` — Claude runs `topic list` and shows you the cluster table, then asks which one you want to resume.
- `/topic <keyword>` — Claude runs `topic --spawn <keyword>`, which opens the resumed session in a **new terminal window**. Your current Claude session stays alive.

The slash command is just a thin shim around the shell command. It uses `topic --spawn`, which detects your terminal (macOS Terminal/iTerm via `osascript`, tmux via `tmux new-window`, WezTerm via `wezterm cli spawn`) and opens a new window or tab. If none of those work (Linux without a graphical detector, SSH session, etc.), it prints the launch command for you to copy and run manually.

You can also call `topic --spawn <keyword>` directly from any shell to get the same new-window behavior without going through the slash command.

## Configuration

By default `topic` looks at the Claude Code project for your home directory (`~/.claude/projects/<sanitized-home>/`). You can override anything via environment variables:

| Variable | Default | What it does |
|---|---|---|
| `TOPIC_PROJECT` | sanitized `$HOME` (e.g. `-Users-jane`) | Which Claude Code project to operate on. Override if you want `topic` to navigate sessions for a project that isn't your home directory. |
| `TOPIC_CLAUDE_BIN` | `which claude`, then `~/.local/bin/claude` | Path to the `claude` CLI binary. |
| `TOPIC_STOP_NAMES` | (empty) | Comma-separated names to filter out of entity extraction. Add your own name, your company name, recurring noise. Example: `TOPIC_STOP_NAMES="Jane,Acme,Loompa"`. |

Persist your settings in `~/.zshrc` or equivalent:

```bash
export TOPIC_STOP_NAMES="Jane,Acme,Loompa"
```

## How it works

**MEMORY.md as topic index.** `topic` parses `~/.claude/projects/<project>/memory/MEMORY.md` looking for `## Heading` blocks. Each H2 is a topic cluster. Markdown links to local `.md` files under that H2 become the cluster's context bundle.

**Session attribution.** For every JSONL session in the project directory, `topic` extracts the first non-junk user message as the title, scans the first 5 user messages for keywords, and matches the session to a cluster if at least 2 distinct keywords overlap. Otherwise the session lives standalone in the picker.

**Entity extraction.** Each session also gets a list of capitalized proper nouns mentioned >=3 times. This is what lets `topic alex` find a session where Alex was the main subject even if "alex" never appears in the title.

**Cache.** Index is cached at `~/.cache/topic/sessions.json`. Auto-refreshed on every run; only changed (mtime newer than cache) sessions get re-scanned. `topic rescan` forces a full rebuild.

**Dashboard.md.** A clean markdown index of every cluster + its files, regenerated on every run. Uses `[[wikilink]]` syntax so Obsidian's graph view can render the topic graph. Files in your memory dir but not referenced under any H2 surface in an "Unfiled" section so nothing gets lost as the vault grows.

## What lives where

```
~/.claude/projects/<project>/                  Claude Code project root
~/.claude/projects/<project>/*.jsonl           individual session transcripts
~/.claude/projects/<project>/memory/MEMORY.md  curated topic index (you write this)
~/.claude/projects/<project>/memory/*.md       per-topic memory files
~/.claude/projects/<project>/memory/Dashboard.md  auto-generated, do not hand-edit
~/.cache/topic/sessions.json                   index cache (auto-managed)
~/bin/topic                                    the script
```

## Adoption tips

1. **Start by writing one H2 in MEMORY.md** with a markdown link to a `project_<topic>.md` file. Run `topic new` to scaffold the structure for you the first time.
2. **Let topic earn it.** Don't try to bulk-organize everything up front. Use `topic` for a week as your normal launcher. Sessions you keep coming back to will reveal themselves as topics worth promoting to a MEMORY.md cluster.
3. **Set `TOPIC_STOP_NAMES`** with your name and company name. Otherwise every session that mentions you gets you as a top "entity" and the entity-search becomes useless.
4. **Use `topic graph`** weekly to scan Obsidian's graph view. Orphan files and tightly-connected clusters jump out visually.

## Privacy

`topic` reads only your local Claude Code session files and your MEMORY.md. It does not send data anywhere. The only network call is when you launch a `claude` session afterward, which is your normal Claude Code traffic.

## License

MIT. Use it, modify it, ship it. No warranty.

## Author

Built by **RSLA**. [RajSinghLA.com](https://rajsinghla.com).

Made this because my Claude Code project folder got too big to navigate and I wanted one keyword to drop me back into the right context. If you're hitting the same wall, it should help.

PRs welcome.

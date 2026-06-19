# roundtable

Chat with a panel of persistent AI advisor "seats" from your terminal.

Each seat is a charter (a short markdown system prompt). A per-room transcript on
disk is the shared memory, so the seats remember the meeting across separate
commands. Every turn, the seats you wake answer **in order**, each one seeing the
earlier replies from the same round (a real discussion, not parallel monologues),
and **any seat may stay silent** instead of padding the conversation.

No API key. Each seat answers through headless `claude -p`, reusing your existing
Claude Code authentication.

```
$ roundtable say board "Should we raise prices 10%?"
Chief Financial Officer: Only if churn elasticity tests under 5% loss; run a cohort price test on one segment first.
Chief Marketing Officer: Lead with added value, not the increase; grandfather existing customers for 60 days to blunt churn.

(silent: Chief Product Officer, VP of Engineering, Head of Design)
```

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed and
  authenticated (the `claude` CLI must be on your `PATH`).
- Python 3.8+ (standard library only, no pip installs).

## Install

```
git clone https://github.com/bkazez/claude-agent-roundtable.git
cd claude-agent-roundtable
ln -s "$PWD/roundtable" ~/bin/roundtable   # or anywhere on your PATH
roundtable init                            # scaffolds config + example seats
```

`init` writes `~/.config/roundtable/config.json` and copies the example seat
charters into `~/.config/roundtable/seats/`.

## Use

```
roundtable say board "your question"                 # wake every seat in the room
roundtable say board --seats finance,growth "..."    # wake specific seats
roundtable continue board                            # seats keep talking; you stay silent
roundtable continue board --rounds 3                 # up to 3 self-driven rounds
roundtable show board                                # print the transcript
roundtable reset board                               # start the meeting over
roundtable rooms                                     # list configured rooms
roundtable seats board                               # list a room's seats
```

`continue` runs rounds with no message from you, so the seats carry the discussion
themselves, each reacting to what was last said. It stops early once a full round
goes silent (nothing left to add). Use it when you want to listen rather than steer.

Drive it from a Claude Code session, a shell script, or by hand. State lives on
disk, so continuity holds no matter what runs the command.

## Configure

`~/.config/roundtable/config.json`:

```json
{
  "model": "sonnet",
  "user_name": "Sam",
  "user_role": "CEO",
  "doc_char_cap": 8000,
  "seats_dir": "~/.config/roundtable/seats",
  "rooms": {
    "board": {
      "seats": ["product", "engineering", "design", "growth", "finance"]
    },
    "launch": {
      "docs": ["~/work/acme/STRATEGY.md", "~/work/acme/ROADMAP.md"],
      "seats": ["product", "growth", "design"]
    },
    "nonprofit": {
      "cwd": "~/work/foundation",
      "tools": ["Read", "Grep", "Glob", "Bash(gh issue list:*)", "Bash(git log:*)"],
      "seats": ["executive-director", "fundraising", "programs", "finance"]
    }
  }
}
```

- **`user_name` / `user_role`** — who *you* are in the room. You can be any role
  (CEO, engineer, PM, solo founder); the seats address you by name and know your
  role. The transcript records your turns under `user_name`.
- **`model`** — any Claude Code model alias or id (`sonnet`, `opus`, `haiku`, ...).
  Override per run with `ROUNDTABLE_MODEL`.
- **`doc_char_cap`** — per-doc truncation limit for injected `docs` (default 8000).
- **`rooms`** — each room has a seat roster plus optional `docs`, `cwd`, and
  `tools` (see below).

## Live context: docs vs. tools

A room gives its seats current context in one of two ways:

- **`docs`** — a list of files injected (read-only, truncated to `doc_char_cap`)
  into every seat's prompt. Simple and cheap; good when the context is one or two
  small files.
- **`cwd` + `tools`** — run the seats in a directory and grant them read-only
  Claude Code tools, so they read the live files themselves, exactly like Claude
  Code working in that folder. Best when context is large, spread across many
  files, or changes between turns (you edit a file in your own session; the panel
  sees the new version next turn). Seats are told their working directory and its
  file listing, so they know where to look.

```json
"vmii": {
  "cwd": "~/work/vmii",
  "tools": ["Read", "Grep", "Glob", "Bash(gh issue list:*)"],
  "seats": ["executive-director", "fundraising", "programs", "finance"]
}
```

Rules of thumb:

- **Keep tools read-only.** Grant `Read`, `Grep`, `Glob`, and narrowly-scoped
  read-only `Bash(...)` patterns. Never grant `Edit`, `Write`, or mutating
  commands. The panel *proposes* changes in its replies; you apply them in your
  own session. That is what makes it safe to run a whole panel against your real
  files.
- **A room with no `tools` is pure text** (one fast model call per seat). Tools
  make a seat mildly agentic (a few read calls per turn), so it costs more; turn
  them on only for the rooms that need live context.
- **Spawned seats inherit your Claude Code setup.** Because each seat is a
  headless `claude` run, it picks up the `CLAUDE.md`, settings, and hooks that
  apply in `cwd`. Useful (project context loads automatically), but a blocking
  `UserPromptSubmit` hook in your config will also block seat calls.

## Add or edit seats

A seat is one markdown file in your `seats_dir`. The filename (minus `.md`) is the
seat slug; the first `#` heading is its display label. Write the charter as a short
prose paragraph in the second person ("You have..."): establish the seat's
seniority and instincts, what it always surfaces, what it refuses, and who it
defers to. Skip headings and bullet scaffolding; every sentence should earn its
place. Keep it tight, a few sentences is plenty.

```markdown
# General Counsel

You have advised companies through fundraising, disputes, and bad headlines, so
you spot the real exposure fast and the theatrical risk just as fast. You name the
cheapest way to de-risk a decision rather than blocking it, you flag what genuinely
needs licensed outside counsel, and you never dress an opinion up as a guarantee.
You defer to the business on appetite for risk, but whether we are about to step on
a rake is your call.
```

Then add `"legal"` to a room's `seats` list. The bundled examples are starting
points; rewrite them freely. They cover a startup panel (product, engineering,
design, growth, finance) and a nonprofit panel (executive-director, fundraising,
programs, plus finance).

## How it works (and what it costs)

- Each seat call sends its charter + the room's docs + the full transcript to
  `claude -p`, then keeps or drops the reply based on the `SILENT` sentinel.
- There is no long-running process. The transcript file *is* the persistence.
- Cost scales with transcript length times the number of woken seats per turn.
  Cheap for normal meetings; for a very long-running room, trim with `reset` or
  shorten the transcript. (Auto-summarization of old turns is a future addition.)

## License

MIT

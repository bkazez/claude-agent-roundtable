# roundtable

Chat with a panel of persistent AI advisor "seats" from your terminal.

Each seat is a charter (a short markdown system prompt). A per-room transcript on
disk is the shared memory, so the seats remember the meeting across separate
commands. Every turn, the seats you wake answer **in parallel**, react to each
other, and **any seat may stay silent** instead of padding the conversation.

No API key. Each seat answers through headless `claude -p`, reusing your existing
Claude Code authentication.

```
$ roundtable say board "Should we raise prices 10%?"
Finance Lead: Only if churn elasticity tests under 5% loss; run a cohort price test on one segment first.
Growth Lead: Lead with added value, not the increase; grandfather existing customers for 60 days to blunt churn.

(silent: Head of Product, Engineering Lead, Design Lead)
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
roundtable show board                                # print the transcript
roundtable reset board                               # start the meeting over
roundtable rooms                                     # list configured rooms
roundtable seats board                               # list a room's seats
```

Drive it from a Claude Code session, a shell script, or by hand. State lives on
disk, so continuity holds no matter what runs the command.

## Configure

`~/.config/roundtable/config.json`:

```json
{
  "model": "sonnet",
  "user_name": "Sam",
  "user_role": "CEO",
  "seats_dir": "~/.config/roundtable/seats",
  "rooms": {
    "board": {
      "docs": [],
      "seats": ["product", "engineering", "design", "growth", "finance"]
    },
    "launch": {
      "docs": ["~/work/acme/STRATEGY.md", "~/work/acme/ROADMAP.md"],
      "seats": ["product", "growth", "design"]
    }
  }
}
```

- **`user_name` / `user_role`** — who *you* are in the room. You can be any role
  (CEO, engineer, PM, solo founder); the seats address you by name and know your
  role. The transcript records your turns under `user_name`.
- **`model`** — any Claude Code model alias or id (`sonnet`, `opus`, `haiku`, ...).
  Override per run with `ROUNDTABLE_MODEL`.
- **`rooms`** — each room has a seat roster and an optional `docs` list. Listed
  files are injected (read-only, truncated) into every seat's system prompt, so
  the panel reasons about your real context.

## Add or edit seats

A seat is one markdown file in your `seats_dir`. The filename (minus `.md`) is the
seat slug; the first `#` heading is its display label.

```markdown
# Legal Counsel

**Optimizes for:** keeping the company out of trouble with the lightest process.
**Voice:** precise, risk-flagging, plain-English. Speaks in first person.
**Always surfaces:** the actual exposure, the cheapest way to de-risk it, what needs a real lawyer.
**Non-negotiables:** never give advice dressed as a guarantee; flag anything that needs licensed counsel.
```

Then add `"legal"` to a room's `seats` list. The bundled examples (product,
engineering, design, growth, finance) are starting points; rewrite them freely.

## How it works (and what it costs)

- Each seat call sends its charter + the room's docs + the full transcript to
  `claude -p`, then keeps or drops the reply based on the `SILENT` sentinel.
- There is no long-running process. The transcript file *is* the persistence.
- Cost scales with transcript length times the number of woken seats per turn.
  Cheap for normal meetings; for a very long-running room, trim with `reset` or
  shorten the transcript. (Auto-summarization of old turns is a future addition.)

## License

MIT

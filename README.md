<h1 align="center">
  t r i c k
  <br>
  <sub>background memory extraction from conversation transcripts</sub>
</h1>

Agents have two kinds of memory: what they choose to remember, and what they forget to. The first kind is handled by explicit writes — the agent decides a fact matters and stores it. The second kind is everything else: the decision made in passing, the preference stated once, the correction that never got logged. Trick handles the second kind.

A background reflector that mines Claude Code transcripts for memories and writes them to [crib](https://github.com/bioneural/crib). Fires on compaction — the moment context is about to be destroyed — and extracts what's worth keeping before it's gone.

---

## How it works

Claude Code conversations generate a JSONL transcript on disk. When the context window fills up (~98%), compaction fires. The `PreCompact` hook triggers trick, which:

1. Snapshots the transcript (compaction may modify it)
2. Spawns `trick --file <snapshot>` in the background
3. Exits immediately (never blocks compaction)

The background process:

1. Parses the JSONL transcript into conversation turns
2. Checks a cursor file — skips turns it already processed
3. Feeds the unprocessed conversation to Ollama
4. Extracts structured memories (decisions, corrections, errors, notes)
5. Writes each memory to crib via `bin/crib write`
6. Updates the cursor

The next retrieval — whether from the post-compaction `SessionStart` or the next `UserPromptSubmit` — picks up the new memories. The system is eventually consistent.

```
Context fills → PreCompact → trick -d → trick --file (background)
                                              ↓
                                           Ollama extracts memories
                                              ↓
                                           crib write
                                              ↓
                                           Next retrieve picks them up
```

---

## Usage

```sh
# Foreground — reflect on a transcript file
bin/trick --file path/to/transcript.jsonl

# Foreground — pipe via stdin
cat transcript.jsonl | bin/trick

# Detach — snapshot transcript, spawn in background, exit immediately
echo '{"transcript_path":"...","session_id":"..."}' | bin/trick -d
```

Trick is not typically called directly. It runs via `-d`, triggered by Claude Code hooks.

### Hook wiring

Add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$TRICK_HOME/bin/trick -d"
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$TRICK_HOME/bin/trick -d"
          }
        ]
      }
    ]
  }
}
```

`PreCompact` is the primary trigger — it fires when context is about to be compacted, and the full transcript is still available. `SessionEnd` is the fallback — it catches any un-reflected turns when the session closes.

### Cursor tracking

Trick tracks its progress in `.claude/trick-cursor.json`:

```json
{
  "session_id": "eb5b0174-...",
  "last_reflected_uuid": "b4575262-...",
  "last_reflected_timestamp": "2026-02-11T15:00:55Z"
}
```

Each run processes only turns after the cursor. A new session ID resets the cursor. This means long-running sessions with multiple compactions are reflected incrementally — each cycle processes only what's new.

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CRIB_HOME` | sibling directory | Path to the crib repo |
| `TRICK_MODEL` | `gemma3:1b` | Ollama model for memory extraction |
| `TRICK_CLAUDE_DIR` | `.claude` in cwd | Path to .claude directory |
| `TRICK_SESSION_ID` | from transcript | Override session ID |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama API base URL |

---

## What it extracts

Trick asks Ollama to identify memories in four categories:

| Type | What it captures |
|------|-----------------|
| `decision` | A choice that was made and why |
| `correction` | Something previously believed that turned out wrong |
| `error` | A failure that occurred and what was learned |
| `note` | Important context, facts, or preferences |

Trivial operational details — file edits, command outputs, routine code changes — are filtered out. The extraction focuses on decisions, rationale, preferences, domain knowledge, and lessons learned.

Each extracted memory is written to crib with its type prefix:

```sh
echo "type=decision Switched from Redis to Memcached for caching" | bin/crib write
```

---

## Timing

```
Session starts
│
├─ Agent works, conversation grows...
│
├─ Agent writes explicit memories (hot path)
│  └─ echo "type=decision ..." | bin/crib write
│
├─ Context hits ~98%
│  ├─ PreCompact fires
│  │  └─ trick -d snapshots + spawns trick in background
│  │
│  ├─ Compaction happens
│  │
│  ├─ SessionStart (compact) fires
│  │  └─ bin/crib retrieve → re-injects memories
│  │
│  └─ Meanwhile, trick processes the snapshot
│     └─ Extracts memories → writes to crib (5–30s)
│
├─ Next prompt's retrieve picks up the new memories
│
├─ Context hits ~98% again
│  └─ (cycle repeats — cursor skips already-processed turns)
│
├─ User closes terminal
│  └─ SessionEnd fires → trick -d on remaining turns
```

The hot path (agent explicitly calling `bin/crib write`) and the cold path (trick extracting from transcripts) complement each other. Explicit writes capture what the agent knows is important. Trick catches what slipped through.

---

## Smoke tests

```
$ bin/smoke-test

Prerequisites
  ✓ crib found
  ✓ Ruby scripts pass syntax check

JSONL parsing
  ✓ JSONL parser finds 4 conversation turns
  ✓ Parser handles both string and array content
  ✓ Empty input exits cleanly
  ✓ Invalid JSONL exits cleanly
  ✓ File flag reads transcript
  ✓ Missing file exits cleanly

Cursor tracking
  ✓ Cursor file created after reflection
  ✓ Cursor points to last turn (aaa-004)
  ✓ Cursor skips already-processed turns
  ✓ Cursor processes only new turns
  ✓ Cursor updated to latest turn (aaa-006)
  ✓ Cursor resets on new session ID

Memory extraction
  ✓ Extraction completes without error
  ✓ Memories written to crib

Detach mode (-d)
  ✓ Detach exits immediately
  ✓ Detach creates transcript snapshot
  ✓ Detach handles empty input
  ✓ Detach handles missing transcript

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
20 passed
```

Tests run against a temporary directory that is created and destroyed on each run. No artifacts are left behind.

---

## Prerequisites

- Ruby 3.x
- [crib](https://github.com/bioneural/crib) — memory storage and retrieval
- [Ollama](https://ollama.com) with the extraction model pulled (default: `gemma3:1b`)

---

## License

MIT — Fort Asset LLC

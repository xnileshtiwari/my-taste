---
name: "my-taste"
description: "Applies Nilesh's learned preferences to work in progress, and records new ones. Load at the start of any task that produces something he will judge — writing prose (chat replies, docs, commit messages, emails, naming), producing visuals (charts, diagrams, UI, images, HTML artifacts), or multi-step work where how much gets built and in what order matters. Also load when he asks what you know about his preferences, refers to his usual style, or runs /my-taste."
compatibility: claude-code opencode
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
---

# My Taste

## On load — do this first

1. Read `META.md`.
2. Read only the domain files its index points to.
3. Do **not** read `candidates.md`. Ever, outside a `/my-taste` run.

Base directory: `/home/nilesh/.claude/skills/my-taste/`

## What this is

Nilesh's preferences, learned from what he actually does rather than from what he took the time to
write down. A preference is observed, counted, and only becomes a rule once it has held up three
separate times.

This file holds the rules of the system. It holds no preferences. Preferences live in the data
files and change constantly; this file should not.

## The files

| Path | What it is | When to read |
|---|---|---|
| `META.md` | Meta preferences + the domain index | On load, always |
| `writing/prose.md` | Preferences about words | When the index points at it |
| `visuals/appearance.md` | Preferences about rendered surfaces | When the index points at it |
| `behavior/actions.md` | Preferences about what you do | When the index points at it |
| `candidates.md` | 1–2 vote entries, and demoted entries | **Only during a `/my-taste` run** |

## Applying preferences

Only graduated entries — the ones in the three domain files — apply. Candidates never do.

- Tag any output shaped by a graduated preference (see Tagging below).
- If an entry is marked `stale`, **ask before applying it.** It has not been confirmed in over 90
  days and he may have moved on.
- On conflict between two entries: the more specific wins over the more general, and a project
  scope wins over `global`.
- A preference is a default, not a command. What he says in this conversation always wins.

## Tagging

When a response was shaped by one or more graduated preferences, end it with a single line:

    [taste: w-3, b-1]

One line, at the end, nothing inline. This is not decoration — it is the only thing that lets a
correction be attributed to the preference that caused it, which is what allows a bad entry to
lose votes. Without the tag, entries can only ever gain.

Do not tag output that no graduated preference touched.

## Writing

Files are written **only** when he runs `/my-taste`. Normal work never edits any of them. Reading
is unlimited; writing has exactly one trigger.

## The five events

These are the only things that count as evidence. Verbatim:

    CORRECTION - model produced X, user replaced it with Y, unprompted. Signal: X->Y.
    SELECTION  - model offered {A, B, C}, user took one unmodified. Signal: chosen vs unchosen.
    REJECTION  - model proposed X, user refused, gave no replacement. Signal: not-X.
    CONSTRAINT - user stated a rule before any output existed. Signal: the rule itself.
    APPROVAL   - model accepted output unmodified after at least one prior correction in the
                 same thread. Signal: the settled form.

REJECTION produces a prohibition — "don't do X" — and never a preference for whatever you did
instead. The replacement only becomes a preference when a later CORRECTION supplies it.

## Routing — which file an entry belongs in

This is the rule the whole system rests on. If a preference can plausibly land in two files, it
lands in two files, each copy collects votes separately, and neither ever reaches three. One
preference, one file, always.

### The ladder — ordered, first match wins

Never evaluate a later step before an earlier one.

1. **Could this be violated before any output exists?** → `behavior/`
   It constrains an action, an order, a tool choice, a permission, or when to ask. "Don't install
   a package without asking" is violated at install time, before a word exists.
2. **What artifact was the event observed in?**
   - prose a human reads — chat reply, commit message, doc, email, code, comment, naming → `writing/`
   - a rendered surface — chart, image, UI, diagram, terminal appearance, HTML artifact, slide → `visuals/`
3. **Tie** → fixed precedence: `behavior` > `writing` > `visuals`.

Step 2 keys on the artifact, which you can observe, not on the nature of the preference, which is
a judgment call. That is what makes it give the same answer twice. Markdown structure inside prose
is `writing` — always — because the artifact is prose.

    "no emoji in commit messages"        step 1 no, artifact = prose      -> writing
    "charts on dark backgrounds"         step 1 no, artifact = chart      -> visuals
    "run tests before saying done"       step 1 yes                       -> behavior
    "bare lists over bold-label bullets" step 1 no, artifact = prose      -> writing
    "ask before any new dependency"      step 1 yes                       -> behavior
    "shorter variable names"             step 1 no, artifact = prose/code -> writing

### Matching never trusts the ladder

Before creating any new entry, search **all three domain files and `candidates.md`** by meaning —
not just the file the ladder picked.

- Match found anywhere → vote there. The existing entry's location wins. Do not move it, do not
  re-route it, even if the ladder disagrees today.
- No match anywhere → new candidate, placed by the ladder.

The ladder only ever places genuinely new entries. A routing mistake then costs one misfiled
entry instead of a permanently split vote.

Ids carry their domain — `w-`, `v-`, `b-` — so a duplicate across files shows up in a grep.

## Entry format

One line per entry, pipe-delimited, same columns in every file:

    id | rule | because | scope | votes | last_confirmed | stale | src

- `because` — filled for `behavior/` entries only. Empty elsewhere.
- `scope` — `global` or a project name.
- `last_confirmed` — ISO date of the most recent vote.
- `stale` — empty, or the word `stale`.
- `src` — a few words quoted from the message that last voted this entry. Used to avoid counting
  the same action twice across runs.

    b-4 | ask before adding any dependency | an unapproved package is his to maintain forever | global | 4 | 2026-08-12 |  | "no, don't install that"
    w-2 | bare lists over bold-labelled bullets |  | global | 3 | 2026-05-01 | stale | "stop bolding every bullet"

## The /my-taste run

In order. Each step is one pass, no revisiting.

1. Read `candidates.md` and all three domain files. This is the only moment `candidates.md` is
   ever read.
2. Scan the transcript for the five events.
3. **Generalization test.** An event survives only if it can be restated as a rule that would
   apply to a future task he has not done yet, without naming this task's specifics, *and* is
   falsifiable — you could look at a future output and say plainly whether it obeyed. If restating
   it requires naming this file, this project, or this request, it is an instruction, not a
   preference. Drop it.
4. Match each surviving event by meaning against everything (see Matching above).
5. **Votes.**
   - Match → +1, refresh `last_confirmed`, overwrite `src`.
   - No match → new candidate at 1 vote, placed by the ladder.
   - Skip a vote whose source action matches that entry's existing `src` — the same action never
     votes the same entry twice.
   - Apply last: a correction to output tagged `[taste: id]` is **−1** on that entry, and
     overrides any upvote it earned in the same run.
6. **Transitions.**
   - Candidate reaches 3 → move it into its domain file.
   - Domain entry drops to 1 → move it back to `candidates.md` with a one-line note of what
     demoted it.
   - An entry at 0 or below stays in `candidates.md` with its note, and never applies.
7. **Contradiction** between a new extraction and a graduated entry → stop and ask him which wins.
   Do not guess, do not record both.
8. Recompute `stale`: `last_confirmed` older than 90 days. This is the only place `stale` is
   written, which is why it can be trusted mid-session.
9. Update `META.md` if a meta preference is newly earned — it needs 3 graduated specifics drawn
   from 2 or more domains. `META.md` caps at 30 lines: at the cap, merge the two most similar meta
   rules into one more general rule carrying both their source ids. Merge, never append, never
   drop one to make room.
10. Print the report: graduated · demoted · stale · new candidates · conflicts. Add any
    `scope-control` or `write-like-me` line that a newly graduated entry now supersedes, so he can
    retire it himself.

## Never

- Never delete anything from any of these files. Demotion moves a line and annotates it.
- Never apply a candidate to output.
- Never read `candidates.md` outside a `/my-taste` run.
- Never edit `scope-control` or `write-like-me`. Those stay authoritative until he retires them
  line by line; a graduated entry only supersedes one once it exists.
- Never write to any of these files outside a `/my-taste` run.

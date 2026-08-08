# Exercise 3 — Memory, State & Execution (Domain 3)

**Goal:** understand where an agent's knowledge actually lives, how a session is resumed,
and which storage mechanism is correct for each kind of state.

**You will create:** nothing — this exercise is analysis and judgement.

**Prerequisite:** [Exercise 2](exercise-02-tool-use-and-environment.md) complete.

**Time:** ~30 minutes

---

## Why an exercise with no files

Exercises 1 and 2 were about configuration. This one is about a mental model, and the exam
tests it through log reading and scenario matching rather than through files you write.

The core idea is uncomfortable but important: **an agent has no memory.** What looks like
memory is a transcript being replayed. Every turn, the agent receives a bundle of text —
instructions, conversation history, file contents it has read — and produces a response.
Nothing carries over except what is put back into that bundle.

Once you internalize this, a lot of agent behaviour stops being mysterious. An agent
"forgets" a constraint because the constraint scrolled out of the bundle. An agent
contradicts an earlier decision because that decision was never written anywhere durable.
The fix is never to ask more nicely; it is to change what goes into the bundle.

---

## The four kinds of state

| Kind | Lives in | Survives |
|------|----------|----------|
| **Session state** | `~/.copilot/session-state/<id>/events.jsonl` | Resuming the same session |
| **Durable artifacts** | PRs, issues, workflow artifacts, logs | Anything — this is the real record |
| **Repository memory** | Instruction files in the repo | Every session, forever, for everyone |
| **Copilot Memory** | GitHub, server-side | Across sessions and features, until unused for 28 days |

### Session state

Copilot writes an append-only event log per session. Resuming with `--resume` or
`--continue` replays that log to rebuild context. This is genuinely useful for continuing
interrupted work, but it is local, tied to one session ID, and invisible to your teammates
and to CI.

### Durable artifacts

This is the category that matters for engineering. A pull request, a review comment, an
uploaded workflow artifact, and a job log are all durable, addressable, and reviewable by
humans and other jobs. When one agent must hand information to another — especially across
workflow jobs — the handoff goes through an artifact, never through anything resembling
shared memory.

### Repository memory

Your `.github/copilot-instructions.md` from Exercise 1 is memory in the most reliable
sense: it is read fresh at the start of every session, by every agent, forever. When you
find yourself repeating a correction, you are looking at something that belongs here.

### Copilot Memory

[Copilot Memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) is the
exception to "an agent has no memory" — and it is worth being precise about what it changes,
because it changes less than the name suggests. It is currently in public preview.

Copilot stores two things server-side:

| Type | Contains | Visible to |
|------|----------|------------|
| **Repository-level facts** | Coding conventions, architectural decisions, build commands, project rules | Anyone with access to Copilot Memory in that repository |
| **User-level preferences** | How *you* like to work with Copilot | Only you, across every repository |

Four properties decide when you can rely on it:

- **It is enabled per user, not per repository.** On individual plans it is on by default. On
  Business and Enterprise plans an administrator must enable the policy before users can opt
  in. So a fact one teammate's session created may simply not exist for another teammate.
- **Facts are stored with citations and revalidated before use.** Copilot checks the citation
  against the current branch and discards the fact if the code no longer supports it. This is
  a direct countermeasure to the context drift described below.
- **Anything unused for 28 days is deleted.** A convention that matters but rarely comes up
  will silently expire.
- **Only users with write access create repository-level facts**, which stops a fork or a
  drive-by contributor from teaching your repository something false.

It is shared across Copilot cloud agent, Copilot code review, and Copilot CLI — so a fact the
cloud agent learns can later inform a code review. Two limits: code review uses repository
facts only and ignores user preferences, and Copilot CLI applies repository facts plus the
preferences of whoever started the run.

**Why this does not replace `copilot-instructions.md`.** Copilot Memory is inferred,
expiring, per-user-enabled, and not in your repository — so it is not reviewable in a pull
request, not versioned with the code, and not guaranteed to be present. Instruction files are
explicit, permanent, and apply to everyone unconditionally. The rule to carry into the exam:
**if a convention is important enough that a build should fail without it, write it in an
instruction file. Let Copilot Memory handle the accumulated detail you would never bother to
write down.**

---

## Task 1 — Read a session log

Here is a Copilot CLI log. Work out what it tells you before reading on.

```
2026-07-27T08:31:03Z session.id=run-42 cwd=/workspace/todo-app
2026-07-27T08:31:04Z ide=Visual Studio Code connected
2026-07-27T08:31:05Z mcp loaded ~/.copilot/mcp-config.json servers=[github,playwright]
2026-07-27T08:31:15Z tool=search args="TodoController"
2026-07-27T08:31:20Z tool=read args="src/api/Controllers/TodoController.cs"
2026-07-27T08:31:25Z tool=edit args="src/api/Controllers/TodoController.cs"
2026-07-27T08:31:30Z tool=execute args="dotnet test"
```

**Questions**

1. Is this a new session or a resumed one?
2. Which IDE is connected?
3. Which MCP servers are loaded?
4. What autonomy level does this agent have?
5. What sequence of tools did it use?

<details>
<summary>Answers</summary>

1. **New session.** There is no `resume=true` and no line loading `events.jsonl`. A resumed
   session looks like this:
   ```
   session.id=run-42 resume=true
   loaded ~/.copilot/session-state/run-42/events.jsonl
   ```
   Absence of evidence is the answer here — the exam expects you to notice what is *not*
   in the log.
2. **Visual Studio Code.**
3. **`github` and `playwright`.**
4. **Medium.** It used `read`, `search`, `edit`, and `execute`. The presence of `edit` and
   `execute` is what makes it medium rather than low.
5. **search → read → edit → execute.**

</details>

### The pattern in that tool sequence

`search → read → edit → execute` is what competent agent behaviour looks like: locate the
relevant code, load it into context, change it, then verify the change. The verification
step is the one that matters. An agent that edits and stops has produced an untested guess.
This is precisely why Exercise 1 put exact test commands into the repository instructions —
it makes that final step possible.

---

## Task 2 — Match each scenario to the right storage

| # | Scenario |
|---|----------|
| 1 | Continue the same agent session later |
| 2 | Share a plan between workflow jobs |
| 3 | Preserve a review decision for audit |
| 4 | Store a long-term repository convention |
| 5 | Track what an agent did and why |

**Options:** A) Session ID / session-state · B) Workflow artifact or job output ·
C) PR comment / review · D) Instructions or repository memory · E) Session logs / PR timeline

<details>
<summary>Answers</summary>

1. **A — Session ID / session-state.** Use `--resume` or `--continue`.
2. **B — Workflow artifact or job output.** `$GITHUB_OUTPUT` for small values,
   `upload-artifact` for files. Separate jobs run on separate machines with no shared
   filesystem, so this is the only mechanism that works.
3. **C — PR comment / review.** Auditable, attributable, visible to every reviewer, and
   permanently attached to the change it describes.
4. **D — Instructions or repository memory.** Persists across all sessions and all agents.
5. **E — Session logs / PR timeline.** The record of what actually happened, as opposed to
   what was intended.

</details>

### The distinction between 3 and 5

Both look like "keep a record," but they answer different questions. A PR review captures a
**decision** — a human judged this acceptable. Session logs and the PR timeline capture
**events** — this is what occurred, in order, with timestamps. Audits need both: the log
proves what happened, the review proves someone accepted it.

---

## Task 3 — Inspect your own Copilot Memory

This one is hands-on, and it takes two minutes.

1. Open **[github.com/settings/copilot/memory](https://github.com/settings/copilot/memory)**.
2. Confirm whether Copilot Memory is enabled for your account. If you are on a Business or
   Enterprise licence and see nothing, the organization policy has not been turned on — note
   that and move on, it does not block the lab.
3. Read whatever preferences are listed. Each one carries a citation showing what it was
   inferred from.
4. Delete any entry that is wrong or stale, and notice that you can do this per entry.

**Questions**

1. A teammate's Copilot session learned that this repository deploys through `ci.yml` rather
   than by hand. Will your session know that?
2. You tell Copilot "always use British spelling in comments." Where does that land, and who
   else is affected?
3. Your team has a rule that every new endpoint needs a test. Copilot Memory or
   `copilot-instructions.md`?
4. Copilot cites a repository fact that was true last month but the code has since changed.
   What happens?

<details>
<summary>Answers</summary>

1. **Probably, but do not depend on it.** Repository-level facts are shared with anyone who
   has access to Copilot Memory in that repository — but only if Memory is enabled for you,
   and only if the fact has been used within the last 28 days.
2. **A user-level preference, affecting only you**, across every repository you work in. Your
   teammates see no change. If you want it to apply to the team, it belongs in an instruction
   file.
3. **`copilot-instructions.md`.** It is a rule you want enforced for everyone, reviewable in
   a PR, and versioned with the code. A memory that expires after 28 days of disuse is the
   wrong home for a standard.
4. **It is discarded.** Facts are stored with citations, and Copilot revalidates the citation
   against the current branch before using the fact. Only validated facts are applied.

</details>

### How this changes the context-drift story

The revalidation step is the interesting part. Ordinary context drift happens because the
agent is reasoning about a file as it was when it read it. Copilot Memory is explicitly
designed not to do that — it re-checks its citations before trusting a stored fact. That
makes it more reliable than session context, and still less reliable than a file in the
repository that everyone can see.

---

## Context drift

Context drift is the failure mode where an agent's assumptions quietly stop matching
reality. It is the most common cause of long agent sessions going wrong.

It happens when:

- The agent read a file early, the file changed, and it is still reasoning about the old
  contents.
- Earlier turns have scrolled out of the usable context window.
- The agent inferred something plausible and never re-checked it.

The symptom is characteristic: the agent behaves *consistently* but *incorrectly*, and
patiently re-explaining does not help, because the flawed assumption is still sitting in
context alongside your correction.

**What works:** start a fresh session, and put the important facts somewhere durable so the
new session picks them up automatically. If the same correction is needed twice, it belongs
in `copilot-instructions.md` — that converts a repeated conversational fix into repository
memory.

---

## Exam notes

### Paths to memorize

```
~/.copilot/session-state/<id>/events.jsonl    Session event log
```

### Resuming

- `--resume` — resume a specific session
- `--continue` — continue the most recent session

### How to tell new from resumed in a log

A resumed session shows `resume=true` **and** a line loading `events.jsonl`. A new session
shows neither. Look for the absence.

### Durable state is not chat memory

The exam repeatedly probes this. Chat history is not durable, not shareable, and not
auditable. Durable state means PRs, issues, artifacts, and logs. If an answer option
proposes passing information between jobs "through the conversation," it is wrong.

### Copilot Memory

| Fact | Value |
|------|-------|
| Enabled per | **User**, not repository |
| Default on | Individual plans. Business/Enterprise need an admin policy first |
| Two types | Repository-level facts, user-level preferences |
| Who can create repository facts | Users with **write access** only |
| Retention | Deleted after **28 days** unused |
| Validated how | Citations re-checked against the current branch |
| Used by | Copilot cloud agent, Copilot code review, Copilot CLI |
| Code review limitation | Repository facts only — ignores user preferences |

If a question asks where a rule belongs and the rule must apply to everyone and be
reviewable, the answer is an instruction file, not Copilot Memory.

### Reading autonomy from a log

Infer the level from the tools that appear:

| Tools observed | Level |
|----------------|-------|
| `search`, `read` | Low |
| plus `edit`, `execute` | Medium |
| plus `agent`, MCP servers | High |

---

## Common pitfalls

**Assuming the agent remembers earlier corrections.** It remembers only what is currently
in context. Repeated corrections signal a missing instruction file entry.

**Passing data between workflow jobs without an artifact.** Jobs are isolated machines.
Without `upload-artifact` / `download-artifact` or `$GITHUB_OUTPUT`, the data does not
travel.

**Treating a plan as a record.** A plan states intent. Only logs and diffs record outcome.

**Relying on Copilot Memory for a team standard.** It is enabled per user, expires after 28
days of disuse, and lives outside your repository. A standard that must hold for everyone
belongs in an instruction file.

**Fighting context drift by explaining harder.** Restart the session and fix the durable
source instead.

---

## What you learned

Where agent state actually lives, how to read a session log for session type and autonomy
level, which storage mechanism fits each need, and where Copilot Memory sits between session
context and repository instructions. Exercise 5 applies this directly: the multi-agent
pipeline passes every handoff through workflow artifacts, for exactly the reasons established
here.

**Next:** [Exercise 4 — Evaluation, Error Analysis & Tuning](exercise-04-evaluation-and-tuning.md)

---
name: antagonist
description: Run an independent adversarial review ("the antagonist") of a plan, diff, design, guard script, or claim through a selectable external model backend (codex, agy/Gemini, claude, anthropic API, moonshot, local) via the `antagonist` CLI. Use when kltm asks for an antagonist, adversarial review, second opinion, or independent check; before acting on a multi-step infra plan, a completeness or safety judgment, or a causal account that no single query can refute; before any step that could sever connectivity or is irreversible; and for security-sensitive code even after merge.
argument-hint: "[backend] <what to review>"
---

# Antagonist review

An independent model reviews material you produced. On every use so far
(eight reviews, August to September 2026) it found real defects adjacent
to where the author was looking, and it also produced some wrong findings
each time. It is a pre-flight gate, not a post-hoc audit: run it before
the mutation, never after the point where a failure would remove the
ability to run it (network loss, session death, credential revocation,
data deletion).

Tool: `antagonist` (source `~/local/src/git/antagonist`, on PATH via
`~/local/bin`). Always start with `antagonist backends`: it shows which
backends are configured (binary or key present) and what model and
effort each defaults to. It does not call any model, so a backend can
still fail at run time; the run's `status` says why.

## 1. Check in first

One line to kltm before spawning: which backend, what is being reviewed,
why now. Parallel backends are fine; name each. A green light for a batch
covers obvious continuations inside the same task.

## 2. Write the prompt file

Put it in the scratchpad. Make it self-contained; the reviewer shares none
of your context. Sections, in order:

1. **Context.** What the material is, who owns it, what it is for, and the
   failure model (what going wrong looks like, who is affected).
2. **Material.** Either pasted inline or named as evidence files (below).
   For code, the exact version under review. For plans, the whole plan.
3. **Claims to test.** Numbered. Every conclusion you are about to act on,
   stated as a falsifiable sentence. Numbering is what makes the answer
   auditable.
4. **Out of scope.** What is already decided, so it is not relitigated.
5. **Constraints.** Runtime, versions, environments, anything that rules
   out otherwise-good fixes (for example: must run on Node 8 and 22).
6. **Questions.** Pointed and specific: the named input classes an
   encoder or validator must handle (encoding, nesting, path separators,
   empty and oversize values), ordering and rollback for a cutover, the
   wrong-instrument check for a PR ("is this repo's automation even using
   this lockfile?"). For encoders and validators include both the
   hostile-input and the legitimate-input matrices.

Framing: write it as a correctness and hardening review from the owner's
side ("cases the guard should handle", "gaps in our own controls"). Do not
use the words bypass, attack, exploit, or "break this"; codex's classifier
has refused that wording twice, and the same substance phrased defensively
runs. The runner warns when a codex prompt contains them.

Evidence: pass every file the review depends on with `--evidence` so it is
pasted byte-for-byte; relative paths resolve against `--cwd`. For a change
under review, capture the exact baseline: `git status --short` first, then
`git diff HEAD > <scratch>/change.diff` (staged plus unstaged), and pass
each untracked file as its own evidence item. Path-only prompts produce
confident reports from model memory.

Secrets: the runner refuses to start when the assembled prompt contains a
known credential value or a credential-shaped string, and it reports only
line numbers. Do not pre-check with `grep -n`, which prints the match. If a
match is a known false positive, pass `--allow-secrets`; a known value
always refuses.

## 3. Run it detached

```
antagonist run -b codex --effort max -f /path/to/prompt.md \
    -e path/relative/to/cwd -e /abs/path --cwd /path/to/repo --label slug --detach
```

Replace every value before showing or running it; pass `--effort`
explicitly (`max` for codex, claude, anthropic, moonshot; `high` for agy)
so a config default cannot lower it silently. The run prints
`antagonist: preflight:` notes on stderr when the backend CLI is behind
its latest release, is off PATH, or the pinned model has a newer sibling
in its family; relay any note to kltm in the check-in or the result
report rather than acting on it (a model change is his call, since it
breaks comparability between runs). `antagonist preflight` shows the full
picture; the check is cached for a day. For agy, `antagonist
backends` must list a `read_file(<repo>/)` grant for the workspace; if it
says `NONE`, run `antagonist agy-allow <repo>` first, or the first file
read ends the run. It prints the run directory. Then poll with a bounded
call:

```
antagonist status <run-dir> --wait 3600    # returns when done, or after 3600 s (exit 3)
antagonist result <run-dir>                # prints result.md, only once status is done
```

Run the `status --wait` call as a **background** Bash task
(`run_in_background`), with a wait at least as long as the run's timeout:
the call returns at once, the harness notifies you the moment the run
finishes, and you stay responsive to kltm in between. This is the default
for codex, which takes 20 to 40 minutes at `max`. A foreground wait is
only for a backend expected within a few minutes (agy, the API
backends), and then **at most 180 seconds per call**: a longer foreground
call shows nothing in the terminal and blocks you from seeing a message.
Exit 3 means still running: do other work and call `status` again. Never
read `result.md` from the run directory yourself; `result` refuses until
the run is `done`, which is the point.

Detached runs live in their own session, so they survive their caller's
process group being killed (the harness reclaiming a background task, the
MCP idle timeout). They do not survive an OOM kill of the worker itself;
`status` reports that as `died`. Do not use the codex MCP tool for long
reviews.

Backend choice:

| backend | when | default effort |
|---|---|---|
| `codex` | default; the track record is here | `max` (never lower for real reviews) |
| `agy` | codex credits short, or a second vendor on the same prompt | `high` (its ceiling) |
| `claude` | cheap fresh-context check; same vendor as this session, weakest independence | `max` |
| `anthropic` | API path when the claude login is unavailable | `max` |
| `moonshot` | third vendor; evidence must be pasted | `max` |
| `local` | only when a local endpoint is configured | n/a |

For anything with a rollback plan or a production surface, run codex and
agy on the same prompt and compare. Timeout default is 3600 s; `max` codex
reviews of a few hundred lines have taken 20 to 40 minutes.

## 4. Verify, then relay

Run `antagonist result <run-dir>` and read its output. For each finding,
before relaying it:

- Test it against the evidence. Run the query it implies. Try the hostile
  and legitimate cases. Some findings are wrong, and a wrong finding
  relayed as fact costs more than the review saved.
- Give each a disposition: confirmed (with what you ran), refuted (with
  the evidence), or deferred (why).
- Report the reviewer's own evidence status (what it verified vs inferred).

Relay in that shape. Do not paste the raw output as the answer.

## 5. Record

The run directory is the durable transcript (prompt as sent, literal
command, raw output, result, metadata). Record its path and the
dispositions where the repo keeps such things: an `events/` entry, the
PR, or the memory note for the work. Do not re-narrate the findings in
several places.

## Failure modes

- Codex refuses ("flagged for possible cybersecurity risk"): rephrase once
  as a hardening review from the owner's side, rerun. If it refuses again,
  tell kltm. Do not silently switch backend; the backends differ.
- agy run fails with "empty response (a tool was denied)": it touched a
  path outside a granted directory or used a search tool. Reads and
  listings need a grant per repo: `antagonist agy-allow <repo>`. Pasted
  evidence needs no grant.
- `status` says `died`: the worker process is gone without writing a
  result. If `status` also prints an `orphan:` line, the backend CLI is
  still running; run the `kill` command it gives before rerunning, or you
  pay for two reviews. Then rerun.
- `status` says `failed` with "incomplete" or "went silent": the transport
  ended early or stalled (a kimi-k3 stream at `max` did this once).
  Nothing partial is presented as a result; rerun, at `high` if it was a
  long-reasoning model.
- Result ends with a truncation marker: the model hit its output cap.
  Rerun with a narrower prompt, or raise `max_tokens` in the config.
- A run refuses to start over credential-shaped text: look at the named
  prompt lines, remove the material, rerun.

## Do not

- Run a review against a live production system; review the plan and the
  commands instead.
- Pass any of the backends' permission-skipping flags.
- Put credentials in evidence, prompts, or labels.

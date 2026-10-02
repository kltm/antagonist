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

antagonist run -b agy --effort high -f /path/to/prompt.md \
    -e path/relative/to/cwd -e /abs/path --cwd /path/to/repo --label slug \
    --split 'C1,C2,C3/C4,C5/*' --max-words 1500 --detach   # `*` = the remaining claims; it must match at least one
```

Replace every value before showing or running it; pass `--effort`
explicitly (`max` for codex, claude, anthropic, moonshot; `high` for agy,
its ceiling) so a config default cannot lower it silently. The run prints
`antagonist: preflight:` notes on stderr when the backend CLI is behind
its latest release, is off PATH, or the pinned model has a newer sibling
in its family; relay any note to kltm in the check-in or the result
report rather than acting on it (a model change is his call, since it
breaks comparability between runs). `antagonist preflight` shows the full
picture; the check is cached for a day.

**agy runs are split.** One agy run thinks for a few minutes whatever the
prompt holds, so a prompt with many claims gets a shallow pass on each.
`--split` runs one review per group of claims in parallel and merges
them: `auto` takes three claims per part, or name the groups
(`C1,C2/C3,C4,C7/*`, where `*` is every claim not named). Put claims that
bear on each other in the same group, because no part sees two groups.
The prompt needs its numbered claims under a heading that contains the
word "claims". Add `--also-model gemini-3.8-flash` for anything with a
production surface: the second model has found defects the first missed.
Give each part a word budget (`--max-words 1500`). If parts fail,
`antagonist retry <run-dir>` reruns only those.

agy reviews with a generated agent that has no write tool. In `exec`
mode its one file tool is a sandboxed shell (no network, the file system
readable, writes only under /tmp); in `read` mode it has agy's file
search and read tools under `--cwd` (add other directories with
`--read-dir`). Both can search the web. `antagonist backends`
shows which mode `auto` resolves to and warns when the sandbox cannot
start on this host; `antagonist agy-check` proves it with one sandboxed
command, and is worth running after agy has updated itself. In `exec`
mode the prompt may ask for checks by execution; say what to test, not
merely that testing is allowed. The run prints the run directory. Then
poll with a bounded call:

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

Backend choice. First decide what the review needs. **Reading** (a diff,
a plan, a guard script, numbered claims, with all the evidence attached):
any backend. **Exploration** (finding what the evidence leaves out) or
**execution** (verifying a finding by running something): codex or agy.
Both read the repository on their own and run checks inside a sandbox;
agy only in `exec` mode. claude can search but not execute; the API
backends can only read what they are given.

| backend | when | default effort |
|---|---|---|
| `agy` | the default; always with `--split`, and `--also-model gemini-3.8-flash` for anything with a production surface; a split review takes 10 to 30 minutes | `high` (its ceiling) |
| `codex` | off the list while its workspace spend cap stands (2026-10-02); when kltm puts it back: the longest single reasoning pass and the longest track record | `max` (never lower for real reviews) |
| `claude` | cheap fresh-context check; same vendor as this session, weakest independence; can search, not execute | `max` |
| `anthropic` | API path when the claude login is unavailable | `max` |
| `moonshot` | third vendor; evidence must be pasted | `max` |
| `local` | only when a local endpoint is configured | n/a |

For anything with a rollback plan or a production surface, run two
reviewers on the same prompt and compare: agy with `--also-model`, plus a
second backend when one is available. Timeout default is 3600 s; `max`
codex reviews of a few hundred lines took 20 to 40 minutes; a split agy
review of 8 to 12 parts takes 10 to 30 (6 parts run at once, and a
per-minute request quota on the Gemini Enterprise service has failed
parts under that load, which `retry` reruns).

A reviewer with a shell reads the review directory, `/tmp`, `~/.cache`,
the system, and agy's read-only mounts of `~/.config` and `~/.docker`
less the paths in `[agy] mask` (the runner hides the `gh`, `gcloud`, `gws`
and docker credential stores by default; `~/.ssh`, `~/.aws`, `~/.netrc`,
`~/local/share` are absent by agy's design). A token file inside a
reviewed checkout is readable, and what it reads goes to the model's
vendor: keep credentials out of checkouts (operations:
`docs/workstation-credentials.md`).

In `exec` mode the runner refuses a `--cwd` or `--read-dir` under `/tmp`,
`/var/tmp` or `~/.cache`: the sandbox lets commands write there, so the
reviewer could change what it reviews. Attach evidence from such places
with `-e` (it is pasted), or use `--tools read`.

## 4. Verify, then relay

Run `antagonist result <run-dir>` and read its output. For each finding,
before relaying it:

- Test it against the evidence. Run the query it implies. Try the hostile
  and legitimate cases. Some findings are wrong, and a wrong finding
  relayed as fact costs more than the review saved.
- Give each a disposition: confirmed (with what you ran), refuted (with
  the evidence), or deferred (why).
- Treat the reviewer's "verified" as a claim, not as evidence. agy has
  marked a point verified after reading the wrong installed copy of a
  library; check what it ran (`status` lists tool calls per part, the
  commands are in each part's `stdout.log`).
- A split result repeats findings across parts and models. Merge
  duplicates before relaying, and say which model or part raised each.
- Read the notes column of a split result (and `note:` lines in
  `status`): a part whose answer may be cut short, one the runner ended
  because agy held the turn open, one with text set aside in
  `result.trailing.md`. A cut-short part is worth a `retry` only if the
  missing piece matters. The cut-short check sees only an unclosed code
  block; prose that stops early is not flagged, so read the ends.
- A split review repeats, and can contradict itself across the two
  models. Nothing merges verdicts: compare the parts on the same claims
  yourself, and treat a defect that spans two groups as the reader's
  job, since no part saw both.
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
- agy run fails with "empty response (a tool was denied ...)": the runner
  already resumed the conversation twice telling the reviewer to carry on
  without the refused tool, and it kept asking. The error names the
  action. `command`: agy's settings lack `"toolPermission":
  "proceed-in-sandbox"`; that file is kltm's to change, or rerun with
  `--tools read`. `read_file` (read mode): the reviewer went outside
  `--cwd`; attach the directory with `--read-dir` and rerun.
- agy run fails with "sandbox failed to start": its shell is broken on
  this host (after an agy update, a missing AppArmor profile). Run
  `antagonist agy-check`, tell kltm what it prints, and rerun with
  `--tools read` if the review can do without execution. Do not accept a
  review whose checks never ran.
- A split run is `failed` with "N of M parts failed": the finished parts
  are in `result.partial.md`; `antagonist retry <run-dir>` reruns the rest.
  One interrupted stream per part is retried automatically.
- agy run fails with "agy exited 3" and `stderr.log` says the response
  "exceeded the output token limit": the answer outgrew the cap. Rerun with
  a word budget in the prompt and fewer, narrower questions; the partial
  draft in `stdout.log` is not a result.
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
- Pass any of the backends' permission-skipping flags, or run agy outside
  the runner: its default agent writes files in its workspace unasked.
- Put credentials in evidence, prompts, or labels.

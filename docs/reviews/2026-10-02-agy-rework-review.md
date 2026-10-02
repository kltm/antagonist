# agy rework reviews, 2026-10-01/02

The change: agy became the default backend (codex lost its workspace spend
cap on 2026-10-01). Commits `c240fed` through `00fd640`: a generated
reviewer agent with no write tool, `read`/`exec` tool modes, `--split` with
`--also-model` and `retry`, corrective turns, answers read from the event
stream (held-open turns, trailing text, cut-short answers, silent attempts),
redaction of every run file, an outer mount namespace over credential
stores. Every review ran on agy itself, with the sandboxed shell; codex was
not available. Runs live under `~/.local/state/antagonist/runs/`.

| review | parts | wall time | outcome | run dir |
|---|---|---|---|---|
| self-review of the rework diff (13 claims, 5 groups × Pro + Flash) | 10 | 19 min + reruns | stdin defect (blocker), 5 majors, several minors; all verified | `20261002-122233-agy-antagonist-agy-rework-selfreview` |
| parity against the codex path (11 claims, 4 groups × Pro + Flash) | 8 | 27 min + a retry (two 429s) | parity holds on capabilities; quality parity unproven until benchmarked; 1 blocker, 6 majors fixed | `20261002-144232-agy-parity-agy-vs-codex` |
| benchmark: the 2026-09-25 codex prompt on the deployed deep tier | 8 | 8 min + a retry | 7 full + 3 partial of codex's 13 defects, 3 missed, 4 found that codex missed | `20261002-151830-agy-parity-benchmark-2026-09-25-prompt` |

## Confirmed and fixed

- `_run_cli` watch loop: polled `communicate()` stopped feeding stdin after
  its first timeout, so a child that had not drained a prompt larger than
  the pipe buffer never saw the rest (CPython registers stdin for writing
  only when `input` is truthy). It hung two parts of the self-review for 37
  minutes with no events; first misread as an agy stall. A thread writes
  stdin now; test with a 400 KB prompt and a slow reader. Found by both
  models.
- Sandbox-failure detection matched the sentinel anywhere in command
  output; reviewers reading the runner's own source were failed. Anchored
  at the start of the output.
- `agy_sandbox_writable` missed parents of a writable root (`--cwd /`,
  `--cwd ~`); `commonpath` now.
- A long reply followed by a tool call counted as a complete answer, so a
  plan before a hung command would have been recorded as the review at the
  stall; a tool step after the longest reply now means work in progress.
- Text agy adds after its answer was moved out of the result; a conclusion
  arriving late could vanish from view. It stays in the result under a
  marker and in `result.trailing.md`.
- A part recorded `done` without `result.md` passed the split as done, and
  once demoted was not written back, so `retry` skipped it. Fixed both.
- `retry` could start over a backend process still running in the same part
  directory; refused now, with the kill command.
- `looks_truncated` and `parse_claims` mishandled `~~~` fences, nested
  fences, inline spans at line start and indented sub-items; one
  CommonMark-style fence tracker serves both; claim items at column 0 only.
- Redaction: `*.tryN`, `*.turnN`, `result.trailing.md` and split top-level
  files were not covered; `meta.json` was redacted as text (JSON escaping
  could hide a value). Every run file is covered, `meta.json` as an object.
- `cmd.txt` named the stdin file relative to the wrong directory; absolute
  now. `agy-check` wrote no `result.md` (so `antagonist result` broke after
  a check) and read only the final turn's log. `invalid model` match was
  case-sensitive. The read-mode note named only `--cwd`, not `--read-dir`.
- Per-minute request quota on `businessaicode.googleapis.com` (HTTP 429)
  under 8 parallel parts; a pause before the retry. agy's model-catalog
  fetch can fail and reject a slug the neighbouring part is using; retried
  once.
- Skill and global instructions still ranked or permitted codex; the
  No-Fable skill carried live state; `profile_codex` in the No-Fable
  toggle used `-m`, which the runner does not accept.
- agy mounts `~/.config` and `~/.docker` read-only into every sandbox
  (measured: `~/.config/gh/hosts.yml`, gcloud's token dbs, `~/.config/gws`,
  `~/.docker/config.json` readable by a reviewer). An outer `bwrap`
  namespace masks them; agy's own sandbox starts inside it.

## Refuted or not adopted

- "Pick the reply before the first system message as the answer": a short
  interim remark ("a command is running, I will wait") would become the
  answer and the real review would be set aside.
- "Stop claim parsing at any subheading": claims grouped under subheadings
  would be dropped silently.
- "Fail closed on a truncated answer": the only signal is an unclosed code
  fence, which also fires on complete answers quoting diffs; it stays a note.
- "Set split concurrency to the part count": the 429 quota argues the other
  way.
- `~/.config/antagonist/config.toml` is readable in the sandbox; it holds
  key paths, not values, and the paths point into an unmounted tree.

## Benchmark detail

Scored against the 13 defect findings of the 2026-09-25 codex review, all of
which had been confirmed. Deployed deep tier (`--split auto`, Pro and Flash,
exec): full on 7, partial on 3, missed 3. The misses: two prose-precision
findings ("12 h+" restore wording; too-definite lifecycle scheduling) and
one library fact (AWS CLI v1 uploads with CRC32, not Content-MD5): both
models reported Content-MD5 as verified after reading the system botocore
1.34.46 instead of the CLI's 1.43.6. Found beyond codex: a `df` call the
host facts ruled out, server-side copies without sha256 metadata, a missing
expired-delete-marker rule, an uploader that copies a key whose upload just
failed. A Pro-only run with explicit groups the day before scored 9 full +
3 partial on the same prompt; single runs vary by about two findings, and on
this run Flash carried the script claims that Pro waved through. No part
read the live repositories or the dispositions record (checked in the
command logs).

## Lessons

- When a failure appears right after a runner change, suspect the runner
  before the backend: the "agy stall" was the watch loop.
- A runner reviewing itself trips on its own sentinel strings; anchor
  detectors to the shape of the real error.
- "Verified" from the reviewer means it read or ran something, not that it
  read the right copy; the botocore miss repeats the 10-01 finding.

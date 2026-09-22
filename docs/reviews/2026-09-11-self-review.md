# Self-review, 2026-09-11

The first use of `antagonist` was on itself: the v1 runner, skill, and
README went to three backends in parallel with the same prompt. Runs live
under `~/.local/state/antagonist/runs/` (paths below are relative to it).

| backend | model | effort | wall time | outcome | run dir |
|---|---|---|---|---|---|
| agy | gemini-3.1-pro-high | high | 3.6 min | 5 findings, 3 claims refuted | `20260911-204602-agy-self-review` |
| codex | account default (gpt-5.6) | max | 33 min | 21 findings, 8 claims refuted | `20260911-204602-codex-self-review` |
| moonshot | kimi-k3 | max | stalled | stream stopped mid-word after 7 min; killed after 16 min of silence | `20260911-204602-moonshot-self-review` |
| moonshot | kimi-k3 | high | 7 min | 17 findings | `20260911-211514-moonshot-self-review-2` |

Round one was on v1. Everything marked fixed below is in v2, which is
what the tests cover. Round two (agy and codex on v2) is recorded at the
end when it finishes.

## Dispositions, round one

Confirmed and fixed in v2:

- `status --wait N` looped forever (agy 1, codex 13). Now a total bound
  with exit 3; the skill says so.
- API backends had no wall-clock bound (agy 3, codex 5, moonshot 4's
  premise). `SIGALRM` in the worker plus a per-read idle timeout; the
  idle timeout is what caught the stalled kimi-k3 stream on rerun.
- agy `SUCCESS` with empty response was marked `done` (agy 4, codex 6,
  moonshot 5-detection). Now `failed` with the stderr note.
- agy `cmd.txt` named `prompt.md` though stdin was a JSON line (agy 5,
  codex 17). The exact stdin line is saved as `prompt.agy.jsonl` and
  named in `cmd.txt`.
- Partial transports accepted as complete (codex 6, moonshot 1):
  `message_stop`, `[DONE]`/`finish_reason`, exit 0, non-empty `-o` file,
  non-empty text are all required now.
- Credential paths (codex 1, 2, 3; moonshot 2): known values redacted
  from errors, tails and logs; CLI children get a scrubbed environment;
  redirects refused; plain http refused; the runner scans the assembled
  prompt and refuses with line numbers only; the `grep -n` advice is
  gone from the skill.
- Timeout kill skipped SIGKILL when the direct child exited (codex 4):
  the group now always gets SIGTERM then SIGKILL; `--timeout 0` rejected.
- Foreground interrupt orphaned the child (moonshot 4): any exception
  out of `communicate` kills the group.
- Fallback attribution and `model_context_window_exceeded` (codex 7).
- Local endpoints get `max_tokens`; kimi-k3 matched as a family, not a
  prefix; blank model rejected (codex 8, moonshot 10).
- agy effort: `--effort` for non-pro models, suffix normalized for pro
  slugs, effective effort recorded (codex 9).
- Evidence resolved against the invoking directory, decoded lossily,
  trailing whitespace stripped (codex 10). Now against `--cwd`, strict
  UTF-8, bytes preserved.
- Reviewer inherited `AGENTS.md` / `CLAUDE.md` (codex 11):
  `project_doc_max_bytes=0` for codex, `--safe-mode --strict-mcp-config
  --disable-slash-commands` for claude, and a preamble line that evidence
  is data.
- Worker death before `running` left `queued` forever; `last` could pick
  a non-run (codex 12, moonshot 5, 13). Pid recorded at spawn, whole
  worker inside the handler, entries filtered on `meta.json`.
- Skill: effort not pinned, `backends` not required, contradictory
  "bypass" wording, `git diff` baseline underspecified, templates shown
  as commands, "reliably" overstated, agy grant precondition buried
  (codex 14, 15, 16, 19; moonshot 14).
- `abc=def==` as a bare token was truncated (codex 18): only an
  upper-case identifier counts as an assignment.
- No tests (codex 20): `tests/test_antagonist.py`, 26 cases, local SSE
  server plus fake CLIs.
- Run directories were 0755 (codex, in its own probing): 0700.
- kimi-k3 `medium` mapped down to `low` (moonshot 7): now `high`.
- Prompt-file read errors dumped a traceback (moonshot 12).
- Key-file path resolved from live config in the worker (codex 21,
  moonshot 8): the path is stored in `opts.json`.

Refuted or not adopted:

- Reasoning discarded (agy 2): the deliverable is the review; reasoning
  is now saved to `reasoning.md` when a provider streams it, not merged
  into the result.
- `read_file` grants do not cover listings (moonshot 3): it relied on a
  stale fact in the prompt; a listing under a granted directory succeeds
  (measured), so the agy note permits listing under the workspace.
- `signal.setitimer` over `signal.alarm` (codex 5): equivalent at
  one-second resolution; `alarm` kept.
- `usage.iterations` cross-check for fallbacks (codex 7): not done; the
  fallback block is the documented signal. Round two asks whether it is
  needed.
- Config fingerprint to detect edits between spawn and start (codex 21):
  storing the resolved key path removes the dependency instead.
- Dropping the framing-word warning (moonshot, complexity): kept; it
  fired usefully on the very first real prompt.

Open after round one:

- The Anthropic `fallback` content-block shape is taken from the API
  docs, not observed (moonshot 9).
- Redaction covers known values only; a backend that prints some other
  on-disk secret would land it in its log (codex 1's broadest reading).
- Kill scope is the process group; a descendant that calls `setsid` is
  out of reach (codex 4).

## Round two on v2

| backend | model | effort | wall time | outcome | run dir |
|---|---|---|---|---|---|
| agy | gemini-3.1-pro-high | high | 3.5 min | 5 findings, 3 claims refuted | `20260911-212913-agy-self-review-r2` |
| codex | account default (gpt-5.6) | max | 40 min | 15 findings, 9 claims refuted | `20260911-212913-codex-self-review-r2` |

agy, confirmed and fixed:

- `meta.json`, `opts.json`, `cmd.txt` were outside the redaction pass, so a
  credential passed as `--model`, `--cwd`, or in `base_url` would land
  in plaintext. Now those flags are scanned before the run starts, the
  files are redacted after it, and `status` redacts what it prints.
- The SIGTERM grace period ended as soon as the direct child exited, so
  grandchildren could be SIGKILLed at once. The grace loop now watches
  the whole process group.
- `describe()` still read a `worker.pid` file that nothing writes.
  Removed.
- No tests for the wait bound, the wall-clock alarm, the idle timeout,
  or log redaction. Four tests added; the alarm test drives a real
  trickling server through the worker subprocess.

agy, not adopted:

- Adding a newline before the closing fence when the file lacks one
  "mutates the bytes". The byte count in the header is the file's, and
  a fence must start on its own line; the alternative (always inserting a
  newline) changes the apparent content of every file that does end with
  one. Documented in the README instead.

codex, confirmed and fixed (v3):

- The process-group timeout test used `$$` inside a subshell, which is the
  parent's pid, so it passed without proving anything. `$BASHPID` now, and
  a second test proves a survivor left behind after a normal exit is
  cleaned up (the runner now sweeps the group on every exit).
- Launcher and worker both wrote `meta.json`; a fast worker's `done` could
  be overwritten with `queued`, and a launcher killed between spawn and
  its second write left no pid. The launcher now writes `launcher.json`
  (pid plus `/proc` start time) and never touches `meta.json` after the
  spawn; liveness compares start times, which also defeats same-boot pid
  reuse.
- A response with no model text but a `max_tokens` stop was `done` with
  only the truncation marker; zero-width-only text passed `.strip()`.
  Visible-text check (whitespace, control, format, ANSI stripped) runs
  before the marker is appended.
- Malformed `data:` payloads and undecodable lines were skipped silently;
  `message_start` and a `stop_reason` were not required. All three now
  fail the run.
- `last` picked by directory name, so two runs in the same second sorted
  by backend name. Ordered by `created_ns` now.
- `--no-preamble` still added a heading and trimmed; prompt files were
  read in text mode. Raw now, `newline=""` on read.
- `--model` on the `local` backend did not satisfy the probe; model
  stripped after effort mapping; IPv6 loopback rejected by a hand
  parser. Resolved model passed to the probe, stripped first,
  `urlsplit().hostname`.
- The runs root was not fsynced after creating a run directory.
- `fallback_credit_token` (a capability Anthropic may return with a
  refusal) reached the log and the error string. Scrubbed on the wire and
  dropped from the error.
- Anthropic fallback signal also read from `usage.iterations`.
- No `run_claude` test; the secret-refusal test depended on a host key;
  no test for the `--wait` bound, the alarm, idle timeout, redaction,
  launcher handshake, no-final-newline evidence, same-second ordering.
  All added; the suite is hermetic (temp key files) and at 43 cases.
- Skill said "Read `result.md`" three sections after forbidding it;
  "usable right now" overstated what `backends` checks; `died` guidance
  did not mention an orphaned CLI. Fixed; `status` now prints the
  orphan's pgid and the `kill` command.
- README: kimi `medium` mapping, `opts.json` "boolean", timeout
  semantics, `launcher.json`.
- `status --wait` slept a minimum of one second past its deadline.

codex, narrowed rather than fixed:

- Raw CLI output is written by the child and redacted after exit; a
  worker killed mid-run leaves it raw. Streaming redaction of a child's
  stdout would mean piping every CLI through the worker; not worth it for
  children that run with a scrubbed environment and read-only sandboxes.
  The invariant in the docstring and README now says exactly this. API
  streams are redacted line by line before logging.
- The credential definition excludes environment values under 8 chars;
  stated in the docs.
- Kill scope remains the process group; a `setsid` descendant is out of
  reach. A worker that dies leaves its child running; `status` reports
  the orphan with the kill command instead of a watchdog.
- Timeout is the backend deadline; cleanup adds up to ~45 s. Documented.

codex, not adopted:

- A startup handshake that blocks the worker until the parent has
  written its identity: the separate-file design removes the write
  conflict without a handshake.
- A reversible encoding for evidence so a missing final newline is
  represented exactly: the reviewer is a language model reading
  markdown; the byte count in the heading records the fact.

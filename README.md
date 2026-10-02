# antagonist

`antagonist` sends something you produced (a plan, a diff, a design, a
guard script, a claim) to a model you did not use to produce it, with a
fixed review prompt, and keeps a durable record of what came back. It is
a command-line tool for people who work with coding agents and want an
independent check before acting on the agent's output.

## Why

An agent reviewing its own work shares its own assumptions, and so does a
subagent that inherits its context. A different model, given only the
material and a prompt that asks it to test numbered claims, finds the
defect next to where the author was looking: the missing case, the wrong
order, the wrong instrument. In the author's own use the review found real
defects on every run, including on this tool (see `docs/reviews/`). It
also produces some wrong findings every time, which is why the tool
records the run and the method insists on verifying each finding before
relaying it.

The tool does three things and nothing else:

1. **One prompt shape for every backend.** A reviewer preamble, your task
   text, and every evidence file pasted byte for byte. The wording is
   identical across backends so their output can be compared.
2. **Backend selection.** Google Antigravity (`agy` CLI, the default),
   OpenAI Codex (`codex` CLI), Claude Code (`claude` CLI), the Anthropic
   API, Moonshot's API, or any OpenAI-compatible local endpoint. Each CLI
   runs in a sandbox with no tool that writes files (agy's shell can write
   under `/tmp`); the API backends see only what you paste.
3. **A run directory per review** with the prompt as sent, the literal
   command, the raw output, the result and metadata, so a finding can be
   traced and a run reproduced.

It never modifies the material under review, never posts anywhere, and
refuses to start if the assembled prompt contains a credential.

A companion skill for Claude Code (`skill/SKILL.md`) carries the method:
what goes in the prompt, how to frame it so a safety classifier does not
refuse it, when to use which backend, how to poll a long run, and the
verify-then-relay rule. Other agents can read it as a method document.

## Requirements

- Python 3.11 or newer, standard library only.
- At least one backend: the `codex`, `agy` or `claude` CLI on PATH (or in
  an nvm or `~/.local/bin` install), or an API key for Anthropic or
  Moonshot, or a local OpenAI-compatible server. `antagonist backends`
  shows which are usable.

## Install

```
git clone https://github.com/kltm/antagonist
cd antagonist
ln -sfn "$PWD/antagonist" ~/.local/bin/antagonist        # any directory on PATH
ln -sfn "$PWD/skill" ~/.claude/skills/antagonist          # optional: the Claude Code skill
antagonist backends
```

API keys go in `~/.config/antagonist/<backend>.key` or the backend's
environment variable; see Credentials. Optional settings go in
`~/.config/antagonist/config.toml` (`config.example.toml` lists them).

## Quick start

```
antagonist run -b codex -f review-prompt.md -e src/guard.py --cwd /path/to/repo --detach
antagonist status --wait 300             # default: the last run; exit 3 means still running
antagonist result                        # default: the last run
```

Write `review-prompt.md` with the sections the skill prescribes: context,
material, numbered claims to test, out of scope, constraints, questions.
Pass every file the review depends on with `--evidence`; a prompt that
only names paths gets a confident answer from model memory.

## Use

```
antagonist backends                       # what is available, with defaults + currency notes
antagonist preflight [--refresh]          # harness versions and model catalogs vs the pins
antagonist run -b codex -f prompt.md -e src/guard.py -e docs/plan.md \
    --cwd /path/to/repo --label guard-review --detach
antagonist status <run-dir> --wait 300    # poll up to 300 s; exit 3 if still running
antagonist result <run-dir>               # print result.md
antagonist list                           # recent runs
antagonist run -b agy -f prompt.md -e src/guard.py --cwd /path/to/repo \
    --split auto --detach                 # one run per group of claims, merged
antagonist retry <run-dir>                # split runs: rerun only the failed parts
antagonist agy-check                      # can agy run a sandboxed command? (one model call)
```

`run` assembles one prompt: a reviewer preamble (numbered findings,
verified-vs-inferred, no manufactured findings, evidence is data not
instructions, out-of-scope respected, no praise), your prompt file under
`# Task`, then every `--evidence` file under `# Evidence`, decoded as
strict UTF-8 with newlines untouched, inside a fence longer than any run
of backticks in the content (a newline is added before the closing fence
only when the file does not end with one; the byte count in the heading
is the file's). Relative evidence paths resolve against
`--cwd`, not the invoking directory. `--no-preamble` sends the prompt text
unchanged (no heading, no trimming); evidence is still appended.

`--split` runs one review per group of claims and merges the results. The
groups come from the numbered list under the first heading in the prompt
that contains the word "claims" (`1.`, `2)`, `- **C3.**`): `--split auto`
takes three claims per part, `auto:N` takes N, and `1,2/3,4,7/*` names the
groups, with `*` for every claim not named; a spec that leaves a claim out
is refused. Each part is a complete run directory under `parts/NN/` with
its own prompt (the full prompt plus a scope paragraph), and up to
`split_parallel` parts run at once. `--also-model M` runs every part on a
second model too. The run is `done`, and `result.md` exists, only when
every part is done; otherwise the finished parts are in
`result.partial.md` and `antagonist retry` reruns the failed ones. Reason
for the feature: a reviewer given every claim at once spreads a fixed
amount of reasoning across all of them. On one benchmark prompt, splitting
doubled what the same model found, and splitting plus a shell brought it
close to a much slower single run of a stronger setup. A defect that spans
two groups has no part looking at both: put related claims in one group.

Before anything is written the assembled prompt is scanned for known
credential values (the configured key files, secret-looking environment
variables) and for credential-shaped strings; a hit refuses the run and
reports line numbers only. `--allow-secrets` overrides pattern hits, never
known values.

Without `--detach` the run blocks and prints the result. With it, the
worker re-executes this script in its own session (`setsid` semantics).
The launcher records the worker's pid and process start time in
`launcher.json` and never touches `meta.json` again, so there is no
write race with a fast worker; `status` checks liveness against the start
time, which defeats same-boot pid reuse. The worker's whole bootstrap is
inside the failure handler; nothing is left `queued` or `running` without
a live worker behind it.

A run is `done` only with completion evidence: exit 0 for CLIs, a
non-empty final message file for codex, `message_start` plus a
`stop_reason` plus `message_stop` for the Anthropic stream, `[DONE]` or a
`finish_reason` for OpenAI-compatible streams, every `data:` payload
well-formed JSON, and visible text from the model itself (whitespace,
zero-width and control characters and ANSI escapes do not count) before
any truncation marker is appended. Anything else is `failed` with the
reason, and nothing partial is written to `result.md`.

Before the run directory is created, a **preflight** currency check runs
(skip with `--no-preflight` or `ANTAGONIST_NO_PREFLIGHT=1`; `--refresh`
ignores the cache; `--strict` refuses to run on any note). It answers two
questions and only warns; it never switches a model or upgrades a CLI:

- Is each CLI harness at its latest release? Installed `--version` against
  the npm registry (codex, claude) or the GitHub latest release (agy). A
  binary missing from PATH but present in an nvm or `~/.local/bin`
  install is reported and used, with its directory first on the child's
  PATH.
- Is the pinned model the newest of its family in the backend's own
  catalog? `/v1/models` for anthropic, `/models` for moonshot and local,
  `agy models` for agy. "Family" is the id with version numbers, date
  stamps and effort suffixes removed, so `claude-opus-5` is told about
  `claude-opus-5-5` but not about a sonnet; codex publishes no catalog and
  its CLI config default is reported instead; claude follows its CLI.

Results are cached in `~/.cache/antagonist/preflight.json` (versions and
model ids only, mode 0600 from creation) for `preflight_ttl` seconds, a
day by default, so a run pays for the fetch at most once a day; a fetch
with any failed component is retried within the hour. The fetch runs its
components in parallel under one `preflight_deadline` (40 s default);
anything slower is recorded as timed out. No fetch error text is kept,
only the exception class and HTTP status, because a redirect target or a
malformed header value can carry key material. Every note is printed as
`antagonist: preflight: ...` on stderr and recorded in the run's
`meta.json`. A fetch failure is a note, not a refusal.

The harness comparison is against the npm `latest` tag (codex, claude)
or the GitHub latest release (agy). An install pinned to a slower release
channel is expected to trail and will be noted; the note says so.

The Claude Code skill in `skill/SKILL.md` carries the method: what goes in
the prompt, how to frame it, when to run which backend, and the
verify-then-relay rule.

## Backends

| backend | mechanism | reads the repo itself | effort scale | default |
|---|---|---|---|---|
| `codex` | `codex exec - -s read-only -C <cwd> --ephemeral -o result -c project_doc_max_bytes=0 -c model_reasoning_effort=<e>`; prompt on stdin | yes, read-only sandbox; `AGENTS.md` not loaded | low, medium, high, xhigh, max | max |
| `agy` | `env -u JAVA_HOME agy -p '' --input-format stream-json --output-format stream-json --sandbox --agent antagonist-reviewer --add-dir <cwd> [--effort <e>] --model <slug>`, run from a workspace inside the run directory; prompt as one JSON line on stdin | yes: file search and read under `--cwd` and `--read-dir`; with `--tools exec`, shell commands in agy's sandbox | low, medium, high (pro models: low, high, carried in the slug) | high |
| `claude` | `claude -p --output-format json --permission-mode plan --tools Read,Grep,Glob --no-session-persistence --safe-mode --strict-mcp-config --disable-slash-commands --effort <e>`; prompt on stdin | yes, read-only tools; no `CLAUDE.md`, hooks, MCP, skills | low, medium, high, xhigh, max | max |
| `anthropic` | `POST /v1/messages`, streaming, adaptive thinking, `output_config.effort`, server-side refusal fallbacks | no | low, medium, high, xhigh, max | max, `claude-opus-5-5` |
| `moonshot` | `POST /v1/chat/completions` (OpenAI-compatible), streaming, `reasoning_effort` on kimi-k3 | no | low, high, max | max, `kimi-k3` |
| `local` | same as moonshot against `base_url` (default ollama) | no | ignored | model required |

`--effort` takes the shared scale and is mapped per backend (`xhigh` and
`max` become `high` on agy; `medium` becomes `high` on kimi-k3, which has no medium).

The literal command for every CLI run is written to `cmd.txt` in the run
directory. For API runs `cmd.txt` holds the endpoint and the request body
with the prompt and the credential elided.

Choose by what the review needs. Reading a diff or plan with all the
evidence attached: any backend. Finding what the evidence leaves out, or
verifying a finding by running something: `codex` or `agy`, the two that
read the repository themselves and execute inside a sandbox (`agy` only
where its sandbox works; `antagonist agy-check` tests it). `claude` can
search but not execute; the API backends only read what they are given.
`--max-words N` appends a word budget, and `status` prints a `hint:` line
naming the fix for known failures.

### Notes per backend

**codex.** `~/.codex/config.toml` sets its own default effort; the runner
always passes one explicitly. Codex's classifier has refused prompts
phrased as "give me the bypass"; phrase reviews as hardening from the
owner's side. The MCP form of codex is not used here: a long review
exceeds the MCP idle timeout while the server keeps running.

**agy (Antigravity CLI).** agy's default agent writes files in its
workspace without asking, headless included, so the runner never uses it.
Every run gets a workspace inside the run directory (`ws/`) holding one
generated agent, `antagonist-reviewer`, whose tool list has no write tool.
The directory under review is attached with `--add-dir`, as is each
`--read-dir`; no standing read grant in agy's settings is needed.

Two modes, chosen with `--tools` or `[agy] tools`, plus `search_web` in
both (switch off with `web = false`):

- `read`: agy's file tools (`view_file`, `grep_search`, `find_by_name`,
  `list_dir`), confined to the attached directories.
- `exec`: `run_command` only, a shell in agy's terminal sandbox (no
  network, most of the file system readable, writes only under `/tmp` and
  cache directories; a request to leave the sandbox is refused headless).
  The file tools are left out on purpose: they are not sandboxed, their
  reach is narrower than the shell's, and one refused read ends a turn.
  Because commands can write under `/tmp`, `/var/tmp` and `~/.cache`, the
  runner refuses a `--cwd` or `--read-dir` under those in this mode.

`auto`,
the default, picks `exec` when agy's own `settings.json` has
`"toolPermission": "proceed-in-sandbox"`, because headless agy refuses
every command otherwise. That setting is global to agy and is yours to
make. What else `exec` needs on Linux:

- User namespaces. Where the kernel restricts them for unconfined
  programs (Ubuntu 24.04: `kernel.apparmor_restrict_unprivileged_userns=1`)
  the sandbox dies with "connecting to sandbox server: ... connection
  reset by peer" until an AppArmor profile grants `userns` to the agy
  binary: `profile agy-sandbox <path-to-agy> flags=(unconfined) { userns, }`.
  `backends` and `run` warn when no profile under `/etc/apparmor.d` names
  the binary.
- No `JAVA_HOME`. With it set, the sandbox tries to add its certificate to
  the JVM trust store, finds it read-only and exits. The runner removes
  the variable from agy's environment.

`antagonist agy-check` runs one sandboxed command through agy and passes
only if the command's own output shows it ran; run it after agy updates
itself. During a review, a command whose output is the sandbox-connection
error (not one that merely prints the phrase from a file) fails the run instead of letting the reviewer fall back to
inference.

agy can exit 0 with status `ERROR` and a complete-looking answer when its
stream is interrupted; the runner reads the status, treats that as a
failure and retries once (`retries`), keeping the failed attempt's logs as
`*.try1`. A refused tool ends the turn `SUCCESS` with an empty response,
because headless agy cannot ask. The runner then resumes the same
conversation (`--conversation <id>`) with a message saying what was
refused and to carry on without it, at most `nudges` times (default 2;
the earlier turn's logs are kept as `*.turnN`); if the response is still
empty the run fails naming the refused action. Prompts go in over stdin because a
single argv string is capped at 128 KiB on Linux. Pro models carry effort
in the slug (`gemini-3.1-pro-low|high`): the runner strips any suffix you
passed and appends the normalized effort; other models get `--effort`.
`high` is the top level these models offer. Usage is attributed to the
GCP project agy is logged into. agy's "verified" is not reliable on its
own (it has checked the wrong installed copy of a library and called the
result verified): check findings before acting on them.

Four things agy does to a turn, and what the runner does about each;
every case is recorded as a note in `meta.json`, shown by `status` and in
the notes column of a merged split result:

- It holds the turn open after answering while a command the reviewer
  started is still running (one that waits on standard input never
  ends), and emits its result only when `--print-timeout` expires: an
  answer at 8 minutes, exit at 60. When the newest stream event closes a
  reply, an answer is in the stream and nothing follows for `linger`
  seconds (default 180; 0 disables), the runner ends the process and
  takes the answer from the stream.
- A turn that goes silent for `stall` seconds (default 600; 0 disables)
  is ended: retried like an interrupted stream when the stream holds no
  complete answer, taken as finished when it does. This is a safeguard;
  the one silent run seen so far was the runner's own fault.
- It adds text after the answer when a background command reports back.
  Text that follows a system message after the longest reply is kept at
  the end of the result below a marker, and alone in `result.trailing.md`,
  so a conclusion that arrived late is still in front of the reader.
- It can stop mid-answer. An answer that ends inside a code block is
  flagged as possibly cut short (any backend).

**claude.** Uses the session login. `--safe-mode` drops `CLAUDE.md`,
hooks, plugins, MCP servers and skills so the reviewer does not inherit
the repository's own instructions; plan mode plus a read-only tool list
keeps it from editing. `--bare` is not used because it also skips the
credential store. `modelUsage` in the JSON result names the models that
served the turn.

**anthropic.** Raw HTTP on purpose (no SDK dependency). Streams SSE and
assembles text deltas; `message_stop` is required. Requests `fallbacks:
"default"` with the matching beta header so a policy refusal is retried
server-side on another model; `--no-fallbacks` turns that off. When a
fallback block arrives the served model is taken from it and
`served_by_fallback` is recorded. A refusal on the final response fails
the run with the category. `max_tokens` and `model_context_window_exceeded`
stops are marked in the result. Default `max_tokens` is 64000.

**moonshot / local.** OpenAI-compatible chat completions. Moonshot gets
`max_completion_tokens` and, for kimi-k3 only, `reasoning_effort`; local
gets `max_tokens`. `stream_options.include_usage` asks for a final usage
chunk; providers that ignore it leave `usage` null. Streamed
`reasoning_content` is saved to `reasoning.md`.

## Credentials

Never on the command line, never in the run directory, never on the
terminal. Key files are read into memory inside the backend call and
accept a bare token, `KEY=value`, or `export KEY=value`. Known values are
redacted from every error message, status line, and metadata the runner
writes, and from API stream lines before they are logged; an Anthropic
`fallback_credit_token` is scrubbed on sight. CLI children start with
secret-looking variables (`*KEY*`, `*TOKEN*`, `*SECRET*`, `*PASSW*`,
`*CREDENTIAL*`; values of 8+ characters join the redaction set) removed
from their environment, and their logs are redacted after they exit, so a
worker killed mid-run can leave raw CLI output behind. Authenticated HTTP
refuses redirects and refuses plain `http://` to anything but loopback.
`--model`, `--cwd`, `--label` and `base_url` are scanned like the prompt.
Defaults:

- anthropic: `~/.config/antagonist/anthropic.key` or `$ANTHROPIC_API_KEY`
- moonshot: `~/.config/antagonist/moonshot.key` or `$MOONSHOT_API_KEY`

Keys kept elsewhere need only a `key_file` line per backend in the
config file; the defaults above are a convention, not a requirement.

Override in the config file.

## Config

`~/.config/antagonist/config.toml` (optional; see `config.example.toml`).
Per-backend `model`, `effort`, `key_file`, `base_url`, `max_tokens`, for
agy also `tools`, `web`, `retries`, `nudges`, `linger`, `stall` and `quota_pause`, plus top-level `default_backend`,
`timeout`, `idle_timeout`, `split_parallel`, `preflight_ttl` and
`preflight_deadline`. `ANTAGONIST_CONFIG`, `ANTAGONIST_RUNS`,
`ANTAGONIST_CACHE` and `ANTAGONIST_AGY_SETTINGS` override the paths. `base_url`
shapes differ by API: anthropic takes the host (`/v1/messages` is
appended; a trailing `/v1` is stripped), moonshot and local take the
OpenAI-style root including `/v1` (`/chat/completions` is appended).

## Run directory

`~/.local/state/antagonist/runs/<YYYYMMDD-HHMMSS>-<backend>[-<label>]/`

| file | content |
|---|---|
| `prompt.md` | the assembled prompt |
| `prompt.agy.jsonl` | agy only: the one-line stream-json message that went to stdin |
| `cmd.txt` | literal command, with the directory it ran in and the absolute path of the file that went to stdin; or endpoint + redacted request |
| `reasoning.md` | API backends: streamed reasoning, when the provider returns any |
| `opts.json` | resolved options (the key file's path, never its contents) |
| `launcher.json` | detached runs: the worker's pid and start time as recorded by the launcher |
| `meta.json` | status, timings, model served, usage, error |
| `stdout.log`, `stderr.log` | raw process output, or the raw event stream for API runs |
| `result.raw.md` | codex only: `--output-last-message` file |
| `result.md` | the review |
| `child.pid` | pid of the CLI backend process (its process group is what a timeout kills) |
| `worker.log` | detached runs only: the worker's own stdout/stderr (normally empty) |
| `ws/` | agy only: the workspace agy runs in; holds the generated reviewer agent |
| `stdout.log.try1`, `stderr.log.try1` | agy only: logs of an attempt that was retried |
| `*.turnN`, `nudgeN.agy.jsonl` | agy only: logs and command of a turn that ended on a refused tool, and the corrective message that followed |
| `result.trailing.md` | agy only: text agy added after its answer, kept out of the result |
| `parts/NN/` | split runs: one complete run directory per part |
| `result.partial.md` | split runs that failed: the parts that finished, merged |

Run directories and the runs root are mode 0700. `status` reports `died`
when a queued or running run's worker is gone, and prints an `orphan:`
line with the exact `kill` command when the backend CLI's process group
outlived it. `last` orders by creation time, not directory name.

Timeouts: `--timeout` is the backend deadline. Cleanup (SIGTERM grace,
SIGKILL grace) and the agy outer margin add up to about 45 s on top of
it; the API alarm fires at `--timeout` + 5 s. `meta.json` records the
requested value.

## Tests

```
python3 -m unittest discover -s tests -v
```

Stdlib `unittest`, hermetic (temp key files, local SSE server, fake CLI executables):
credential redaction and refusal, redirect refusal, stream completion
gating, fallback attribution, evidence byte-exactness and cwd resolution,
process-group kill on timeout, agy effort dispatch, died detection, the
agy reviewer agent (no write tool, review directory attached, `JAVA_HOME`
removed), status and sandbox-failure gating, retry, corrective turns
after a refused tool, a prompt larger than the pipe buffer under the
watch loop, a turn held open after the answer, a silent attempt, text
added after the answer, an answer cut short, material in a directory
the sandbox can write, claim parsing, split, merge and part retry.

## Known limits

- Timeout kill covers the backend's process group. A descendant that
  called `setsid` itself is out of reach.
- Redaction covers known values. A backend that prints some other secret
  it found on disk would land in its log; the read-only sandboxes and the
  scrubbed environment are the mitigation, not a guarantee.

- API-backend runs have two bounds: `--idle-timeout` (default 300 s, the
  longest the stream may go silent; a kimi-k3 stream stalled mid-word for
  good in one review) and `--timeout` wall clock via `SIGALRM`
  in the worker.
- agy `exec` mode depends on host state the runner does not own: agy's
  `toolPermission` setting, an AppArmor profile where user namespaces are
  restricted, and agy's own sandbox, which changes between releases. agy
  updates itself; `antagonist agy-check` is the test.
- agy's sandbox is not read-only: commands can write under `/tmp` and
  cache directories, and can read most files the user can, including
  credential files that sit in a repository checkout. Its output goes to
  the model.
- Ending a held-open turn is a judgment from the event stream. A reviewer
  that answers and then waits longer than `linger` for a command, meaning
  to add to its answer, loses the addition.
- The answer is taken as every reply up to the longest one. A short
  remark before it ("a command is running, I will wait") stays in the
  result, and text after it is set aside only when a system message
  separates the two and it is shorter than the answer.
- `--split` merges by concatenation. Duplicate findings across parts and
  models are left for the reader, and a defect that needs two claims from
  different groups can be missed (each part sees every claim but reports
  on its own).
- The cut-short check sees an unclosed code block only; an answer that
  stops early in prose is not flagged.
- A per-minute request quota on the Gemini Enterprise service has failed
  parts of an 8-part run (HTTP 429); the attempt is retried after
  `quota_pause` seconds and `retry` reruns what still failed. The runner
  does not pace requests.
- No cost accounting. Usage is recorded where the backend reports it.
- The preflight compares versions and catalog ids only. It cannot tell
  whether a newer release changed behaviour, whether a newer model is
  better for review, or what the codex account actually serves.
- The preflight deadline bounds its worker threads, not every descendant
  of a probed CLI. Two concurrent cold starts both fetch (no lock); the
  atomic replace keeps the cache file whole either way.
- `local` is probed by `GET <base_url>/models`; nothing is bundled for it.
  Point it at your own server (ollama, llama.cpp, vLLM) in the config.

## License

BSD 3-Clause; see `LICENSE`.

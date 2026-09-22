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
2. **Backend selection.** OpenAI Codex (`codex` CLI), Google Antigravity
   (`agy` CLI), Claude Code (`claude` CLI), the Anthropic API, Moonshot's
   API, or any OpenAI-compatible local endpoint. Each CLI runs read-only
   in a sandbox; the API backends see only what you paste.
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
antagonist agy-allow <dir>                # grant agy headless read access
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
| `agy` | `agy -p '' --input-format stream-json --output-format stream-json --sandbox [--effort <e>] --model <slug>`; prompt as one JSON line on stdin | reads and listings under a granted directory only | low, medium, high (pro models: low, high, carried in the slug) | high |
| `claude` | `claude -p --output-format json --permission-mode plan --tools Read,Grep,Glob --no-session-persistence --safe-mode --strict-mcp-config --disable-slash-commands --effort <e>`; prompt on stdin | yes, read-only tools; no `CLAUDE.md`, hooks, MCP, skills | low, medium, high, xhigh, max | max |
| `anthropic` | `POST /v1/messages`, streaming, adaptive thinking, `output_config.effort`, server-side refusal fallbacks | no | low, medium, high, xhigh, max | max, `claude-opus-5-5` |
| `moonshot` | `POST /v1/chat/completions` (OpenAI-compatible), streaming, `reasoning_effort` on kimi-k3 | no | low, high, max | max, `kimi-k3` |
| `local` | same as moonshot against `base_url` (default ollama) | no | ignored | model required |

`--effort` takes the shared scale and is mapped per backend (`xhigh` and
`max` become `high` on agy; `medium` becomes `high` on kimi-k3, which has no medium).

The literal command for every CLI run is written to `cmd.txt` in the run
directory. For API runs `cmd.txt` holds the endpoint and the request body
with the prompt and the credential elided.

### Notes per backend

**codex.** `~/.codex/config.toml` sets its own default effort; the runner
always passes one explicitly. Codex's classifier has refused prompts
phrased as "give me the bypass"; phrase reviews as hardening from the
owner's side. The MCP form of codex is not used here: a long review
exceeds the MCP idle timeout while the server keeps running.

**agy (Antigravity CLI).** Headless mode cannot prompt, so any tool without
an allow rule is soft-denied and the run ends with "no output produced".
`read_file(<dir>/)` rules work and cover both file reads and directory
listings under that directory (`antagonist agy-allow <dir>` adds one,
backing up `settings.json` first). Listing or reading outside a granted
directory, and the search tools, are denied, so the preamble confines the
model to the workspace and forbids searching. Prompts go in
over stdin because a single argv string is capped at 128 KiB on Linux.
Pro models carry effort in the slug (`gemini-3.1-pro-low|high`): the
runner strips any suffix you passed and appends the normalized effort;
other models get `--effort`. Usage is attributed to the GCP project agy is
logged into.

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
Per-backend `model`, `effort`, `key_file`, `base_url`, `max_tokens`, plus
top-level `default_backend`, `timeout`, `idle_timeout`, `preflight_ttl`
and `preflight_deadline`. `ANTAGONIST_CONFIG`, `ANTAGONIST_RUNS` and
`ANTAGONIST_CACHE` override the paths. `base_url`
shapes differ by API: anthropic takes the host (`/v1/messages` is
appended; a trailing `/v1` is stripped), moonshot and local take the
OpenAI-style root including `/v1` (`/chat/completions` is appended).

## Run directory

`~/.local/state/antagonist/runs/<YYYYMMDD-HHMMSS>-<backend>[-<label>]/`

| file | content |
|---|---|
| `prompt.md` | the assembled prompt |
| `prompt.agy.jsonl` | agy only: the one-line stream-json message that went to stdin |
| `cmd.txt` | literal command (with the file that went to stdin), or endpoint + redacted request |
| `reasoning.md` | API backends: streamed reasoning, when the provider returns any |
| `opts.json` | resolved options (the key file's path, never its contents) |
| `launcher.json` | detached runs: the worker's pid and start time as recorded by the launcher |
| `meta.json` | status, timings, model served, usage, error |
| `stdout.log`, `stderr.log` | raw process output, or the raw event stream for API runs |
| `result.raw.md` | codex only: `--output-last-message` file |
| `result.md` | the review |
| `child.pid` | pid of the CLI backend process (its process group is what a timeout kills) |
| `worker.log` | detached runs only: the worker's own stdout/stderr (normally empty) |

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
process-group kill on timeout, agy effort dispatch, died detection.

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
- An agy run whose model tried a denied tool ends `SUCCESS` with an empty
  response; the runner turns that into a failed run with the stderr note.
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

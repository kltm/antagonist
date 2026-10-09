# 2026-10-09: the `cborg` backend

Two runs. The first used the new backend itself, on an unrelated change,
before its own diff was reviewed; the second was agy reviewing this diff.

## Run 1: first real use of `cborg` (reviewing operations-cborg's free-tier overlay)

Run dir: `~/.local/state/antagonist/runs/20261009-145706-cborg-cborg-free-tier-first-run`
`-b cborg --effort max --split auto --max-words 1500`, 2 parts (claims 1-3; 4),
both done in 140 s, parallel=5 recorded in meta.json, $0 (unmetered).

Quality: part 2 refuted the reviewed repo's "any assignment is caught"
claim about its pre-commit hook with four reproducible gaps (colon-form
JSON/YAML assignments, key-bearing JSON filenames outside the backstop,
rename evasion, `grep -q` + `pipefail` SIGPIPE on a large diff); all four
reproduced against the old hook in a scratch repo and are fixed in
operations-cborg `f1a019b`. Part 1's major (compact window with zero
headroom) was answered from the Claude Code docs (a window, not a trigger)
and recorded there rather than changed. One minor factual slip in the
probe that preceded this run (`/var/tmp` called "frequently a tmpfs").

## Run 2: agy on this diff

Run dir: `~/.local/state/antagonist/runs/20261009-145643-agy-antagonist-cborg-backend-review`
`-b agy --effort high --split auto --max-words 1200`, 2 parts, 300 s.

| finding | disposition |
|---|---|
| `read_key_file` takes the first non-comment line even when it is not an assignment (`set -a` would become the key) | fixed: the first `_ASSIGN_RE` match wins wherever it sits; raw-token fallback only when no assignment exists; tests added |
| explicit `split_parallel = 0` swallowed by `or`, falls back to the global | fixed: `split_parallel_for()` distinguishes absent from 0 (0 floors to 1); tests added |
| a raw token beginning with `#` is now read as a comment | accepted; documented in the docstring (no real token starts with `#`) |
| `norm_effort` would `KeyError` on an unknown effort string | refuted: `norm_effort` dies on any value outside `EFFORTS` before the backend branches |
| `run_openai_compat(provider="anthropic")` now raises "no API key" instead of proceeding keyless | accepted: never invoked that way; the new behavior is the safer one |
| `reasoning_effort` "always passed" overstated: only when truthy | accepted as a wording error in the claim; the config default is `max` |
| untested: cborg `high`/`max` mapping, the cborg catalog job in `fetch_preflight`, the split-parallel override | fixed: tests added (87 total) |

## Run 3: cborg on its own backend commit (`488520e`) and the no-fable `profile_cborg`

Run dir: `~/.local/state/antagonist/runs/20261009-151101-cborg-cborg-reviews-its-own-backend`
`-b cborg --effort max --split auto --max-words 1500`, 2 parts, 169 s, $0.
Evidence: the commit diff and the bash profile; the live config and the
executor were not in evidence, which accounts for the refuted majors.

| finding | disposition |
|---|---|
| `read_key_file` returns an inline trailing comment as part of an unquoted value (`K=abc # note` → `abc # note`) | fixed: values are read as a shell would (quotes honoured, unquoted word ends at whitespace); tests |
| first assignment may not be the token when an env file sets other variables first | fixed: an assignment whose name ends in TOKEN/KEY/SECRET wins, else the first; tests. For the live file the token was already first |
| lowercase assignment names fall through to the raw-token path and return the whole line | fixed for `export name=…` (any case); bare `name=…` stays a raw token because `abc=def==` is a valid raw token (existing test) |
| `split_parallel` is recorded in meta.json but maybe not enforced | refuted: the split executor sizes its pool from `meta["split"]["parallel"]`; `--also-model` parts share that pool |
| `profile_cborg` validates `cborg.env` while the runner reads `cborg.key` / `$CBORG_API_KEY` | refuted for this host (the live config points `[cborg] key_file` at the same env file), but the gap was real in general: the profile now also requires `antagonist backends` to list cborg as `ok` |
| the profile's "other 403 = key problem" branch | fixed: CBORG answers 401 for a bad key (probed); 401 is now the key branch, 403 is IP (`ip_not_authorized`) or other |
| an unset `ANTHROPIC_AUTH_TOKEN` under `set -u` is reported as "HTTP none" | fixed: NOKEY and no-response are distinct outcomes; missing curl is reported |
| probe endpoint `/user/info` differs from the backend's base path | fixed: the probe uses `/v1/models` under the backend's base URL |
| `split_parallel_for` global fallback can raise on a non-numeric global | fixed: tries backend, global, then 6 |
| effort map keyed to the backend, not the model | fixed: `reasoning_effort` only for `cborg-coder-max*`; other models through the gateway get none |
| a `length` finish with partial content passes as success | accepted as designed: the result carries the truncation marker and `raw_status=length`; tests added for both the empty-content failure and the partial-content marker |
| no run-time explanation of `ip_not_authorized` | fixed: the cborg path re-raises a 403 of that shape with the 30-day-entry explanation (the generic path already included the gateway's body) |
| stale comment "a 403 is reported as an IP problem, not a key problem" | fixed |
| `-b`, `--split auto`, `--max-words` validity | closed by use: every run in this record used them |
| reasoning arrives inline as `<think>` rather than `reasoning_content`? | closed: 6292 `reasoning_content` deltas and a 72 KB `reasoning.md` in part 01 of this very run |

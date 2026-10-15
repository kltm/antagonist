# 2026-10-14: the `reflection` backend

One real run so far, at the lowest effort, on the day the backend was
written. No external review of the diff yet (agy or cborg on this change
is still to do); the hermetic suite passed (87 tests) before the run.

## Run 1: first real use of `reflection` (a two-claim review of a shell one-liner)

Run dir: `~/.local/state/antagonist/runs/20261014-155736-reflection-reflection-smoke3`
`-b reflection --effort low --max-words 150`, no split, 25 s, free (open-beta
allowance). Usage as reported by the final chunk: 1,856 tokens, 661 of them
reasoning, 448 prompt tokens cached. `reasoning_content` arrived as stream
deltas and was saved to `reasoning.md` (3 KB); the result carried one verdict
per claim, a refuted list and open questions, as asked.

Quality: it refuted the claim that
`sed -i "s|REPLACE_ME|$(xclip -o -selection clipboard)|" <file>` tolerates a
`|` in the pasted value (the `|` is the `sed` delimiter; `&` and `\` would also
be interpreted), which is right and was the point of the test. It held the
shell-history claim with the correct reason (the history stores the command,
the substitution happens at execution) and named the gap it did not check
(argv and process listings).

## Observed on the way

- Chat completions returned a bare Cloudflare `error code: 502` (text/plain,
  no `x-request-id`) for both of the org's keys from about 15:40 to 15:57
  PDT, while `/models` answered in 0.2 s. Two runner attempts failed with
  `HTTP 502 ...` in 34 to 38 s; the generic HTTP error path reported it
  correctly and no hint fired (none is needed: it was an outage). Expect this
  on a beta; `status` shows it plainly.
- Both keys are JWTs of 580 characters; `read_key_file` takes them from an
  `export REFLECTION_API_KEY=...` line in an env file (the configured
  `key_file` is the work key under `~/local/share/secrets/go/env/`).

## Untested

- `--split` on this backend, and whether a split deep review at `high` or
  `max` fits the 100k output tokens per UTC day the beta allows the whole
  org (reasoning counts; this one run spent 807 output tokens at `low`).
- The live shape of `daily_allowance_exhausted` (429), `insufficient_credits`
  and `prepaid_balance_exhausted` (402): the re-raise and the two `HINTS`
  entries match on the code strings from the vendor's `errors.md`, exercised
  only by the unit test's synthetic error event.
- `fetch_preflight`'s reflection catalog job against the live endpoint (the
  unit test stubs it).

## Run 2: reflection on its own backend commit (`2cb4e94`)

Run dir: `~/.local/state/antagonist/runs/20261014-173728-reflection-reflection-self-review`
`-b reflection --effort high --max-words 1500`, no split, 37 s, free.
Evidence: the commit diff (script, example config, tests), the docs diff,
and the unchanged functions and tables the change depends on, extracted by
name. Eight claims, one verdict each; eight further findings.

| finding | disposition |
|---|---|
| (major) an existing but unreadable key file raises before the `$REFLECTION_API_KEY` fallback | confirmed (`read_key_file` on a `chmod 000` file raises `PermissionError`), but pre-existing for every API backend and untouched by this commit; failing loudly on a present-but-unreadable key file is arguably right. Deferred: if changed, report "key file unreadable" rather than fall back silently |
| several `*KEY`/`*TOKEN`/`*SECRET` assignments: the first wins, no preference for the backend's own name | confirmed as the documented rule (README, Credentials); the operator's file has one assignment. No change |
| a vendor error body that echoed the request header would carry the key into the exception text | refuted in effect: `meta["error"]` is written through `secrets.redact` (line 2431) and every `status` print redacts; configured key values are in the redaction set |
| the other documented codes (403, 409, 503) reach the operator as the raw HTTP error | accepted: the vendor's `message` text in the body names the fix; only the three codes a reviewer is likely to hit got a rewrite |
| a benign `daily_allowance_exhausted` in `stderr.log` would fire the hint | confirmed for CLI backends in principle (every `HINTS` entry shares this), negligible for API runs: `_write_api_cmd` empties `stderr.log` (line 1426) |
| `length` finish with reasoning but no content gives "no visible text" and no allowance hint | confirmed, pre-existing (the cborg tests cover it). Deferred: a `HINTS` entry for "no visible text" naming `max_tokens` and effort would help every reasoning backend |
| where `norm_effort` is called, and with which model | verified: `cmd_run` resolves `args.model or b.get("model")` before calling it (line 2159), so the default `Beam-1.0-501B-A23B` matches `^beam-` under `re.I`; the smoke run's `cmd.txt` shows `reasoning_effort` sent with no `--model` given |
| whether `--split` applies to API backends | **doc defect found**: the `--split` help text says "CLI backends only", but `cmd_run` never refuses it for an API backend, and the 2026-10-09 cborg runs used `--split auto`. Claim 6's premise was wrong; the rate-limit arithmetic still holds (6 parts at once against 30 requests/minute), and the real cost of a split on this backend is the shared output allowance. Fix pending: the help string and the skill table |
| `~` in `key_file` | verified: `backends` prints the expanded path |

Untested after this run: a live 429 or 402, the live catalog job, `--split` on
this backend. Reviewer's own evidence status: it cited line numbers from the
pasted excerpts for every holding claim and marked the three it could not
see as undetermined, which matched what was missing from the evidence.

## Run 3: the same prompt at `max`

Run dir: `~/.local/state/antagonist/runs/20261014-174353-reflection-reflection-self-review-max`
`-b reflection --effort max --max-words 1500`, 56 s, free. Same prompt and
evidence as run 2, for a like-for-like comparison of the two efforts.

| | run 2 (`high`) | run 3 (`max`) |
|---|---|---|
| duration | 37 s | 56 s |
| prompt tokens | 14,573 (448 cached) | 14,573 (448 cached) |
| output tokens | 8,112, of which 7,138 reasoning | 11,877, of which 10,932 reasoning |

Findings: the same set as run 2 (unreadable key file as the one major;
first key-ish assignment wins; `length` finish with no content lacks a
diagnostic; `norm_effort` call site, `~` expansion, `status` hint path and
`--split` gating marked as not determinable from the evidence), plus:

| finding | disposition |
|---|---|
| claim 6 refuted on a different ground: `split_parallel` caps concurrency, nothing throttles requests over time, so parts could in principle exceed 30 per minute | accepted as literally true, immaterial in practice: a split has a handful of parts (three for an eight-claim prompt at `auto`, doubled by `--also-model`), far below 30 in any minute |
| no `HINTS` entry for the bare Cloudflare `HTTP 502` seen in the outage | accepted as a cheap improvement, deferred with the "no visible text" hint: "the vendor's inference origin is down while `/models` answers; wait" |
| claim 5 "cannot determine absolutely" (another capture layer might log the key) | honest; the redaction layer (line 2431 and the `status` prints) is outside the excerpts and answers it |

The extra effort bought one additional minor finding and about half again
as many reasoning tokens. Both runs left the prompt cache almost untouched
(448 cached tokens on an identical 14.5k prompt six minutes apart), which
is worth knowing before anyone proposes this model for a chat bot.

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

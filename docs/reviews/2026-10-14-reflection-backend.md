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

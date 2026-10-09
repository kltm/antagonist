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

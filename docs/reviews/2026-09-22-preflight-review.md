# Preflight currency check review, 2026-09-22

The change: a cached pre-flight in `run`, `backends` and a new
`preflight` subcommand that reports whether each CLI harness is at its
latest release and whether each pinned model is the newest of its family
in the backend's own catalog; a PATH fallback for nvm-installed CLIs; the
Anthropic default moved from `claude-opus-5` to `claude-opus-5-5`. Two
backends reviewed the same prompt in parallel, twice. Runs live under
`~/.local/state/antagonist/runs/`.

| round | backend | model | effort | wall time | outcome | run dir |
|---|---|---|---|---|---|---|
| 1 | agy | gemini-3.1-pro-high | high | 4.8 min | 7 findings, 5 claims refuted | `20260922-143405-agy-preflight-review` |
| 1 | codex | gpt-5.6-sol (config default) | max | 28 min | 13 findings, 8 claims refuted | `20260922-143405-codex-preflight-review` |
| 2 | agy | gemini-3.1-pro-high | high | 9.4 min | 4 minor findings, 0 claims refuted | `20260922-150759-agy-preflight-review-r2` |
| 2 | codex | gpt-5.6-sol (config default) | max | 32 min | 11 findings, 5 claims refuted | `20260922-150759-codex-preflight-review-r2` |

The codex run in round one was the first real use of the PATH fallback:
codex was not on the calling shell's PATH and ran from its nvm directory.

## Dispositions, round one

Confirmed and fixed (agy = a, codex = c, by finding number):

- Fetch error text reached `meta.json`, stderr and `cmd_preflight` output
  unredacted; truncation before redaction; escaped newlines in a key
  defeating exact-match redaction (a1, c1). Design change: no error text
  is kept, only exception class plus HTTP status (`_err_tag`);
  `RedirectRefused` is its own class; the fetched dict is serialised,
  redacted and parsed back once so every caller sees the redacted object.
- Cache temp file created under umask and chmodded after rename (a2, c3);
  fixed `.tmp` name shared by concurrent writers (c3). `write_text_atomic`
  now uses `mkstemp` in the same directory, `fchmod` before writing,
  cleanup on failure; the cached path re-tightens 0600.
- nvm directories sorted lexically, `v9` before `v20` (a3, c7). Sorted by
  parsed version; the test has both.
- Cache validation: non-dict JSON, non-numeric epoch, clock backwards,
  future stamp, invalid UTF-8, missing keys (a4, c4). `_cache_fresh`
  requires schema `v`, dict sections, finite epoch, `0 <= age < ttl`;
  `read_json` catches `ValueError`.
- No total bound on a cold fetch; sequential worst case 180 s or more
  (c2). Parallel daemon threads under one `preflight_deadline` (40 s);
  bodies capped at 2 MB; a fetch with failures is cached one hour at
  most. Cold fetch on this host went from 3.8 s to 1.6 s.
- Parser: effort word stripped mid-id (a5, c5); two letter-prefixed
  numbers flattened, hyphenated dates read as versions, unversioned pins
  ordered, presence ignoring dates (c5). Anchored suffix; ambiguous
  shapes are "not assessed"; `Y-M-D` runs become a date; presence is
  `(version, date)`.
- `parse_version` required a dot (a6). Bare majors parse.
- `_NoRedirect()` instance vs class convention (a7). Class.
- "current" printed for an unassessed CLI default (c6). "not assessed".
- `installed_version` ignored exit status and parsed stray numbers;
  `catalog_agy` accepted any slug-shaped line on any exit (c9). Zero exit
  required, stdout first line only, tab-separated lines only, separate
  `installed_error` / `latest_error`.
- `run_codex` filed the config default under `model_served` (c13). Read
  before launch, stored under `extra.codex_config_at_launch`.
- The new lifecycle test could reach a real local model server (c12).
  Its config points `[local]` at port 1 and asserts the run fails.
- Anthropic `/v1/models` is paginated (c open question). Follows
  `has_more`/`last_id`, five pages max.

Refuted:

- `agy models --output-format json` (c9, last bullet): agy 1.2.8 rejects
  the flag. Inferred from a changelog, not run.

Accepted, no code change:

- A cold `--strict` refusal writes the cache file (c10). Claim narrowed
  to "no run directory"; the cache is wanted.
- `--no-preflight` still adds the `preflight` metadata field and the PATH
  fallback applies regardless (c11). Documented as additive.
- npm `latest` can trail a deliberately pinned release channel (c8). The
  note text says so; `--strict` still counts it.

Tests: 43 before, 57 after round two.

## Dispositions, round two

Round two took the fixed diff plus the list above. agy: 9.4 min, all
ten claims hold, 4 minor findings. codex: 32 min, 11 findings, 5 claims
refuted in their absolute forms.

Confirmed and fixed:

- `write_text_atomic` could leak the descriptor if `fchmod` raised before
  `fdopen` (a1). Chmod inside the `with`.
- `ANTAGONIST_NO_PREFLIGHT=0` disabled the check (a2). Falsey strings
  count as off.
- Date-only families (`gpt-4o-2024-05-13`) were never ordered (a3).
  Ordered by date when no version exists.
- `quote(after)` on a non-string `last_id` (a4). `str()`.
- Object-level redaction: `json.dumps` escapes quotes, backslashes and
  newlines, so a literal-match redaction on the serialised text can miss
  a key containing them (c1). Redaction now walks the object; the
  serialised form is derived from the redacted object.
- A fresh cache hit was returned unredacted and `cmd_preflight` printed
  most fields raw (c2). Cache objects are redacted on read, the file is
  chmodded before reading, the schema is bumped to 2 so caches from
  earlier code are discarded, and every `cmd_preflight` line goes
  through redaction. (First attempt at that shadowed `print` and crashed
  the command; caught by running it, not by the suite: the suite never
  invokes `cmd_preflight`. Fixed with a plain `out()` helper.)
- A 200 with the wrong shape (`{}`, missing `data`, `has_more` after
  five pages) became a clean result and a false "not in catalog" (c3).
  `_require` raises `BadResponse`.
- `shutil.which` can return a relative path when PATH holds `.` (this
  host's PATH does), and `_run_cli` let the child resolve the bare name
  again in the review's cwd (c4). The checked absolute path is the one
  that runs, for PATH hits too; an empty inherited PATH falls back to
  `os.defpath`. Test chdirs into the fake's directory with `.` on PATH.
- Unorderable explicit pins were declared present without looking; an
  explicit claude pin produced no note at all (c5). Presence is exact
  even when unordered; codex/claude explicit pins get a "cannot be
  assessed" note.
- Malformed `ttl_s` (`"bad"`, `Infinity`, `0`) crashed `antagonist
  preflight`; bad numeric config values could raise out of the fetch
  (c6, c9 part). Guarded conversions, treated as stale or default.
- "ahead" printed as "latest"; `1.0` sorted below `1.0.0` (c7).
  `compare_versions` pads and distinguishes ahead.
- Date-shaped non-dates (`2025-99-99`, `20259999`) got a fabricated
  order (c8). Validated with `datetime.date`; invalid means not assessed.

Refuted or corrected:

- The prompt said "50 tests"; the reviewed revision had 56 (c11). The
  prompt text was stale. 57 at round two, 57 after these fixes.
- `agy --output-format json models` (root-flag form) does work on 1.2.8
  (c open question; the subcommand-flag form does not). It wraps the same
  tab-separated text in a JSON `response` field, so parsing is unchanged.

Accepted, no code change:

- The deadline bounds the threads, not descendants of a probed CLI, and
  `codex_defaults()` (a local TOML read) runs after it (c9).
- No single-flight lock: two concurrent cold starts can both fetch and
  race to write (c10). Rare; the atomic replace keeps the file whole.
- Release-channel awareness (c7 wrong-instrument): npm `latest` is the
  npm tag only. Documented in the note text and README.

No third round was run.

# tgrep for coding agents

A short guide for AI agents (and the humans who wire them up) that want to
use `tgrep` as a fast search tool inside a repository. It complements the
[README](README.md), which documents every flag; this file covers the few
things an agent has to get right.

## The mental model

tgrep is ripgrep with a pre-built trigram index and an optional server.

```
tgrep index .        # once: build the index into ./.tgrep
tgrep serve .        # once per session: keep the index warm and watch for changes
tgrep "pattern" .    # every search: finds the server, answers in milliseconds
```

A search resolves in this order:

1. **Server** running for this tree: query it over TCP. Fastest, and always
   current because the server watches the filesystem.
2. **On-disk index** but no server: read `.tgrep/` directly. Fast, but only as
   fresh as the last `tgrep index` run.
3. **No index**: scan every file, like grep. Correct but slow on large trees.
   tgrep prints a warning on stderr when this happens.

An agent never has to choose between these. It runs the same command and gets
the same output; only the latency differs.

## Setup

```bash
brew install tgrep                      # macOS, Linux
cargo install --path tgrep-cli --locked # from a checkout
```

Then, from the repository root, once:

```bash
tgrep serve . &
```

`serve` builds the index if none exists and answers queries while it builds.
It writes `.tgrep/serve.json` (PID and port) so clients can find it. Do not
commit `.tgrep/`; add it to `.gitignore`.

Check that a server is up:

```bash
tgrep status .
```

If your agent framework cannot keep a background process alive, skip `serve`
and run `tgrep index .` instead. Searches then use the on-disk index. Re-run
`tgrep index .` after large changes (branch switch, generated code).

## Searching

The command line is a ripgrep subset. Anything you already do with `rg` should
work unchanged.

```bash
tgrep "fn parse_config" .                 # regex, default
tgrep -F "Vec<Option<T>>" .               # literal string
tgrep -w "handle" . -t rust               # whole word, Rust files only
tgrep "TODO|FIXME" . -g "src/**" -C 2     # glob scope, 2 lines of context
tgrep "impl .* for Server" . -l           # file names only
tgrep "deprecated" . -c                   # count per file
tgrep --files . -t py                     # list searchable Python files
```

Rules of thumb for agents:

- **Quote the pattern** and pass the search root explicitly (`.` or a path).
- **Prefer `-F`** when the query is a symbol or a string the user typed. It
  avoids regex-escaping mistakes.
- **Narrow with `-t` or `-g`** before adding `-m`. The index makes scoping
  cheap; `-m` only trims output.
- **Use `-l` first** on a broad query, then search the specific files. This
  keeps output small.
- **Use `-C 2` or `-C 3`** when you need to read the surrounding code.
- **Use `-q`** when you only need a yes/no answer; read the exit code.

## Machine-readable output

`--json` emits one JSON object per line, in ripgrep's format. Record types
are `begin`, `match`, `context`, `end`, and `summary`.

```bash
tgrep "fn main" . --json
```

```json
{"data":{"path":{"text":"src/main.rs"}},"type":"begin"}
{"data":{"absolute_offset":38226,"line_number":998,"lines":{"text":"fn main() {\n"},"path":{"text":"src/main.rs"},"submatches":[{"end":7,"match":{"text":"fn main"},"start":0}]},"type":"match"}
{"data":{"binary_offset":null,"path":{"text":"src/main.rs"},"stats":{"bytes_printed":269,"bytes_searched":50644,"elapsed":{"human":"0.000019s","nanos":18541,"secs":0},"matched_lines":1,"matches":1,"searches":1,"searches_with_match":1}},"type":"end"}
{"data":{"elapsed_total":{"human":"0.000834s","nanos":833792,"secs":0},"stats":{"bytes_printed":529,"bytes_searched":50644,"elapsed":{"human":"0.000834s","nanos":833792,"secs":0},"matched_lines":1,"matches":1,"searches":1,"searches_with_match":1}},"type":"summary"}
```

Any parser written for `rg --json` works as is.

`--vimgrep` gives `file:line:col:text`, one row per match, which is the
easiest format to feed into a "jump to location" step.

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | At least one match |
| `1` | No match |
| `2` | Error (unreadable path, bad regex, ...) |

A match plus an error yields `2`, unless `-q` is set, which yields `0`. Same
as ripgrep.

## When the index is not used

These fall back to a full scan even with a server running, because they widen
the file set the index was built over:

- `--hidden`, `--no-ignore` and variants, `-u`/`-uu`/`-uuu`
- `-a`/`--text`, `--binary`, `-E`/`--encoding`
- `--no-index` (explicit)
- naming a single file instead of a directory

Avoid these on large repositories unless you need them.

## Freshness

- With a **server**, results reflect the filesystem as of the last watcher
  event. Edits made a moment ago are visible.
- With only an **on-disk index**, results reflect the last `tgrep index`.
  Files created since then are not found. Run `tgrep index .` again, or
  start `tgrep serve .`.
- `tgrep --files` reads from the index too. Add `--no-index` to list what is
  on disk right now.

## Keep `index`, `serve` and search flags aligned

`--index-path`, `--exclude`, and `--max-filesize` describe the index. Pass the
same values to `tgrep index`, `tgrep serve` and each search. If they differ,
the client either cannot find the server or silently searches a different set
of files.

```bash
tgrep index . --index-path /tmp/idx --exclude vendor
tgrep serve . --index-path /tmp/idx --exclude vendor
tgrep "pattern" . --index-path /tmp/idx
```

## Repositories without `.git`

tgrep refuses to index a tree that is not a Git repository, to avoid indexing
a home directory by mistake. For a plain directory pass `--no-require-git` to
both `index` and `serve`.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `warning: no index at ... - scanning every file` | No index and no server | Run `tgrep index .` or `tgrep serve .` |
| `Server unreachable, falling back to local index` | Server died or `serve.json` is stale | Restart `tgrep serve .` |
| A new file is not found | On-disk index is stale | Run `tgrep index .` or use a server |
| Search is slow despite a server | Flag bypasses the index (see above) | Drop the flag or scope with `-g`/`-t` |

## Tool definition sketch

If you expose tgrep to a model as a tool, a minimal schema is:

```json
{
  "name": "tgrep",
  "description": "Fast regex search over the repository. ripgrep-compatible flags. Use -F for literal strings, -t/-g to scope, -l for file names only, -C N for context.",
  "parameters": {
    "pattern": {"type": "string"},
    "path": {"type": "string", "default": "."},
    "flags": {"type": "array", "items": {"type": "string"}}
  }
}
```

Run `tgrep <flags...> -- <pattern> <path>` and return stdout. Treat exit code
`1` as "no results", not as a failure.

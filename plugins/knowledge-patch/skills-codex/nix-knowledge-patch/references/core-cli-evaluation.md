# Core CLI, Evaluation, and Configuration

## Paths and process environment

### Isolated XDG directories

Nix-specific variables override the corresponding general XDG variables (since
2.25.0). Use `NIX_CACHE_HOME`, `NIX_CONFIG_HOME`, `NIX_DATA_HOME`, and
`NIX_STATE_HOME` to isolate Nix without changing the rest of a process's XDG
layout.

```sh
export NIX_CACHE_HOME="$PWD/.nix/cache"
export NIX_CONFIG_HOME="$PWD/.nix/config"
export NIX_DATA_HOME="$PWD/.nix/data"
export NIX_STATE_HOME="$PWD/.nix/state"
```

Temporary build directories no longer follow `TMPDIR` (since 2.30.0).
`build-dir` defaults to `builds` under `NIX_STATE_DIR`, normally
`/nix/var/nix/builds`. Update disk provisioning, monitoring, cleanup, and
debugging tools accordingly.

Temporary build-directory names are opaque (since 2.32.0); never derive a
derivation name from one.

## Evaluator behavior

### Integer range

Signed 64-bit integer overflow fails evaluation rather than wrapping (since
2.25.0). `builtins.fromJSON` also rejects integers above the signed 64-bit
maximum. Flake `nixConfig` rejects negative values for configuration options.

```console
$ nix eval --expr '9223372036854775807 + 1'
error: integer overflow in adding 9223372036854775807 + 1
```

### Structured derivations

Creating structured attributes by placing serialized JSON in the `__json`
environment variable is deprecated (since 2.30.0). Use the supported evaluator
mechanism:

```nix
builtins.derivation (attrs // {
  __structuredAttrs = true;
})
```

### Dynamic attributes in `let`

Early 2.32 releases accidentally rejected the special case that permits a
simple string-literal dynamic attribute in a `let` expression. That behavior
is restored in 2.32.5. Other dynamic attributes in `let` remain unsupported.

## Command semantics

### Formatting

`nix fmt` no longer inserts an implicit `.` argument (since 2.25.0). A
formatter can distinguish a generic no-argument invocation from `nix fmt .`,
for example to make the former format an entire tree.

`nix formatter build` builds and links the configured formatter without
running it (since 2.29.0), then prints the formatter executable's full path
rather than only its enclosing store path.

### Raw and JSON output

`nix-instantiate --eval --raw` requires a string and prints the string verbatim
without quotes or escaping (since 2.26.0).

```console
$ nix-instantiate --eval --raw --expr '"hello"'
hello
```

Commands using `--json` pretty-print when stdout is a terminal and remain
single-line when redirected (since 2.29.0). Force the desired mode when
automation may allocate a pseudoterminal:

```sh
nix eval --json --pretty --expr '{ answer = 42; }' > result.json
nix eval --json --no-pretty --expr '{ answer = 42; }'
```

Human-readable commands and progress displays choose size units dynamically
(since 2.33.0). A parser must not assume MiB, a fixed unit, or one common unit
per line.

### Profiles

`nix profile install` was renamed to `nix profile add` (since 2.30.0). The old
spelling remains an alias, but new scripts should use:

```sh
nix profile add nixpkgs#hello
```

## Machine-readable store and derivation data

### `nix path-info`

Pass `--json-format` with `nix path-info --json` (since 2.33.0). Omitting it
currently warns and selects version 1, but is intended to become an error.

- Version 1 uses absolute store-path keys and references, string hashes, and
  string content addresses.
- Version 2 wraps results in `version`, `storeDir`, and `info`; keys and
  references are store-path basenames, and `ca` contains a method and SRI hash.

```json
{
  "version": 2,
  "storeDir": "/nix/store",
  "info": {
    "abc...-foo": {
      "ca": {
        "method": "nar",
        "hash": "sha256-..."
      }
    }
  }
}
```

### Derivation JSON

The unstable derivation JSON used by `nix derivation` changed store paths from
absolute paths to basenames in 2.32.0. Consumers must join them with the
reported store directory rather than assuming `/nix/store`.

`nix derivation show` emits version 4 JSON as of 2.33.0:

- The root contains `version` and `derivations`.
- `inputSrcs` and `inputDrvs` moved to `inputs.srcs` and `inputs.drvs`.
- Fixed-output content addresses are structured objects.
- `nix derivation add` rejects version 3 and earlier.

Migrate producers and consumers together.

## REPL and diagnostics

`:reload` reloads flakes introduced with `:load-flake` as well as files from
`:load` and command-line-loaded items (since 2.29.0).

The REPL accepts semicolon-separated bindings, nested attribute bindings, and
`inherit` statements (since 2.34.0):

```console
nix-repl> a = { x = 1; y = 2; }
nix-repl> inherit (a) x y
nix-repl> p = 1; q = 2;
```

The evaluator can produce collapsed stack samples for flamegraphs, speedscope,
and compatible tools (since 2.30.0). Use `--eval-profiler flamegraph`,
`--eval-profile-file` (default `nix.profile`), and
`--eval-profiler-frequency` (default 99 Hz).

```sh
nix eval --eval-profiler flamegraph \
  --eval-profile-file nix.profile \
  --expr 'builtins.length (builtins.genList (x: x) 1000)'
```

Set `trace-import-from-derivation = true` to warn for each IFD without
disabling it (since 2.30.0). This supports observation and gradual removal
while `allow-import-from-derivation` remains enabled.

## Configuration changes

`build-cores = 0` detects the available processor count and exports that count
through `NIX_BUILD_CORES` (since 2.31.0), matching an unset setting. Builders
no longer receive literal zero.

The old Boolean `warn-short-path-literals` introduced in 2.31.0 warned for
paths such as `foo/bar`; prefer explicit `./foo/bar`. It was superseded in
2.34.0 by tri-state lints:

```ini
lint-url-literals = fatal
lint-short-path-literals = warn
lint-absolute-path-literals = warn
```

Each accepts `ignore` (default), `warn`, or `fatal`. `lint-url-literals`
replaces the `no-url-literals` experimental feature.

The experimental `external-builders` setting lets helper programs build
derivations for selected systems, such as through QEMU (since 2.32.0).

## Client, daemon, and installation compatibility

Nix 2.32.0 raises the daemon worker-protocol floor to protocol version 18,
first spoken by Nix 2.0. Upgrade both client and daemon peers to at least Nix
2.0 before combining them with Nix 2.32.

The Rust installer rewrite is beta in 2.34.0. It can install over an existing
script-based installation without preparation, and its uninstall removes the
entire installation even when that installation predates the Rust installer.

```sh
curl -sSfL https://artifacts.nixos.org/nix-installer | sh -s -- install
/nix/nix-installer uninstall
```

Treat uninstall as destructive and verify the target installation first.

On Linux, `libexec/nix-nswrapper` can run the daemon with full sandboxing in an
unprivileged user namespace (since 2.34.0). Allocate its build-user UID and GID
ranges in `/etc/subuid` and `/etc/subgid`; Nixpkgs supplies `nix.daemonUser`
and `nix.daemonGroup` for the configuration.

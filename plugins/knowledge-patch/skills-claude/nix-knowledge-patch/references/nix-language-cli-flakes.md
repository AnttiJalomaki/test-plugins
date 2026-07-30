# Nix Language, CLI, Flakes, and APIs

Use this reference for evaluation semantics, command output, flake input and
lock behavior, interactive workflows, installer operations, and native APIs.

## Evaluation and expression behavior

### Integer overflow is an evaluation error

Since 2.25.0, signed 64-bit integer overflow fails instead of wrapping.
`builtins.fromJSON` rejects integers above the signed 64-bit maximum, and flake
`nixConfig` rejects negative values for configuration options.

```console
$ nix eval --expr '9223372036854775807 + 1'
error: integer overflow in adding 9223372036854775807 + 1
```

### Supported structured-attribute construction

Since 2.30.0, serializing structured derivation attributes into the `__json`
environment variable is deprecated. Use the supported flag on
`builtins.derivation`:

```nix
builtins.derivation (attrs // { __structuredAttrs = true; })
```

### Warnings for short path literals

In 2.31.0, `warn-short-path-literals = true` warns about relative path literals
such as `foo/bar` that do not begin with `.` or `/`; spell them `./foo/bar`.
This Boolean is superseded by the tri-state settings below.

### Simple-string dynamic attributes restored in 2.32.5

Early 2.32 releases accidentally rejected the special case for simple
string-literal dynamic attributes in `let`. Release 2.32.5 restores it. Other
dynamic attributes in `let` remain unsupported.

### Tri-state path-literal lints

Since 2.34.0, stable `lint-url-literals` replaces the `no-url-literals`
experimental feature, `lint-short-path-literals` replaces the deprecated
`warn-short-path-literals`, and `lint-absolute-path-literals` covers `/...` and
`~/...`. Each accepts `ignore` (default), `warn`, or `fatal`.

```ini
lint-url-literals = fatal
lint-short-path-literals = warn
lint-absolute-path-literals = warn
```

## Flake inputs, locks, and source handling

### Relative path flake inputs

Since 2.26.0, a flake can use a sibling flake through a relative `path:` URL:

```nix
inputs.foo.url = "path:./foo";
```

This changes the lock-file format. Older Nix versions cannot consume lock files
that contain relative-path input locks.

### Lock generation ignores local flake registries

Since 2.26.0, resolving an indirect reference while generating a lock file uses
only the global registry and command-line `--override-flake` values, not system
or user registries. Use explicit URLs when resolution must be reproducible.

### Flakes can require Git submodules

Since 2.27.0, a Git-backed flake can declare its own submodule requirement:

```nix
inputs.self.submodules = true;
```

Callers no longer need to pass `submodules = true`.

### Git fetcher support for LFS

Since 2.27.0, enable Git LFS materialization with `inputs.self.lfs = true` or
`lfs=1` on a Git URL:

```sh
nix flake prefetch 'git+ssh://git@example.com/repo.git?lfs=1'
```

### Output links from `nix flake prefetch`

Since 2.27.0, `nix flake prefetch --out-link ./result <flake-reference>` creates
an output link for the prefetched source.

### Partial `nix flake show` output around IFD

Since 2.29.0, `nix flake show` skips outputs that require
import-from-derivation and continues displaying the remaining outputs instead
of failing the entire command.

### Source information for non-flake inputs

Since 2.30.0, inputs with `flake = false` expose the parent source's
`sourceInfo`, distinguishing that source from a nested input. They can select a
source subdirectory with `?dir=subdir`.

```nix
inputs.data = {
  url = "path:./vendor?dir=subdir";
  flake = false;
};
```

### Parallel flake-input prefetching

Since 2.31.0, `nix flake prefetch-inputs .` fetches all inputs in parallel. It
avoids serialized on-demand fetches but can fetch inputs evaluation would not
use.

### Nested locks are preserved when updating inputs

Since 2.31.0, when an input reference changes during lock update, Nix consults
that input's lock file for nested inputs instead of fetching their latest
versions. This preserves the versions chosen by the updated input.

### SHA-256 Git hashing

Since 2.31.0, experimental Git-hashed store objects support SHA-256 in addition
to SHA-1.

### `nix flake check` can leave substitutable outputs unrealized

Since 2.32.0, a derivation available from a substituter is not downloaded just
because `nix flake check` sees it. A successful check need not leave every
checked output in the local store.

### Resolving registry references

Since 2.33.0, `nix registry resolve NAME` prints the flake reference selected
for an indirect registry name without fetching or evaluating the flake.

### Cloning non-Git flake inputs

Since 2.33.0, `nix flake clone` supports arbitrary input types, including
tarball-backed flakes.

### Channel URL migration

Since 2.33.0, built-in channel URLs use `https://channels.nixos.org/` rather
than `https://nixos.org/channels/`. The old endpoint redirects for now; migrate
stored URLs and allowlists before the redirect is retired.

### Relative `file:` tarball paths are rejected

Since 2.34.0, `file:` tarball references cannot contain relative paths. Use an
absolute path or an unambiguous alternative reference.

## Command behavior and automation

### Nix-specific XDG location overrides

Since 2.25.0, `NIX_CACHE_HOME`, `NIX_CONFIG_HOME`, `NIX_DATA_HOME`, and
`NIX_STATE_HOME` override the corresponding XDG variables for Nix only.

```sh
export NIX_CACHE_HOME="$PWD/.nix/cache"
export NIX_CONFIG_HOME="$PWD/.nix/config"
export NIX_DATA_HOME="$PWD/.nix/data"
export NIX_STATE_HOME="$PWD/.nix/state"
```

### Zero-argument `nix fmt` is formatter-defined

Since 2.25.0, `nix fmt` does not implicitly pass `.`. The formatter can treat a
no-argument invocation differently from `nix fmt .`.

### Raw output from `nix-instantiate --eval`

Since 2.26.0, `nix-instantiate --eval --raw` requires a string result and emits
it without quotes or escaping.

### Terminal-aware JSON formatting

Since 2.29.0, commands using `--json` pretty-print on a terminal and remain
single-line when redirected. Use `--pretty` or `--no-pretty` explicitly,
especially under a pseudoterminal.

### Reloading flakes in the REPL

Since 2.29.0, `:reload` reloads flakes added with `:load-flake` as well as files
added with `:load` and command-line inputs.

### Building a flake formatter

Since 2.29.0, `nix formatter build` builds and links the configured formatter
without running it, then prints the full executable path.

### `nix profile add`

Since 2.30.0, `nix profile install` is named `nix profile add`; the old spelling
remains an alias.

### Stack-sampling evaluation profiles

Since 2.30.0, `--eval-profiler flamegraph` emits collapsed evaluator call
stacks. `--eval-profile-file` selects the output (default `nix.profile`) and
`--eval-profiler-frequency` selects the sampling rate (default 99 Hz).

### Archiving flakes without signature checks

Since 2.30.0, `nix flake archive --no-check-sigs` can copy directly to a remote
store when signature verification would otherwise block the archive.

### Tracing import-from-derivation

Since 2.30.0, `trace-import-from-derivation = true` warns for every IFD without
denying it, allowing CI to inventory IFD while
`allow-import-from-derivation` remains enabled.

### Multiple bindings and `inherit` in the REPL

Since 2.34.0, the REPL accepts semicolon-separated bindings, nested attribute
bindings, and `inherit`:

```text
a = { x = 1; y = 2; }
inherit (a) x y
p = 1; q = 2;
```

### Dynamic size units in human-readable output

Since 2.33.0, commands choose human-readable size units dynamically. Parsers
must not assume MiB or assume that one line uses a single unit.

## JSON schemas

### Derivation JSON uses store-path basenames

Since 2.32.0, the unstable JSON used by `nix derivation` represents store paths
by basename rather than absolute store-directory paths.

### Versioned `nix path-info` JSON

Since 2.33.0, automation should pass `--json-format` with
`nix path-info --json`. Format 1 retains absolute path keys, string hashes and
content addresses; format 2 wraps data in `version`, `storeDir`, and `info`,
uses basenames, and structures `ca` as method plus SRI hash. Omitting the
format currently warns and selects 1, but is planned to become an error.

### Derivation JSON version 4

Since 2.33.0, `nix derivation show` emits a version 4 envelope with `version`
and `derivations`. `inputSrcs` and `inputDrvs` move to `inputs.srcs` and
`inputs.drvs`, and fixed-output content addresses are objects.
`nix derivation add` rejects version 3 and earlier.

## Native API and source-build compatibility

### Namespaced C++ headers and configuration macros

Since 2.28.0, include installed C++ headers through
`nix/<component>/...`. pkg-config supplies `-I${includedir}`, configuration
headers need not be force-included, and remaining public macros use `NIX_`.

```cpp
#include <nix/store/derived-path.hh>
#include <nix/util/configuration.hh>
#if NIX_SUPPORT_ACL
// ...
#endif
```

### Builder-scoped flake settings in the C API

Since 2.28.0, `nix_flake_init_global` is removed. Add settings to each evaluator
builder with `nix_flake_settings_add_to_eval_state_builder`.

### C API flake loading and locking

Since 2.29.0, C callers can load and perform basic locking of flakes. Choose
lock modes with `nix_flake_lock_flags_set_mode_check`, `_virtual`, or
`_write_as_needed`; adding an input override also enables virtual mode. The
`nix-fetchers-c` library manages `nix.conf` settings for built-in fetchers.

### Mutable values for indexed C API access

Since 2.32.0, `nix_get_attr_name_byidx` and `nix_get_attr_byidx` take mutable
`nix_value *` because access may modify a value. The change is ABI-compatible
but can require const-correctness source fixes.

### Lazy C API collection access

Since 2.32.0, use `nix_get_list_byidx_lazy`,
`nix_get_attr_byname_lazy`, and `nix_get_attr_byidx_lazy` to retrieve members
without forcing them, such as when forwarding a sub-value into another
collection or function call.

### C primop errors are sticky by default

Since 2.34.0, a C primop error is remembered in its thunk; forcing it again does
not retry. Mark an intentionally retryable failure recoverable:

```c
nix_set_err_msg(context, NIX_ERR_RECOVERABLE, msg);
```

## Installation

### Beta Rust installer and complete uninstall

Since 2.34.0, the Rust installer rewrite is beta and can run over an existing
script-installed Nix without preparation. `/nix/nix-installer uninstall`
removes the complete installation, including one that predates the Rust
installer.

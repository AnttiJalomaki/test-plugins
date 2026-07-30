# Native APIs and Source Builds

## Building Nix from source

Nix uses Meson and Ninja as of 2.26.0. The Make-based build system was removed,
so source-build automation, packaging, flags, and CI targets must use Meson.

## C++ consumer migration

Installed public headers are namespaced under `nix/<component>/...` (since
2.28.0). `pkg-config` supplies `-I${includedir}` rather than an include
directory ending in `/nix`. Configuration headers no longer need to be
force-included, and remaining public configuration macros use the `NIX_`
prefix.

```cpp
#include <nix/store/derived-path.hh>
#include <nix/util/configuration.hh>

#if NIX_SUPPORT_ACL
// ...
#endif
```

Update include directives and macro checks together; do not compensate for the
new pkg-config include directory by retaining old unnamespaced paths.

## Flake settings and locking in C

The process-global `nix_flake_init_global` function was removed in 2.28.0.
Attach flake settings to each evaluator-state builder:

```c
nix_eval_state_builder *builder =
    nix_eval_state_builder_new(ctx, store);
HANDLE_ERROR(ctx);
nix_flake_settings_add_to_eval_state_builder(ctx, settings, builder);
HANDLE_ERROR(ctx);
```

The C API can load flakes and perform basic locking as of 2.29.0, avoiding
workarounds through `builtins.getFlake`. Select lock behavior with:

- `nix_flake_lock_flags_set_mode_check`
- `nix_flake_lock_flags_set_mode_virtual`
- `nix_flake_lock_flags_set_mode_write_as_needed`

Calling `nix_flake_lock_flags_add_input_override` also enables virtual mode.
The separate `nix-fetchers-c` library owns `nix.conf` settings for built-in
fetchers.

## Collection access and mutability

`nix_get_attr_name_byidx` and `nix_get_attr_byidx` accept mutable
`nix_value *` rather than `const nix_value *` as of 2.32.0, because the lookup
may modify the value. This is ABI-compatible but can require source changes
for const-correct compilation.

Lazy accessors added in 2.32.0 retrieve list and attribute-set members without
forcing evaluation:

- `nix_get_list_byidx_lazy`
- `nix_get_attr_byname_lazy`
- `nix_get_attr_byidx_lazy`

Use them when passing an unevaluated child into another collection or function
call.

## Store operations in C

The C API adds two store operations in 2.34.0:

- `nix_store_query_path_from_hash_part()` resolves a hash fragment to a full
  store path.
- `nix_store_copy_path()` copies a path between stores with controls for
  repair and signature checking.

Respect each store's directory and verification policy; do not manufacture a
full path by assuming `/nix/store`.

## Primop error behavior

An error returned by a C API primop is sticky by default as of 2.34.0.
Re-forcing the thunk reuses the remembered failure rather than calling the
primop again and potentially succeeding. Classify an intentionally retryable
failure as recoverable:

```c
nix_set_err_msg(context, NIX_ERR_RECOVERABLE, msg);
```

Return the default nonrecoverable classification for deterministic errors so
evaluation does not repeat side effects.

# Language and Runtime

## Comprehensions and generators

### Strict generators

OTP 28.0 adds strict comprehension generators. A relaxed generator silently
skips a value that does not match its pattern; a strict generator fails on
that value. Use `<:-` for list and map generators and `<:=` for binary
generators.

```erlang
[X || {ok, X} <:- [{ok, 1}, error, {ok, 3}]].
%% ** exception error: no match of right hand side value error
```

Choose strict generators when malformed input is an invariant violation, and
keep the older operators when filtering by pattern is intentional.

### Zip generators

OTP 28.0 also lets generators joined by `&&` advance in parallel. This
produces a zip rather than a Cartesian product. Any number of list, binary, or
map generators may be zipped, and the zipped group may be mixed with other
generators and filters.

```erlang
[{X, Y} || X <- [1, 2] && Y <- [a, b]].
%% [{1,a},{2,b}]
```

### Assignments and multiple output values

In OTP 29.0, a match used as a comprehension qualifier is a compile error by
default. Previously it could compile and fail later because the matched value
was treated as a non-Boolean filter. Assignment qualifiers are experimental;
enable them per module or runtime:

```erlang
-feature(compr_assign, enable).
%% or: erl -enable-feature compr_assign

hashes(Items) ->
    [H || Item <- Items, H = erlang:phash2(Item), H rem 10 =:= 0].
```

With `compr_assign` enabled, `P = E` behaves as the strict generator
`P <-:- [E]`. A comprehension may also emit several comma-separated values on
each iteration:

```erlang
[I, -I || I <- lists:seq(1, 3)].
%% [1,-1,2,-2,3,-3]
```

Syntax Tools 4.0.2 in OTP 28.2 annotates map comprehensions and map generators
in abstract syntax. Abstract-form visitors must recognize them rather than
assuming every comprehension or generator is a list or binary construct.

## Native records

OTP 29.0 introduces experimental native records. A native record is a runtime
type rather than the tagged tuple used for a traditional record. The leading
`#` in the declaration distinguishes it:

```erlang
-record #vec{x = 0.0, y = 0.0}.
-export_record([vec]).

make_vec(X, Y) -> #vec{x = X, y = Y}.
```

Construction, update, matching, and field access use familiar record syntax.
Values print with the defining module, for example `#geom:vec{...}`.
Definitions are private by default. Code in another module can perform a
field-free match such as `#geom:vec{}`, but construction and field-aware
matching require the defining module to declare `-export_record([vec])`.

Do not treat the experiment as representation-stable. OTP 29.0.1 corrects
native-record programs that could crash the compiler and comparisons that
could return the wrong value or crash ERTS. It also fixes a rare compiler
optimization that could invert a Boolean result. Use 29.0.1 as the minimum
patch level for a native-record deployment.

OTP 29.0.2 further corrects native-record analysis in Dialyzer, formatting by
`io_lib:bformat/2`, and a crash caused by a tuple-record operation inside a
native-record anonymous update. Update the complete OTP installation rather
than selectively patching one affected application.

## Functions, guards, numbers, and types

Function application is left associative in OTP 29.0. An expression such as
`f(X)(Y)` is accepted and means `(f(X))(Y)`.

The guard BIF `is_integer(Term, LowerBound, UpperBound)` returns `true` only
when all three arguments are integers and the term is within the inclusive
bounds. It avoids the common bug where comparison-based range guards also
accept floats:

```erlang
is_digit(C) -> is_integer(C, $0, $9).
```

OTP 28.0 adds based floating-point literals. A second `#` can introduce a
based exponent:

```erlang
2#0.011.       %% 0.375
16#0.011#e5.  %% 4352.0
```

This notation can preserve an exact non-decimal or bit-level representation,
as in `2#0.10101#e8`.

Dialyzer in OTP 28.0 supports nominal declarations:

```erlang
-nominal meter() :: integer().
-nominal foot() :: integer().
```

The two nominal types are incompatible in function input and output specs
despite identical representations. A nominal type remains compatible with a
non-opaque, non-nominal type of the same structure, such as `integer()`.

## Compiler diagnostics and analysis

OTP 28.0 provides the opt-in compiler option `warn_deprecated_catch` for
finding `catch Expr`. A module can temporarily suppress a project-wide
setting with `-compile(nowarn_deprecated_catch).`, but a targeted
`try ... catch` prevents unrelated runtime errors from being swallowed.

The old-style `catch` warning is enabled by default in OTP 29.0. Two more
default warnings cover a variable exported from a subexpression and a match
whose aliased patterns unify constructors. Their migration escape hatches are
`nowarn_export_var_subexpr` and `nowarn_match_alias_pats`.

The opt-in `warn_obsolete_bool_op` finds eager `and` and `or` uses that should
usually be `andalso` and `orelse`, or `,` and `;` in guards.

Old-style guard type tests such as `integer` and `atom` are deprecated in OTP
29.0 and scheduled for removal in OTP 30. The `odbc` application and `ftp` and
`ct_ftp` modules have the same removal schedule.

Functions may be marked with `-unsafe`. The compiler warns by default about
calls to OTP functions classified as always unsafe. Enable
`warn_possibly_unsafe_function` to also report conditional hazards such as
functions that create atoms.

`xref:analyze/2` adds the predefined `unsafe_function_calls`,
`undocumented_function_calls`, and `private_function_calls` analyses. `xref`
now applies `ignore_xref` declarations itself as a post-analysis filter; build
tools should not duplicate that filtering.

Compiling with `to_abstr` in OTP 29.0 retains source `-doc` attributes in the
generated `.abstr` file. Abstract-code consumers and BEAM-targeting languages
can preserve the documentation metadata.

When a BEAM file has no debug information and has `moduledoc(false)`, OTP
29.0.2 makes `xref` return an error rather than crash. Callers must handle the
error result.

## Process and signal behavior

`erlang:hibernate/0` in OTP 28.0 minimizes the calling process's memory while
it waits for the next message. Unlike `erlang:hibernate/3`, it retains the
call stack.

Priority messages in OTP 28.0 require a capability-bearing alias:

```erlang
PrioAlias = alias([priority]),
erlang:send(PrioAlias, Message, [priority]).
```

The `priority` send option places the message ahead of ordinary messages while
preserving signal order. A send through the same alias without the option is
ordinary, and `unalias/1` revokes the capability. Use
`exit(PrioAlias, Reason, [priority])` for a priority exit signal. For link and
monitor signals generated by events, give the `priority` option to
`erlang:link/2` or `erlang:monitor/3`.

## Collections and persistent structures

OTP 29.0 expands `array` with `prepend/2`, `append/2`, `concat/1,2`,
`slice/3`, `shift/2`, `from/2,3`, index-bounded traversals such as `foldl/5`,
and map-fold families including `mapfoldl/3` and `sparse_mapfoldr/5`.

The array's internal representation also changed. An array term serialized
with `term_to_binary/1` on an older OTP release is incompatible with OTP 29;
migrate or regenerate stored values rather than carrying them across the
upgrade.

Map order remains undefined, but OTP 29.0 makes all standard iteration forms
produce a given map's elements in the same order. This aligns
`maps:keys/1`, `maps:values/1`, `maps:to_list/1`, default iterators, and map
comprehensions. Do not infer that the order is sorted or stable across maps.

`gb_sets:from_ordset/1` and `gb_trees:from_orddict/1` now check their input and
reject unordered values instead of building invalid structures. For example,
`gb_sets:from_ordset([3,2,1])` raises `badarg` with reason `not_ordset`.

The new `graph` module is a persistent functional counterpart to `digraph`
and `digraph_utils`. Modifications return new graphs while older values remain
usable:

```erlang
G0 = graph:new(),
G1 = graph:add_vertex(G0, a),
G2 = graph:add_vertex(G1, b),
G3 = graph:add_edge(G2, a, b).
```

In OTP 28.4, `persistent_term:put_new/2` returns quickly if the same key and
value are already present. If the key exists with a different value, it
raises `badarg`:

```erlang
persistent_term:put_new(config, Config).
```

## Code loading

The current working directory is the last entry in the OTP 29.0 default code
path. A BEAM in `.` no longer takes precedence over an OTP or application
module unless code explicitly changes the path. Make development overrides
and test doubles explicit.

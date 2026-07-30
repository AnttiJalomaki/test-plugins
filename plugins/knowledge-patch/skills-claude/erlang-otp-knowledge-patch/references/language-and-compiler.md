# Language and compiler

## Comprehensions and generators

### Strict generators

Since 28.0, strict generators fail when an input does not match their pattern
instead of silently skipping it. Use `<:-` for list and map generators and
`<:=` for binary generators. Existing generator operators remain relaxed.

```erlang
[X || {ok, X} <:- [{ok, 1}, error, {ok, 3}]].
%% ** exception error: no match of right hand side value error
```

### Zip generators

Since 28.0, join generators with `&&` to consume them in parallel rather than
produce a Cartesian product. Any number of list, binary, or map generators
can be zipped, and zipped generators can be combined with other generators
and filters.

```erlang
[{X, Y} || X <- [1, 2] && Y <- [a, b]].
%% [{1,a},{2,b}]
```

### Abstract syntax annotations

Syntax Tools 4.0.2 in 28.2 annotates map comprehensions and map generators in
Erlang abstract syntax. Abstract-form visitors must handle these constructs
explicitly rather than assuming every comprehension or generator is a list
or binary form.

### Assignment qualifiers

In 29.0, a match used as a comprehension qualifier is a compile error by
default instead of compiling and later failing as a non-Boolean filter.
Enable the experimental `compr_assign` feature with
`-feature(compr_assign, enable).` or `erl -enable-feature compr_assign`.
After activation, `P = E` behaves like the strict generator `P <-:- [E]`.

```erlang
-feature(compr_assign, enable).
hashes(Items) ->
    [H || Item <- Items, H = erlang:phash2(Item), H rem 10 =:= 0].
```

### Multiple emitted values

Since 29.0, a comprehension can emit multiple comma-separated values for
each iteration:

```erlang
[I, -I || I <- lists:seq(1, 3)].
%% [1,-1,2,-2,3,-3]
```

## Syntax, literals, and guards

### Shell fun syntax

Since 28.0, the shell accepts the usual `fun Name/Arity` form for
auto-imported BIFs and shell-local functions:

```erlang
F = fun is_atom/1,
true = F(example).
```

### Based floating-point literals

Since 28.0, floating-point literals can use bases other than ten. A second
`#` introduces a based exponent:

```erlang
2#0.011.       %% 0.375
16#0.011#e5.  %% 4352.0
```

This can preserve an exact non-decimal or bit-level representation, as in
`2#0.10101#e8`.

### Left-associative application

Since 29.0, function application is left associative, so `f(X)(Y)` is
accepted and means `(f(X))(Y)`.

### Bounded integer guard

The 29.0 guard BIF `is_integer(Term, LowerBound, UpperBound)` returns `true`
only when all three values are integers and
`LowerBound =< Term =< UpperBound`. Use it instead of range guards that can
accept floats.

```erlang
is_digit(C) -> is_integer(C, $0, $9).
```

## Types and native records

### Nominal types

Dialyzer supports nominal type declarations since 28.0:

```erlang
-nominal meter() :: integer().
-nominal foot() :: integer().
```

`meter()` and `foot()` are incompatible in input and output specifications
even though they share a representation. A nominal type remains compatible
with a non-opaque, non-nominal type of the same structure, such as
`integer()`.

### Experimental native records

Native records introduced in 29.0 are runtime types rather than tagged
tuples. Declare them with `-record #name{...}.`; familiar construction,
update, matching, and field syntax applies. Printed values include the
defining module, for example `#geom:vec{...}`.

```erlang
-record #vec{x = 0.0, y = 0.0}.
-export_record([vec]).

make_vec(X, Y) -> #vec{x = X, y = Y}.
```

Definitions are private by default. A different module can make a field-free
match such as `#geom:vec{}`, but construction and field-aware matching need
`-export_record([vec])` in the defining module. The feature remains
experimental and may change incompatibly.

Use at least 29.0.1 for native-record deployments: it fixes compiler crashes,
incorrect comparisons, and ERTS crashes, along with a rare optimization that
could invert a Boolean result. Release 29.0.2 further fixes Dialyzer analysis,
`io_lib:bformat/2`, and a crash caused by a tuple-record operation inside a
native-record anonymous update. Update the complete installation rather than
selectively patching one application.

## Warnings, unsafe calls, and scheduled removals

The old-style `catch Expr` warning was opt-in as
`warn_deprecated_catch` in 28.0 and is enabled by default in 29.0. A module
can suppress a project-wide opt-in setting with
`-compile(nowarn_deprecated_catch).`, but targeted `try ... catch` clauses
avoid swallowing unrelated runtime errors.

The compiler also warns by default when:

- a variable is exported from a subexpression; use
  `nowarn_export_var_subexpr` only as a migration escape hatch;
- a match aliases patterns that unify constructors; the corresponding escape
  hatch is `nowarn_match_alias_pats`.

Enable `warn_obsolete_bool_op` to find eager `and` and `or`. Prefer
`andalso` and `orelse`, or `,` and `;` in guards.

Functions can carry `-unsafe` attributes. The compiler warns by default about
calls to functions classified as always unsafe.
`warn_possibly_unsafe_function` also reports conditionally unsafe operations,
including atom-creating functions.

Old guard type tests such as `integer` and `atom`, the `odbc` application,
and the `ftp` and `ct_ftp` modules are deprecated in 29.0 and scheduled for
removal in OTP 30.

## Documentation in abstract output

Since 29.0, compiling with `to_abstr` preserves source `-doc` attributes in
the generated `.abstr` file. Abstract-form tooling and BEAM-targeting
languages can retain this documentation metadata.


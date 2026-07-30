# Shell, I/O, and Testing

## Standard input and terminal modes

Standard input is lazy in OTP 28.0. The runtime waits for `io:get_line/2` or an
equivalent operation before reading, rather than consuming input greedily in
advance. Remove `-noinput` workarounds that existed only to prevent those
unrequested reads.

`noshell` mode remains cooked by default. For applications that need
individual keystrokes, no line editing, and no output echo, start it in raw
mode:

```erlang
shell:start_interactive({noshell, raw}).
```

Raw mode lets the application read input without waiting for Enter, so the
application must provide any desired editing and echo behavior.

## Shell functions and remote shells

The OTP 28.0 shell accepts the normal `fun Name/Arity` form for auto-imported
BIFs and shell-local functions:

```erlang
F = fun is_atom/1,
true = F(example).
```

Starting with OTP 28.1, closing a remote shell's input stream exits that shell
without terminating the remote node. The default tracer recognizes a remote
shell and directs trace output to the remote group leader, keeping diagnostic
output attached to the user's shell.

## Terminal styling

OTP 29.0 adds `io_ansi`, which uses the local terminfo database to format or
write terminal colors and styles. A remote call uses the destination
terminal's capabilities.

```erlang
io_ansi:fwrite([bold, red, "wrong answer: ", "~p~n"], [99]).
```

Use `io_ansi:format/2` when the caller needs the encoded sequence as a binary
instead of writing it immediately.

## Common Test output

As of OTP 28.1, Common Test prints map keys in the same order produced by
`maps:iterator(Map, ordered)`. Golden output, log parsers, and comparison tools
that inspect rendered maps must expect this ordering.

This is separate from the OTP 29.0 map-iteration alignment: default map
iteration forms agree with one another for a given map, but map order is still
undefined rather than sorted or globally stable.

## Documentation tests

OTP 29.0 adds `ct_doctest` for running shell-style examples embedded in Erlang
module documentation and documentation files. It supports examples whose
expected result is a failure, can compile example modules for the test shell,
and accepts pluggable parsers for documentation formats such as EDoc and
AsciiDoc.

Use doctests for executable examples, but keep environment-specific setup and
large integration scenarios in ordinary Common Test suites.

## Windows source-tree workflows

OTP 28.1 allows NIFs and linked-in drivers to load on Windows while Erlang is
running inside an Erlang source tree. Native-code build and test workflows no
longer need to leave that layout solely to load their artifacts.

# Migration and Rego

## Rego v1 migration (`1.0-migration`)

Upgrade bundle producers before consumers. Producers using OPA v0.64.0 or
later put `rego_version` in the bundle manifest, and that embedded value takes
precedence over `--v1-compatible`.

While v0 consumers remain, policy must remain v0-compatible and v1 producers
must use `--v0-compatible`, unless modules import `rego.v1`. In the opposite
direction, a v1 consumer of bundles from a v0 producer needs
`--v0-compatible`, because the old bundle cannot declare its Rego version.

Rego v1 treats `in`, `every`, `if`, and `contains` as keywords by default, so
their `future.keywords` imports are no-ops. A rule with a body requires `if`;
a multi-value rule requires `contains`; value assignment does not require
`if`; and a solitary reference head such as `p.a` is invalid.

```rego
package example

allow if {
	input.user == "alice"
}

reasons contains "missing role" if {
	not input.role
}

limit := 10
```

Compilation also rejects duplicate or shadowing imports and refuses `input`
or `data` as a rule or variable name. Replacements using `with input as ...`
or `with data as ...` remain valid.

The removed deprecated built-ins are:

- `any`, `all`, `re_match`, `net.cidr_overlap`, and `set_diff`
- `cast_array`, `cast_set`, `cast_string`, `cast_boolean`, `cast_null`, and
  `cast_object`

Use a v1.0-or-later binary to check and rewrite v0 source in dual-version mode,
then lint it:

```sh
opa check --v0-v1
opa check --v0-v1 --strict
opa fmt --write --v0-v1
regal lint
```

## Compatibility metadata

Capabilities made for `--v0-compatible` include the `rego_v1` feature (since
1.4.0). Do not infer from that feature's presence that the policy is running
without v0 compatibility.

REST policy uploads use the runtime's configured Rego version (since 1.0.0).
This keeps API-loaded policy aligned with the server's v0 or v1 setting.

## Reference and numeric syntax

Dotted references can contain keyword-named segments such as `package`, `if`,
`else`, and `not` without bracket notation (since 1.6.0):

```rego
allow if {
	input.package.source == "internal"
}
```

Primitive numbers with surplus leading zeros always fail to parse (since
1.6.0). Replace values such as `0123` with `123`.

## String interpolation

Prefix a quoted template with `$` and use `{...}` for expressions (since
1.12.0). An undefined expression inserts `<undefined>` rather than making the
rule undefined.

```rego
message := $"User {input.username} has role {input.role}"
```

## Improved negation

`import future.keywords.not` opts into improved negation semantics (since
1.17.0). The compiler puts all expanded parts of a composite expression inside
the negated body, so an undefined input or nested call makes `not` succeed
rather than making the containing rule fail.

```rego
package example

import future.keywords.not

blocked(name) if startswith(name, "blocked-")

allow if {
	not blocked(input.user)
}
```

Use this import whenever a policy uses `not` and should receive the new
behavior. Unlike the earlier future-keyword imports under Rego v1, it is not a
no-op.

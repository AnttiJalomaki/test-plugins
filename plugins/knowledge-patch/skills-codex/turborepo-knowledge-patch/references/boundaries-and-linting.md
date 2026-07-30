# Package Boundaries and Environment Linting

## Boundary checks

Run the experimental boundary analyzer to find imports that escape a package
directory and imports of packages absent from that package's dependencies
(since 2.4.0):

```bash
turbo boundaries
```

Package rules, implicit dependencies, and TypeScript configuration path aliases
are understood by the analyzer (since 2.5.0). It also detects circular package
dependencies and analyzes dynamic imports (since 2.10.0).

## Circular-dependency diagnostics

Use cycle errors to identify actionable edge-removal choices (since 2.4.0).
Diagnostics list sets of dependency edges where removing any one complete set
breaks the Package Graph cycle, instead of listing only involved packages.

Do not confuse a Package Graph cycle with an invalid Task Graph (behavior
changed in 2.9.0). Turborepo no longer exits automatically for a package cycle.
Tasks without cyclic task dependencies can run; task relationships such as
`^build` still fail when they create a Task Graph cycle:

```json
{
  "tasks": {
    "simple-task": {},
    "build": { "dependsOn": ["^build"] }
  }
}
```

## ESLint Flat Config

Use Flat Config with `eslint-config-turbo` and `eslint-plugin-turbo` while
retaining compatibility with ESLint 8 (since 2.4.0). Spread the shareable
configuration into the exported array:

```js
export default [
  ...turboConfig,
];
```

For direct plugin use, register `turbo` in the flat-config `plugins` object:

```js
export default [
  {
    plugins: { turbo },
    rules: {
      "turbo/no-undeclared-env-vars": "error",
    },
  },
];
```

## Biome undeclared-environment checks

With Biome 2.3.10 or newer, allow project detection from repository
dependencies, then explicitly enable the nursery `noUndeclaredEnvVars` rule
(since 2.7.0). The rule catches environment-variable use that could otherwise
produce incorrect cache hits:

```json
{
  "linter": {
    "rules": {
      "nursery": {
        "noUndeclaredEnvVars": "error"
      }
    }
  }
}
```

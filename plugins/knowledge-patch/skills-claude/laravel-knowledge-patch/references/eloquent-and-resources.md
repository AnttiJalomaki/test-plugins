# Eloquent Models, Relationships, Casts, and Resources

## Contents

- [Model identity, state, and serialization](#model-identity-state-and-serialization)
- [Casts and attribute values](#casts-and-attribute-values)
- [Scopes and query behavior](#scopes-and-query-behavior)
- [Relationships and loading](#relationships-and-loading)
- [Creation and factories](#creation-and-factories)
- [API resources](#api-resources)

## Model identity, state, and serialization

- **UUIDv7 model IDs** `[12.0-upgrade]`: `HasUuids` generates ordered
  UUIDv7-compatible IDs. Use `HasVersion4Uuids` for ordered UUIDv4 strings;
  replace the removed `HasVersion7Uuids` trait with `HasUuids`.
- **Partition result type** `[12.0.0]`: `Eloquent\Collection::partition()`
  returns an `Illuminate\Support\Collection`, not an Eloquent collection.
- **Virtual properties in serialization** `[2025-05]`: model array and JSON
  output includes PHP virtual properties, including property hooks without
  backing storage.
- **Previous state after save** `[2025-05]`: `getPrevious()` returns values from
  immediately before the most recent save; `getChanges()` returns the new
  values.
- **Nested model construction during boot** `[13.0-upgrade]`: constructing the
  same model while its `boot` or trait `boot*` method is running throws
  `LogicException`; move construction outside the boot cycle.
- **Restored collection relations** `[13.0-upgrade]`: serialized and restored
  Eloquent collections, including those in queued jobs, restore eager-loaded
  relations.
- **Morph-map serialization** `[2026-02]`: serialized model identifiers use
  configured morph-map aliases rather than concrete class names.
## Casts and attribute values

- **Unicode JSON casts** `[2025-03]`: use the `json:unicode` cast to encode with
  `JSON_UNESCAPED_UNICODE`.
- **HTML string casts** `[2025-03]`: `AsHtmlString` returns HTML string values
  from model attributes.
- **Mapped collection casts** `[2025-04]`: use
  `AsCollection::of(Option::class)` to map decoded items into a declared type.
- **URI casts** `[2025-06]`: `AsUri` converts URI attributes into Laravel URI
  objects.
- **Discarded-change cast state** `[2025-06]`: discarding model changes clears
  cached cast values, so the next access reflects restored attributes.
- **Fluent casts** `[2025-06]`: `AsFluent` exposes a structured attribute as a
  `Fluent` value.
- **Mass-assigned value objects** `[2025-09]`: value-object casts work through
  `fill()`, `create()`, and related mass-assignment paths.
- **Castable enums** `[2025-09]`: enums may implement `Castable` to select their
  own Eloquent caster.
- **Binary casts** `[2026-01]`: use `AsBinary` for binary-valued model
  attributes.

## Scopes and query behavior

- **Attribute-declared scopes** `[2025-03]`: local named scopes may use a PHP
  attribute instead of only the `scopeXxx` method convention.
- **Attributes without conditions** `[2025-03]`: call
  `withAttributes($values, asConditions: false)` to set pending creation
  attributes without adding matching `where` clauses.
- **Void local scopes** `[2025-04]`: a local scope may mutate the supplied
  builder and return `void`.
- **Filtering attached models** `[2025-04]`: `whereAttachedTo($model)` filters
  through a many-to-many relation.
- **Scope attribute rename** `[2025-10]`: replace the earlier `NamedScope`
  attribute with `Scope`.
- **Retaining selected global scopes** `[2025-09]`:
  `withoutGlobalScopesExcept([...])` removes all global scopes except the
  listed security or tenancy scopes.
- **Reporting non-unique `sole()`** `[2026-06]`:
  `MultipleRecordsFoundException` from `sole()` is reported.

## Relationships and loading

- **One-of-many through relations** `[2025-03]`: `HasOneThrough` supports
  `CanBeOneOfMany`, including `latestOfMany()`.
- **Force-create many** `[2025-04]`: `forceCreateMany()` bypasses mass
  assignment for bulk related models; `forceCreateManyQuietly()` also
  suppresses events.
- **Automatic relationship loading** `[2025-04]`: enable globally with
  `Model::automaticallyEagerLoadRelationships()` or per model with
  `withRelationshipAutoloading()` to propagate lazy relationship loading
  across related models.
- **Nested loaded checks** `[2025-04]`: `relationLoaded()` accepts paths such as
  `posts.comments`.
- **Nested `loadMissing` arrays** `[2025-08]`: pass nested relationship arrays
  as an alternative to flattened paths.
- **SQLite morph exclusions** `[2025-11]`: SQLite supports
  `whereNotMorphedTo()`.
- **Associative eager-loading keys** `[2026-02]`: eager-loaded relation results
  retain associative keys instead of being reindexed.
- **Failing pivot transactions** `[2026-03-laravel-12]`: `BelongsToMany`
  provides `*OrFail` methods, such as `syncOrFail()`, for atomic pivot changes.
- **Polymorphic pivot inference** `[13.0-upgrade]`: inferred table names for
  custom polymorphic pivot models are pluralized. Set `$table` explicitly to
  keep an older singular table name.

## Creation and factories

- **Filled bulk inserts** `[2025-04]`: `Model::fillAndInsert()` merges model
  attributes before inserting multiple rows.
- **Factory parent expansion** `[2025-07]`: call
  `Factory::dontExpandRelationshipsByDefault()` to disable automatic expansion
  of parent relationship definitions globally.
- **Factory sequence context** `[2025-09]`: sequence callbacks receive pending
  `$attributes` and `$parent`.
- **Optional Faker dependency** `[2025-09]`: Laravel can run without
  `fakerphp/faker`; retain it only when factories or generated fake data need
  it.
- **Direct factory inserts** `[2025-11]`: `Factory::insert()` persists generated
  rows and handles hidden and array-cast attributes.
- **Lazy creation values** `[2026-02]`: `firstOrCreate()` and `createOrFirst()`
  accept a closure for values, evaluated only when insertion is needed.
- **Suppressing factory callbacks** `[2026-02]`: use
  `withoutAfterMaking()` or `withoutAfterCreating()` for one operation.
- **More lazy creation values** `[2026-04]`: `firstOrNew()` and
  `updateOrCreate()` accept a values closure.

## API resources

- **Convention-based conversion** `[2025-04]`: models and Eloquent collections
  expose `toResource()` and `toResourceCollection()` using conventional
  discovery.
- **Explicit resource attributes** `[2025-09]`: use `#[UseResource(...)]` and
  `#[UseResourceCollection(...)]` instead of convention where explicit mapping
  is preferable.
- **JSON:API resources** `[2026-01]`: `JsonApiResource` supports JSON:API
  serialization. Resource handling deduplicates circular references, and
  `ModelInspector` results include the model's `JsonResource`.

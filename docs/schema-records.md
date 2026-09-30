# Schema-Based Records

Making `ember-validated-form` work with WarpDrive's schema-based records. This
doc records why the boilerplate app cannot move to them yet, which parts of the
problem belong to this add-on, and how far the work has got. The work happens
on the `schema-records` branch.

Items marked **⚠ verify** are inferred from reading the code, not from running
it. Confirm before acting.

## Background

The boilerplate web app (`vite-upgrade` branch) runs every model as a legacy
`Model` class. Its store is built with `useLegacyStore` and registers no
schemas.

A schema for `page` was drafted there in `5ce4547` ("Replace ember-data with
warp-drive"). It was commented out in the same commit and never enabled. It was
deleted in `4e42815` ("Remove page schema"). Neither commit records why.

The draft used `withDefaults` from `@warp-drive/legacy/model/migration-support`.
That makes a legacy-mode schema record. Such a record still has the `Model`
methods the app calls: `save`, `rollbackAttributes`, `isNew`, `destroyRecord`
and `constructor.modelName`. So those are not blockers.

## How a schema record fails on unknown properties

In development builds, a schema record throws `No field named X on <type>` when
code reads a property that is not in its schema. Only a few names are exempt:
`then`, `length`, `nodeType`, `setInterval` and `document`.

A `Model` instance returns `undefined` for the same read. Code that checks
"does this object have X?" by reading X works on a `Model` and throws on a
schema record. That difference is behind the blockers below.

## Blockers in this add-on

### Reading `validate` and `error` off the model ⚠ verify

This add-on reads properties off `@model` that only a changeset has:

- `validated-form` reads `this.args.model.validate` in its constructor and in
  `submit` (`addon/components/validated-form.js`).
- `validated-button` reads `model.validate` in `click`
  (`addon/components/validated-button.js`).
- `validated-input` reads `error.<name>.validation` in its `errors` getter
  (`addon/components/validated-input.js`).

When `@model` is a changeset, these reads are fine. When `@model` is a schema
record passed directly, each one throws. The add-on should check for these
properties without reading them off a schema record.

## Blockers outside this add-on

These live in the boilerplate app or in `ember-changeset`. They are listed so
the whole path to schema records is in one place.

### Changeset probing

The `changeset` helper from `ember-changeset` reads `__changeset__` off the
object it is given, to check whether it is already a changeset. The check is
`isChangeset` in `validated-changeset`. Every admin form in the boilerplate
passes a record to `changeset`, so every admin form would throw.

This is confirmed. It is reported upstream as
[ember-changeset#710](https://github.com/adopted-ember-addons/ember-changeset/issues/710)
("Warp Drive Schema throws errors", opened 2026-06-14, no replies). No open pull
request or branch in `ember-changeset` or `validated-changeset` addresses it.
`ember-changeset`'s last release is v5.0.1 (May 2025), so an upstream fix may be
slow to land.

A likely fix: check `'__changeset__' in obj` before reading it. A schema
record's `has` trap returns `false` for unknown names instead of throwing.

### Key naming ⚠ verify

The API uses dasherized keys, such as `created-at`. The boilerplate uses
camelCase names, such as `createdAt`.

Today the legacy `JSONAPISerializer` converts between them. Its
`keyForAttribute` dasherizes names on the way out and matches them on the way
in. So the cache stores `createdAt`.

The draft schema also set `sourceKey: 'created-at'` on `createdAt` and
`updatedAt`. WarpDrive reads `sourceKey` as the key to look up in the cache, not
the key the API sends. The serializer has already renamed the key by then, so
the record reads a key that does not exist, and those fields come back empty.

One layer has to own the conversion:

- **Keep the serializer.** Drop `sourceKey` from the schemas. This is the
  smaller change while the app still uses legacy requests.
- **Drop the serializer.** Keep `sourceKey`, and move to request handlers that
  put the API's keys straight into the cache. This is the longer-term shape.

### Mirage models ⚠ verify

The boilerplate's `mirage/helpers/import-ember-data-models.ts` builds Mirage
models from an `import.meta.glob` of `app/models/`. A resource whose model file
is replaced by a schema drops out of Mirage, and its tests lose their fake API.

## Progress

- [ ] Reproduce the add-on's failures with a schema record in a test
- [ ] Make the add-on's property checks safe for schema records
- [ ] Make `ember-changeset` work with schema records
- [ ] Settle key naming for `page` in the boilerplate
- [ ] Update the boilerplate's Mirage model import
- [ ] Roll the approach out to the other boilerplate models

## Findings

Record what each step shows here, with the commit that shows it.

- **2026-09-30. No one else is working on this.** Upstream
  `adfinis/ember-validated-form` has no branch, pull request or issue about
  schema records. The nearest is issue #1213, "Convert to v2 addon". The
  changeset side is covered only by ember-changeset#710.

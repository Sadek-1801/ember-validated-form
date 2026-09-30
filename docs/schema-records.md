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

### Key and type naming

The API uses dasherized keys, such as `created-at`. The boilerplate uses
camelCase names, such as `createdAt`. One layer has to own the conversion:

- **Legacy requests** (`store.findRecord`, `store.query`) go through the legacy
  `JSONAPISerializer`. It renames `created-at` to `createdAt` before the data
  reaches the cache. A schema used this way must not set `sourceKey`.
- **Request builders** (`store.request(findRecord(...))`) skip the serializer.
  The cache holds the API's keys as sent, so the schema needs
  `sourceKey: 'created-at'`.

The boilerplate branch `warpdrive-page-schema-migration` (commit `bfa46b0`)
takes the second path, so its `sourceKey` is correct.

Skipping the serializer also skips its type conversion. Mirage sent plural
types (`pages`), and the schema is registered as `page`. `bfa46b0` fixed this
for Mirage with `typeKeyForModel` in the Mirage serializer.

⚠ verify: the real API must also send the singular type. If it sends `pages`,
the schema lookup fails in production even though the tests pass.

### Mirage models ⚠ verify

The boilerplate's `mirage/helpers/import-ember-data-models.ts` builds Mirage
models from an `import.meta.glob` of `app/models/`. A resource whose model file
is replaced by a schema drops out of Mirage, and its tests lose their fake API.

## Progress

- [ ] Reproduce the add-on's failures with a schema record in a test
- [ ] Make the add-on's property checks safe for schema records
- [x] Make `ember-changeset` work with schema records (uncommitted, branch
      `schema-record-support` in `validated-changeset` and `ember-changeset`)
- [ ] Commit, push to the forks, and open upstream pull requests linking
      ember-changeset#710
- [ ] Fix the boilerplate's `getModelName` on destroyed records
- [x] Settle key naming for `page` in the boilerplate (`bfa46b0`)
- [ ] Update the boilerplate's Mirage model import
- [ ] Roll the approach out to the other boilerplate models

## Findings

Record what each step shows here, with the commit that shows it.

- **2026-09-30. No one else is working on this.** Upstream
  `adfinis/ember-validated-form` has no branch, pull request or issue about
  schema records. The nearest is issue #1213, "Convert to v2 addon". The
  changeset side is covered only by ember-changeset#710.
- **2026-09-30. Reading pages already works on a boilerplate branch.**
  `warpdrive-page-schema-migration` (`bfa46b0`, 2026-06-09) registers
  `PageSchema` and a `date` transformation. It fetches pages with request
  builders and fixes the naming described above. It also stops `model-utils`
  from probing `.proxy` and `.constructor.modelName`, which throw on a schema
  record. Its commit message names the one thing left: the page forms break on
  `ember-changeset`. It suggests native mutation (`checkout` plus
  `updateRecord`) as the follow-up. `warpdrive-schema-migration` is the same
  commit with `vite-upgrade` merged in (2026-06-14). Neither branch was merged
  into `vite-upgrade`. The schema was deleted from `vite-upgrade` the next day.
- **2026-09-30. Test run on `warpdrive-page-schema-migration`.**
  - `ember test --path dist` crashed before any test ran. This add-on's
    `index.js` read `app.options` with no host app. It is fixed here in
    `index.js`. `vite-upgrade` works around it with a pnpm patch.
  - Admin Page: the index test passes. All four Create tests fail with
    `No field named __changeset__ on page`, thrown from `isChangeset`. This
    confirms the changeset blocker.
  - Admin Embed File: Destroy fails with `<model::embed-file:4> is not a
ReactiveResource or Model known to WarpDrive`. That is a regression from
    `bfa46b0`: `getModelName` now calls `recordIdentifierFor`, which throws on
    a destroyed record. The old `constructor.modelName` did not. ⚠ verify the
    exact caller.
- **2026-09-30. Changeset fixes, one layer at a time.** Each fix below moved
  the page Create tests one step further. Each has a failing test first, and
  each package's full suite passes with it.
  1. `validated-changeset`, branch `schema-record-support`:
     `isChangeset` checks `'__changeset__' in obj` before reading it.
  2. `validated-changeset` and `ember-changeset` (both on branch
     `schema-record-support`): the changeset Proxy turned every key into a
     string with `key.toString()`. Ember's `get` reads the Symbol
     `PROXY_CONTENT`, which became the string `"Symbol(PROXY_CONTENT)"` and was
     looked up on the record. Setting a Symbol key also recorded a bogus
     change. Symbol keys now go to the changeset object itself.
  3. `No field named unknownProperty on page`. When a field has no value yet,
     Ember's `get` checks the changeset for an `unknownProperty` hook. The
     changeset forwarded any key it did not know to its content, so the record
     was asked and threw. `set`, `save` and validation also read the content
     this way. Fixed at the root: `validated-changeset` now asks the content
     only through `contentHasKey` (`key in content`). `ember-changeset`
     overrides it to also allow `ObjectProxy` content and content with its own
     `unknownProperty`, since `in` cannot see keys those forward.
  4. `No field named relationshipFor on page`, in `ember-changeset`'s
     `safeGet` when `ember-data` is installed. Fixed with an `in` check.

  Result: all 8 Admin Page Create and Update tests pass. The one failure left
  is Destroy, from the `getModelName` regression above. Suites:
  `validated-changeset` 406 passed and 2 skipped (as before), `ember-changeset`
  210 passed and 1 todo (as before). Lint is clean in both.

- **2026-09-30. Boilerplate wired to the forks, full suite green except Admin
  Role.** On `warpdrive-page-schema-migration`, `pnpm-workspace.yaml` overrides
  `validated-changeset`, `ember-changeset` and `ember-validated-form` to the
  `Sadek-1801` forks and allowlists their `prepare` scripts. `getModelName`
  now reads `constructor.modelName` first, which legacy Models and legacy-mode
  schema records both answer even after they are unloaded, and falls back to
  `recordIdentifierFor`. `getInternalModelName` unwraps a `proxy` wrapper again,
  checking with `in` first. 83 of 83 tests pass, including all 11 Admin Page
  tests and the `FormTitle` tests that `bfa46b0` had broken.
  - Admin Role still fails, unrelated to this work: `LazyModel` passes
    `ember-concurrency-retryable`'s `retryable` option, which is not
    registered. The `lazy-model-tracked-function` branch replaces that task.

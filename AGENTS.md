# AGENTS.md

Primary instructions for **all** AI agents working on this project.

Documentation lives in two places. Read the relevant part before making
changes, and keep it up to date when a change makes it wrong.

- [`tests/dummy/app/templates/docs/`](tests/dummy/app/templates/docs/) — the
  user-facing docs: usage, customization, configuration and migration guides.
  They are published as the documentation site.
- [`docs/`](docs/) — work in progress: what is being changed, why, and how far
  it has got. Start with
  [`docs/schema-records.md`](docs/schema-records.md).
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — install, and the commands to lint,
  test and run the demo app
- [`README.md`](README.md) — what the add-on is and where its docs are

This project is an Ember add-on published to npm. It runs inside other people's
apps, so its components' arguments and yielded values are a public API. A
change that breaks them is a breaking change, even when this repo's tests still
pass.

Releases are cut automatically from commit messages, so every commit follows
Conventional Commits (`fix:`, `feat:`, `fix!:` for a breaking change). A commit
hook checks this.

When a change alters behaviour a doc describes, update that doc as part of the
change.

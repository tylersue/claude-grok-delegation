# Contributing

Contributions are welcome and maintainer-reviewed. The scope of this plugin is deliberately lean — a courier agent, a skill, and docs. Features that grow a runtime or add other vendors likely belong in a separate plugin.

## Bug fixes

Open a PR directly — bug fixes are welcome. Every bug-fix PR must include reproduction steps, or a grok-doc citation for any behavioral claim it relies on.

## Enhancements / features

Open an issue first, and wait for maintainer go-ahead before writing code. Unsolicited feature PRs may be closed.

## Quality bar

This repo's history is adversarially verified — behavioral claims were checked against grok's documentation or reproduced empirically. PRs that change behavioral claims must cite the doc line or show the repro.

## License

This project is MIT-licensed. Per GitHub's Terms of Service (inbound=outbound), content you submit in a PR is automatically licensed under the same MIT terms. There is no CLA or DCO.

## Housekeeping

Never include `.planning/` paths in PRs — that directory is maintainer-local planning and is excluded from the repo.

Every PR must pass the `manifest-sanity` and `claude-validate` checks before it can be merged.

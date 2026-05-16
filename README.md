# pr-audit-plugin

A Claude Code plugin marketplace. Currently hosts **pr-audit** — automated
pull request review.

## Install

In Claude Code:

```
/plugin marketplace add Crscristi28/pr-audit-plugin
/plugin install pr-audit@pr-audit-plugin
```

That adds this marketplace and installs the plugin. Run `/pr-audit` in any git
repository to use it.

## Plugins in this marketplace

### pr-audit

Automated pull request review for Claude Code. It commits your changes, opens a
PR on GitHub, reviews the code with specialized reviewers running in parallel,
and posts findings as inline comments — with one-click fix suggestions where
the fix is simple.

- **Security** — injection, hardcoded secrets, missing authorization, unsafe input
- **Correctness** — logic bugs, null crashes, missing error handling, race conditions
- **Performance** — N+1 queries, unbounded loops, blocking calls
- **Plus** — accessibility, API contracts, architecture, and test coverage on
  larger or higher-risk changes

After the review it prints a copyable handoff brief and offers to apply the
fixes in the same session.

See [`pr-audit/README.md`](./pr-audit/README.md) for full usage, configuration,
and the optional custom-rules file.

## Repository layout

```
pr-audit-plugin/
├── .claude-plugin/
│   └── marketplace.json     # marketplace catalog
├── .github/workflows/       # plugin + version validation on every push
└── pr-audit/                # the plugin (skills, agents, references)
```

## Versioning

Each plugin carries an explicit `version` in its `plugin.json`. Users only
receive an update when that version is bumped. Plugin changes that ship without
a version bump are rejected by the `Check Version Bump` workflow.

## License

[MIT](./LICENSE)

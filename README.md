# dev-gatekeepers

**The development environment for the [Dynamic Agents](https://github.com/dynamicagents)
gatekeeper side.**

The gatekeeper and the wire contract it speaks, pinned together at a known-good
combination, plus the agent skills they share — so a laptop, a cloud session, or a machine
that has never seen this project all reach the same place from one clone.

```bash
git clone --recurse-submodules git@github.com:dynamicagents/dev-gatekeepers.git
cd dev-gatekeepers
npm install && npm run bootstrap
```

Then launch your agent from this directory.

| repo | |
| ---- | --- |
| [`g2a-protocol`](https://github.com/dynamicagents/g2a-protocol) | the gatekeeper↔agent wire contract |
| [`slack-gatekeeper`](https://github.com/dynamicagents/slack-gatekeeper) | the Slack-anchored gatekeeper |

`slack-gatekeeper` depends on the contract and imports none of the agent runtime — the
two must never share one. The agent side is a separate workspace,
[`dev-agents`](https://github.com/dynamicagents/dev-agents), which pins `g2a-protocol`
too.

[`AGENTS.md`](AGENTS.md) is the working guide. It is symlinked to `CLAUDE.md`, because
Claude Code reads that name.

---

## Skills

Skills live in `.agents/skills/`, installed with the [`skills`](https://skills.sh) CLI and
recorded in `skills-lock.json`. Claude Code reads `.claude/skills/`, which holds one
symlink per skill and is generated:

```bash
npx skills add <pack>    # writes to .agents/skills/ only
npm run skills           # link it where Claude Code will look
```

`npm run check` fails when those two drift, which is the point: the CLI knows nothing
about `.claude/skills/`, and a skill that never got linked fails silently — nothing
announces a skill it did not find.

## Commands

| | |
| --- | --- |
| `npm run bootstrap` | submodules on a branch, `node_modules`, and skill links. Safe to re-run. |
| `npm run check` | skill links and submodule structure are intact |
| `npm run skills` | re-link `.claude/skills/` after adding or removing a skill |
| `npm run sync` | put every submodule on its branch and fast-forward it |
| `node scripts/submodules.mjs --pushed` | every pinned commit is on a remote — needs network, so it is not in `check` |

## The submodule pointers

A pointer is a known-good combination, not a mirror. Every commit in a subrepo makes its
pointer here stale, and that is the design — bump one deliberately rather than on every
commit. `npm run check` never fails on staleness; it checks only that the submodules would
survive a clone.

`bootstrap` and `sync` leave the submodules on `main`. Plain `git submodule update` checks
out the recorded *commit* and so detaches HEAD; these check out the branch and fast-forward
it, and skip any submodule that is dirty or on a feature branch.

## License

Apache 2.0. Each submodule carries its own copy.

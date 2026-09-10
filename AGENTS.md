# AGENTS.md — the Dynamic Agents gatekeeper workspace

The gatekeeper half of the system: the service agents talk *through*, and the wire
contract that keeps it and the agent runtime from drifting apart.

| repo | what it is |
| ---- | ---------- |
| [`g2a-protocol`](g2a-protocol/AGENTS.md) | the gatekeeper↔agent wire contract: constants and pure functions, no dependencies |
| [`slack-gatekeeper`](slack-gatekeeper/AGENTS.md) | the Slack-anchored gatekeeper — routing, registration, the A2A crossing |

`slack-gatekeeper` depends on `@dynamicagents/g2a-protocol` from the registry and
**imports none of the agent runtime**. That is the whole point of the split: the
gatekeeper and `@dynamicagents/core` must never share a runtime, and they can only
both depend on the contract while depending on it costs nothing. Every rule in
`g2a-protocol` follows from that one fact.

The agent side lives in a separate workspace (`dev-agents`: `core`, `plugins`,
`starter`). `g2a-protocol` is a submodule of both, pinned independently — each records
what was green for its own side, so the two pins may legitimately differ.

---

## Where a change goes

| You are changing… | It goes in |
| ----------------- | ---------- |
| a value both sides must spell identically | `g2a-protocol` |
| anything that signs, verifies, fetches, or reads config | the consumer, never the protocol |
| Slack routing, registration, the A2A crossing | `slack-gatekeeper` |
| the agent runtime's half of verification | `core`, in the other workspace |

The test for `g2a-protocol` is two questions, and both must point that way: **must two
repos spell it identically**, and **can they both import it from somewhere that already
owns it?** If the second is yes, it belongs there and not here.

The card and endpoint checks stay in the gatekeeper; the verification chain stays in
core. They are not the same check and must not be made to look like one.

---

## A change to `g2a-protocol` is a change to the wire

The two sides do not interoperate across it in either direction — a mismatched claim
name is a total outage, not a degraded mode. So:

- **Bump the minor**, with `npm version minor`. On 0.x, npm reads `^0.1.0` as `0.1.x`,
  so a minor is a hard break nobody picks up by accident: someone has to type the new
  range and notice why.
- **Ship both consumers in the same release.** There is no ordering where one goes first
  safely.
- **Update `src/claims.spec.ts` by hand.** Those assertions are literals, not references,
  deliberately — importing a constant and asserting it equals itself tests nothing.
  Typing the string twice is the point; the second time is when the cost registers.

A version bump reaching `main` is what ships it: on the first green Test run for a commit
carrying that version, `release.yml` publishes over OIDC and only then cuts the tag.

---

## Working here

```bash
npm run bootstrap    # submodules on a branch, node_modules, skill links. Run this first.
npm run check        # skill links and submodule structure are intact
npm run skills       # re-link .claude/skills after adding or removing a skill
npm run sync         # put every submodule on its branch and fast-forward it
```

**Verify with `npm run check` in the repo you touched, not `npm test`.** Vitest
transpiles specs without typechecking them, so a type error passes a green suite. Where
`wrangler.jsonc` bindings or compat settings moved, run `npm run types` first and commit
the regenerated `worker-configuration.d.ts` — it is generated but committed, and
`check` fails when it goes stale.

**`npm run cf`** in `slack-gatekeeper` is a thin Cloudflare API proxy for inspecting the
deployed Worker — logs, Workflow instances, AI Gateway calls. It reads credentials from a
gitignored `.cf.env` and redacts them from all output, so the token never lands in shell
history or in an agent's context. Prefer it to pasting a token anywhere.

### The submodule pointers

A pointer is a **known-good combination**, not a mirror of each submodule's `main`. Every
commit in a subrepo makes its pointer stale, and that is the design: bump one
deliberately rather than on every commit. `npm run sync` moves the checkouts and reports
which are ahead of their pin.

`bootstrap` and `sync` both leave the submodules **on `main`**, not on a detached HEAD.
Plain `git submodule update` — and `git clone --recurse-submodules` — check out the
recorded *commit*, and a commit is not a branch, so they detach you and the next commit
you write goes somewhere no branch can see. Neither touches a submodule with uncommitted
changes, or one you have checked out onto a feature branch.

---

## Pull requests

**Every Copilot review comment ends resolved.** Copilot reviews the PRs in these repos and
in this workspace, and a PR is not ready to hand over while one of its threads is open.

Resolved does not mean accepted. Read each comment against the code first — Copilot is
often right, and sometimes confidently wrong about an API it has not read or behaviour it
cannot run. Then fix it and reply naming the commit, or reply saying why not, and resolve
the thread either way, so it records what happened. The review summary can raise points
that are not threads; read those too.

Resolving a thread is GraphQL-only:

```bash
# The open threads, with the id both mutations take. `--paginate` walks every page of
# threads — without it, a long review can look clean.
gh api graphql --paginate -F o=dynamicagents -F r=<repo> -F n=<pr> -f query='
  query($o:String!,$r:String!,$n:Int!,$endCursor:String){repository(owner:$o,name:$r){pullRequest(number:$n){
    reviewThreads(first:100,after:$endCursor){pageInfo{hasNextPage endCursor} nodes{id isResolved path line
      comments(first:1){nodes{author{login} body}}}}}}}' \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved|not)'

gh api graphql -f id=<thread> -f body='<the fix and its commit, or why not>' -f query='
  mutation($id:ID!,$body:String!){addPullRequestReviewThreadReply(
    input:{pullRequestReviewThreadId:$id,body:$body}){comment{url}}}'

gh api graphql -f id=<thread> -f query='
  mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}'
```

### Knowing Copilot has finished

The review is requested automatically, and exactly when is worth knowing:

- **Opening a PR ready for review into `main` requests it** — the `protect-main` ruleset
  asks, and it covers no other branch.
- **A draft gets no request** while it is a draft.
- **A Dependabot PR gets no request**, ever.
- **A push requests nothing.** The review of an earlier commit is the last one a PR gets
  unless someone asks again — fixing Copilot's comments does not bring it back.

Finished is an event on the PR's timeline, not an absence of comments: a review can finish
having left none. Copilot's latest review event answers it:

```bash
gh api repos/dynamicagents/<repo>/issues/<pr>/timeline --paginate --jq '
  .[] | select((.event=="review_requested" and .requested_reviewer.login=="Copilot")
            or (.event=="reviewed" and .user.login=="Copilot")) | .event' | tail -n 1
```

`reviewed` means done. `review_requested` means still working. No output means it was
never asked, and waiting will not change that — a draft or a Dependabot PR, usually.
Reviews here have landed from under two to about seven minutes after the request, so poll
no faster than every thirty seconds, and stop after fifteen minutes rather than wait on a
review that never started.

---

## Skills

Skills live in `.agents/skills/`. Claude Code reads `.claude/skills/`, which holds a
symlink per skill and is **generated** — `npm run skills` after any change, and
`npm run check` fails when the two drift.

```bash
npx skills add <pack>    # writes to .agents/skills/ only
npm run skills           # then link it where Claude Code will find it
```

The CLI knows nothing about `.claude/skills/`, and a skill that never got linked fails
silently: nothing announces a skill it did not find. That is the whole reason the check
exists.

**The Cloudflare skills are the reference for the platform — use them instead of
recalling it.** `cloudflare`, `wrangler`, `workers-best-practices`, `durable-objects`,
`agents-sdk`, `cloudflare-email-service` and `sandbox-sdk` are installed here and retrieve
current documentation rather than relying on training data, which for Workers APIs and
limits goes stale fast. Reach for them before answering from memory about bindings, limits,
compatibility dates, or Durable Object and Workflow rules. Neither this file nor a repo's
own `AGENTS.md` should grow a third copy of what they already say.

---

## Running Claude here

Launch from this directory. Skill discovery walks parent directories only as far as the
repository root, and each submodule is its own repository root — so a session started
inside `slack-gatekeeper/` may not see the workspace skills, while one started here sees
both these and, on demand, each repo's own `AGENTS.md` as you work in it.

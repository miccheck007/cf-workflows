# cf-workflows

The one workflow every Worker repo runs: [`worker.yml`](.github/workflows/worker.yml).
Worker repos only carry the caller, [`caller/cloudflare.yml`](caller/cloudflare.yml), copied as is
to `.github/workflows/cloudflare.yml`. Onboarding, manifests and the broker are in
[cf-platform](https://github.com/miccheck007/cf-platform).

Worker repos hold **no Cloudflare credential**. Each run asks the platform's broker for a token
scoped to one Worker and valid for 15 minutes, proving who it is with GitHub's OIDC token; the
broker only answers this workflow (`@main`), for a repo with a manifest, and for production only on
`main`'s current head merged from a reviewed pull request.

| Event in the Worker repo | Jobs | Result |
|---|---|---|
| push to `main` | `build` → `deploy` (environment `production`) → `report` | `wrangler deploy` with the commit as message/tag, plus the `WORKER_` secrets |
| pull request opened / updated | `build` → `preview` (environment `preview`) → `report` | the Worker's [Preview](https://developers.cloudflare.com/workers/previews/) `pr-<n>` (`wrangler preview`), listed under the Worker in the dashboard; URL on the PR's deployment |
| pull request closed | `build` → `preview-delete` (no environment) → `report` | the Preview is deleted and the PR's preview deployments are marked inactive |

**What the PR shows.** The `report` job is the only one that comments, and it holds no token and
runs no repo code:
- one comment per PR, edited on every push: the Preview's URL (latest push) and the fixed URL of
  the commit it was built from; on close, that the Preview was deleted;
- after the merge: *Deployed to production* with the version, or *Not deployed to production*;
- on any failure: which step failed (the build, the broker, `wrangler`, …) or the broker's reason,
  and what to do next.

Nothing retries a failure later: it's reported where it happened. Fix the cause, then use
*Re-run failed jobs* on the run (production: while `main` hasn't moved) or push a fix. The only
automatic retry is the broker call itself, twice, on timeouts, 5xx or refused connections; a
refusal is never retried.

The PR page itself only shows the PR branch's deployments, which are previews. The caller grants
`pull-requests: write` (the `report` job) and `deployments: write` (marking closed PRs' preview
deployments inactive); the job that runs the repo's code keeps only `contents: read`. A shared
workflow can't use more than its caller grants, so a caller missing one makes every run fail to
start: update callers before merging a change that needs more.

`build` runs the repo's code (install, `wrangler deploy --dry-run`) and holds nothing: no secrets,
no permission to request an OIDC token. `deploy`/`preview` run no repo code: a pinned Wrangler on
build's bundle with a config written from `wrangler.jsonc` minus anything that runs code or picks
the target (`build`, `name`, `account_id`, `routes`, `env`, `containers`, …). Static assets
(`assets.directory`) are deployed from the repo tree; symlinks are refused. `wrangler.toml` isn't
supported.

KV namespace IDs come from the broker (production's, or the `_PREVIEW` copies for previews) and are
written into every upload. A Preview inherits nothing from production and takes its bindings only
from the `previews` block, which the workflow writes: the repo's bindings under the same names, the
`_PREVIEW` KV IDs, and `previews.vars` if the repo sets it (top-level `vars` otherwise). It runs
with `--ignore-base-config`, so settings edited in the dashboard don't apply. Previews support
vars, KV, services, AI, Analytics Engine, browser, version metadata, static assets and the Worker's
own Durable Objects (a fresh namespace per Preview); not yet D1, R2, Queues or Workflows. Every job
revokes its token when done.

**Secrets**: GitHub secrets named `WORKER_<NAME>` become the Worker secret `<NAME>` on the next
deploy (`WORKER_API_KEY` -> `API_KEY`), and deleting one deletes it from the Worker. GitHub has no
event for a changed secret, so to ship one, merge a PR (an empty commit is fine) or *Run workflow*
in the repo. Put them in the
repo's `production` environment (main only); `preview` works the same for previews. Nothing else
ships: the caller passes `secrets: inherit` (environment secrets arrive empty otherwise), so every
repo and org secret reaches the workflow, and the prefix keeps them out of the Worker. A
repository-level `WORKER_` secret ships to production *and* previews; the environment's value wins
when both exist. The broker returns the names the run must see; a run missing one stops before
shipping or deleting anything. A Preview keeps its secrets from one push to the next, so a secret
deleted from `preview` stays in open Previews until the PR closes.

This repo holds no secrets and no state. It is still security-critical: the broker trusts
whatever runs as `worker.yml@main`, so `main` only changes by reviewed PRs, and every merge reaches
all Worker repos on their next run. There is no version line yet: a breaking change needs one
(a new branch, trusted by the broker alongside `main` while callers move).

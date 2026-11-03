# cf-workflows

The one workflow every Worker repo runs: [`worker.yml`](.github/workflows/worker.yml).
Worker repos only carry the caller, [`caller/cloudflare.yml`](caller/cloudflare.yml), copied as is
to `.github/workflows/cloudflare.yml`. Onboarding, manifests and tokens are in
[cf-platform](https://github.com/miccheck007/cf-platform).

| Event in the Worker repo | Job | Environment | Result |
|---|---|---|---|
| push to `main`, or *Run workflow* | `deploy` | `production` (main only) | `wrangler deploy` with the commit as message/tag, plus the `WORKER_` secrets |
| pull request opened / updated | `preview` | `preview` | version on `<name>-preview`, alias `pr-<n>`; URL on the PR's deployment |
| pull request closed | `preview` | `preview` | the alias answers 410 (aliases can't be deleted) |

**Secrets**: GitHub secrets named `WORKER_<NAME>` become the Worker secret `<NAME>` on the next
deploy (`WORKER_API_KEY` -> `API_KEY`), and deleting one deletes it from the Worker. Put them in the
repo's `production` environment; `preview` works the same for previews. Nothing else ships: the
caller passes `secrets: inherit` (environment secrets arrive empty otherwise), so every repo and org
secret reaches the workflow, and the prefix is what keeps them out of the Worker. A repository-level
`WORKER_` secret ships to production *and* previews; the environment's value wins when both exist.

This repo holds no secrets and no state. It is still security-critical: whatever `worker.yml` runs,
runs with every Worker's deploy token. Changes go through review; callers pin `@v1`, which only
moves when a release is tagged.

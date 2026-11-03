# cf-workflows

The one workflow every Worker repo runs: [`worker.yml`](.github/workflows/worker.yml).
Worker repos only carry the caller, [`caller/cloudflare.yml`](caller/cloudflare.yml), copied as is
to `.github/workflows/cloudflare.yml`. Onboarding, manifests and tokens are in
[cf-platform](https://github.com/miccheck007/cf-platform).

| Event in the Worker repo | Job | Environment | Result |
|---|---|---|---|
| push to `main`, or *Run workflow* | `deploy` | `production` (main only) | `wrangler deploy` with the commit as message/tag, plus the environment's secrets |
| pull request opened / updated | `preview` | `preview` | version on `<name>-preview`, alias `pr-<n>`; URL on the PR's deployment |
| pull request closed | `preview` | `preview` | the alias answers 410 (aliases can't be deleted) |

**Secrets**: every secret in the Worker repo's `production` environment becomes a Worker secret of
the same name on the next deploy; deleting one there deletes it from the Worker. `preview` works the
same for previews. Names starting with `CFP_` are the platform's and never shipped.

This repo holds no secrets and no state. It is still security-critical: whatever `worker.yml` runs,
runs with every Worker's deploy token. Changes go through review; callers pin `@v1`, which only
moves when a release is tagged.

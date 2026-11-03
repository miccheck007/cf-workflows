# cf-workflows

The one workflow every Worker repo runs: [`worker.yml`](.github/workflows/worker.yml).
Worker repos only carry the caller, [`caller/cloudflare.yml`](caller/cloudflare.yml), copied as is
to `.github/workflows/cloudflare.yml`. Onboarding, manifests and the broker are in
[cf-platform](https://github.com/miccheck007/cf-platform).

Worker repos hold **no Cloudflare credential**. Each run asks the platform's broker for a token
scoped to one Worker and valid for 15 minutes, proving who it is with GitHub's OIDC token; the
broker only answers this workflow (`@v2`), for a repo with a manifest, and for production only on
`main`'s current head merged from a reviewed pull request.

| Event in the Worker repo | Jobs | Result |
|---|---|---|
| push to `main`, or *Run workflow* | `build` → `deploy` (environment `production`) | `wrangler deploy` with the commit as message/tag, plus the `WORKER_` secrets |
| pull request opened / updated | `build` → `preview` (environment `preview`) | version on `<name>-preview`, alias `pr-<n>`; URL on the PR's deployment |
| pull request closed | `build` → `preview` | the alias answers 410 (aliases can't be deleted) |

`build` runs the repo's code (install, `wrangler deploy --dry-run`) and holds nothing: no secrets,
no permission to request an OIDC token. `deploy`/`preview` run no repo code: a pinned Wrangler on
build's bundle with a config written from `wrangler.jsonc` minus anything that runs code or picks
the target (`build`, `name`, `account_id`, `routes`, `env`, `containers`, …). Static assets
(`assets.directory`) are deployed from the repo tree; symlinks are refused. `wrangler.toml` isn't
supported.

**Secrets**: GitHub secrets named `WORKER_<NAME>` become the Worker secret `<NAME>` on the next
deploy (`WORKER_API_KEY` -> `API_KEY`), and deleting one deletes it from the Worker. Put them in the
repo's `production` environment (main only); `preview` works the same for previews. Nothing else
ships: the caller passes `secrets: inherit` (environment secrets arrive empty otherwise), so every
repo and org secret reaches the workflow, and the prefix keeps them out of the Worker. A
repository-level `WORKER_` secret ships to production *and* previews; the environment's value wins
when both exist. The broker returns the names the run must see; a run missing one stops before
shipping or deleting anything.

This repo holds no secrets and no state. It is still security-critical: the broker trusts
whatever runs as `worker.yml@v2`. `v2` is a branch that only moves by reviewed fast-forward.

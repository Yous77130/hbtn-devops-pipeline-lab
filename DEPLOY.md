# Deploy runbook

This is the operational runbook for the staging deployment of
`hbtn-devops-pipeline-lab`. It covers how a release reaches staging,
how to verify it, how to roll back, and how to clean up the lab's
temporary resources.

## Target

Staging runs on [Render](https://render.com) as a Web Service built from
the image published to GHCR (`ghcr.io/yous77130/hbtn-devops-pipeline-lab`).
A separate, disposable Render PostgreSQL database (`hbtn-pipeline-db`)
backs the service.

The GHCR package is public, so Render pulls the image without a
registry credential.

## Trigger

The `deploy` job in `.github/workflows/ci.yml` runs after `test` and
`build` succeed, and only for a `push` to `main` (never for pull
requests). It calls the Render deploy API for the existing service:

```bash
curl -f -X POST "https://api.render.com/v1/services/$RENDER_SERVICE_ID/deploys" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"clearCache":"do_not_clear"}'
```

`RENDER_API_KEY` and `RENDER_SERVICE_ID` are stored as GitHub repository
secrets; `STAGING_URL` is a repository variable
(`https://hbtn-devops-pipeline-lab.onrender.com`, no trailing slash).
None of these values are written in the workflow file, the repository,
or the workflow logs.

Render always redeploys the `:latest` tag of the image, which the
`build` job just pushed, so the deploy always ships the commit that
triggered the pipeline.

## Database configuration

The web service's `DATABASE_URL` environment variable points to the
Postgres instance's **internal** connection string (Render's private
network, not the external one). It is set directly in the Render
dashboard (Environment tab of the web service), never in the workflow
or the repository.

On startup, `src/server.js` calls `runMigrationsWithRetry()`, which
applies any pending SQL migrations from `src/db/migrations/` before the
server starts accepting the database-backed routes. No separate
migration step is needed in the pipeline.

## Verification

After triggering the deploy, the `deploy` job polls both endpoints in a
bounded retry loop (30 attempts, 10 seconds apart, 5 minutes total) and
fails the job if either check has not returned HTTP 200 by the last
attempt:

- `GET $STAGING_URL/health` — process liveness only.
- `GET $STAGING_URL/items` — confirms the deployed API can query
  PostgreSQL (proves the database connection and migrations worked).

The same two requests can be run independently, outside the pipeline,
from any machine:

```bash
curl -s -o /dev/null -w "HTTP /health: %{http_code}\n" https://hbtn-devops-pipeline-lab.onrender.com/health
curl -s -o /dev/null -w "HTTP /items: %{http_code}\n"  https://hbtn-devops-pipeline-lab.onrender.com/items
```

Both must return `200`.

## Rollback

Every image the `build` job publishes is tagged with the immutable
commit SHA (`ghcr.io/yous77130/hbtn-devops-pipeline-lab:<sha>`), in
addition to the moving `:latest` tag. To roll back:

1. Identify the last known-good commit SHA (from `git log` or the
   Actions run history).
2. In the Render dashboard, open the web service's **Settings** →
   **Image**, and change the image reference to the SHA-tagged image
   (`ghcr.io/yous77130/hbtn-devops-pipeline-lab:<good-sha>`) instead of
   `:latest`.
3. Trigger a manual deploy from the Render dashboard, or re-run the
   `Trigger Render deploy` API call.
4. Re-run the two verification requests above to confirm the rollback
   is healthy.
5. Once the underlying issue is fixed on `main`, redeploy `:latest`
   (or point the service back at it) so future pushes resume normal
   flow.

## Credential cleanup

When the lab is no longer needed:

1. In Render, delete the web service `hbtn-devops-pipeline-lab` and
   the database `hbtn-pipeline-db` (Settings → Delete, on each).
2. Revoke the Render API key used for `RENDER_API_KEY`
   (Render → Account Settings → API Keys).
3. Delete the GitHub repository secrets `RENDER_API_KEY` and
   `RENDER_SERVICE_ID`, and the repository variable `STAGING_URL`
   (repo Settings → Secrets and variables → Actions).
4. Leave the GHCR package (`hbtn-devops-pipeline-lab`) public, since it
   contains no sensitive data and was published on purpose; delete it
   only if the whole lab repository is being retired.

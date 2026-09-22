# Deployment Runbook

## Deployment target

The staging environment uses **Render**.

The application is deployed as a Web Service from the public GHCR image:

```
ghcr.io/crypoune/hbtn-devops-pipeline-lab:latest
```

The Web Service and PostgreSQL database are deployed in the same Render region so the application can use the database's internal connection string.

## Database configuration

- The staging Web Service uses the `DATABASE_URL` environment variable.
- `DATABASE_URL` is configured directly in the Render Web Service using the PostgreSQL database's Internal Database URL.
- The database credentials are stored by Render and are not committed to the repository, workflow, or runbook.
- The PostgreSQL database is disposable and is intended only for the staging environment.

## Deployment trigger

Deployments are triggered by pushes to the `main` branch.

The GitHub Actions workflow runs the following release chain:

```
test → build → deploy
```

The `deploy` job requires the `build` job to succeed and triggers a Render deployment through the Render API.

The Render API credentials are stored as GitHub repository secrets:

- `RENDER_API_KEY`
- `RENDER_SERVICE_ID`

The staging URL is stored as the GitHub repository variable:

- `STAGING_URL`

No credential values are stored in this repository.

## Staging verification

After triggering the Render deployment, the workflow performs bounded health checks against both staging endpoints.

### Application liveness

**Request:**

```
GET ${STAGING_URL}/health
```

**Expected result:** `HTTP 200`

The response should indicate that the application is healthy.

### Database-backed API

**Request:**

```
GET ${STAGING_URL}/items
```

**Expected result:** `HTTP 200`

This endpoint verifies that the deployed application can communicate with PostgreSQL and query the database.

---

The workflow performs a maximum of **12 verification attempts**, waiting between attempts when necessary.

The deploy job succeeds only when both `/health` and `/items` return `HTTP 200`.

If both endpoints do not return HTTP 200 within the allowed attempts, the verification step exits with a non-zero status and the deploy job fails.

## Rollback

Docker images are also published with an immutable commit-SHA tag.

For a rollback, identify the commit SHA corresponding to the known-good release and deploy the corresponding image instead of `latest`:

```
ghcr.io/crypoune/hbtn-devops-pipeline-lab:<COMMIT_SHA>
```

Using the commit-SHA tag avoids relying on the mutable `latest` tag and allows a specific published image to be selected.

## Cleanup

When the lab is no longer needed:

1. Remove the Render staging Web Service.
2. Remove the disposable Render PostgreSQL database.
3. Revoke the temporary Render API key used by GitHub Actions.
4. Remove the `RENDER_API_KEY` repository secret.
5. Remove the `RENDER_SERVICE_ID` repository secret.
6. Remove the `STAGING_URL` repository variable.
7. Keep the GHCR package visibility consistent with the intended cleanup policy.

No credentials should remain in the repository, workflow files, runbook, or logs.

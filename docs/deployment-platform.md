# IIS-Design Deployment Platform Runbook

Operational documentation for the IIS-Design self-hosted deployment platform.

## Overview

The platform provides a push-to-deploy workflow for IIS Design applications.

```text
Developer
   |
   | git push
   v
GitHub
   |
   v
ipsos-devops-01
GitHub Actions Runner
Podman build + smoke test
   |
   v
GitHub Container Registry
Immutable SHA image
   |
   v
ipsos-apps-01
Docker Compose
   |
   v
design-proxy
   |
   v
design.go.ar
```

Developers only need GitHub access. They do not need VPN access, SSH access to production, Docker, Podman or deployment credentials.

## Components

### GitHub

Organization:

```text
IIS-Design
```

Shared workflow repository:

```text
IIS-Design/.github
```

Reusable deployment workflow:

```text
.github/workflows/container-deploy.yml
```

### Build runner

Host:

```text
ipsos-devops-01
```

Responsibilities:

- Receive GitHub Actions jobs
- Checkout application source
- Build containers with rootless Podman
- Run local smoke tests
- Publish validated images to GHCR
- Trigger production deployments

The runner is not a production application server.

### Production

Host:

```text
ipsos-apps-01
```

Responsibilities:

- Run application containers with Docker Compose
- Pull immutable images from GHCR
- Pin deployed applications by image digest
- Validate application health
- Automatically roll back failed deployments

Applications are connected to the external Docker network:

```text
ipsos-edge
```

Traffic is routed through:

```text
design-proxy
```

and exposed through:

```text
https://design.go.ar/
```

## Deployment lifecycle

A normal deployment follows this sequence:

1. A developer pushes a commit to GitHub.
2. GitHub assigns the job to `ipsos-devops-01`.
3. Podman builds the application container.
4. The container starts locally on a temporary loopback port.
5. The configured smoke-test URL is checked.
6. The image is published to GHCR using the Git commit SHA.
7. The runner requests a production deployment.
8. Production pulls the SHA image.
9. The image is resolved to an immutable registry digest.
10. Docker Compose is updated to use that digest.
11. The application is started.
12. The production health check is validated.
13. If the health check fails, the previous digest is restored automatically.

A green GitHub Actions run means the complete deployment process succeeded.

## Application workflow

Application repositories contain only a small caller workflow.

Example for an application using `main`:

```yaml
name: CI

on:
  push:

jobs:
  deploy:
    permissions:
      contents: read
      packages: write

    uses: IIS-Design/.github/.github/workflows/container-deploy.yml@main

    with:
      app: example-app
      container-port: 80
      smoke-path: /example-app/_health
```

For an application deploying from `master`:

```yaml
    with:
      app: example-app
      container-port: 80
      smoke-path: /example-app/_health
      deploy-branch: master
```

Pushes from other branches still build, smoke-test and publish an immutable SHA image, but production deployment is limited to the configured deployment branch.

## Production application configuration

Each application has a directory under:

```text
/srv/apps/<app>
```

The normal Compose file defines the service and connects it to `ipsos-edge`.

Example:

```yaml
services:
  example-app:
    image: ghcr.io/iis-design/example-app:main
    container_name: example-app
    restart: unless-stopped
    networks:
      - ipsos-edge

networks:
  ipsos-edge:
    external: true
```

Each application also has a root-managed deployment configuration:

```text
/srv/apps/<app>/.iis-deploy
```

Example:

```text
IMAGE_REPO=ghcr.io/iis-design/example-app
SERVICE=example-app
HEALTH_URL=http://example-app/example-app/_health
```

Production deployment writes:

```text
/srv/apps/<app>/compose.override.yaml
```

The override pins production to the exact image digest:

```yaml
services:
  example-app:
    image: ghcr.io/iis-design/example-app@sha256:...
```

The digest override is the authoritative image running in production.

## Adding a new application

Before onboarding an application:

- The application must have a working Containerfile.
- The application must start without interactive configuration.
- A stable local smoke-test URL must exist.
- A stable production health URL must exist.
- The production service must be reachable from `design-proxy`.
- The application must join the `ipsos-edge` network.

Then:

1. Create `/srv/apps/<app>` on production.
2. Add the production `compose.yaml`.
3. Add the root-managed `.iis-deploy` configuration.
4. Add the route in `design-proxy`.
5. Add the reusable workflow caller to the application repository.
6. Push the application branch.
7. Confirm the GitHub Actions run is green.
8. Confirm `compose.override.yaml` contains an immutable digest.
9. Confirm the production health URL responds successfully.
10. Confirm the public `design.go.ar` route works.

## Health checks

There are two health validation stages.

### Local smoke test

Executed on `ipsos-devops-01`.

The newly built image is started temporarily with Podman and tested before it can be published or deployed.

### Production health check

Executed after Docker Compose starts the new image.

The health URL is configured per application in `.iis-deploy`.

A failed production health check triggers automatic rollback.

## Automatic rollback

Before replacing an application, the deployment engine records the current image digest.

If the new application does not become healthy within the configured timeout:

1. Container status is printed.
2. Recent application logs are printed.
3. The previous image digest is restored.
4. Docker Compose recreates the service.
5. The previous application health is validated.

The GitHub Actions run remains red even when rollback succeeds. This is intentional: the requested release failed, while production was safely restored.

Automatic rollback was validated with a deliberate failed deployment before the platform was considered production-ready.

## Checking a deployed application

On `ipsos-apps-01`:

```bash
cd /srv/apps/<app>
docker compose ps
```

Check the pinned image:

```bash
cat compose.override.yaml
```

Verify the effective Compose configuration:

```bash
docker compose config | grep 'image:'
```

Check application logs:

```bash
docker compose logs --tail=100
```

## Checking GitHub Actions

From a repository with GitHub CLI access:

```bash
gh run list --workflow CI --limit 5
```

Watch a running deployment:

```bash
gh run watch <RUN_ID>
```

Inspect individual step timing:

```bash
gh api repos/IIS-Design/<repo>/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[].steps[] | {name, started_at, completed_at, conclusion}'
```

GitHub queue or dispatch time can vary independently of the actual deployment time.

Once a job reaches the self-hosted runner, normal build and deployment execution is typically much faster than the total elapsed workflow time shown by GitHub.

## Runner status

On `ipsos-devops-01`:

```bash
sudo systemctl status actions.runner.IIS-Design.ipsos-devops-01.service --no-pager
```

Recent logs:

```bash
sudo journalctl \
  -u actions.runner.IIS-Design.ipsos-devops-01.service \
  --since "10 minutes ago" \
  --no-pager
```

A healthy runner should remain connected and receive jobs without requiring manual intervention.

## Podman preflight

The runner executes a job-start preflight before each GitHub Actions job.

The preflight:

- verifies the rootless Podman environment
- continues immediately when Podman is healthy
- repairs the known stale pause-process condition when detected
- fails closed for unexpected Podman errors

Do not manually reset Podman state unless the preflight reports a failure that it cannot repair.

## Image cache

Runner image layers are intentionally preserved between jobs.

This significantly improves repeat-build performance.

Temporary CI containers are removed after every run, but build images and reusable layers are kept as cache.

Periodic image pruning should be performed as maintenance and must not be added to the critical deployment path.

## GHCR

Images are published under:

```text
ghcr.io/iis-design/<app>
```

Every successful build publishes an immutable commit tag:

```text
ghcr.io/iis-design/<app>:<git-sha>
```

After a successful production deployment, the convenience `main` tag is also updated.

Production does not depend on the moving `main` tag. It runs the immutable digest recorded in `compose.override.yaml`.

## Migrated applications

Current migrated applications:

| Application | Branch | Production route |
| --- | --- | --- |
| UXIpsos Pages | `main` | `/uxipsos-pages/` |
| Core | `main` | `/core/` |
| Mustang | `master` | `/mustang/` |
| Prototypes | `master` | `/prototypes/` |
| UI Library | `master` | `/ui-library/` |
| Astra | `main` | `/astra/` |

Legacy direct references to `herokuapp.com` were removed from these repositories during migration.

## Production validation

The platform was validated with:

- successful container builds
- local smoke tests
- GHCR SHA publication
- immutable production digest pinning
- production health checks
- automatic rollback after a deliberately unhealthy release
- successful rollback health validation
- multiple applications using different stacks
- repeated production deployments using the shared workflow

## Developer experience

The intended developer workflow is deliberately simple:

```bash
git add .
git commit -m "..."
git push
```

Everything after `git push` is handled by the deployment platform.

Developers should not need to:

- SSH into production
- manage containers manually
- access GHCR credentials
- use a VPN for deployment
- know the production network topology

## Migration status

The self-hosted platform is operational for the migrated applications.

The previous hosting environment should remain available as a temporary fallback until final parity validation and migration sign-off are complete.

Do not remove fallback infrastructure solely because the new deployment path is operational.

# 1. CI pipeline and container images

*2026-11-22 · .github#25*

Goal: every pull request and every merge goes through a build pipeline, and a
merge to `main` publishes a container image that the cluster can run.

## What already existed

- Every repository's `.github/workflows/ci.yml` calls the shared
  `reusable-ci.yml` on pull requests and on pushes to `main`: PR title check,
  EditorConfig, `npm ci`, lint, test and build, and a "Container image" job
  that builds the `Dockerfile` without pushing it.
- `main` is protected by a ruleset: changes only through pull requests, the
  checks above must pass, no force-push or deletion.
- `reusable-publish.yml` could already build and push an image, but no
  repository called it and none had a `Dockerfile`.

## What we added

### A `Dockerfile` per repository

`service-identity` and `gateway` (Node.js):

```dockerfile
FROM node:24-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts

FROM node:24-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY package.json ./
COPY src ./src
USER 1000:1000
CMD ["node", "src/server.js"]
```

- **What it does:** a *multi-stage* build. The first stage installs only
  runtime dependencies; the final image copies them plus the source, so no
  build tools, tests or dev dependencies ship.
- **Why these details:**
  - `USER 1000:1000` is a *number*, not the name `node`. The Helm chart sets
    `runAsNonRoot: true`, and Kubernetes can only verify that for a numeric
    user. With `USER node` the pod would refuse to start
    (`CreateContainerConfigError`).
  - `CMD ["node", ...]` instead of `npm start`: npm does not pass the stop
    signal (SIGTERM) on to Node, so containers would be killed after the grace
    period instead of shutting down cleanly.
  - `service-identity` runs `npm ci` **with** install scripts, because its
    Kafka client downloads a native library (librdkafka) during install. It
    also copies `migrations/`, since it migrates the database at startup.
- **If done differently:** a single-stage image would be larger and contain
  dev tooling. Running as root would be rejected by the chart's security
  settings.

`web` (static React build):

- Build stage: `npm ci && npm run build` (Vite) on `node:24-alpine`.
- Runtime: `nginxinc/nginx-unprivileged:stable-alpine` (runs as user 101 on
  port 8080), our `nginx/default.conf`, and `nginx/40-runtime-config.sh`.
- **Why:** nginx is the standard way to serve static files. (Only the
  Kubernetes *ingress-nginx* controller was retired in 2026; nginx itself is
  maintained. Our Ingress controller is Traefik.)
- The start-up script writes `/tmp/config.js` from environment variables
  (`OIDC_AUTHORITY`, ...), so **one image runs in every environment**. With
  `GATEWAY_URL` set (local Compose only) it also proxies `/api` to the gateway;
  without it `/api` answers 404 instead of returning the app's HTML.
- Everything nginx writes goes to `/tmp`, so the container runs with a
  read-only root filesystem, as the chart requires.

### A `publish` job in each `ci.yml`

```yaml
publish:
  needs: checks
  if: github.event_name == 'push'
  permissions:
    contents: read
    packages: write
  uses: RSOcc26/.github/.github/workflows/reusable-publish.yml@main
```

- **What it does:** after all checks pass on a push to `main`, builds the
  image and pushes it to `ghcr.io/rsocc26/<repository>` with the tags
  `sha-<commit>` and `main` (version tags add `X.Y.Z`).
- **Why:** images are only published from reviewed, tested code. The commit
  tag lets a deploy pin an exact version; `main` always points at the latest.
- **If done differently:** without `needs: checks` a broken commit could be
  published. Without the `if`, every pull request would push an image.

### Making the images public

GitHub created the new packages as **private**. We changed each package to
public (Package settings → Change visibility). If that option is greyed out,
the organization does not allow public packages yet: Organization settings →
Packages → allow Public.

- **Why:** the source repositories are public and the images contain no
  secrets, so public images cost nothing and need no credentials.
- **If kept private:** every puller needs a token. In Kubernetes that means a
  personal GitHub token stored as an `imagePullSecret` in every namespace,
  which expires and has to be recreated after each cluster rebuild. Locally,
  everyone would need `docker login ghcr.io`.

## How we checked it

- Each image was run with `--read-only --cap-drop ALL` as a non-root user,
  matching the chart's security context.
- The Compose stack ran from the **published** images: a user signed in with
  Keycloak reached `/api/identity/v1/me` through web → gateway → identity.
- `web/test/unit/runtime-config.test.js` tests the start-up script. It caught
  a real bug on its first run: `echo` dropped backslashes in some shells, so
  the script now uses `printf`.

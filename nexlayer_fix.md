# Nexlayer working build fix

This file is the authoritative, pinned build solution for this repo. Nexlayer uses it verbatim on every run and will not override it. If a future build with this fix fails, Nexlayer appends/updates it rather than regenerating.

## CRITICAL BUILD CONSTRAINTS

1. **Download the pre-compiled gitea binary from GitHub releases** — building gitea from source requires Go + Node.js + pnpm + Make and the multi-stage source build fails (golang:1.26-alpine3.24 does not exist; platform args are not supported in Kaniko). Use `curl` to fetch the official pre-compiled amd64 binary from GitHub releases into an Alpine base image.

2. **Base image must be `mirror.gcr.io/library/alpine:3.20`** — confirmed available on Nexlayer's mirror. Do NOT use Alpine 3.24 (untested) or `--platform=$BUILDPLATFORM` (breaks Kaniko).

3. **Port is 3000** — gitea HTTP listens on 3000 by default. Keep `servicePorts: [3000]`.

4. **SQLite database via env var** — set `GITEA__database__DB_TYPE=sqlite3`. No separate DB pod needed.

5. **Volume at /data required** — gitea stores repos, config, and SQLite DB at /data. Without a persistent volume all data resets on pod restart.

6. **Run as non-root git user** — gitea requires a `git` Unix user for SSH operations. Create it in the Dockerfile.

## Fixed Dockerfile

```dockerfile
FROM mirror.gcr.io/library/alpine:3.20

RUN apk add --no-cache ca-certificates bash git curl sqlite tini

ARG GITEA_VERSION=1.22.6
RUN curl -fsSL -o /usr/local/bin/gitea \
    "https://github.com/go-gitea/gitea/releases/download/v${GITEA_VERSION}/gitea-${GITEA_VERSION}-linux-amd64" && \
    chmod +x /usr/local/bin/gitea

RUN addgroup -S git && adduser -S -G git git && \
    mkdir -p /data/gitea /app/gitea && \
    chown -R git:git /data /app

ENV USER=git HOME=/data/gitea GITEA_CUSTOM=/data/gitea/custom GITEA_WORK_DIR=/data/gitea

USER git
WORKDIR /app/gitea
EXPOSE 3000
ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/gitea"]
CMD ["web", "--port", "3000"]
```

## Fixed nexlayer.yaml

```yaml
application:
  name: gitea
  pods:
    - name: app
      image: "."
      path: /
      servicePorts:
        - 3000
      vars:
        GITEA__database__DB_TYPE: sqlite3
        GITEA__server__HTTP_PORT: "3000"
        GITEA__server__ROOT_URL: "<%URL%>"
        GITEA__server__DOMAIN: "<%DOMAIN%>"
        USER_UID: "1000"
        USER_GID: "1000"
      volumes:
        - name: data
          mountPath: /data
          size: 10Gi
```

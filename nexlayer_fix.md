# Nexlayer working build fix

This file is the authoritative, pinned build solution for this repo. Nexlayer uses it verbatim on every run and will not override it. If a future build with this fix fails, Nexlayer appends/updates it rather than regenerating.

## CRITICAL BUILD CONSTRAINTS

1. **Use the pre-built gitea image — do NOT build from source** — Building from source takes 5+ minutes and fails. Use the official pre-built image via GCR mirror.

2. **Image: `mirror.gcr.io/gitea/gitea:1.22`** — Avoids Docker Hub rate limiting.

3. **Port is 3000** — gitea HTTP listens on 3000. Keep `servicePorts: [3000]`. Do NOT use port 22 (that's SSH only).

4. **SQLite database** — set `GITEA__database__DB_TYPE=sqlite3`. No separate DB pod needed.

5. **Volume at /data required** — gitea stores repos, config, and SQLite DB here.

## Fixed Dockerfile

```dockerfile
FROM mirror.gcr.io/gitea/gitea:1.22
```

## Fixed nexlayer.yaml

```yaml
application:
  name: gitea
  pods:
    - name: app
      image: "mirror.gcr.io/gitea/gitea:1.22"
      path: /
      servicePorts:
        - 3000
      vars:
        GITEA__database__DB_TYPE: sqlite3
        GITEA__server__HTTP_PORT: "3000"
        GITEA__server__ROOT_URL: "<%URL%>"
        GITEA__server__DOMAIN: "<%DOMAIN%>"
      volumes:
        - name: data
          mountPath: /data
          size: 10Gi
```

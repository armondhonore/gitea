# Nexlayer working build fix

This file is the authoritative, pinned build solution for this repo. Nexlayer uses it verbatim on every run and will not override it. If a future build with this fix fails, Nexlayer appends/updates it rather than regenerating.

## CRITICAL BUILD CONSTRAINTS

1. **Do NOT build from source** — the gitea repo has a Dockerfile but building gitea from source requires Go + Node.js + pnpm + Make and is extremely complex. Use the pre-built `gitea/gitea:1.22` image directly.

2. **Port is 3000** — gitea HTTP server listens on 3000 by default. Keep `servicePorts: [3000]`.

3. **SQLite database** — use `GITEA__database__DB_TYPE: sqlite3`. No separate DB pod required.

4. **Volume at /data required** — gitea stores repositories, config, and the SQLite DB at /data. Without a volume all data is lost on restart.

5. **ROOT_URL must match the deployment URL** — set `GITEA__server__ROOT_URL` to the live Nexlayer URL so OAuth redirects and clone URLs work correctly.

## Fixed Dockerfile

```dockerfile
FROM gitea/gitea:1.22
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
        GITEA__server__SSH_DOMAIN: "<%DOMAIN%>"
        USER_UID: "1000"
        USER_GID: "1000"
      volumes:
        - name: data
          mountPath: /data
          size: 10Gi
```

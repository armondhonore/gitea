# Nexlayer working build fix

This file is the authoritative, pinned build solution for this repo. Nexlayer uses it verbatim on every run and will not override it. If a future build with this fix fails, Nexlayer appends/updates it rather than regenerating.

## Fixed Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
# Stage 1: Build frontend
# Use node-alpine as base to ensure pnpm and node are compatible
FROM --platform=$BUILDPLATFORM mirror.gcr.io/library/node:22-alpine AS frontend-build

# Install make and other build tools required by the Gitea Makefile
RUN apk add --no-cache make git gcc g++ musl-dev

# Install pnpm via npm as per project requirements
RUN npm install -g pnpm@11.5.3

WORKDIR /src
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./ 
COPY .npmrc* ./

# Install dependencies
RUN --mount=type=cache,target=/root/.local/share/pnpm/store pnpm install --no-frozen-lockfile

COPY . .
# Now make is available to build the frontend assets
RUN make frontend

# Stage 2: Build backend
FROM mirror.gcr.io/library/golang:1.26-alpine3.24 AS build-env

ARG GITEA_VERSION
ARG TAGS=""
ENV TAGS="bindata timetzdata $TAGS"

# Install build-base and git for Go compilation and make
RUN apk --no-cache add build-base git make

WORKDIR ${GOPATH}/src/gitea.dev
COPY go.mod go.sum ./
RUN go mod download

COPY . .
# Copy the compiled frontend assets from the first stage
COPY --from=frontend-build /src/public/assets public/assets

# Build the Go binary
RUN --mount=type=cache,target="/root/.cache/go-build" \
    --mount=type=bind,source=".git/",target=".git/" \
    make backend

# Prepare the root filesystem for the final image
COPY docker/root /tmp/local
RUN chmod 755 /tmp/local/usr/bin/entrypoint \
              /tmp/local/usr/local/bin/* \
              /tmp/local/etc/s6/gitea/* \
              /tmp/local/etc/s6/openssh/* \
              /tmp/local/etc/s6/.s6-svscan/* \
              /go/src/gitea.dev/gitea

# Stage 3: Final Runtime Image
FROM mirror.gcr.io/library/alpine:3.24 AS gitea

EXPOSE 22 3000

# Install runtime dependencies
RUN apk --no-cache add \
    bash \
    ca-certificates \
    curl \
    gettext \
    git \
    linux-pam \
    openssh \
    s6 \
    sqlite \
    su-exec \
    gnupg

# Create git user and group
RUN addgroup -S -g 1000 git && \
    adduser -S -H -D -h /data/git -s /bin/bash -u 1000 -G git git && \
    echo "git:*" | chpasswd -e

# Copy the prepared rootfs and the binary
COPY --from=build-env /tmp/local /
COPY --from=build-env /go/src/gitea.dev/gitea /app/gitea/gitea

ENV USER=git
ENV GITEA_CUSTOM=/data/gitea
VOLUME ["/data"]

ENTRYPOINT ["/usr/bin/entrypoint"]
CMD ["/usr/bin/s6-svscan", "/etc/s6"]

```

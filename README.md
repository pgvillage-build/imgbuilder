# imgbuilder

Usually all images are built as part of the tool itself.
For some tools (for now it is just wal-g-pg) it isn't. We will build that ourselves in this repo.

Each image lives in `images/<name>/`. Pull requests build every image for linux/amd64 and linux/arm64 on native runners;
a published release pushes them to `ghcr.io/pgvillage-build/<name>`.

## wal-g-pg

[wal-g](https://github.com/wal-g/wal-g) for PostgreSQL, built from source from
[pgvillage-tools/wal-g](https://github.com/pgvillage-tools/wal-g), with the same recipe as the wal-g-pg packages in
[aptbuilder](https://github.com/pgvillage-build/aptbuilder) and [rpmbuilder](https://github.com/pgvillage-build/rpmbuilder)
(brotli, libsodium and lzo support, linked statically).

```
ghcr.io/pgvillage-build/wal-g-pg:<wal-g version>   # e.g. 3.0.9b
ghcr.io/pgvillage-build/wal-g-pg:latest
ghcr.io/pgvillage-build/wal-g-pg:sha-<commit>
```

The image is based on `gcr.io/distroless/base-debian13:nonroot`: glibc and ca-certificates, no shell and no package manager.
It runs as the `nonroot` user (UID 65532) and the entrypoint is `/usr/local/bin/wal-g-pg`, so the container's arguments are the
wal-g command:

```sh
podman run --rm ghcr.io/pgvillage-build/wal-g-pg:latest --version
```

### Running it as a Kubernetes Job

Configure wal-g with [environment variables](https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md)
(`WALG_S3_PREFIX`, `AWS_*`, `PGHOST`, …) and pass the command in `args`. There is no shell in the image, but Kubernetes expands
`$(VAR)` in `args` itself.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: wal-g-backup-push
spec:
  template:
    spec:
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
      containers:
        - name: wal-g
          image: ghcr.io/pgvillage-build/wal-g-pg:3.0.9b
          args: ["backup-push", "$(PGDATA)"]
          envFrom:
            - secretRef:
                name: wal-g-config  # WALG_S3_PREFIX, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, PGHOST, PGUSER, ...
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
```

`backup-push` reads the files of the data directory, so the Job needs it mounted at `$(PGDATA)` (left out above).
Commands that only talk to the storage, like `backup-list` or `delete`, don't.

### Updating

The wal-g tag, the go version and the base images are pinned in `images/wal-g-pg/Dockerfile`; Renovate opens PRs to bump them.
After merging, create a release to publish the new image.

To build it locally:

```sh
podman build -t wal-g-pg images/wal-g-pg
```

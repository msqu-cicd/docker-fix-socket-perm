# docker-fix-socket-perm

Composite action that makes the Docker host socket usable for the runner
user. It looks up the GID that owns the socket and adds the runner user to
the matching group, creating a workaround group first when no group owns
that GID yet.

## How it works

1. Fail early when the socket does not exist.
2. Exit successfully when the socket is already readable and writable.
3. Exit successfully when the user's groups already include the socket
   owner GID.
4. When a group with the socket owner GID exists (resolved via `getent`,
   so NSS sources count too), add the user to it.
5. Otherwise create the workaround group with that GID and add the user.
   If a group with the workaround name already exists under a different
   GID, the action fails with a clear error instead of running `groupadd`
   into a conflict.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `socket-path` | `/var/run/docker.sock` | Location of the Docker socket |
| `workaround-group` | `hostdocker` | Name of the group to create when no group owns the socket GID |

Both inputs are optional.

## Usage

```yaml
jobs:
  build:
    runs-on: some-host-runner
    steps:
      - uses: CICD/docker-fix-socket-perm@main
      - run: docker info
```

## Requirements and limitations

- The action must run as root or as a user with passwordless sudo
  (`sudo -n` is used; it fails instead of prompting for a password).
- `usermod -aG` only updates the group database. Processes that are
  already running keep their old supplementary groups, so on runners
  where every step runs inside the same long-lived session the new
  membership may not take effect until the runner process is restarted.
  Container-based runners that resolve groups per job pick it up on the
  next job at the latest; if a later step in the same job still gets
  "permission denied" on the socket, run the Docker command via
  `sg <group> -c '...'`.

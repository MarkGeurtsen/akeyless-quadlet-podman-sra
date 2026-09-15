# Akeyless Compose -> Podman Quadlet

## 1. Rootful only - rootless does not work for this stack

This started as a rootless setup and was migrated to rootful partway
through. **Rootless is not viable here** - `akeyless-sra-ssh`'s
entrypoint needs to write to `/etc/passwd` and `/etc/ssh/ca.pub` as
real root, and in rootless Podman `--privileged` does **not** grant
elevated host privileges (it only lifts some internal restrictions) -
a rootless container can never have more real capability than the
launching user. That failure (`cp: cannot create regular file
'/etc/ssh/ca.pub': Permission denied`, `usermod: cannot lock
/etc/passwd`) is unfixable under rootless; do not spend time
retrying it there.

**Where files go (rootful, system-wide):**

- `.container` / `.network` / `.volume` units -> `/etc/containers/systemd/`
- `.target` units -> `/etc/systemd/system/` (Quadlet's directory only
  scans its own extensions - plain `.target` files dropped into
  `/etc/containers/systemd/` are silently ignored)

On the Mac/podman-machine setup, `podman machine ssh` (no
`--username` flag) already connects as **root** - that's the rootful
session to use for all of this. Do not use `--username core` /
`systemctl --user` for these services; that was the rootless path
that didn't work.

```bash
podman machine ssh -- 'mkdir -p /etc/containers/systemd /etc/systemd/system'
podman machine ssh -- 'cp /Users/user/akeyless/*.container /Users/user/akeyless/*.network /Users/user/akeyless/*.volume /etc/containers/systemd/'
podman machine ssh -- 'cp /Users/user/akeyless/*.target /etc/systemd/system/'
podman machine ssh -- 'systemctl daemon-reload'
```

Quadlet auto-generates a matching `.service` for every `.container` (e.g.
`akeyless-gateway.container` -> `akeyless-gateway.service`). Manage
everything with plain `systemctl` / `podman ps` (no `--user`, no
`--username core`) from here on.

## 2. Fix up the paths

Compose let you use relative paths (`./certs/cert.crt`, `./gateway.env`, ...)
because everything was resolved relative to the compose file. Quadlet units
have no such context, so every host path must be absolute.

**On the Mac/Podman-machine setup**, these are hardcoded to
`/Users/user/akeyless/...` rather than using the `%h` specifier. That's
deliberate: `%h` expands to the home directory of whichever account
your rootless `systemd --user` instance runs as *inside the VM* (on the
default Fedora CoreOS machine image that's `core`, home
`/var/home/core`) - completely unrelated to `/Users/user`, which is
the actual virtiofs mount point of your Mac home. Since the mount is
always at `/Users/<your-mac-username>/akeyless`, hardcoding it is more
reliable than `%h` for this setup.

Current expected layout, matching what's actually in `/Users/user/akeyless`
right now:

```
/Users/user/akeyless/
  gateway.env
  cache.env
  sra.env
  certs/
    cert.crt
    key.pem
  sshd-config/        <- only needed if you enable SRA
    ca.pub
  metrics/             <- only needed if you enable metrics
    prometheus/
      prometheus.yml
```

`sshd-config/` and `metrics/` don't exist yet in your directory listing -
create those (with `ca.pub` and `prometheus.yml` respectively) before
starting `akeyless-sra-ssh.service` or `prometheus.service`. Also
double check the two filenames inside `certs/` actually match
`cert.crt` / `key.pem` - adjust the `Volume=` lines in
`akeyless-gateway.container` if yours are named differently.

If you ever run these on a different Mac account or move the folder,
update the `/Users/user/akeyless/...` paths in every `.container` file
accordingly - there's no path-independent shortcut here given the `%h`
mismatch.

## 3. The "tricky" part: SRA depending on a *healthy* gateway

Compose's `condition: service_healthy` (used by `akeyless-web` /
`akeyless-ssh` waiting on `akeyless-gateway`) doesn't have a direct
Quadlet key - ordinary `After=`/`Requires=` only waits for a unit to be
*started*, not *healthy*.

The fix is `Notify=healthy` in `akeyless-gateway.container`: Podman then
only tells systemd `READY=1` once the container's own `HealthCmd` reports
healthy. Since `akeyless-sra-web.service` and `akeyless-sra-ssh.service`
both do `After=akeyless-gateway.service` + `Requires=akeyless-gateway.service`,
systemd will now genuinely block them until the gateway is healthy -
functionally identical to the compose behavior. `TimeoutStartSec=180` on
the gateway service gives systemd enough slack to not time out during the
health check's `start_period` + retries.

## 4. Networks

- `internal-net.network` -> joined by cache, gateway, and both SRA
  containers (matches compose).
- `internal-metrics.network` -> joined by gateway, prometheus, and
  grafana (matches compose).
- The gateway container has two `Network=` lines - Quadlet supports
  joining multiple networks this way.
- Quadlet-managed Podman networks get DNS for free (aardvark-dns), so
  containers can still reach each other by container name exactly like
  compose's default network aliasing did - no need to port the old
  `links:` entries.

## 5. Emulating `--profile`

Quadlet has no `profiles` concept - instead just control *which* units
you start:

| Compose | Quadlet equivalent |
|---|---|
| `--profile gateway` | `systemctl start akeyless-gateway.service` |
| `--profile sra` | `systemctl start akeyless-sra.target` (pulls in gateway + web + ssh) |
| `--profile metrics` | `systemctl start akeyless-metrics.target` (pulls in prometheus + grafana) |
| combined profiles | start multiple targets/services together |

`akeyless-cache.service` is pulled in automatically wherever it's needed
via `Requires=` on the gateway unit, so you never start it directly.

## 6. Things worth double-checking

- **`platform: linux/amd64`** -> passed through as `PodmanArgs=--platform=linux/amd64`
  (Quadlet has no first-class `Platform=` key). On Apple Silicon this
  means every process in the container runs under QEMU emulation
  (`qemu-x86_64-static`), which is dramatically slower than native -
  gateway/SRA startup alone can take 2-3+ minutes on a cold pull. See
  `TimeoutStartSec` below. If `akeyless/gateway` or
  `akeyless/zero-trust-bastion` ever publish an arm64 image, dropping
  this flag would remove the emulation overhead entirely - worth
  checking periodically.
- **`TimeoutStartSec=600`** is set in `[Service]` on
  `akeyless-gateway.container`, `akeyless-sra-web.container`, and
  `akeyless-sra-ssh.container` specifically to survive the QEMU
  slowness above combined with `Notify=healthy` blocking start
  completion until the health check passes. The default (~90s) is not
  enough on this setup and the unit will be killed mid-startup
  (`start operation timed out. Terminating.`) even though the app was
  about to succeed.
- **`akeyless-sra-ssh.container` needs `User=0:0` in `[Container]`.**
  The `zero-trust-bastion` image sets `USER 1001` in its Dockerfile,
  so the container runs as an unprivileged user by default regardless
  of host engine or `--privileged` - `--privileged` only lifts
  host-side restrictions, it does not change which UID the image's
  own entrypoint runs as. Without `User=0:0`, the ssh-proxy entrypoint
  fails every time trying to write `/etc/ssh/ca.pub` and lock
  `/etc/passwd` (`Permission denied`). `akeyless-sra-web` does not
  need this override - its entrypoint doesn't touch those paths.
- **`privileged: true`** on `akeyless-ssh` -> `PodmanArgs=--privileged`
  (no first-class `Privileged=` key). Kept for parity with compose,
  but note it is `User=0:0` above - not this flag - that actually
  fixes the permission errors.
- Gateway's TLS port is published as **`8443:8000`**, not `443:8000`
  as in the original compose file. `443` requires
  `net.ipv4.ip_unprivileged_port_start` to be lowered for rootless
  Podman to bind it - since this stack ended up rootful anyway (root
  can bind privileged ports natively), reverting to `443` is an option
  if exact compose parity matters more than the current working port.
- **`restart: unless-stopped`** -> mapped to `Restart=always`. Systemd
  doesn't have an exact "unless-stopped" mode, but in practice `Restart=always`
  behaves the same way: it restarts on any exit, and a manual
  `systemctl stop` is always respected regardless of the Restart= policy.
- Grafana's optional dashboard/datasource provisioning bind mounts are
  commented out in `grafana.container`, same as they were commented out
  in your compose file - uncomment and adjust paths if you use them.
- Named volumes `prometheus_data` / `grafana_data` become
  `prometheus-data.volume` / `grafana-data.volume` with `VolumeName=`
  set to the original compose volume names, so existing data isn't
  orphaned if you're migrating a live stack.

## 7. Env file gotchas (Podman's `--env-file` parser is stricter than Compose's)

Compose tolerates `KEY="value"    # trailing comment` lines. Podman's
`EnvironmentFile=` parser does not - it takes everything after `=` up
to the newline as the literal value, quotes and comment text included.
This silently corrupted `GATEWAY_ACCESS_ID`, `AKEYLESS_URL`, and
similar values (e.g. `GATEWAY_ACCESS_ID` became the literal string
`"p-ll1wg9mzzf3aas"        # The Access ID to authenticate the
Gateway`), producing garbled URLs and `PANIC`/config-parse errors deep
in the gateway logs that don't obviously point back to the env file.
When copying values from an Akeyless-provided template `.env`, strip
all quotes and move every trailing `# comment` to its own line above
the value before use.

Also double check internal URLs reference the container's actual
`ContainerName=`, not the old compose *service* key - e.g.
`REMOTE_ACCESS_SSH_SERVICE_INTERNAL_URL` must point at
`akeyless-sra-ssh` (the real container name), not `akeyless-ssh` (the
old compose service name) - Podman network DNS resolves by container
name only.
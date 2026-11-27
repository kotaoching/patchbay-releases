# Shayan Relay

English | [简体中文](README.zh-CN.md)

Shayan (/ˈʃɑːjɛn/ SHAH-yen; 沙燕, *shāyàn*) is named after the Beijing swallow kite. Its Chinese name is 纸鸢.

The Relay runs on a server with a public address. Phones reach `shayand` on a Computer through it. `shayand` connects out to the Relay, so the Computer does not open any inbound port.

The Relay only forwards Noise-encrypted traffic and does not serve the page: the Web that phones open has its own domain and its own install ([Web guide](../web/README.md)). The Relay sees metadata, not content: which Computer is online, the addresses of Computers and phones, when and for how long they connect, how many bytes each channel carries, and the size of the offline copies. It has no database. It keeps one small file in its state directory, `hosts.json`: the allowlist of Computers that may connect, each with its `hostId`, public key and the time it joined. Only the Relay writes this file.

It runs from its own package, `shayan-relay-<version>-<platform>.tar.gz`, signed and published in the releases of [`kotaoching/shayan-releases`](https://github.com/kotaoching/shayan-releases). The package carries its own Node, so the server does not need Node. The same package runs in one of two ways: as the system service `shayan-relay.service` (systemd runs it as a dynamic user and starts it at boot), or as the Docker container `shayan-relay` (the image is built on the server from the package).

## Requirements

- A public address with inbound TCP 80 and 443 open, and a domain whose A record (and AAAA record, if you have IPv6) points to it.
- Debian 12 or 13, or Ubuntu 24.04 LTS or 26.04 LTS, on x86_64 or arm64, and root or a user who can `sudo`.
- `sudo apt install caddy curl`. With `minisign` installed as well, every install and upgrade verifies the package's signature against the release key; without it, only HTTPS is relied on. Caddy runs as a system service on ports 80 and 443, and `/etc/caddy/Caddyfile` contains the line `import sites/*.caddy` (the same line the Web needs; add it once). You write the Relay's site yourself, as `/etc/caddy/sites/shayan-relay.caddy` (change the port if it is not 8787):

  ```sh
  sudo mkdir -p /etc/caddy/sites
  printf '<relay domain> {\n\treverse_proxy 127.0.0.1:8787\n}\n' | sudo tee /etc/caddy/sites/shayan-relay.caddy
  sudo systemctl reload caddy
  ```

  Caddy obtains the certificate itself, forwards WebSockets, keeps the `Host` header and appends the phone's address to `X-Forwarded-For`, which `--trust-proxy` reads. The installer does not touch Caddy. It installs even if the site or the certificate is not ready yet, and then only reports that HTTPS does not answer.
- The system service needs systemd. The Docker way needs Docker Engine with its Compose plugin (`docker compose version` must answer): on Debian 13 `sudo apt install docker.io docker-cli docker-compose`, on Ubuntu `sudo apt install docker.io docker-compose-v2`. Debian 12's own repository has no Compose plugin; use Docker's repository ("Install Docker Engine on Debian" at docs.docker.com).

## Install

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay/install.sh | sudo sh -s -- --domain <relay domain> [--port <port>] [--docker <dir>]
```

Without `--docker` it is installed as a system service in `/opt/shayan-relay`; with it, as a container in that directory. `--port` defaults to 8787. To install a specific version, including one not yet marked as the latest:

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay-v<version>/install.sh | sudo sh -s -- --version <version> --domain <relay domain>
```

The script downloads that version's package, verifies its signature with the release key built into the script (when `minisign` is installed), unpacks it and runs the package's own `install.sh` with `bash`. That installer:

1. Runs one install at a time.
2. Checks that what the chosen way needs is present, and that a Relay running the other way is not running (both use the same port). If it cannot tell, it stops and changes nothing.
3. Puts the version in `<dir>/versions/<version>/`, read-only. For Docker, it builds the image `shayan-relay:<version>` there from the package's `Dockerfile` (the base image is pinned by digest). If the build fails, it stops and leaves the running Relay alone.
4. On the first install, writes `relay.env` from the options.
5. Points `current` at the new version.
6. System service: writes the unit for `DIR`, then runs `daemon-reload`, `restart` and `enable`. Docker: puts the package's `compose.yaml` in `<dir>` and runs `docker compose up -d`.
7. Counts the install as done only if `127.0.0.1:<port>/relay/info`, asked with `Host: <domain>`, answers within 30 seconds.

When it finishes, it prints the installed version, the configuration file, that the Relay answered, the command to read the logs, the command to roll back, and whether `https://<domain>/relay/info` can be reached from this server (if not, the install has still succeeded). After the first install, it also explains how to let a Computer connect (see "Which Computers may connect" below).

After the Relay restarts (an upgrade, a server reboot), the offline copies come back only as each Computer reconnects, and invitations that were not used yet no longer work: create a new one.

## Upgrade

```sh
sudo shayan-relay update                      # the latest version, if it is newer than the installed one
sudo shayan-relay update --version <version>  # that version, older ones too
```

`/usr/local/bin/shayan-relay` hands `update` to the `fetch.sh` of the installed version. For Docker too, it runs on the server, not in the container. If the command itself is broken, run it directly: `sudo <dir>/current/fetch.sh [--version <version>]`. With `minisign` installed, it trusts only the release keys in the installed version.

## Moving from Patchbay 0.6.1

Shayan was called Patchbay up to 0.6.1. Patchbay does not update itself into the new name, and Shayan does not read Patchbay's data, so the move is made once by hand:

1. Remove Patchbay's Relay as the Uninstall section below says, with `patchbay` in place of `shayan` in each name, its Caddy site file `/etc/caddy/sites/patchbay-relay.caddy` included. Its `relay.env` and state directory stay where they are; delete them when you no longer need them.
2. If the Relay moves to a new domain, point the domain at this server first.
3. Install Shayan's Relay as the Install section above says, then invite each Computer again (`sudo shayan-relay invite`).

## Configuration

`/etc/shayan-relay/relay.env` holds one `KEY=value` per line. The first install writes it from its options; after that you edit it yourself. After editing, run `sudo <DIR>/current/install.sh` again: it restarts the Relay and checks that it answers. Once the file exists, an option that differs from it has no effect, and the installer says so. If a key is missing, the installer stops and tells you which line to add. The service's unit (`EnvironmentFile`) and the container's `compose.yaml` (`env_file`) read the same file.

| Key | Meaning |
| --- | --- |
| `WAY` | `service` (system service) or `docker` (container) |
| `DIR` | The install directory, an absolute path: `/opt/shayan-relay` by default for the service, the `--docker` directory for Docker |
| `DOMAIN` | The Relay's domain. `https://<DOMAIN>` is its origin, and each Computer's signature is bound to it |
| `PORT` | The port it listens on at `127.0.0.1`, which the Caddy site forwards to; 8787 by default |
| `UPDATE_SOURCE` | Where upgrades come from: `https://github.com/kotaoching/shayan-releases/releases/download/relay`, or a local directory with the same layout |

- Changing the port: edit `PORT` and run the installer again; change the line in the Caddy site to `reverse_proxy 127.0.0.1:<new port>` and run `sudo systemctl reload caddy`.
- Changing the domain: edit `DOMAIN` and run the installer again. The allowlist stays as it is, since it knows Computers by their keys, not by the domain. Point the Caddy site at the new domain. On each Computer run `shayan relay set https://<new domain>`; no new invitation is needed. Each phone has to pair again, because the pairing invitation names the Relay.

## Roll back

The server keeps the version that was replaced, so you can go back to it without a download:

```sh
sudo <dir>/versions/<previous version>/install.sh
```

For an older version, install it by its version number (the second command under "Upgrade"). The allowlist does not change between versions, and there is no database to restore. If an install fails (no answer within 30 seconds), the installer exits with status 1 and prints the log command and the rollback command. It leaves the service as it is and does not switch back on its own.

## Which Computers may connect

Only the running Relay changes its allowlist. Joining and revoking take effect at once, without a restart. `/usr/local/bin/shayan-relay` talks to the running Relay as root (the service through `admin.sock` in its state directory, the container through `docker exec`):

| Task | Command |
| --- | --- |
| Create an invitation | `sudo shayan-relay invite` |
| List the Computers | `sudo shayan-relay hosts` |
| Remove a Computer | `sudo shayan-relay revoke <hostId>` |

Each Computer is independent and may belong to a different person. Computers cannot see each other, and cannot add or remove one another. A `hostId` on the allowlist cannot be taken over by another key. Only the person who runs the Relay can add or remove Computers.

- **Adding a Computer**: create an invitation. It prints the line `shayan relay join --url https://<domain> --code <code>`. An invitation is for one Computer, can be used once, is valid for 5 minutes and is kept only in the Relay's memory. Run the line on that Computer: it sends the Computer's `hostId` and public key to the Relay, then sets the Computer's Relay to `https://<domain>`. If the answer was lost, run it again on the same Computer. For those 5 minutes the line is a single-use secret, so use it only on the Computer that is joining. When `shayan relay` shows `connected`, run `shayan devices pair`.
- **Listing**: one line per Computer, `<hostId> <fingerprint> <joined at|-> <on|off>`. On a Computer, `shayan relay key` prints its `hostId` and fingerprint so you can compare them.
- **Removing a Computer**: it leaves the allowlist and is disconnected at once, and so are its phones' connections through this Relay. From then on it gets `not_allowed`. Its offline copy is deleted at once.
- **A Computer whose key changed** (its data directory was recreated): first `revoke` its old entry, then invite it again.

## State directory and backup

The only thing to keep is `hosts.json` in the state directory (mode 0600, written only by the Relay). The package, the image and the unit can all be installed again; if the allowlist is lost, every Computer has to be invited again.

| Way | State directory | Owner |
| --- | --- | --- |
| System service | `/var/lib/shayan-relay/` (with `DynamicUser`, actually `/var/lib/private/shayan-relay/`) | The dynamic user systemd creates for it, mode 0700 |
| Docker | `<dir>/state/`, mounted in the container as `/var/lib/shayan-relay` | uid 1000 (the user in the image), mode 0700 |

To back up, copy `hosts.json`. To restore it, stop the Relay, put the file back with mode 0600 and the same owner as the state directory (`sudo install -m 600 -o "$(sudo stat -L -c %u <state dir>)" -g "$(sudo stat -L -c %g <state dir>)" hosts.json <state dir>/`), then start the Relay.

## The two ways in detail

- System service: the unit is the package's `shayan-relay.service`, with `@DIR@` replaced by `DIR`, written to `/etc/systemd/system/`. It has `EnvironmentFile=/etc/shayan-relay/relay.env`, `DynamicUser=yes`, `StateDirectory=shayan-relay` (the only place it can write), `ProtectSystem=strict`, `ProtectHome` and other hardening options, and `Restart=always`. Logs: `journalctl -u shayan-relay`. Do not put `DIR` under `/home`, because of `ProtectHome`.
- Docker: one image per version, `shayan-relay:<version>`, built on this server from `<dir>/versions/<version>/` and containing only that package; it runs as uid 1000, and no registry is used. `<dir>/compose.yaml` is the package's own: the container is named `shayan-relay`, uses `restart: unless-stopped`, is read-only, has `<dir>/state/` as its only volume, publishes only on `127.0.0.1:<port>`, and logs with `json-file` (10 MB × 3). To run Compose yourself, use `sudo env -u DOMAIN -u PORT docker compose --env-file /etc/shayan-relay/relay.env -f <dir>/compose.yaml <command>`: for `${DOMAIN}` and `${PORT}` in the compose file, Compose takes variables of the same name from the calling environment before `--env-file`. Logs: `docker compose -f <dir>/compose.yaml logs`. The images of `current` and of the replaced version are kept; the others are removed together with their version directories.

The logs contain no frame or blob content, no reads of offline copies, no source addresses, no signatures and no invitation codes.

## Switching between the two ways

Both ways use the same port and cannot run at the same time. The installer does not switch for you; these are the steps:

1. Stop the current one. Service: `sudo systemctl disable --now shayan-relay.service`. Docker: `sudo env -u DOMAIN -u PORT docker compose --env-file /etc/shayan-relay/relay.env -f <dir>/compose.yaml stop`.
2. Change `WAY` and `DIR` in `relay.env`.
3. Move the allowlist, so that Computers do not have to join again. To Docker: `sudo install -d -m 700 -o 1000 -g 1000 <dir>/state && sudo install -m 600 -o 1000 -g 1000 /var/lib/shayan-relay/hosts.json <dir>/state/`. To the service: after installing, restore it as described in "State directory and backup".
4. Install the new way: `curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay-v<version>/install.sh | sudo sh -s -- --version <version>` (the installed version is in the old directory; the new one has none yet).

## Uninstall

System service:

```sh
sudo systemctl disable --now shayan-relay.service
sudo rm -f /etc/systemd/system/shayan-relay.service /usr/local/bin/shayan-relay && sudo systemctl daemon-reload
sudo rm -r /opt/shayan-relay
```

Docker:

```sh
sudo env -u DOMAIN -u PORT docker compose --env-file /etc/shayan-relay/relay.env -f <dir>/compose.yaml down
sudo docker image ls --format '{{.Repository}}:{{.Tag}}' shayan-relay | xargs -r sudo docker image rm
sudo rm -f /usr/local/bin/shayan-relay
sudo rm -r <dir>/versions && sudo rm -f <dir>/compose.yaml <dir>/current
```

`relay.env` and the state directory (the allowlist) are kept. If you do not need them, delete `/etc/shayan-relay/` and `/var/lib/shayan-relay/` (or `<dir>/state/`) too. Remove `/etc/caddy/sites/shayan-relay.caddy` and run `sudo systemctl reload caddy`. On each Computer, run `shayan relay clear`.

## Troubleshooting

- Logs: the command for each way, above.
- Ask the Relay without Caddy: `curl -H 'Host: <domain>' http://127.0.0.1:<port>/relay/info`. The Relay only answers requests whose `Host` is its own domain.
- HTTPS does not answer: check `/etc/caddy/sites/shayan-relay.caddy`, the import line in the Caddyfile, the domain's A record, ports 80 and 443, and Caddy's certificate (`journalctl -u caddy`).
- On the Computer, run `shayan relay`: `lastError` shows why the last attempt failed. `not_allowed`: the Computer is not on the allowlist, or the key on the allowlist is not its current one; check with `sudo shayan-relay hosts` and invite it again. `protocol`: the Relay and the Computer run versions that cannot talk to each other; upgrade the one that is behind.

## Without the installer

The unpacked package runs as it is, behind a reverse proxy that handles TLS:

```
shayan-relay serve --origin https://<domain> --state-dir <dir> --listen 127.0.0.1:<port> --trust-proxy
```

`--state-dir` is where it keeps the allowlist and `admin.sock` (mode 0700, owned by the user that runs it). A new directory starts with an empty allowlist, so no Computer can connect until one joins with an invitation. The commands are then `shayan-relay invite --state-dir <dir>` and so on, run as that user.

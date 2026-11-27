# Shayan releases

English | [简体中文](README.zh-CN.md)

Shayan (/ˈʃɑːjɛn/ SHAH-yen; 沙燕, *shāyàn*) is named after the Beijing swallow kite. Its Chinese name is 纸鸢. Up to 0.6.1 it was called Patchbay.

This repository holds the release packages of Shayan, their signatures and their installers. The source code is not here.

Shayan lets you run coding agents (Claude Code, Codex, pi) on your own computer and work with them from your phone or a browser: start them, follow what they do, answer their approval requests and take over their terminal.

## Components

Shayan has three components. Each has its own package and its own version numbers.

| Component | What it does | Package |
| --- | --- | --- |
| Host | Runs on the Computer where your coding agents run, as the user who runs them | `shayan-host-<version>-linux-<x64 or arm64>.tar.gz` |
| Relay | Runs on a server with a public address and relays the connection between your phone and the Computer | `shayan-relay-<version>-linux-<x64 or arm64>.tar.gz` |
| Web | Runs on a server with a public address and serves the interface you open on your phone or in a browser | `shayan-web-<version>-any.tar.gz` |

The Relay and the Web can run on the same server or on two. Each component's guide covers its requirements, configuration, upgrades, rollback, uninstalling and troubleshooting:

- [Host guide](host/README.md)
- [Relay guide](relay/README.md): as a system service or in Docker
- [Web guide](web/README.md)

## Install

You need:

- A server (or two) running Debian 12 or 13, or Ubuntu 24.04 or 26.04, on x86_64 or arm64, with TCP ports 80 and 443 open, and a domain for the Relay and one for the Web, each pointing to the server it runs on.
- A Computer running one of the same systems, used as the user who runs the agents, with Claude Code, Codex or pi installed, logged in and on that shell's `PATH`. The first install saves that `PATH` for the agents.

The commands below use example domains. Set `RELAY_DOMAIN` and `WEB_DOMAIN` to your own in the shell where you run them.

**1. Each server.** On every server that runs the Relay or the Web, install Caddy:

```sh
sudo apt install caddy curl minisign           # minisign is optional; with it, every install verifies the package's signature
sudo mkdir -p /etc/caddy/sites
```

Then make sure `/etc/caddy/Caddyfile` contains this line once, at the top level (not inside a site block), so that Caddy loads the sites in `/etc/caddy/sites/`:

```
import sites/*.caddy
```

Add it if it is missing; the Web's installer stops without it. Do not add it a second time: a site imported twice makes Caddy refuse the configuration.

**2. The Relay**, on its server. Configure Caddy for the Relay, then install it:

```sh
RELAY_DOMAIN=relay.example.com
printf '%s {\n\treverse_proxy 127.0.0.1:8787\n}\n' "$RELAY_DOMAIN" | sudo tee /etc/caddy/sites/shayan-relay.caddy
sudo systemctl reload caddy
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay/install.sh | sudo sh -s -- --domain "$RELAY_DOMAIN"   # add --docker /opt/shayan-relay-docker to run it in Docker
```

**3. The Web**, on its server. Its installer configures Caddy itself:

```sh
WEB_DOMAIN=app.example.com
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/web/install.sh | sudo sh -s -- --domain "$WEB_DOMAIN"
```

**4. The Host**, on the Computer, as that user (only `apt` needs sudo):

```sh
sudo apt install curl minisign                 # minisign is optional, as on the server
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/host/install.sh | sh -s --
loginctl enable-linger                         # needed over ssh, and for agents to keep running after you log out
```

If the installer says that `~/.local/bin` is not on your `PATH`, run the `export PATH=...` line it prints before going on. Then add a root folder the agents may work in. It must be a folder that already exists; use your own, and add more if you like:

```sh
shayan roots add "$HOME/projects"
```

If `loginctl enable-linger` is not allowed, an administrator can run `sudo loginctl enable-linger <user>`. `shayan doctor` checks the Computer item by item and says what to fix.

**5. Join the Relay.** When the Host is installed, create an invitation on the Relay's server:

```sh
sudo shayan-relay invite
```

It prints a line `shayan relay join --url https://… --code …` with an invitation code. Run that line on the Computer within 5 minutes; it works once.

**6. Pair the phone.** When `shayan relay` on the Computer shows `connected`, run `shayan devices pair` there. It shows a pairing code as a QR code. On the phone, open the Web's address (for example `https://app.example.com`), add it to the Home Screen, and in it scan the pairing code. The phone then shows a confirmation code: type it into the Computer's terminal, where `shayan devices pair` asks for it.

## Update

If a machine runs Patchbay 0.6.1 or earlier, it does not update itself into Shayan. Move it once as each guide's "Moving from Patchbay 0.6.1" section says: remove Patchbay, install Shayan, then join the Relay and pair the phone again. After that, update with these commands.

On each machine, this installs the latest version if it is newer than the installed one. With `--version <version>`, it installs that version, older ones too.

```sh
shayan update                                # the Computer
sudo shayan-relay update                     # the Relay
sudo /srv/shayan-web/current/fetch.sh        # the Web
```

## Releases

- **One release per version**, tagged `<component>-v<version>` (for example `host-v0.7.0`). It contains the packages, a signature (`.minisig`) for each, the version's `install.sh` and its guides. Its release notes describe the changes, including any manual upgrade steps.
- **One release per component**, tagged `host`, `relay` or `web`. It contains the `install.sh` that a first install uses, and `latest`, a single line with the latest version number. A new version is published first, tested on real machines, and then marked as the latest.

To install a version by its number, including one not yet marked as the latest, replace `<component>` and `<version>`:

```
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/<component>-v<version>/install.sh | sh -s -- --version <version>
```

For the Relay and the Web, use `sudo sh` and add `--domain`. Each `install.sh` downloads the version's package, verifies its signature when `minisign` is installed (otherwise it relies on HTTPS alone and says so), and runs the installer inside the package with the options you gave.

## Signatures

Every package is signed with [minisign](https://jedisct1.github.io/minisign/). The public key:

```
RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

The installers have it built in and, when `minisign` is installed, check everything below themselves. To verify a package by hand, put its `.minisig` in the same directory and run:

```sh
minisign -Vm shayan-host-0.7.0-linux-x64.tar.gz -P RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

minisign checks the content, not the file name, so a renamed file still verifies. After it succeeds, make sure the trusted comment it prints is exactly the package's file name.

# Patchbay releases

English | [简体中文](README.zh-CN.md)

This repository holds the release packages of Patchbay, their signatures and their installers. The source code is not here.

Patchbay lets you run coding agents on your own computer and work with them from your phone or a browser: start them, follow what they do, answer their approval requests and take over their terminal. The phone connects through a Relay.

> No version has been published here yet. The first will be the one after 0.6.1; until then the commands below do not work and the guides' links are empty.

## Components

Each component is installed on its own machine from its own package and has its own version numbers.

| Component | Where it runs | Package |
| --- | --- | --- |
| Host | Each Computer whose agents you run, as the user who runs them | `patchbay-host-<version>-linux-<x64 or arm64>.tar.gz` |
| Relay | A server with a public address; phones reach the Computer through it | `patchbay-relay-<version>-linux-<x64 or arm64>.tar.gz` |
| Web | A server with a public address; the page a phone opens | `patchbay-web-<version>-any.tar.gz` |

Each component's guide covers its requirements, configuration, upgrades, rollback, uninstalling and troubleshooting:

- [Host guide](host/README.md)
- [Relay guide](relay/README.md): as a system service or in Docker
- [Web guide](web/README.md)

## Install

You need:

- A server running Debian 12 or 13, or Ubuntu 24.04 or 26.04 (x86_64 or arm64), with TCP ports 80 and 443 open and two domains pointing to it: one for the Relay and one for the Web. The two can also be on separate servers.
- A Computer running one of the same systems, used as the user who runs the agents, with Claude Code, Codex or pi installed, logged in and on that shell's `PATH`. The first install saves that `PATH` for the agents.

**1. The server.** Once: install Caddy and write the Relay's site (the Web's installer writes its own):

```sh
sudo apt install caddy curl                    # with minisign as well, each package's signature is verified
# add this line to /etc/caddy/Caddyfile, once:  import sites/*.caddy
sudo mkdir -p /etc/caddy/sites
printf '<relay domain> {\n\treverse_proxy 127.0.0.1:8787\n}\n' | sudo tee /etc/caddy/sites/patchbay-relay.caddy
sudo systemctl reload caddy
```

Then install the Relay and the Web, and create an invitation for the Computer:

```sh
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/relay/install.sh | sudo sh -s -- --domain <relay domain>   # add --docker <dir> to run it in Docker
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/web/install.sh | sudo sh -s -- --domain <web domain>
sudo patchbay-relay invite                     # prints the command for the Computer; valid once, for 5 minutes
```

**2. The Computer**, as that user (only `apt` needs sudo):

```sh
sudo apt install curl                          # and minisign, as on the server
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/host/install.sh | sh -s --
loginctl enable-linger                         # needed over ssh, and for runs to continue after you log out
patchbay roots add <dir>                       # a folder where runs may start
patchbay relay join --url https://<relay domain> --code <code>   # the command that invite printed
patchbay devices pair                          # once `patchbay relay` shows connected
```

If `loginctl enable-linger` is not allowed, an administrator can run `sudo loginctl enable-linger <user>`. `patchbay doctor` checks the Computer item by item and says what to fix.

**3. The phone.** Open `https://<web domain>`, add it to the Home Screen, and in it scan the pairing code that `patchbay devices pair` shows. Then type the code the phone shows into the Computer's terminal, where `patchbay devices pair` asks for it.

## Update

On each machine, this installs the latest version if it is newer than the installed one. With `--version <version>`, it installs that version, older ones too.

```sh
patchbay update                                # the Computer
sudo patchbay-relay update                     # the Relay
sudo /srv/patchbay-web/current/fetch.sh        # the Web
```

## Releases

- **One release per version**, tagged `<component>-v<version>` (for example `host-v0.7.0`). It contains the packages, a signature (`.minisig`) for each, the version's `install.sh` and its guides, and its release notes describe the changes, including any manual upgrade steps.
- **One release per component**, tagged `host`, `relay` or `web`. It contains the `install.sh` used by a first install, and `latest`, a single line with the latest version number. A new version is published first, tested on real machines, and then marked as the latest.

To install a version by its number, including one not yet marked as the latest:

```sh
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/<component>-v<version>/install.sh | sh -s -- --version <version>   # for the Relay and the Web: sudo sh, with --domain
```

Each `install.sh` downloads the version's package, verifies its signature when `minisign` is installed (otherwise it relies on HTTPS alone and says so), and runs the installer inside the package with the options you gave.

## Signatures

Every package is signed with [minisign](https://jedisct1.github.io/minisign/). The public key:

```
RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

The installers have it built in. To verify a package yourself, with its `.minisig` in the same directory:

```sh
minisign -Vm patchbay-host-<version>-linux-x64.tar.gz -P RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

The trusted comment it prints is the package's file name, so a signature cannot be reused for a different package.

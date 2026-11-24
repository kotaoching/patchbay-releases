# Patchbay releases

The packages of Patchbay, their signatures and their installers. The source is not in this repository.

Patchbay is a local-first control space for coding agents that run on your machine: start them, follow what they do, answer their approvals, and take over their terminal, from your phone or from a browser, through a Relay.

> No version has been published here yet: the first will be the one after 0.6.1. Until then the commands below find nothing.

## The three parts

Each is installed on its own machine from its package here, and has its own versions.

| Part | Where it runs | Package |
| --- | --- | --- |
| Host | each Computer whose agents you run, as the user who runs them | `patchbay-host-<version>-linux-<x64 or arm64>.tar.gz` |
| Relay | a server with a public address: phones reach the Computer through it | `patchbay-relay-<version>-linux-<x64 or arm64>.tar.gz` |
| Web | a server with a public address: the page a phone opens | `patchbay-web-<version>-any.tar.gz` |

## Install

You need:

- a server running Debian 12 or 13, or Ubuntu 24.04 or 26.04 (x86_64 or arm64), with TCP 80 and 443 open and two domains pointing to it, one for the Relay and one for the Web (the two can also be on two servers);
- a Computer on one of the same systems, used as the user who runs the agents, with Claude Code, Codex or pi installed, logged in and on that shell's `PATH` (the first install keeps that `PATH` for the agents).

**1. The server.** Once, Caddy and the Relay's site, written by hand (the Web's installer writes its own):

```sh
sudo apt install caddy curl                    # with minisign too, each package's signature is checked
# add this line to /etc/caddy/Caddyfile, once:  import sites/*.caddy
sudo mkdir -p /etc/caddy/sites
printf '<relay domain> {\n\treverse_proxy 127.0.0.1:8787\n}\n' | sudo tee /etc/caddy/sites/patchbay-relay.caddy
sudo systemctl reload caddy
```

Then the Relay, the Web, and an invitation for the Computer:

```sh
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/relay/install.sh | sudo sh -s -- --domain <relay domain>   # --docker <dir>: in Docker
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/web/install.sh | sudo sh -s -- --domain <web domain>
sudo patchbay-relay invite                     # prints the line for the Computer, for one use within 5 minutes
```

**2. The Computer**, as that user (only `apt` takes sudo):

```sh
sudo apt install curl                          # and minisign, as on the server
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/host/install.sh | sh -s --
loginctl enable-linger                         # used over ssh, or runs go on after you log out
patchbay roots add <dir>                       # the folder runs may start in
patchbay relay join --url https://<relay domain> --code <code>   # the line invite printed
patchbay devices pair                          # once `patchbay relay` says connected
```

Where `loginctl enable-linger` is denied, an administrator runs `sudo loginctl enable-linger <user>`. `patchbay doctor` checks the Computer item by item, with what to do.

**3. The phone.** Open `https://<web domain>`, add it to the Home Screen, and in it scan the QR code `patchbay devices pair` shows; then type the code the phone shows where `patchbay devices pair` asks for it.

## Update

Each on its own machine: the latest version when it is newer than the one installed, or with `--version <version>` that version, an older one too.

```sh
patchbay update                                # the Computer
sudo patchbay-relay update                     # the Relay
sudo /srv/patchbay-web/current/fetch.sh        # the Web
```

## What is here

- **A release for each version**, tagged `<part>-v<version>` (`host-v0.7.0`): its packages, each with its signature (`.minisig`), and its `install.sh`. Its text is what that version changed, with the steps of an upgrade that needs some by hand.
- **A release for each part**, tagged `host`, `relay` and `web`: the `install.sh` a first install runs, and `latest`, one line with the number of the latest version. A version is put up first, tried, and then made the latest.

To install a version by its number, one that is not the latest yet too:

```sh
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/<part>-v<version>/install.sh | sh -s -- --version <version>   # the Relay and the Web: sudo sh, and their --domain
```

Each `install.sh` fetches that version's package, checks its signature when `minisign` is installed (without it, it says in a line that only HTTPS was relied on), and runs the installer inside the package with the options given.

## Signatures

Every package is signed with [minisign](https://jedisct1.github.io/minisign/). The release key's public half:

```
RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

The installers have it written in them. To check a package by hand, with its `.minisig` beside it:

```sh
minisign -Vm patchbay-host-<version>-linux-x64.tar.gz -P RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

The trusted comment it prints is the package's file name: a signature is of that one package, and does not pass for another.

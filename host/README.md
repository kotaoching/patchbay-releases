# Shayan Host

English | [简体中文](README.zh-CN.md)

Shayan (/ˈʃɑːjɛn/ SHAH-yen; 沙燕, *shāyàn*) is named after the Beijing swallow kite. Its Chinese name is 纸鸢.

The Host runs on your own Computer, installed as your user, without sudo. It consists of two systemd user services, `shayand.service` and `shayan-agents.service`. Only `shayand` is enabled, and it starts the agents service. Phones and browsers reach it through a Relay ([Relay guide](../relay/README.md)), so the Computer opens no inbound port.

It runs from its own package, `shayan-host-<version>-<platform>.tar.gz`, signed and published in the releases of [`kotaoching/shayan-releases`](https://github.com/kotaoching/shayan-releases). The package carries its own Node, `shayand`, the CLI, the agents service and the hook helper, so the Computer does not need Node.

## Requirements

- Debian 12 or 13, or Ubuntu 24.04 LTS or 26.04 LTS, on x86_64 or arm64.
- A systemd user instance for your user: `systemctl --user` must answer. A desktop login or an ssh login provides one.
- `sudo apt install curl`. With `minisign` installed as well (`sudo apt install minisign`), every install and upgrade verifies the package's signature against the release key. Without it, the installer relies on HTTPS alone and says so.
- If you use the Computer over ssh, or want runs to continue after you log out, enable lingering: `loginctl enable-linger` (where that is not allowed, `sudo loginctl enable-linger <user>`). Without it, systemd stops Shayan and its runs when you log out, and after a reboot nothing starts until you log in.
- The coding agents you want to use (Claude Code, Codex or pi; the docs call them providers) installed, logged in and on the `PATH` of the shell you install from. The first install saves that `PATH` in `host.env`, and `shayand` finds the providers with it.

## Install

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/host/install.sh | sh -s -- [--data-dir <dir>]
```

Without `--data-dir`, data is kept in `~/.local/share/shayan`. To install a specific version, including one not yet marked as the latest:

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/host-v<version>/install.sh | sh -s -- --version <version>
```

The script downloads that version's package, verifies its signature with the release key built into the script (when `minisign` is installed), unpacks it and runs the package's own `install.sh` with `bash`. That installer:

1. Runs one install at a time.
2. Puts the version in `~/.local/lib/shayan/versions/<version>/`, read-only.
3. Reads `host.env`. On the first install it writes the file from the options and the current shell's `PATH`, but only after the steps below succeed.
4. Stops `shayand`. The agents service and its runs keep going; no new run can start from this point. If the agents program in this version is different and runs are open, it stops and asks you to end them first (`shayan list` shows them, `shayan stop <runId>` ends one). If there is a database and the version changes, it backs the database up with this version's `shayand` (`<data dir>/backups/shayan-<time>-<version>.db`), and stops if that fails. Whenever it stops here, it starts `shayand` again as it was and changes nothing else.
5. Points `current` at the new version, and `~/.local/bin/shayan` at the `shayan` in `current` (an unrelated file already at that path is left alone).
6. Writes the two units from `host.env`, restarts the agents service if its program changed (it has no runs at this point, see step 4), starts `shayand`, and enables only `shayand`. At boot, its `Wants=` starts the agents service.
7. Counts the install as done only if `shayand` answers on `daemon.sock` within 30 seconds. It checks with the package's `shayan list`, using the credentials from `host.env`, not `SHAYAN_TOKEN` or `SHAYAN_SERVICE_ENV` from your shell.

When it finishes, it prints the installed version, the configuration file, that `shayand` answered, the command to read the logs and the command to roll back. If lingering is off, or `~/.local/bin` is not on your `PATH`, it prints the command that fixes it.

After the first install:

- Join a Relay with `shayan relay join`: run the line the Relay's administrator gives you (they get it from `sudo shayan-relay invite`).
- When `shayan relay` shows `connected`, run `shayan devices pair` to pair a phone.
- `shayan doctor` checks the setup item by item.

## Upgrade

```sh
shayan update                      # the latest version, if it is newer than the installed one
shayan update --version <version>  # that version, older ones too
```

`shayan update` runs the `fetch.sh` of the installed version. If `shayan` itself is broken, run it directly: `~/.local/lib/shayan/current/fetch.sh [--version <version>]`. With `minisign` installed, it trusts only the release keys in the installed version.

During an upgrade `shayand` is down for a few seconds, while the agents service and running runs continue. If the new version's agents program is different and runs are open, the upgrade stops without installing anything (step 4 above): end those runs, then upgrade.

## Moving from Patchbay 0.6.1

Shayan was called Patchbay up to 0.6.1. Patchbay does not update itself into the new name, and Shayan does not read Patchbay's data, so the move is made once by hand:

1. Remove Patchbay. Its services have the names they had in 0.6.1; the first line ends its runs:

   ```sh
   systemctl --user disable --now patchbay-core.service patchbay-supervisor.service
   rm -f ~/.config/systemd/user/patchbay-core.service ~/.config/systemd/user/patchbay-supervisor.service && systemctl --user daemon-reload
   rm -f ~/.local/bin/patchbay
   chmod -R u+w ~/.local/lib/patchbay && rm -r ~/.local/lib/patchbay
   ```

   Its data directory and `~/.config/patchbay/` stay where they are; delete them when you no longer need them.
2. Install Shayan as the Install section above says.
3. Join the Relay again with a new invitation from its administrator, then pair your phone again.

## Configuration

`~/.config/shayan/host.env` (under `$XDG_CONFIG_HOME` when that is set) holds one `KEY=value` per line, without quotes. The first install writes it from its options; after that you edit it yourself. After editing, run `~/.local/lib/shayan/current/install.sh` again: it rewrites the two units from the file, restarts `shayand` and checks that it answers.

Once the file exists, an option that differs from it has no effect, and the installer says so. If a required key is missing, or a line is not one of these keys, the installer stops and tells you which line to add.

| Key | Meaning |
| --- | --- |
| `DATA_DIR` | The data directory, an absolute path. Without `--data-dir` it is `~/.local/share/shayan`. |
| `UPDATE_SOURCE` | Where `shayan update` gets versions from: `https://github.com/kotaoching/shayan-releases/releases/download/host`, or a local directory with the same layout. `shayand` reads `latest` from it once a day and shows a notice in the Web's Settings when a newer version is available. |
| `PATH` | The `PATH` that `shayand` uses to find providers, written into its unit. The first install takes it from your shell. |
| `SERVICE_ENV` | The file where `shayand` writes its credentials. Present only if `SHAYAN_SERVICE_ENV` was set during the install; otherwise the file is `service.env` next to `host.env`. |

The units contain only `--data-dir`, `PATH` and `SHAYAN_UPDATE_SOURCE`, plus `SHAYAN_SERVICE_ENV` when `SERVICE_ENV` is set. They do not load all of `host.env`, because the agents service passes `shayand`'s environment on to every run.

- A new provider, or one that moved: add its directory to the `PATH` line, then run `current/install.sh`.
- Moving the data directory ends all runs, so first make sure none is still needed:

  ```sh
  systemctl --user stop shayand.service shayan-agents.service
  mv <old data dir> <new data dir>
  # set DATA_DIR in host.env to the new directory
  ~/.local/lib/shayan/current/install.sh
  ```

## Roll back

The Computer keeps the version that was replaced, so you can go back to it without a download. The database was backed up when the version changed. Restore that backup first, because the newer `shayand` may have changed the database in a way the older one cannot read. The installer prints these commands with the actual paths on your Computer; run them in order:

```sh
systemctl --user stop shayand.service
~/.local/lib/shayan/versions/<previous version>/bin/shayand --data-dir <data dir> --restore <data dir>/backups/shayan-<time>-<version>.db --force
~/.local/lib/shayan/versions/<previous version>/install.sh
```

The second command uses the previous version's `shayand` to put the database back as it was before the install, and keeps the current one beside it (`shayan-<time>-replaced.db`).

- With `--force`, open runs stay open; they were open in the backup too.
- If the current database cannot be read, all paired devices are revoked and each phone has to pair again.
- Whatever happened after the install, such as new runs or new pairings, is lost.
- If the database did not change between the two versions, the third command alone is enough.

For an older version, install it by its version number (the second command under "Upgrade").

If an install fails (`shayand` does not answer within 30 seconds), the installer exits with status 1 and prints the log command and these commands. It leaves the `shayand` unit as it is (systemd keeps restarting it) and does not switch back on its own.

## Uninstall

There is no uninstall command. The first line ends all runs:

```sh
systemctl --user disable --now shayand.service shayan-agents.service
rm -f ~/.config/systemd/user/shayand.service ~/.config/systemd/user/shayan-agents.service && systemctl --user daemon-reload
rm -f ~/.local/bin/shayan
chmod -R u+w ~/.local/lib/shayan && rm -r ~/.local/lib/shayan
```

The agents unit is not enabled on its own, so `systemctl` reports in the first line that it has no installation config; that is expected. The version directories are read-only, hence the `chmod`.

The data directory and `~/.config/shayan/` (`host.env` and `shayand`'s credentials) are kept. Delete them too if you do not need them; they hold the run history and the paired devices. The Computer also stays on the Relay's allowlist until the Relay's administrator runs `sudo shayan-relay revoke <hostId>`.

## Troubleshooting

- Logs: `journalctl --user -u shayand -u shayan-agents`.
- `shayan doctor` checks the platform, the package, systemd, lingering, the two services, disk space, the update source and each provider, and says what to do about each check that fails.
- `shayand` does not answer: `systemctl --user status shayand.service`. After a failed install, see "Roll back".
- A provider shows as `not found`: it is not on the `PATH` in `host.env` (see "Configuration").
- Shayan stops when you log out, or does not start after a reboot: lingering is off (see "Requirements").
- The installer stops with `systemctl --user does not answer`: log in as this user directly (desktop or ssh) and run it again. A shell entered with `su` or `sudo` cannot reach the user's systemd.

## Without the installer

The unpacked package runs as it is: `<package>/bin/shayand --data-dir <dir>` starts `shayand`, which starts the agents service itself when it is not run from its unit. `<package>/bin/shayan --data-dir <dir> <command>` is its CLI.

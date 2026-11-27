# 纸鸢 Host

[English](README.md) | 简体中文

纸鸢的英文名是 Shayan（/ˈʃɑːjɛn/，读作 SHAH-yen），取自北京的沙燕风筝。

Host 装在你自己的电脑上，以你的用户身份安装，不需要 sudo。它由两个 systemd 用户服务组成：`shayand.service` 和 `shayan-agents.service`。只有 `shayand` 设为开机启动，由它启动 agents 服务。手机和浏览器通过 Relay 连接到它（[Relay 说明](../relay/README.zh-CN.md)），所以电脑不需要开放任何入站端口。

Host 使用自己的安装包运行：`shayan-host-<版本>-<平台>.tar.gz`，经过签名，发布在 [`kotaoching/shayan-releases`](https://github.com/kotaoching/shayan-releases) 的发行版中。安装包自带 Node、`shayand`、命令行工具、agents 服务和 hook 程序，电脑上不需要另装 Node。

## 系统要求

- Debian 12 或 13，或 Ubuntu 24.04 LTS 或 26.04 LTS，x86_64 或 arm64。
- 当前用户有 systemd 用户实例：`systemctl --user` 能正常响应。桌面登录和 ssh 登录都会提供。
- `sudo apt install curl`。如果同时安装了 `minisign`（`sudo apt install minisign`），每次安装和升级都会用发布公钥验证安装包的签名；没有安装时只依赖 HTTPS，安装程序会给出提示。
- 如果通过 ssh 使用这台电脑，或者希望退出登录后任务（run，Agent 的一次运行）继续进行，需要开启 linger：`loginctl enable-linger`（没有权限时用 `sudo loginctl enable-linger <用户>`）。不开启的话，退出登录时 systemd 会停止纸鸢和它的任务，电脑重启后也要等这个用户登录才会启动。
- 要用的编程 Agent（Claude Code、Codex 或 pi，文档中称为 provider）已经安装、登录，并且在运行安装命令的 shell 的 `PATH` 中。首次安装会把这个 `PATH` 写入 `host.env`，`shayand` 用它查找 provider。

## 安装

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/host/install.sh | sh -s -- [--data-dir <目录>]
```

不指定 `--data-dir` 时，数据保存在 `~/.local/share/shayan`。安装指定版本（包括尚未设为最新版本的）：

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/host-v<版本>/install.sh | sh -s -- --version <版本>
```

这个脚本会下载该版本的安装包，在装有 `minisign` 时用脚本内置的发布公钥验证签名，解压后用 `bash` 运行安装包里的 `install.sh`。安装程序会依次：

1. 确保同一时间只有一个安装在进行。
2. 把这个版本放到 `~/.local/lib/shayan/versions/<版本>/`，设为只读。
3. 读取 `host.env`。首次安装时，根据选项和当前 shell 的 `PATH` 生成这个文件，但要等下面各步都成功后才写入。
4. 停止 `shayand`。agents 服务和其中的任务继续进行，从这一步起不会再有新的任务启动。如果这个版本的 agents 程序不同，而且还有任务在进行，安装会停止，并提示先结束这些任务（`shayan list` 查看，`shayan stop <runId>` 结束一个）。如果已有数据库且版本发生变化，会用这个版本的 `shayand` 备份数据库（`<数据目录>/backups/shayan-<时间>-<版本>.db`），备份失败也会停止。在这一步停止时，`shayand` 会按原样重新启动，其他内容都不改动。
5. 把 `current` 指向新版本，并让 `~/.local/bin/shayan` 指向 `current` 中的 `shayan`（该位置已有其他文件时不会改动它）。
6. 根据 `host.env` 写入两个 unit；agents 程序有变化时重启 agents 服务（此时它没有任务，见第 4 步）；启动 `shayand`，并且只把 `shayand` 设为开机启动，开机时由它的 `Wants=` 启动 agents 服务。
7. `shayand` 在 30 秒内通过 `daemon.sock` 响应，才算安装成功。检查时使用安装包中的 `shayan list` 和 `host.env` 中的凭据，不使用当前 shell 里的 `SHAYAN_TOKEN`、`SHAYAN_SERVICE_ENV`。

安装完成后会输出：安装的版本、配置文件、`shayand` 已响应、查看日志的命令和回滚的命令。如果 linger 没有开启，或者 `~/.local/bin` 不在 `PATH` 中，也会给出对应的命令。

首次安装后：

- 用 `shayan relay join` 加入一个 Relay：运行 Relay 管理员给你的那一行命令（管理员通过 `sudo shayan-relay invite` 生成）。
- `shayan relay` 显示 `connected` 后，运行 `shayan devices pair` 配对手机。
- `shayan doctor` 会逐项检查安装状态。

## 升级

```sh
shayan update                      # 升级到最新版本（比已安装的新时才安装）
shayan update --version <版本>      # 安装指定版本，也可以是更旧的版本
```

`shayan update` 运行的是已安装版本中的 `fetch.sh`。如果 `shayan` 本身坏了，可以直接运行：`~/.local/lib/shayan/current/fetch.sh [--version <版本>]`。装有 `minisign` 时，它只信任已安装版本中的发布公钥。

升级期间 `shayand` 会停几秒，agents 服务和进行中的任务不受影响。如果新版本的 agents 程序不同，而且还有任务在进行，升级会停止，不安装任何内容（见上面第 4 步）：结束这些任务后再升级。

## 从 Patchbay 0.6.1 换过来

纸鸢到 0.6.1 为止叫 Patchbay。Patchbay 不会自己升级成新名字，纸鸢也不读 Patchbay 的数据，所以要手工换一次：

1. 卸掉 Patchbay。它的服务用的还是 0.6.1 的名字；第一行会结束它的运行：

   ```sh
   systemctl --user disable --now patchbay-core.service patchbay-supervisor.service
   rm -f ~/.config/systemd/user/patchbay-core.service ~/.config/systemd/user/patchbay-supervisor.service && systemctl --user daemon-reload
   rm -f ~/.local/bin/patchbay
   chmod -R u+w ~/.local/lib/patchbay && rm -r ~/.local/lib/patchbay
   ```

   它的数据目录和 `~/.config/patchbay/` 留在原处，不再需要时再删。
2. 照上面的「安装」一节装上纸鸢。
3. 向 Relay 的管理员要一个新的邀请，重新加入 Relay，再重新配对手机。

## 配置

配置文件是 `~/.config/shayan/host.env`（设置了 `$XDG_CONFIG_HOME` 时在其下），每行一个 `KEY=值`，值不加引号。首次安装根据选项生成，之后由你自己修改。修改后再运行一次 `~/.local/lib/shayan/current/install.sh`：它会按文件重写两个 unit，重启 `shayand`，并确认它能响应。

配置文件存在后，与它不一致的命令行选项不会生效，安装程序会给出提示。缺少必需的键，或者某一行不是下面这些键时，安装程序会停止，并说明需要添加哪一行。

| 键 | 含义 |
| --- | --- |
| `DATA_DIR` | 数据目录，绝对路径。不指定 `--data-dir` 时为 `~/.local/share/shayan`。 |
| `UPDATE_SOURCE` | `shayan update` 获取版本的位置：`https://github.com/kotaoching/shayan-releases/releases/download/host`，或本机上布局相同的目录。`shayand` 每天从这里读取一次 `latest`，有更新的版本时在 Web 的设置页中提示。 |
| `PATH` | `shayand` 查找 provider 时使用的 `PATH`，写入它的 unit。首次安装时取自当前 shell。 |
| `SERVICE_ENV` | `shayand` 写入凭据的文件。只有安装时设置了 `SHAYAN_SERVICE_ENV` 才会有这一行；否则这个文件是 `host.env` 旁边的 `service.env`。 |

两个 unit 中只包含 `--data-dir`、`PATH` 和 `SHAYAN_UPDATE_SOURCE`，设置了 `SERVICE_ENV` 时还有 `SHAYAN_SERVICE_ENV`。unit 不会加载整个 `host.env`，因为 agents 服务会把 `shayand` 的环境变量传给每个任务。

- 新装了 provider，或者 provider 换了位置：把它所在的目录加到 `PATH` 那一行，再运行 `current/install.sh`。
- 迁移数据目录会结束所有任务，请先确认没有需要保留的任务：

  ```sh
  systemctl --user stop shayand.service shayan-agents.service
  mv <原数据目录> <新数据目录>
  # 把 host.env 中的 DATA_DIR 改为新目录
  ~/.local/lib/shayan/current/install.sh
  ```

## 回滚

电脑上会保留被替换的上一个版本，回滚不需要重新下载。切换版本时数据库已经备份过；回滚时要先恢复这份备份，因为新版本的 `shayand` 可能已经修改了数据库，旧版本无法读取。安装程序会输出下面这些命令，并带上这台电脑上的实际路径，按顺序运行即可：

```sh
systemctl --user stop shayand.service
~/.local/lib/shayan/versions/<上一个版本>/bin/shayand --data-dir <数据目录> --restore <数据目录>/backups/shayan-<时间>-<版本>.db --force
~/.local/lib/shayan/versions/<上一个版本>/install.sh
```

第二条命令用上一个版本的 `shayand` 把数据库恢复到这次安装之前的状态，当前的数据库会保留在旁边（`shayan-<时间>-replaced.db`）。

- 使用 `--force` 时，进行中的任务仍保持进行中，它们在备份中也是进行中的。
- 如果当前数据库无法读取，所有已配对的设备都会被撤销，手机需要重新配对。
- 安装之后发生的事情，比如新的任务、新的配对，都会丢失。
- 如果两个版本之间数据库没有变化，只运行第三条命令即可。

更早的版本可以按版本号安装（见"升级"中的第二条命令）。

安装失败（`shayand` 在 30 秒内没有响应）时，安装程序以状态 1 退出，并输出查看日志的命令和上面这些命令。它不会改动 `shayand` 的 unit（systemd 会继续重启它），也不会自动切回旧版本。

## 卸载

没有卸载命令。第一行会结束所有任务：

```sh
systemctl --user disable --now shayand.service shayan-agents.service
rm -f ~/.config/systemd/user/shayand.service ~/.config/systemd/user/shayan-agents.service && systemctl --user daemon-reload
rm -f ~/.local/bin/shayan
chmod -R u+w ~/.local/lib/shayan && rm -r ~/.local/lib/shayan
```

agents 服务的 unit 没有单独设为开机启动，所以第一行中 `systemctl` 会提示它没有安装配置，这是正常的。版本目录是只读的，所以要先 `chmod`。

数据目录和 `~/.config/shayan/`（`host.env` 和 `shayand` 的凭据）会保留。如果不再需要，也可以删除，任务的历史记录和已配对的设备都在其中。这台电脑还会留在 Relay 的允许列表中，需要 Relay 管理员运行 `sudo shayan-relay revoke <hostId>` 移除。

## 故障排查

- 日志：`journalctl --user -u shayand -u shayan-agents`。
- `shayan doctor` 会检查平台、安装包、systemd、linger、两个服务、磁盘空间、更新源和每个 provider，并说明每个未通过的检查项该怎么处理。
- `shayand` 没有响应：`systemctl --user status shayand.service`；安装失败后见"回滚"。
- provider 显示为 `not found`：它不在 `host.env` 的 `PATH` 中（见"配置"）。
- 退出登录后纸鸢停止，或者重启后没有启动：linger 没有开启（见"系统要求"）。
- 安装程序停止并提示 `systemctl --user does not answer`：直接以这个用户登录（桌面或 ssh）后再运行。通过 `su` 或 `sudo` 进入的 shell 无法连接这个用户的 systemd。

## 不使用安装程序

解压后的安装包可以直接运行：`<安装包>/bin/shayand --data-dir <目录>` 启动 `shayand`，不是通过它的 unit 运行时，它会自己启动 agents 服务。`<安装包>/bin/shayan --data-dir <目录> <命令>` 是它的命令行工具。

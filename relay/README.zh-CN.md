# 纸鸢 Relay

[English](README.md) | 简体中文

纸鸢的英文名是 Shayan（/ˈʃɑːjɛn/，读作 SHAH-yen），取自北京的沙燕风筝。

Relay 装在一台有公网地址的服务器上，手机通过它连接电脑上的 `shayand`。`shayand` 主动连接 Relay，所以电脑不需要开放任何入站端口。

Relay 只转发经 Noise 加密的流量，不提供网页：手机打开的 Web 有自己的域名和安装方式（[Web 说明](../web/README.zh-CN.md)）。Relay 能看到的是元数据，看不到内容：哪台电脑在线、电脑和手机的地址、什么时候连接、连接多久、每条通道传输了多少字节、离线副本的大小。它没有数据库，只在状态目录中保存一个小文件 `hosts.json`：允许接入的电脑列表，每台电脑记录 `hostId`、公钥和加入时间。这个文件只由 Relay 自己写入。

Relay 使用自己的安装包运行：`shayan-relay-<版本>-<平台>.tar.gz`，经过签名，发布在 [`kotaoching/shayan-releases`](https://github.com/kotaoching/shayan-releases) 的发行版中。安装包自带 Node，服务器上不需要另装 Node。同一个安装包有两种运行方式：系统服务 `shayan-relay.service`（systemd 以动态用户运行它，开机自动启动），或 Docker 容器 `shayan-relay`（镜像在服务器上由安装包构建）。

## 系统要求

- 一个公网地址，开放入站 TCP 80 和 443 端口；一个域名，A 记录（有 IPv6 时再加 AAAA 记录）指向这个地址。
- Debian 12 或 13，或 Ubuntu 24.04 LTS 或 26.04 LTS，x86_64 或 arm64；root 或可以使用 `sudo` 的用户。
- `sudo apt install caddy curl`。如果同时安装了 `minisign`，每次安装和升级都会用发布公钥验证安装包的签名；没有安装时只依赖 HTTPS。Caddy 作为系统服务监听 80 和 443 端口，`/etc/caddy/Caddyfile` 中要有一行 `import sites/*.caddy`（Web 也需要这一行，只加一次）。Relay 的站点配置需要你自己编写，放在 `/etc/caddy/sites/shayan-relay.caddy`（端口不是 8787 时改成实际端口）：

  ```sh
  sudo mkdir -p /etc/caddy/sites
  printf '<Relay 域名> {\n\treverse_proxy 127.0.0.1:8787\n}\n' | sudo tee /etc/caddy/sites/shayan-relay.caddy
  sudo systemctl reload caddy
  ```

  Caddy 会自动申请证书、转发 WebSocket、保留 `Host` 请求头，并把手机的地址追加到 `X-Forwarded-For` 末尾，`--trust-proxy` 会读取它。安装程序不会改动 Caddy。站点或证书还没准备好时也可以安装，安装程序只会提示 HTTPS 暂时无法访问。
- 系统服务方式需要 systemd。Docker 方式需要 Docker Engine 和 Compose 插件（`docker compose version` 能正常输出）：Debian 13 用 `sudo apt install docker.io docker-cli docker-compose`，Ubuntu 用 `sudo apt install docker.io docker-compose-v2`。Debian 12 的官方源中没有 Compose 插件，请使用 Docker 官方源（见 docs.docker.com 上的 "Install Docker Engine on Debian"）。

## 安装

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay/install.sh | sudo sh -s -- --domain <Relay 域名> [--port <端口>] [--docker <目录>]
```

不加 `--docker` 时安装为系统服务，安装在 `/opt/shayan-relay`；加上时以容器运行，安装在指定目录。`--port` 默认为 8787。安装指定版本（包括尚未设为最新版本的）：

```sh
curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay-v<版本>/install.sh | sudo sh -s -- --version <版本> --domain <Relay 域名>
```

这个脚本会下载该版本的安装包，在装有 `minisign` 时用脚本内置的发布公钥验证签名，解压后用 `bash` 运行安装包里的 `install.sh`。安装程序会依次：

1. 确保同一时间只有一个安装在进行。
2. 检查所选方式需要的组件是否齐全，并确认另一种方式的 Relay 没有在运行（两种方式使用同一个端口）。无法确认时停止，不做任何改动。
3. 把这个版本放到 `<目录>/versions/<版本>/`，设为只读。Docker 方式会在这里用安装包中的 `Dockerfile` 构建镜像 `shayan-relay:<版本>`（基础镜像按 digest 固定）。构建失败时停止，正在运行的 Relay 不受影响。
4. 首次安装时，根据选项写入 `relay.env`。
5. 把 `current` 指向新版本。
6. 系统服务：按 `DIR` 写入 unit，然后执行 `daemon-reload`、`restart` 和 `enable`。Docker：把安装包中的 `compose.yaml` 放到 `<目录>`，执行 `docker compose up -d`。
7. 以 `Host: <域名>` 请求 `127.0.0.1:<端口>/relay/info`，30 秒内有响应才算安装成功。

安装完成后会输出：安装的版本、配置文件、Relay 已响应、查看日志的命令和回滚的命令，以及从这台服务器能否访问 `https://<域名>/relay/info`（无法访问不算安装失败）。首次安装后，还会说明如何让电脑接入（见下面的"允许哪些电脑接入"）。

Relay 重启后（升级、服务器重启），离线副本要等各台电脑重新连接后才会恢复；尚未使用的邀请也会失效，需要重新生成。

## 升级

```sh
sudo shayan-relay update                      # 升级到最新版本（比已安装的新时才安装）
sudo shayan-relay update --version <版本>      # 安装指定版本，也可以是更旧的版本
```

`/usr/local/bin/shayan-relay` 会把 `update` 交给已安装版本中的 `fetch.sh` 执行。Docker 方式也是在服务器上执行，而不是在容器里。如果这个命令本身坏了，可以直接运行：`sudo <目录>/current/fetch.sh [--version <版本>]`。装有 `minisign` 时，它只信任已安装版本中的发布公钥。

## 从 Patchbay 0.6.1 换过来

纸鸢到 0.6.1 为止叫 Patchbay。Patchbay 不会自己升级成新名字，纸鸢也不读 Patchbay 的数据，所以要手工换一次：

1. 照下面的「卸载」一节卸掉 Patchbay 的 Relay，每个名字里的 `shayan` 换成 `patchbay`，包括它的 Caddy 站点文件 `/etc/caddy/sites/patchbay-relay.caddy`。它的 `relay.env` 和状态目录留在原处，不再需要时再删。
2. 如果 Relay 换到新域名，先把域名指向这台服务器。
3. 照上面的「安装」一节装上纸鸢的 Relay，再给每台电脑重新发邀请（`sudo shayan-relay invite`）。

## 配置

配置文件是 `/etc/shayan-relay/relay.env`，每行一个 `KEY=值`。首次安装根据选项生成，之后由你自己修改。修改后再运行一次 `sudo <DIR>/current/install.sh`：它会重启 Relay，并确认它能响应。配置文件存在后，与它不一致的命令行选项不会生效，安装程序会给出提示。缺少某个键时，安装程序会停止，并说明需要添加哪一行。系统服务的 unit（`EnvironmentFile`）和容器的 `compose.yaml`（`env_file`）读取的也是这个文件。

| 键 | 含义 |
| --- | --- |
| `WAY` | `service`（系统服务）或 `docker`（容器） |
| `DIR` | 安装目录，绝对路径：系统服务默认为 `/opt/shayan-relay`，Docker 方式为 `--docker` 指定的目录 |
| `DOMAIN` | Relay 的域名。`https://<DOMAIN>` 是它的源，每台电脑的签名都与它绑定 |
| `PORT` | 它在 `127.0.0.1` 上监听的端口，Caddy 站点反向代理到这里；默认为 8787 |
| `UPDATE_SOURCE` | 升级时获取版本的位置：`https://github.com/kotaoching/shayan-releases/releases/download/relay`，或本机上布局相同的目录 |

- 更换端口：修改 `PORT` 后重新运行安装程序；把 Caddy 站点中的那一行改为 `reverse_proxy 127.0.0.1:<新端口>`，再执行 `sudo systemctl reload caddy`。
- 更换域名：修改 `DOMAIN` 后重新运行安装程序。允许列表保持不变，因为它按电脑的密钥识别，而不是按域名。把 Caddy 站点改为新域名。在每台电脑上运行 `shayan relay set https://<新域名>`，不需要新的邀请。每部手机都要重新配对，因为配对邀请中写明了 Relay 的地址。

## 回滚

服务器上会保留被替换的上一个版本，回滚不需要重新下载：

```sh
sudo <目录>/versions/<上一个版本>/install.sh
```

更早的版本可以按版本号安装（见"升级"中的第二条命令）。允许列表不随版本变化，也没有数据库需要恢复。安装失败（30 秒内没有响应）时，安装程序以状态 1 退出，并输出查看日志的命令和回滚的命令。它不会改动当前的服务，也不会自动切回旧版本。

## 允许哪些电脑接入

允许列表只能由正在运行的 Relay 修改，加入和移除都会立即生效，不需要重启。`/usr/local/bin/shayan-relay` 以 root 身份与正在运行的 Relay 通信（系统服务通过状态目录中的 `admin.sock`，容器通过 `docker exec`）：

| 操作 | 命令 |
| --- | --- |
| 生成邀请 | `sudo shayan-relay invite` |
| 列出电脑 | `sudo shayan-relay hosts` |
| 移除电脑 | `sudo shayan-relay revoke <hostId>` |

每台电脑都是独立的，可以属于不同的人：它们互相看不到，也不能添加或移除彼此。允许列表中已有的 `hostId` 不会被另一把密钥顶替。只有管理 Relay 的人可以添加或移除电脑。

- **添加电脑**：生成一个邀请，它会输出一行 `shayan relay join --url https://<域名> --code <邀请码>`。一个邀请只用于一台电脑，只能使用一次，5 分钟内有效，只保存在 Relay 的内存中。在那台电脑上运行这一行：它会把这台电脑的 `hostId` 和公钥发送给 Relay，然后把这台电脑的 Relay 设为 `https://<域名>`。如果没有收到响应，在同一台电脑上再运行一次即可。这一行在 5 分钟内是一次性凭据，只在要加入的那台电脑上使用。`shayan relay` 显示 `connected` 后，运行 `shayan devices pair`。
- **列出电脑**：每台一行，格式为 `<hostId> <指纹> <加入时间|-> <on|off>`。在电脑上运行 `shayan relay key` 可以查看它的 `hostId` 和指纹，用来对照。
- **移除电脑**：它会立即从允许列表中删除并断开连接，它的手机经这台 Relay 的连接也会断开，之后它会收到 `not_allowed`。它的离线副本也会立即删除。
- **电脑的密钥变了**（数据目录重建过）：先 `revoke` 列表中它原来的记录，再重新邀请。

## 状态目录与备份

需要保留的只有状态目录中的 `hosts.json`（权限 0600，只由 Relay 写入）。安装包、镜像和 unit 都可以重新安装；允许列表丢失后，每台电脑都要重新邀请。

| 方式 | 状态目录 | 所有者 |
| --- | --- | --- |
| 系统服务 | `/var/lib/shayan-relay/`（使用 `DynamicUser` 时实际位置为 `/var/lib/private/shayan-relay/`） | systemd 为它创建的动态用户，权限 0700 |
| Docker | `<目录>/state/`，在容器中挂载为 `/var/lib/shayan-relay` | uid 1000（镜像中的用户），权限 0700 |

备份时复制 `hosts.json` 即可。恢复时先停止 Relay，以 0600 权限和与状态目录相同的所有者放回（`sudo install -m 600 -o "$(sudo stat -L -c %u <状态目录>)" -g "$(sudo stat -L -c %g <状态目录>)" hosts.json <状态目录>/`），再启动 Relay。

## 两种运行方式的细节

- 系统服务：unit 是安装包中的 `shayan-relay.service`，把其中的 `@DIR@` 替换为 `DIR` 后写入 `/etc/systemd/system/`。其中包括 `EnvironmentFile=/etc/shayan-relay/relay.env`、`DynamicUser=yes`、`StateDirectory=shayan-relay`（它唯一可写的位置）、`ProtectSystem=strict`、`ProtectHome` 等加固选项，以及 `Restart=always`。日志：`journalctl -u shayan-relay`。不要把 `DIR` 放在 `/home` 下（受 `ProtectHome` 限制）。
- Docker：每个版本一个镜像 `shayan-relay:<版本>`，在这台服务器上从 `<目录>/versions/<版本>/` 构建，只包含这个安装包，以 uid 1000 运行，不使用镜像仓库。`<目录>/compose.yaml` 就是安装包中的那一份：容器名为 `shayan-relay`，`restart: unless-stopped`，只读，唯一的卷是 `<目录>/state/`，只发布到 `127.0.0.1:<端口>`，日志使用 `json-file`（10 MB × 3）。需要自己运行 Compose 时，使用 `sudo env -u DOMAIN -u PORT docker compose --env-file /etc/shayan-relay/relay.env -f <目录>/compose.yaml <命令>`：对于 compose 文件中的 `${DOMAIN}` 和 `${PORT}`，Compose 会优先使用调用环境中的同名变量，其次才是 `--env-file`。日志：`docker compose -f <目录>/compose.yaml logs`。会保留 `current` 和被替换版本的镜像，其他镜像随版本目录一起删除。

日志中不包含帧和 blob 的内容、离线副本的读取记录和来源地址，也不包含签名和邀请码。

## 切换运行方式

两种方式使用同一个端口，不能同时运行。安装程序不会自动切换，步骤如下：

1. 停止当前的方式。系统服务：`sudo systemctl disable --now shayan-relay.service`。Docker：`sudo env -u DOMAIN -u PORT docker compose --env-file /etc/shayan-relay/relay.env -f <目录>/compose.yaml stop`。
2. 修改 `relay.env` 中的 `WAY` 和 `DIR`。
3. 迁移允许列表，这样电脑不需要重新加入。切换到 Docker：`sudo install -d -m 700 -o 1000 -g 1000 <目录>/state && sudo install -m 600 -o 1000 -g 1000 /var/lib/shayan-relay/hosts.json <目录>/state/`。切换到系统服务：安装完成后，按"状态目录与备份"中的方法恢复。
4. 用新的方式安装：`curl -fsSL https://github.com/kotaoching/shayan-releases/releases/download/relay-v<版本>/install.sh | sudo sh -s -- --version <版本>`（已安装的版本在原来的目录中，新目录中还没有）。

## 卸载

系统服务：

```sh
sudo systemctl disable --now shayan-relay.service
sudo rm -f /etc/systemd/system/shayan-relay.service /usr/local/bin/shayan-relay && sudo systemctl daemon-reload
sudo rm -r /opt/shayan-relay
```

Docker：

```sh
sudo env -u DOMAIN -u PORT docker compose --env-file /etc/shayan-relay/relay.env -f <目录>/compose.yaml down
sudo docker image ls --format '{{.Repository}}:{{.Tag}}' shayan-relay | xargs -r sudo docker image rm
sudo rm -f /usr/local/bin/shayan-relay
sudo rm -r <目录>/versions && sudo rm -f <目录>/compose.yaml <目录>/current
```

`relay.env` 和状态目录（允许列表）会保留。如果不再需要，也可以删除 `/etc/shayan-relay/` 和 `/var/lib/shayan-relay/`（或 `<目录>/state/`）。删除 `/etc/caddy/sites/shayan-relay.caddy`，再执行 `sudo systemctl reload caddy`。在每台电脑上运行 `shayan relay clear`。

## 故障排查

- 日志：见上面每种方式各自的命令。
- 绕过 Caddy 直接访问 Relay：`curl -H 'Host: <域名>' http://127.0.0.1:<端口>/relay/info`。Relay 只响应 `Host` 为自己域名的请求。
- HTTPS 无法访问：检查 `/etc/caddy/sites/shayan-relay.caddy`、Caddyfile 中的 import 行、域名的 A 记录、80 和 443 端口，以及 Caddy 的证书（`journalctl -u caddy`）。
- 在电脑上运行 `shayan relay`：`lastError` 显示最近一次失败的原因。`not_allowed`：这台电脑不在允许列表中，或者列表中的公钥不是它当前的公钥；用 `sudo shayan-relay hosts` 检查后重新邀请。`protocol`：Relay 和这台电脑的版本无法互通，先升级版本较旧的一方。

## 不使用安装程序

解压后的安装包可以直接运行，前面放一个处理 TLS 的反向代理：

```
shayan-relay serve --origin https://<域名> --state-dir <目录> --listen 127.0.0.1:<端口> --trust-proxy
```

`--state-dir` 是它保存允许列表和 `admin.sock` 的目录（权限 0700，属于运行它的用户）。新的目录中允许列表为空，在有电脑凭邀请加入之前，任何电脑都无法接入。此时的命令是 `shayan-relay invite --state-dir <目录>` 等，以运行它的用户执行。

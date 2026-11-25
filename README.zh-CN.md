# Patchbay releases

[English](README.md) | 简体中文

这个仓库存放 Patchbay 的发布安装包、签名和安装脚本，不包含源代码。

Patchbay 让你在自己的电脑上运行编程 Agent（Claude Code、Codex、pi），并在手机或浏览器上使用它们：启动 Agent、查看它在做什么、处理它的授权请求、接管它的终端。

> 这里还没有发布任何版本，第一个版本会是 0.6.1 之后的下一个版本。下面各组件的说明，会在该组件的首个版本设为最新版本后可用。

## 组件

Patchbay 由三个组件组成，各组件独立打包，版本号也各自独立。

| 组件 | 作用 | 安装包 |
| --- | --- | --- |
| Host | 安装在运行编程 Agent 的电脑上，以运行 Agent 的用户身份安装 | `patchbay-host-<版本>-linux-<x64 或 arm64>.tar.gz` |
| Relay | 运行在有公网地址的服务器上，在手机和电脑之间转发连接 | `patchbay-relay-<版本>-linux-<x64 或 arm64>.tar.gz` |
| Web | 运行在有公网地址的服务器上，提供在手机或浏览器中打开的界面 | `patchbay-web-<版本>-any.tar.gz` |

Relay 和 Web 可以部署在同一台服务器上，也可以分开部署。各组件的说明包括系统要求、配置、升级、回滚、卸载和故障排查：

- [Host 说明](host/README.zh-CN.md)
- [Relay 说明](relay/README.zh-CN.md)：系统服务或 Docker
- [Web 说明](web/README.zh-CN.md)

## 安装

需要准备：

- 一台（或两台）服务器，运行 Debian 12 或 13，或 Ubuntu 24.04 或 26.04，x86_64 或 arm64，开放 TCP 80 和 443 端口，并有两个指向它的域名：一个给 Relay，一个给 Web。
- 一台运行上述系统之一的电脑，以运行 Agent 的用户身份操作，已安装并登录 Claude Code、Codex 或 pi，并且它们在这个 shell 的 `PATH` 中。首次安装会为 Agent 保存这个 `PATH`。

下面的命令使用示例域名。请在运行命令的 shell 中，把 `RELAY_DOMAIN` 和 `WEB_DOMAIN` 设为你自己的域名。

**1. 每台服务器。** 在运行 Relay 或 Web 的每台服务器上安装 Caddy，并让它加载 `/etc/caddy/sites/` 中的站点。缺少这一行时，Web 的安装程序会停止。

```sh
sudo apt install caddy curl minisign           # minisign 是可选的；安装后，每次安装都会验证安装包的签名
grep -qxF 'import sites/*.caddy' /etc/caddy/Caddyfile || echo 'import sites/*.caddy' | sudo tee -a /etc/caddy/Caddyfile
sudo mkdir -p /etc/caddy/sites
```

**2. Relay**，在它的服务器上。先为 Relay 配置 Caddy，再安装：

```sh
RELAY_DOMAIN=relay.example.com
printf '%s {\n\treverse_proxy 127.0.0.1:8787\n}\n' "$RELAY_DOMAIN" | sudo tee /etc/caddy/sites/patchbay-relay.caddy
sudo systemctl reload caddy
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/relay/install.sh | sudo sh -s -- --domain "$RELAY_DOMAIN"   # 加上 --docker /opt/patchbay-relay-docker 则在 Docker 中运行
```

**3. Web**，在它的服务器上。它的安装程序会自己配置 Caddy：

```sh
WEB_DOMAIN=app.example.com
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/web/install.sh | sudo sh -s -- --domain "$WEB_DOMAIN"
```

**4. Host**，在电脑上，以那个用户身份操作（只有 `apt` 需要 sudo）：

```sh
sudo apt install curl minisign                 # minisign 是可选的，与服务器相同
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/host/install.sh | sh -s --
loginctl enable-linger                         # 通过 ssh 使用，或希望退出登录后 Agent 继续运行时需要
patchbay roots add "$HOME/projects"            # 允许 Agent 工作的工作区根目录；换成你自己的目录，可以添加多个
```

如果不允许运行 `loginctl enable-linger`，可以请管理员运行 `sudo loginctl enable-linger <用户>`。`patchbay doctor` 会逐项检查这台电脑，并说明需要修复什么。

**5. 加入 Relay。** Host 安装好之后，在 Relay 的服务器上生成邀请：

```sh
sudo patchbay-relay invite
```

它会输出一行带邀请码的命令：`patchbay relay join --url https://… --code …`。请在 5 分钟内到电脑上运行这一行，它只能使用一次。

**6. 配对手机。** 电脑上的 `patchbay relay` 显示 `connected` 后，在电脑上运行 `patchbay devices pair`，它会以二维码显示配对码。在手机上打开 Web 的地址（例如 `https://app.example.com`），添加到主屏幕，然后在其中扫描这个配对码。手机随后会显示一个确认码：在电脑的终端里，`patchbay devices pair` 提示的位置输入它。

## 升级

如果某台机器运行的是 0.6.1 或更早的版本，它是从原来的下载地址安装的，下面的命令仍会去那里查找。请按这里发布的第一个版本的发行说明中的升级步骤，完成一次迁移；之后再用下面的命令升级。

在各自的机器上运行：有比已安装版本更新的最新版本时安装它；加上 `--version <版本>` 时安装指定版本，也可以是更旧的版本。

```sh
patchbay update                                # 电脑
sudo patchbay-relay update                     # Relay
sudo /srv/patchbay-web/current/fetch.sh        # Web
```

## 发行版

- **每个版本一个发行版**，Git 标记为 `<组件>-v<版本>`（例如 `host-v0.7.0`）。其中包括安装包、每个安装包的签名（`.minisig`）、这个版本的 `install.sh` 和说明。它的发行说明介绍这个版本的改动，包括需要手动操作的升级步骤。
- **每个组件一个发行版**，Git 标记为 `host`、`relay` 或 `web`。其中是首次安装使用的 `install.sh`，以及 `latest`：只有一行，内容是最新版本号。新版本会先发布，在真实机器上测试后，再设为最新版本。

按版本号安装（包括尚未设为最新版本的），把 `<组件>` 和 `<版本>` 换成实际的值：

```
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/<组件>-v<版本>/install.sh | sh -s -- --version <版本>
```

Relay 和 Web 需要用 `sudo sh`，并加上 `--domain`。每个 `install.sh` 都会下载该版本的安装包，在装有 `minisign` 时验证签名（没有安装时只依赖 HTTPS，并给出提示），然后用你提供的选项运行安装包里的安装程序。

## 签名

每个安装包都用 [minisign](https://jedisct1.github.io/minisign/) 签名。公钥：

```
RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

安装脚本已经内置了这个公钥，并会自动完成下面的检查。如果要手动验证安装包，把它的 `.minisig` 放在同一目录下，运行：

```sh
minisign -Vm patchbay-host-0.7.0-linux-x64.tar.gz -P RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

minisign 只验证文件内容，不验证文件名，所以改名后的文件仍能通过验证。验签成功后，请确认输出中的可信注释与安装包文件名完全一致。

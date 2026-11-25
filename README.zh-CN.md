# Patchbay releases

[English](README.md) | 简体中文

这个仓库存放 Patchbay 的发布安装包、签名和安装脚本，不包含源代码。

Patchbay 让你在自己的电脑上运行编程 agent，并在手机或浏览器上使用它们：启动 agent、查看它在做什么、处理它的授权请求、接管它的终端。手机通过 Relay 连接。

> 这里还没有发布任何版本，第一个版本会是 0.6.1 之后的下一个版本。在此之前，下面的命令无法使用，各组件说明的链接也是空的。

## 组件

每个组件安装在自己的机器上，使用各自的安装包，版本号也各自独立。

| 组件 | 运行在哪里 | 安装包 |
| --- | --- | --- |
| Host | 运行 agent 的每台电脑，以运行 agent 的用户身份安装 | `patchbay-host-<版本>-linux-<x64 或 arm64>.tar.gz` |
| Relay | 有公网地址的服务器，手机通过它连接电脑 | `patchbay-relay-<版本>-linux-<x64 或 arm64>.tar.gz` |
| Web | 有公网地址的服务器，手机打开的网页 | `patchbay-web-<版本>-any.tar.gz` |

各组件的说明包括系统要求、配置、升级、回滚、卸载和故障排查：

- [Host 说明](host/README.zh-CN.md)
- [Relay 说明](relay/README.zh-CN.md)：系统服务或 Docker
- [Web 说明](web/README.zh-CN.md)

## 安装

需要准备：

- 一台服务器，运行 Debian 12 或 13，或 Ubuntu 24.04 或 26.04（x86_64 或 arm64），开放 TCP 80 和 443 端口，并有两个指向它的域名：一个给 Relay，一个给 Web。两者也可以分别放在不同的服务器上。
- 一台运行上述系统之一的电脑，以运行 agent 的用户身份操作，已安装并登录 Claude Code、Codex 或 pi，并且它们在这个 shell 的 `PATH` 中。首次安装会为 agent 保存这个 `PATH`。

**1. 服务器。** 首次需要安装 Caddy 并编写 Relay 的站点配置（Web 的安装程序会自己写入它的站点）：

```sh
sudo apt install caddy curl                    # 同时安装 minisign 时，会验证每个安装包的签名
# 在 /etc/caddy/Caddyfile 中加入这一行（只需一次）：import sites/*.caddy
sudo mkdir -p /etc/caddy/sites
printf '<Relay 域名> {\n\treverse_proxy 127.0.0.1:8787\n}\n' | sudo tee /etc/caddy/sites/patchbay-relay.caddy
sudo systemctl reload caddy
```

然后安装 Relay 和 Web，并为电脑生成邀请：

```sh
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/relay/install.sh | sudo sh -s -- --domain <Relay 域名>   # 加上 --docker <目录> 则在 Docker 中运行
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/web/install.sh | sudo sh -s -- --domain <Web 域名>
sudo patchbay-relay invite                     # 输出给电脑运行的命令，只能使用一次，5 分钟内有效
```

**2. 电脑**，以那个用户身份操作（只有 `apt` 需要 sudo）：

```sh
sudo apt install curl                          # 以及 minisign，与服务器相同
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/host/install.sh | sh -s --
loginctl enable-linger                         # 通过 ssh 使用，或希望退出登录后 run 继续运行时需要
patchbay roots add <目录>                       # 允许启动 run 的文件夹
patchbay relay join --url https://<Relay 域名> --code <邀请码>   # invite 输出的那一行命令
patchbay devices pair                          # 在 `patchbay relay` 显示 connected 之后运行
```

如果不允许运行 `loginctl enable-linger`，可以请管理员运行 `sudo loginctl enable-linger <用户>`。`patchbay doctor` 会逐项检查这台电脑，并说明需要修复什么。

**3. 手机。** 打开 `https://<Web 域名>`，添加到主屏幕，然后在其中扫描 `patchbay devices pair` 显示的配对码，再在电脑的终端里输入手机上显示的确认码（`patchbay devices pair` 会提示输入）。

## 升级

在各自的机器上运行：有比已安装版本更新的最新版本时安装它；加上 `--version <版本>` 时安装指定版本，也可以是更旧的版本。

```sh
patchbay update                                # 电脑
sudo patchbay-relay update                     # Relay
sudo /srv/patchbay-web/current/fetch.sh        # Web
```

## Releases 的内容

- **每个版本一个 release**，tag 为 `<组件>-v<版本>`（例如 `host-v0.7.0`）。其中包括安装包、每个安装包的签名（`.minisig`）、这个版本的 `install.sh` 和说明；release 说明介绍这个版本的改动，包括需要手动操作的升级步骤。
- **每个组件一个 release**，tag 为 `host`、`relay` 或 `web`。其中是首次安装使用的 `install.sh`，以及 `latest`：只有一行，内容是最新版本号。新版本会先发布，在真实机器上测试后，再设为最新版本。

按版本号安装（包括尚未设为最新版本的）：

```sh
curl -fsSL https://github.com/kotaoching/patchbay-releases/releases/download/<组件>-v<版本>/install.sh | sh -s -- --version <版本>   # Relay 和 Web 需要 sudo sh，并加上 --domain
```

每个 `install.sh` 都会下载该版本的安装包，在装有 `minisign` 时验证签名（没有安装时只依赖 HTTPS，并给出提示），然后用你提供的选项运行安装包里的安装程序。

## 签名

每个安装包都用 [minisign](https://jedisct1.github.io/minisign/) 签名。公钥：

```
RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

安装脚本中已经内置了这个公钥。如果要自己验证安装包，把它的 `.minisig` 放在同一目录下，运行：

```sh
minisign -Vm patchbay-host-<版本>-linux-x64.tar.gz -P RWQca4FgH3lic/sEVYQezYMwc0E1g+AUX2uPojhiy5dVjJEqKYdcmbgR
```

命令输出的可信注释是安装包的文件名，所以一个签名不能被挪用到别的安装包上。

# ECS 控制台

一个简单的阿里云 ECS 管理面板，可以查看实例状态和流量，并在流量达到阈值时自动关机，避免产生额外费用。

> 阿里云流量数据可能存在延迟，自动关机不能保证绝对不会产生费用。

## 部署

准备一台安装了 Docker 的公网 Linux 服务器。如果使用域名，请先将域名解析到服务器 IP。

在服务器上运行：

```bash
curl -fsSL https://raw.githubusercontent.com/weandy/ecs-controller/main/install.sh \
  | env ECS_COOKIE_SECURE=1 bash
```

安装脚本会从本仓库拉取代码，并尽量使用预构建镜像启动。如果 `ghcr.io/weandy/ecs-controller` 上还没有对应镜像，脚本会在服务器上本地构建。服务默认只监听服务器本机的 `43211` 端口，请通过你现有的 HTTPS 访问方式打开面板，不要直接开放这个端口。

公网使用 Passkey 时必须使用 HTTPS。

### 本地测试

在自己的电脑上测试时，可以使用 Docker Desktop 或 Colima，安装后访问 `http://127.0.0.1:43211`。

## 第一次使用

1. 安装完成后，脚本会显示一个初始化 token。
2. 打开网页，输入 token 并设置管理员密码，密码至少 6 位。
3. 进入“账号管理”，添加阿里云账号。
4. 在“系统设置”中设置流量阈值、自动关机和通知方式。

如果忘记了初始化 token，可以查看日志：

```bash
docker logs ecs-controller 2>&1 | grep '初始化 token'
```

## 阿里云权限

请在阿里云 RAM 中创建一个用于本项目的用户，开启“OpenAPI 调用访问”，然后把 AccessKey ID 和 AccessKey Secret 填入账号管理。

项目会根据功能使用以下权限：

- **实例管理**：ECS
- **实例流量**：CMS
- **账号流量**：CDT
- **创建实例和 EIP**：VPC、ECS
- **消费情况**：账单服务

如果出现“CMS 不可用”或“CDT 不可用”，通常是对应的 RAM 权限没有开启。只查看和管理已有实例时，不需要开通创建实例、EIP 和账单权限。

## 主要功能

- 查看 ECS 状态、规格、IP 和流量。
- 查看 CMS 实例流量和 CDT 账号流量。
- CMS 或 CDT 任一流量达到阈值时自动关机。
- 定时开关机和抢占式实例自动恢复。
- 创建、启动、停止和释放 ECS。
- Telegram、邮件和 Webhook 通知。
- Passkey 免密码登录。

## 升级已有安装

已经在跑旧版或原作者版本时，在**那台 Linux 服务器**上重新执行安装命令即可：

```bash
curl -fsSL https://raw.githubusercontent.com/weandy/ecs-controller/main/install.sh \
  | env ECS_COOKIE_SECURE=1 bash
```

脚本会把现有目录的 `origin` 指到本仓库、快进拉取 `main`，再拉取镜像并重启。账号、配置和流量数据在 Docker volume 里，不会被清掉。端口仍是本机 `127.0.0.1:43211`，原来的 HTTPS 反代不用改。

如果记得安装目录，也可以不用 curl：

```bash
cd ~/ecs-controller
git remote set-url origin https://github.com/weandy/ecs-controller.git
git fetch origin
git pull --ff-only origin main
./deploy.sh
```

如果提示某个 `sha-...` 镜像 `not found`，说明当前 commit 的预构建镜像还没推上去。重新执行一次安装命令即可：脚本会改用 `latest`，再没有就在服务器本地构建。

升级后检查：

```bash
docker ps --filter name=ecs-controller
docker logs ecs-controller --tail 50
```

容器应显示 `healthy`。如果 Telegram 仍报 webhook 冲突，浏览器打开一次：

```text
https://api.telegram.org/bot<TOKEN>/deleteWebhook
```

然后给 Bot 发 `/start`。新版本在轮询时也会自动清除 webhook。

## 在线更新

进入“关于 → 版本中心”，点击检查更新即可。

本 fork 检查的是 `weandy/ecs-controller`，镜像仓库是 `ghcr.io/weandy/ecs-controller`。只有对应 commit 的 Docker 镜像已经推送完成后才会在线更新。更新不会删除账号、配置和流量数据。

GitHub Container Registry 默认经常是私有的。服务器要直接拉镜像，需要先把两个 Package 改成 **Public**（改成公开后不能再改回私有）：

1. [ecs-controller](https://github.com/users/weandy/packages/container/package/ecs-controller)
2. [ecs-controller-updater](https://github.com/users/weandy/packages/container/package/ecs-controller-updater)

每个页面：右侧 **Package settings** → 最下面 **Danger Zone** → **Change visibility** → **Public** → 确认。两个都要改。可用下面命令确认，应返回 `200` 而不是 `401`：

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  https://ghcr.io/v2/weandy/ecs-controller/manifests/latest
```

镜像还没公开时，上面的安装/升级命令仍能用：拉镜像失败后会在服务器本地构建，只是第一次会慢一些。设成 Public 之后，就可以继续用面板里的在线更新。

## 安全提醒

- 不要把 AccessKey、AccessKey Secret 或初始化 token 发给别人。
- 公网访问请使用 HTTPS，不要直接暴露 `43211` 端口。
- 建议使用 RAM 子账号，不要直接使用阿里云主账号密钥。

## Passkey

登录后进入“系统设置 → 全局设置”，点击“设置 Passkey”。以后可以使用指纹、面容、系统 PIN 或安全密钥登录。

公网使用 Passkey 时必须配置 HTTPS；本地测试可以使用 `http://localhost`。

## 请喝咖啡

如果这个工具对你有帮助，欢迎支持项目：

![项目赞赏码](./static/donation-code.jpg)

- USDT-TRC20：`TMBmXngHcKtNo9nLxTCJcGTfNutsbb9rp3`
- USDT-Polygon / BSC / ERC20 / ArbitrumOne：`0x846ad38ed7159aee70b0ebd8ec39ad2d7b32835b`

## 相关链接

- 项目地址：https://github.com/weandy/ecs-controller
- 上游项目：https://github.com/Kori1c/ecs-controller
- 问题反馈：https://github.com/weandy/ecs-controller/issues
- 许可证：[MIT](./LICENSE)

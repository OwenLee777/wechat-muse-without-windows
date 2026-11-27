# 不用电脑Windows部署，也能微信连Muse

> 2026-11-27 实测报告 · 云电脑 Ubuntu 24.04 · Python 桥 v1.3.2
> 作者：Owen（李龙图）+ Awen（Muse）

## 结论先说

不用 Windows 常开电脑，也能把 Muse 接进微信。

今天在云电脑上实测通了：扫码登录、绑定主人、订阅更新提醒、手动收发、每 5 秒自动回复，全都跑通了。微信里发“在吗”，Muse 自动回“在呢，Owen。”

这不是 Muse 官方功能。Muse 官方的消息通道只有 WhatsApp，没有微信入口。下面这条路是第三方开源桥，原理是腾讯 iLink 协议。

## 用了什么

- 桥：`gfydfhcf/wechat-ilink-muse-bridge`，v1.3.2，MIT 开源
  - 腾讯官方 iLink 协议，长轮询秒级收消息
  - 连发消息合并、主动推送、媒体断点续传
- 云电脑：Ubuntu 24.04 LTS，Python 3.12.3，git 2.43.0
- 依赖：只有 `cryptography`，实测装的是 50.0.2
- 驱动：仓库自带的 Muse hook 脚本 `examples/muse-hook/ilink-watch.sh`，每 5 秒跑一轮

## 今天是怎么跑通的

### 1. 装桥

```bash
git clone --depth 1 --branch v1.3.2 \
  https://github.com/gfydfhcf/wechat-ilink-muse-bridge.git
cd wechat-ilink-muse-bridge
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt   # 只有 cryptography
```

预检 `PREFLIGHT_OK`：Python、cryptography、网络、数据目录都通过。

### 2. 扫码登录：码只有 1-2 分钟

```bash
ILINK_DATA_DIR=$PWD/data .venv/bin/python ilink.py qrcode
ILINK_DATA_DIR=$PWD/data .venv/bin/python ilink.py login <qrcode-id>
```

- 前两次都超时了，不是因为扫不上，是人忙着没看到，码等了 5 分钟就过期了。
- 第三次扫完，微信弹提示：当前微信已连了一个 OpenClaw，继续连接会解除原来的连接。
  - 一个微信一次只能连一个 OpenClaw。旧的那个是 2026-11-16 微信里那个 Awen（当时显示“暂无法连接 OpenClaw”）。
  - 点“继续连接”，就是把旧的顶掉，换成 Muse。
- 第四张码 `bee7c4eb`，12:44 登录成功：`LOGIN_OK`，`session.json` 落盘。

教训：扫完看到“继续连接”要马上点，别等，码的有效期只有 1-2 分钟左右。

### 3. 绑定主人：10 分钟窗口

```bash
ILINK_DATA_DIR=$PWD/data .venv/bin/python ilink.py bind
```

12:45 绑定成功：`BIND_OK`，当前微信号设为 Owner。之后只有 Owner 和手动批准的账号能跟机器人聊天。

机器人连发了四条欢迎：欢迎使用、绑定成功、现在可以直接发消息了、要不要订阅更新提醒。

回“订阅”就订上了，机器人回“已订阅，有新版时我会在微信上告诉你”。

### 4. 真收真发验收

- 手动查收：微信发“你是谁”，后台 `poll` 收到，回了“我是 Awen，你的 Muse 助手，现在在微信里也能跟你聊天了。刚才那条‘你是谁’我收到了，这就是真收真发。”，`SEND_OK`。
- 再发“你好呀 awen”“你怎么不回复我”：当时还没接自动驱动，手动查才回得了，所以慢了。这两条也合并成一条回了。

### 5. 接自动回复：每 5 秒查一次

把仓库的 hook 脚本装成 Muse hook，每 5 秒跑一轮长轮询：

- 脚本：`~/hooks/scripts/wechat-ilink-watch.sh`
  - 从仓库 `examples/muse-hook/ilink-watch.sh` 拷来
  - 固定了云电脑的路径和 venv Python
  - 修了一个小 bug：dry-run 时 version-check 那一步会直接退出，改成只记日志、继续往下跑
- 干跑（dry-run）20 秒，“no new messages”，通过。
- 启用后，微信发“在吗”，hook 自动回了“在呢，Owen。”，`SEND_OK`。自动回复的 wake 路径实测通了。

## 关键机制，简单说

- 收消息是两段式：`poll` 只拉取、不推进游标；消息先落盘到待处理队列，再 `commit` 推进游标。崩了也不会丢消息，最多重复。
- 回消息带幂等键：用第一条消息的 `message_id` 当 `--key`，重复调用直接回 `SEND_ALREADY_SENT`，不重发。
- 连发合并：几条连发的消息，只回一条，不是一条回一条。
- 驱动二选一：hook 和 daemon 不能同时跑，共享一把驱动锁，先跑起来的赢。
- 订阅、更新安装这些，先判再回，不当普通聊天回。

## 限制和风险，如实写

1. **云 IP 容易被拦**：小红书那篇帖子也提醒了，云服务器 IP 扫码容易触发风控。Windows 常开电脑那条路线仍是备选主力。
2. **云电脑换机会停**：进程停了，hook 也停了，得重起。登录态还在的话不用重扫。
3. **登录态有效期不固定**：社区说法从 24 小时到 3 天以上都有，过期了要重扫。hook 会报一次过期提醒，不会一直催。
4. **不是官方同步**：微信里的 Muse 和 App 里的 Muse 是同一套记忆和任务吗？这次只验证了收发通，没验证全部同步。
5. **`session.json` 就是登录凭证**：只放本地，绝不提交、绝不发给别人。

## 复现步骤（给想自己搭的人）

1. 云电脑或 Linux 机器，Python 3.10+，装 `cryptography`。
2. 克隆 v1.3.2，按上面第 1 节装好。
3. `ilink.py qrcode` 出码，手机微信扫码，看到“继续连接”马上点。
4. `ilink.py login <qrcode-id>`，等 `LOGIN_OK`。
5. 马上 `ilink.py bind`，10 分钟内在微信里给机器人发任意一条消息，完成绑定。
6. 回“订阅”订更新提醒（可选）。
7. 接 hook 驱动，每 5 秒查一次，微信发一条测试消息验收自动回复。
8. 有问题先跑 `ilink.py health`，看登录、token、队列是不是正常。

## 相关链接

- 桥仓库：https://github.com/gfydfhcf/wechat-ilink-muse-bridge
- 这次实测的云电脑部署思路，也适用于 LibTV 那套云部署（见 `libtv-cli-cloud-login`）。

---
*本报告只记录 2026-11-27 当天在云电脑上的实测结果，不含任何登录凭证、二维码和账号 ID。*

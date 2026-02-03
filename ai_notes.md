# AI 相关笔记

> 目的：把一些「AI / 本地助手 / 自动化」的经验做成可复用的备忘。

## OpenClaw：开机自启动（systemd user service）

我当前环境检查结果：

- 可执行文件：`/home/heyu/.npm-global/bin/openclaw`
- 配置文件：`~/.openclaw/openclaw.json`
- systemd 用户服务：`~/.config/systemd/user/openclaw-gateway.service`
- 端口：`18789`（环境变量 `OPENCLAW_GATEWAY_PORT=18789`）
- Dashboard：<http://127.0.0.1:18789/>

### 1) 启用并立即启动（自启动）

```bash
systemctl --user enable --now openclaw-gateway.service
```

### 2) 查看状态

```bash
systemctl --user status openclaw-gateway.service --no-pager
openclaw gateway status
```

### 3) 重启 / 停止

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user stop openclaw-gateway.service
```

### 4) 关闭开机自启动

```bash
systemctl --user disable --now openclaw-gateway.service
```

### 5) 关键点：开机后即使不登录也能跑（linger）

如果希望机器重启后、用户**未登录**时也能自动运行 user service，需要打开 lingering：

```bash
loginctl show-user $USER -p Linger
sudo loginctl enable-linger $USER
```

我当前检查：`Linger=yes`。

### 6) 日志排查

```bash
journalctl --user -u openclaw-gateway.service -f
journalctl --user -u openclaw-gateway.service --since "today"
```

以及 OpenClaw 自己的状态排查：

```bash
openclaw status
```

---

## 备忘：常见习惯

- 尽量用 `systemctl --user` 管 OpenClaw 服务；用 `openclaw gateway status` 快速确认 gateway 监听/探活。
- 默认 gateway 绑在 `127.0.0.1`，属于本机可访问；如果要远程访问需要额外配置（安全起见不要随便暴露）。

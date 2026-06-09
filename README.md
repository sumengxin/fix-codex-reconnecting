# 修复 Codex 反复 Reconnecting 1/5

这是一个面向 Windows 用户的 Codex Skill，用于诊断和修复：

```text
Reconnecting 1/5
Reconnecting 2/5
...
Reconnecting 5/5
```

适用于正在使用 v2rayN、Clash、Mihomo 等本地代理，但 Codex 后台进程没有稳定继承代理设置的情况。

## 使用方法

1. 打开本仓库的 [SKILL.md](./SKILL.md)。
2. 点击右上角的 `Raw`。
3. 保存文件，或把网页地址交给 Codex。
4. 对 Codex 说：

```text
请阅读并执行这个 SKILL.md，诊断并修复 Reconnecting 1/5。
先检测本机代理端口，确认有效后再修改。
```

## 安装为长期 Skill

在目标电脑创建文件夹：

```text
C:\Users\你的用户名\.codex\skills\fix-codex-reconnecting\
```

把 `SKILL.md` 放入该文件夹，然后彻底退出并重新打开 Codex。

之后可以直接说：

```text
使用 fix-codex-reconnecting 修复 Codex 反复重连。
```

## Skill 会执行什么

- 自动检测本机 Windows 系统代理；
- 检查代理端口是否正在监听；
- 备份原有代理环境变量；
- 设置 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 和 `NO_PROXY`；
- 连续进行 5 次网络测试；
- 提醒重新启动 Codex；
- 提供恢复原配置的方法。

## 安全说明

- 不使用固定代理端口；
- 不照抄其他电脑的 `7897`、`10808` 等端口；
- 只修改当前 Windows 用户的环境变量；
- 不修改账号、API Key、对话文件或 `config.toml`；
- 修改前会生成备份；
- 支持回滚。

## 注意

测试请求收到 `401` 或 `403`，通常表示已经成功到达服务器，只是测试请求没有登录认证，不一定代表代理失败。

如果修复后仍然重连，应继续检查代理节点质量、网络丢包、WebSocket/HTTP CONNECT 支持，以及多个代理软件之间的冲突。

## License

MIT

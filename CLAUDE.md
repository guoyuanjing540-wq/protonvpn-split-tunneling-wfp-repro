# CLAUDE.md

本文件为 Claude Code 在本仓库的常驻上下文，每次新会话自动读取。

## 仓库性质

这是一个**纯文档仓库**，没有源代码、构建系统或测试。
唯一内容是 `README.md`——一份 Proton VPN 分流（Split Tunneling）故障的复现记录。

因此：不要去找 package.json / Makefile / 测试命令，它们不存在。

## 沟通约定

- 与仓库作者交流使用中文
- 本仓库是**公开**仓库，且 README 刻意面向搜索引擎优化。
  不要在提交内容中写入真实姓名、邮箱、客服工单号等可被索引的个人信息。

## 记录的问题

Proton VPN Windows 客户端 v5.1.8，WireGuard (UDP)，Split Tunneling 的 **Exclude 模式**：

若某进程在被加入排除列表**之前**就已在运行，则 VPN 连接后该进程仍可能被拦截，
出站连接返回 `WSAEACCES / 10013`。

- 受影响进程：`com.vortex.helper.exe`（Mihomo/Clash 内核，由 `service.exe` 拉起）
- WFP 证据：安全事件 `5157`，`FilterRTID` 匹配到 `ProtonVPN block IPv4`
- 恢复方式：结束进程 → 重启 Windows → 重新启动进程 → 连接 VPN
- 根因**未确认**；是否必须重启系统（而非仅重启进程）也未隔离验证

维护 README 时请保留这种"只陈述观察、不断言根因"的措辞，这是作者有意为之。
同理，README 末尾的中英文检索关键词区块是有意保留的，勿删。

## 状态

该问题已通过 Proton VPN 官方客服渠道上报，截至 2026-09 仍未结案。
具体工单编号不记录在此公开文件中，需要时向作者确认。

## 工作约定

- 所有改动提交到分支 `claude/untitled-session-b0mymt`，除非另行说明
- 未经明确要求不创建 Pull Request

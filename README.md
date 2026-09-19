# Proton VPN Split Tunneling / WFP Reproduction Record

> **Note:** This is a user-reported reproduction record, not an official Proton VPN diagnosis.
>
> The exact root cause has not been confirmed. This repository documents the observed behavior, WFP evidence, and recovery sequence.

## Environment

- OS: Windows
- Proton VPN: v5.1.8
- Protocol: WireGuard (UDP)
- Split tunneling mode: Exclude
- Third-party proxy core: Mihomo/Clash-based
- Process:
  - `com.vortex.helper.exe`
  - launched by `service.exe`

Example path:

`C:\Users\Administrator\.config\com.vortex.helper\com.vortex.helper.exe`

---

## Issue

When `com.vortex.helper.exe` is already running and is then added to Proton VPN's Split Tunneling exclusion list, the process can still lose outbound connectivity after Proton VPN connects.

The process receives:

`WSAEACCES / Windows socket error 10013`

Disconnecting Proton VPN restores connectivity immediately.

---

## Steps to reproduce

1. Start the third-party proxy application.
2. Confirm `com.vortex.helper.exe` is already running.
3. Add `com.vortex.helper.exe` to Proton VPN's Split Tunneling exclusion list.
4. Apply the configuration / reconnect Proton VPN.
5. Connect using WireGuard (UDP).
6. Attempt an outbound connection from the proxy process.

### Actual result

The process fails to connect and reports socket error `10013`.

### Expected result

The excluded process should continue using the normal network path and should not be blocked by Proton VPN.

---

## Recovery behavior observed

The issue stopped occurring after the following sequence:

1. Fully terminate the proxy process.
2. Restart Windows.
3. Launch the proxy process again.
4. Connect Proton VPN.

After this sequence, Proton VPN and the proxy process were able to work at the same time.

I have **not yet isolated whether restarting only the process is sufficient**, so the reboot should not be considered a confirmed requirement.

---

## WFP observations

Windows Filtering Platform auditing was enabled during troubleshooting.

Security Event ID `5157` showed outbound connections from:

`com.vortex.helper.exe`

being blocked.

Observed destinations included:

- `31.56.91.3:22701`
- `31.56.91.4:22701`
- `1.12.12.12:443`
- `223.5.5.5:443`

A live WFP filter dump taken immediately after one of the failures matched the event's `FilterRTID` to:

`ProtonVPN block IPv4`

Example:

```xml
<name>ProtonVPN block IPv4</name>
<description>Block all IPv4 traffic</description>
<layerKey>FWPM_LAYER_ALE_AUTH_CONNECT_V4</layerKey>
<weight>
    <type>FWP_UINT8</type>
    <uint8>1</uint8>
</weight>
<filterCondition/>
<action>
    <type>FWP_ACTION_BLOCK</type>
</action>
```

The runtime filter ID changes when Proton recreates its WFP rules, so the specific `FilterRTID` is not stable between sessions.

---

## Additional observation

In the WFP export, I could find Windows Firewall permit rules referencing:

`com.vortex.helper.exe`

under:

`FWPM_PROVIDER_MPSSVC_WF`

However, I did not find a corresponding Proton-specific `ProtonVPN permit app` entry for that executable in the snapshot I inspected.

The `ProtonVPN permit app` entries I found referenced Proton's own executables, such as:

- `protonvpn.client.exe`
- `protonvpnservice.exe`
- `protonvpn.wireguardservice.exe`

---

## Important note

I am **not claiming that WFP rule refresh behavior is definitively the root cause**.

The confirmed observations are:

- the process was excluded in Proton VPN settings;
- outbound connections still received `WSAEACCES / 10013`;
- WFP Event ID `5157` showed the process being blocked;
- the corresponding live filter matched `ProtonVPN block IPv4`;
- the issue disappeared after terminating the process, rebooting Windows, relaunching the process, and reconnecting Proton VPN.

This may indicate an edge case involving split-tunnel rule application and process lifetime, but further testing would be required to confirm the exact cause.

I can provide the full WFP filter export, Event ID `5157` records, and additional logs if useful.
Search keywords / 检索关键词

English: Proton VPN, ProtonVPN, split tunneling, exclude mode, WFP, Windows Filtering Platform, WSAEACCES, Windows socket error 10013, Mihomo, Clash, WireGuard UDP, com.vortex.helper.exe, ProtonVPN block IPv4.

中文： Proton VPN 分流失败、分流排除不生效、代理被阻断、Windows 网络错误 10013、套接字访问被拒绝、WFP 防火墙规则、Clash 与 Proton VPN 冲突、Mihomo 代理连接失败。

Related symptoms / 相关症状：

A third-party proxy process may receive WSAEACCES 10013 after connecting Proton VPN, even when the process has been added to the split tunneling exclusion list.

第三方代理进程即使已经加入 Proton VPN 分流排除列表，连接 VPN 后仍可能出现 10013 错误，导致代理无法正常连接。

This repository documents one observed case and its recovery procedure. The exact root cause has not been confirmed.

本仓库记录的是一次实际遇到的故障及恢复过程，尚未确认最终根因。
## Related terms / Search keywords
WSAEACCES, error 10013, WFP, Windows Filtering Platform,
ProtonVPN split tunneling, Mihomo, Clash core, com.vortex.helper.exe,
分流失败, 代理被阻断, VPN连接后代理进程失联
Acknowledgments / 致谢

Investigation and testing were performed by the repository author, with assistance from AI tools (ChatGPT/Codex and Claude) for troubleshooting suggestions, log analysis, and documentation review.

本案例的实际操作与测试由仓库作者完成，ChatGPT/Codex 和 Claude 协助提供排查建议、日志分析及文档整理。

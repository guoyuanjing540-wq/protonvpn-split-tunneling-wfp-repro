# Proton VPN Split Tunneling / WFP Reproduction Record

> **Note:** This is a user-reported reproduction record, not an official Proton VPN diagnosis.
>
> The exact root cause has not been confirmed. This repository documents the observed behavior, WFP evidence, and recovery sequence.

## Environment

- OS: Windows
- Proton VPN: v5.1.8 (running processes under `C:\Program Files\Proton\VPN\v5.1.8\`)
- Protocol: WireGuard (UDP)
- Split tunneling mode: Exclude
- Third-party proxy core: Mihomo/Clash-based
- Process:
  - `com.vortex.helper.exe`
  - launched by `service.exe`

Example path:

`C:\Users\Administrator\.config\com.vortex.helper\com.vortex.helper.exe`

Both `service.exe` and `com.vortex.helper.exe` were added to the Split Tunneling exclusion list.

---

## Issue

Two symptoms were observed, and they appear to be connected.

### 1. The excluded process is blocked anyway

When `com.vortex.helper.exe` is already running and is then added to Proton VPN's Split Tunneling exclusion list, the process can still lose outbound connectivity after Proton VPN connects.

The process receives:

`WSAEACCES / Windows socket error 10013`

Disconnecting Proton VPN restores connectivity immediately.

### 2. Proton VPN itself can get stuck at "Connecting"

On this network, Proton VPN's own servers are only reachable **through** the third-party proxy. So when the proxy is blocked during Proton's connection phase, Proton loses the upstream it needs to complete its own connection.

Observed sequence:

```
Proton VPN starts connecting
  → Proton's dynamic WFP provider installs its filters
  → "ProtonVPN block IPv4" blocks non-permitted IPv4 traffic
  → the proxy process gets WSAEACCES / 10013
  → the proxy's upstream dies
  → Proton, which was reaching its own servers through that proxy, loses its upstream too
  → Proton retries, never completes, stays at "Connecting"
```

During this state:

- `ProtonVPN Service` — Running
- `ProtonVPN WireGuard` — **Stopped**
- No WireGuard / Wintun / TUN virtual adapter present
- `ProtonVPNCallout` driver — running

In other words, the filtering layer is already active before the tunnel exists.

Note that the block filters are not permanent leftovers — they appear during the connection phase and disappear when Proton VPN is disconnected.

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

## A/B test

To rule out routing and interface problems, the proxy's outbound interface was pinned to the physical adapter (`interface-name = WLAN 2`) instead of automatic detection. The test then varied only Proton VPN's state:

| Proton VPN state | Same interface, same destination | Result |
|---|---|---|
| Connecting | `WLAN 2` → `31.56.91.5:20302` | `WSAEACCES / 10013` |
| Disconnected | `WLAN 2` → `31.56.91.5:20302` | `CONNECTED` |

Node latency was normal (~190–210 ms) whenever Proton VPN was not connecting, so the proxy itself was healthy throughout.

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

## No Proton permit rule for the excluded process

In the WFP export, the only filters referencing `com.vortex.helper.exe` came from the Windows Firewall provider:

- provider: `FWPM_PROVIDER_MPSSVC_WF`
- layer: `FWPM_LAYER_ALE_RESOURCE_ASSIGNMENT_V4`

That layer governs socket **bind**, whereas Proton's block filter sits at `FWPM_LAYER_ALE_AUTH_CONNECT_V4`, which governs **connect**. A permit at the bind layer therefore does not counteract a block at the connect layer.

No `ProtonVPN permit app` entry for the excluded executable was found. A plain text search of the export returned exactly three `ProtonVPN permit app` filters, all referencing Proton's own executables:

- `protonvpn.client.exe`
- `protonvpnservice.exe`
- `protonvpn.wireguardservice.exe`

---

## Open question: permit filters reference an older install path

<!-- TODO: fill in after capturing a WFP dump while Proton VPN is stuck at "Connecting".
     Command used:
       netsh wfp show filters file=C:\wfp_during_connect.xml
       Select-String -Path C:\wfp_during_connect.xml -Pattern "v5.1.5","v5.1.8","block" -SimpleMatch

     Question to settle: do v5.1.8 permit filters exist at connect time, or only v5.1.5 ones?
       - If v5.1.8 entries are present  → the earlier export simply predates the upgrade; no issue here.
       - If only v5.1.5 entries appear  → the permit filters point at paths that no longer exist,
                                          and would not match the running v5.1.8 binaries.
     Do not state a conclusion here until the dump is captured. -->

The `ProtonVPN permit app` filters found in the export reference executables under:

`C:\Program Files\Proton\VPN\v5.1.5\`

while the Proton VPN processes actually running at the time were under:

`C:\Program Files\Proton\VPN\v5.1.8\`

`ProtonVPN permit app` filters match on `FWPM_CONDITION_ALE_APP_ID`, which is a full executable path. A filter naming a `v5.1.5` path would therefore not match a process running from `v5.1.8`.

**This is not yet established as a defect.** The export may simply have been taken before the upgrade to v5.1.8. Settling this requires a WFP dump captured while Proton VPN is stuck in the connecting state, checked for both version paths. That capture has not been done yet, and no conclusion is drawn here.

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

## Ruled out

- **Fake-IP / TUN mode.** An earlier note treated this as a confirmed cause. Detailed records show `tun = false` at the time, so it is not the cause. Recorded here because the incorrect note circulated first.
- **Routing and adapter selection.** Pinning the proxy to the physical adapter (`WLAN 2`) did not change the outcome; see the A/B test above.
- **Proxy health.** Node latency stayed normal (~190–210 ms) except while Proton VPN was connecting.

A separate, earlier problem — a stale `ALL_PROXY` environment variable interfering with Proton VPN login — was resolved and is unrelated to the block described here.

---

## Important note

I am **not claiming that WFP rule refresh behavior is definitively the root cause**.

The confirmed observations are:

- the process was excluded in Proton VPN settings;
- outbound connections still received `WSAEACCES / 10013`;
- WFP Event ID `5157` showed the process being blocked;
- the corresponding live filter matched `ProtonVPN block IPv4`;
- the only filters naming the excluded process came from the Windows Firewall provider, at the bind layer rather than the connect layer;
- with the interface pinned, the only variable that changed the outcome was Proton VPN's connection state;
- Proton VPN itself could not complete a connection while its own upstream proxy was blocked;
- the issue disappeared after terminating the process, rebooting Windows, relaunching the process, and reconnecting Proton VPN.

This may indicate an edge case involving split-tunnel rule application and process lifetime, but further testing would be required to confirm the exact cause.

I can provide the full WFP filter export, Event ID `5157` records, and additional logs if useful.

Search keywords / 检索关键词

English: Proton VPN, ProtonVPN, split tunneling, exclude mode, WFP, Windows Filtering Platform, WSAEACCES, Windows socket error 10013, Mihomo, Clash, WireGuard UDP, com.vortex.helper.exe, ProtonVPN block IPv4, stuck at connecting.

中文： Proton VPN 分流失败、分流排除不生效、代理被阻断、Windows 网络错误 10013、套接字访问被拒绝、WFP 防火墙规则、Clash 与 Proton VPN 冲突、Mihomo 代理连接失败、Proton 卡在正在连接。

Related symptoms / 相关症状：

A third-party proxy process may receive WSAEACCES 10013 after connecting Proton VPN, even when the process has been added to the split tunneling exclusion list. On networks where Proton VPN's own servers are only reachable through that proxy, Proton VPN may then fail to finish connecting at all.

第三方代理进程即使已经加入 Proton VPN 分流排除列表，连接 VPN 后仍可能出现 10013 错误，导致代理无法正常连接。如果 Proton VPN 本身也需要经由该代理才能连上服务器，还会连带卡在"正在连接"。

This repository documents one observed case and its recovery procedure. The exact root cause has not been confirmed.

本仓库记录的是一次实际遇到的故障及恢复过程，尚未确认最终根因。
## Related terms / Search keywords
WSAEACCES, error 10013, WFP, Windows Filtering Platform,
ProtonVPN split tunneling, Mihomo, Clash core, com.vortex.helper.exe,
分流失败, 代理被阻断, VPN连接后代理进程失联, 卡在正在连接
Acknowledgments / 致谢

Investigation and testing were performed by the repository author, with assistance from AI tools (ChatGPT/Codex and Claude) for troubleshooting suggestions, log analysis, and documentation review.

本案例的实际操作与测试由仓库作者完成，ChatGPT/Codex 和 Claude 协助提供排查建议、日志分析及文档整理。

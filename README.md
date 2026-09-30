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

The `ProtonVPN permit app` filters found in the first export reference executables under:

`C:\Program Files\Proton\VPN\v5.1.5\`

while the Proton VPN processes actually running at the time were under:

`C:\Program Files\Proton\VPN\v5.1.8\`

`ProtonVPN permit app` filters match on `FWPM_CONDITION_ALE_APP_ID`, which is a full executable path. A filter naming a `v5.1.5` path would therefore not match a process running from `v5.1.8`.

The question was whether this is a defect or just a stale export taken before the upgrade. A second dump was captured while Proton VPN was stuck at "Connecting".

### Second capture (stuck at "Connecting")

Conditions: WireGuard (UDP), a US server, **Kill Switch off**.

```
netsh wfp show filters file=C:\wfp_during_connect.xml   (~1.8 MB)
```

Plain-text search results:

| Search term | Matches |
|---|---|
| `v5.1.5` | 0 |
| `v5.1.8` | 0 |
| `permit app` | 0 |
| `ProtonVPN` | 2 (provider names only: `ProtonVPN Dynamic Provider`, `ProtonVPN Permanent Provider`) |

So while stuck, this dump contained **no ProtonVPN block or permit filters at all**. The providers were registered, but no filters were installed under them. Proton VPN then connected to a Japan server on its own a little later.

The older export (`wfp_filters.xml`) contains `v5.1.5` in four executable paths (`protonvpn.wireguardservice.exe`, `protonvpnservice.exe`, `protonvpn.client.exe`, `resources\openvpn.exe`), all under `program files\proton\vpn\v5.1.5\`. Those are consistent with leftovers from before the upgrade to v5.1.8.

### What this does and does not show

- With Kill Switch off, getting stuck at "Connecting" happened **without any Proton WFP filter present**. That instance of the hang is therefore not explained by WFP blocking; it looks more like the connection itself not completing, which fits Proton support's statement that connections from mainland China are increasingly blocked (hostnames first, IP addresses more recently).
- The "permit paths point at an old version" idea is **neither confirmed nor ruled out**. With Kill Switch off there were no block or permit filters to inspect, so this capture could not test it. Testing it would need a dump taken while the Kill Switch is on and the client is stuck.
- Whether Kill Switch was on during the original 10013 failures above is not recorded here.

---

## Recovery behavior observed

The issue stopped occurring after the following sequence:

1. Fully terminate the proxy process.
2. Restart Windows.
3. Launch the proxy process again.
4. Connect Proton VPN.

After this sequence, Proton VPN and the proxy process were able to work at the same time.

I have **not yet isolated whether restarting only the process is sufficient**, so the reboot should not be considered a confirmed requirement for the WireGuard case above. A later test on a different protocol (next section) suggests reconnecting the VPN may be enough.

---

## Follow-up: other protocols and the exclusion list

Tested later on a Japan server with Kill Switch off, from mainland China.

### Protocol connectivity

| Protocol | Result |
|---|---|
| Stealth | Would not connect, with or without another proxy running |
| WireGuard (TCP) | Would not connect; the Cancel button became unresponsive and the client had to be force-closed in Task Manager; after relaunch the login page spun until another proxy was enabled |
| OpenVPN (TCP) | **Connected** (Tokyo), with the other proxy turned off |

WireGuard (UDP) connected intermittently in earlier sessions and not at other times.

### Does the exclusion list apply without reconnecting?

Setup: OpenVPN (TCP), Tokyo, Exclude mode, Microsoft Edge in the excluded apps list. To remove two confounders from an earlier inconclusive attempt, every `msedge.exe` process was ended in Task Manager (Edge keeps background processes after its windows close) and the Windows system proxy was confirmed off.

| Action after adding Edge to the exclusion list | IP shown in Edge |
|---|---|
| Edge fully restarted, VPN **not** reconnected | Tokyo VPN IP (exclusion had no effect) |
| VPN disconnected, then reconnected | Real ISP IP, immediately (exclusion worked) |

On this protocol, changes to the excluded apps list only took effect after the VPN connection was re-established. Restarting the excluded application alone was not enough. This resembles the original WireGuard report, where a full Windows restart was needed. Not tested here: whether the same holds for WireGuard (UDP), and whether a running process behaves differently from one started after the change.

Practical workaround observed: after editing the split tunneling list, disconnect and reconnect Proton VPN.

### Support-side notes

- Proton support stated that mainland China blocks Proton VPN hostnames and has more recently started blocking IP addresses, which may explain why some servers are unreachable while others (for example certain US ones) still connect.
- A bug report with logs was submitted from the Windows client. The first attempt failed to send, presumably because Proton's servers were unreachable; it succeeded on retry through another proxy.

---

## Ruled out

- **Fake-IP / TUN mode.** An earlier note treated this as a confirmed cause. Detailed records show `tun = false` at the time, so it is not the cause. Recorded here because the incorrect note circulated first.
- **Routing and adapter selection.** Pinning the proxy to the physical adapter (`WLAN 2`) did not change the outcome; see the A/B test above.
- **Proxy health.** Node latency stayed normal (~190–210 ms) except while Proton VPN was connecting.
- **Restarting only the excluded app.** In the OpenVPN (TCP) test above, restarting Edge alone did not apply the exclusion; reconnecting the VPN did.
- **WFP blocking as the cause of every "Connecting" hang.** One hang was captured with no Proton WFP filters present (Kill Switch off), so at least that instance is not a WFP block.

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
- the issue disappeared after terminating the process, rebooting Windows, relaunching the process, and reconnecting Proton VPN;
- on OpenVPN (TCP), a changed exclusion list took effect only after disconnecting and reconnecting the VPN;
- a "Connecting" hang was captured with Kill Switch off and no Proton WFP filters present, so not every hang involves a WFP block.

This may indicate an edge case involving split-tunnel rule application and process lifetime, but further testing would be required to confirm the exact cause.

I can provide the full WFP filter export, Event ID `5157` records, and additional logs if useful.

Search keywords / 检索关键词

English: Proton VPN, ProtonVPN, split tunneling, exclude mode, WFP, Windows Filtering Platform, WSAEACCES, Windows socket error 10013, Mihomo, Clash, WireGuard UDP, com.vortex.helper.exe, ProtonVPN block IPv4, stuck at connecting.

中文： Proton VPN 分流失败、分流排除不生效、代理被阻断、Windows 网络错误 10013、套接字访问被拒绝、WFP 防火墙规则、Clash 与 Proton VPN 冲突、Mihomo 代理连接失败、Proton 卡在正在连接。

Related symptoms / 相关症状：

A third-party proxy process may receive WSAEACCES 10013 after connecting Proton VPN, even when the process has been added to the split tunneling exclusion list. On networks where Proton VPN's own servers are only reachable through that proxy, Proton VPN may then fail to finish connecting at all.

第三方代理进程即使已经加入 Proton VPN 分流排除列表，连接 VPN 后仍可能出现 10013 错误，导致代理无法正常连接。如果 Proton VPN 本身也需要经由该代理才能连上服务器，还会连带卡在"正在连接"。改完分流排除列表后，需要断开并重新连接 VPN 才生效，仅重启被排除的程序无效（OpenVPN TCP 下实测）。

This repository documents one observed case and its recovery procedure. The exact root cause has not been confirmed.

本仓库记录的是一次实际遇到的故障及恢复过程，尚未确认最终根因。
## Related terms / Search keywords
WSAEACCES, error 10013, WFP, Windows Filtering Platform,
ProtonVPN split tunneling, Mihomo, Clash core, com.vortex.helper.exe,
分流失败, 代理被阻断, VPN连接后代理进程失联, 卡在正在连接
Acknowledgments / 致谢

Investigation and testing were performed by the repository author, with assistance from AI tools (ChatGPT/Codex and Claude) for troubleshooting suggestions, log analysis, and documentation review.

本案例的实际操作与测试由仓库作者完成，ChatGPT/Codex 和 Claude 协助提供排查建议、日志分析及文档整理。

# Windows settings affected by external power

Research date: 2026-09-19. Scope: this laptop and the proposed intentional 80% to 76% discharge interval. Microsoft documentation and read-only local inspection; no power settings changed.

## Findings

Performance mode and sleep are the main controls, but display, lid actions, device power management and background activity can also differ. Preserving selected preferences is feasible. Preserving every consequence of external power is not established: Windows and applications will still observe battery operation through [power-source status](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-system_power_status).

| Area | What can depend on power source | Evidence and relevance |
| --- | --- | --- |
| Performance mode | Windows has separately selected AC and battery power modes, above the legacy power plan. | The [AC](https://learn.microsoft.com/en-us/windows/win32/api/powrprof/nf-powrprof-powergetuserconfiguredacpowermode) and [DC](https://learn.microsoft.com/en-us/windows/win32/api/powrprof/nf-powrprof-powergetuserconfigureddcpowermode) APIs expose these selections. This laptop selects Best performance on AC and Balanced on battery. This is a first-version priority. |
| Processor policy | Energy/performance preference, boost, frequency limits and core parking can differ underneath the mode. | Microsoft documents [processor policies](https://learn.microsoft.com/en-us/windows-hardware/customize/power-settings/configure-processor-power-management-options). Lower EPP favors performance; actual operation depends on processor support and other policy. [Tuning explanation](https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/hardware/power/power-performance-tuning). |
| Screen, sleep and lid | Display timeout, sleep, hibernation, wake timers and lid/button actions have AC/DC values. | [Windows settings](https://support.microsoft.com/en-us/windows/experience/power-battery/power-settings-in-windows-11) and the local inventory below. The lid difference matters if working with an external monitor. |
| Display appearance and video | Brightness, content-dependent brightness, HDR and video playback preferences can change. | Content-dependent brightness offers a battery-only mode. [Brightness documentation](https://support.microsoft.com/en-US/Windows/Hardware/Display-Graphics/change-display-brightness-and-color-in-windows). HDR normally switches off on battery on supported laptops unless configured otherwise. [HDR documentation](https://support.microsoft.com/en-gb/windows/hardware/display-graphics/hdr-settings-in-windows). |
| Energy Saver | Can reduce brightness, limit power-mode selection and change background activity. | On Windows 11 24H2 and later it can also run while plugged in. Microsoft distinguishes general effects from additional unplugged/low-battery restrictions on app sync, background apps, updates and scheduled tasks. [Energy Saver](https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/energy-saver). Ordinary battery operation does not imply all these restrictions are active. |
| Devices and connectivity | Wi-Fi power saving, storage idle policies, PCIe link power management and USB policies can have AC/DC preferences. | See the local inventory, [disk settings](https://learn.microsoft.com/en-us/windows-hardware/customize/power-settings/disk-settings) and [USB selective suspend](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/usb-selective-suspend). Microsoft recommends keeping USB selective suspend enabled. Driver support determines actual effects. |
| Modern Standby | Networking, wake activity, updates and app background activity can differ during sleep. | [Network connectivity](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby-network-connectivity) and [wake sources](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby-wake-sources). This laptop supports S0 Modern Standby. Matching idle timeouts does not reproduce all AC sleep behavior. |
| Independent applications and tasks | Apps can react to battery status; scheduled tasks can refuse to start on battery. | [Task Scheduler battery condition](https://learn.microsoft.com/en-us/windows/win32/taskschd/tasksettings-disallowstartifonbatteries). These conditions do not disappear when power-plan values match. |
| Battery protection | Low/critical thresholds, notifications and actions apply while actually discharging. | The local critical action is Hibernate on battery. Preserve these protections; copying the AC critical action would disable it. |

Dynamic refresh rate also saves display power, but it responds to activity and supported hardware. It is not evidence of a universal unplug-triggered refresh-rate switch. HDR, content-dependent brightness and refresh-rate support on this laptop remain unverified. [Refresh-rate documentation](https://support.microsoft.com/en-us/windows/hardware/display-graphics/change-the-refresh-rate-on-your-monitor-in-windows).

## Verified local snapshot

Windows 11 25H2, build 26200.9457. Active legacy plan: Balanced. Read `powercfg /query`, `powercfg /qh` and `powercfg /a`, plus the documented AC/DC power-mode getter APIs. Both getters succeeded. [Powercfg reference](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options).

These are saved settings, not measured performance or proof that hardware implements every entry. For example, the plan enables hybrid sleep, but `/a` reports hybrid sleep unavailable.

| Setting | Plugged in | On battery |
| --- | --- | --- |
| User-selected power mode | Best performance | Balanced |
| Screen off after | 15 minutes | 5 minutes |
| Sleep after | Never | 20 minutes |
| Hibernate after | Never | 72 hours |
| Lid close | Do nothing | Sleep |
| Wake timers | Enabled | Disabled |
| Processor EPP, base and efficiency class 1 | 33 | 55 |
| System cooling policy | Active | Passive |
| Wi-Fi power saving | Maximum performance | Medium saving |
| Video playback | Optimize video quality | Balanced |
| Desktop slideshow | Available | Paused |
| Energy Saver charge-level setting | 0% | 30% |
| Critical battery action | Do nothing | Hibernate |

Saved values that match: brightness 90%; adaptive brightness off; processor minimum 5%, maximum 100%; boost mode Aggressive; USB selective suspend enabled; PCIe maximum power savings. Other hidden differences include NVMe idle timing and processor parking. Equal saved values do not prove equal runtime behavior. No load tests or physical unplug test were performed.

## Recommended first-version scope

Offer selected controls: preserve the plugged-in power mode; screen and sleep timeouts; brightness where supported; and optionally lid action. Decide hibernation and wake timers separately. Keep Wi-Fi and video preferences as optional follow-up controls unless the user experiences a problem.

Do not copy every hidden setting. The [power-mode setter](https://learn.microsoft.com/en-us/windows/win32/api/powrprof/nf-powrprof-powersetuserconfigureddcpowermode) is a user preference that other system signals may override. Processor settings, hardware limits, firmware and driver policies can prevent AC-equivalent performance on battery. No laptop-specific hard limit was measured here.

Treat preservation as temporary, only during a confirmed intentional pause. Record the previous selected settings and restore them when preservation ends, including recovery after service restart. Account for user changes made during a pause instead of blindly restoring stale values. Keep low-battery warnings and critical actions intact, and retain Energy Saver's low-battery protection. At the proposed 76% to 80% band, the saved 30% threshold alone should not activate it.

These are design recommendations, not implemented controls. Capability checks and an actual pause/resume test must verify each selected setting before the application promises to preserve it.

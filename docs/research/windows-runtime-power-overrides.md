# Temporary Windows power overrides

Research discussed during the implementation interview and recorded with the user's subsequent permission. No settings were changed and no runtime prototype has been run.

Microsoft documents CallNtPowerInformation with SystemPowerPolicyDc as a way to update the current DC system policy without storing changes in a power scheme. SystemPowerPolicyAc is the corresponding AC operation. SystemPowerPolicyCurrent is query-only. Scheme changes can overwrite these runtime values. Source: [CallNtPowerInformation](https://learn.microsoft.com/en-us/windows/win32/api/powerbase/nf-powerbase-callntpowerinformation).

SYSTEM_POWER_POLICY includes LidClose, IdleTimeout and VideoTimeout. These provide a documented candidate for three of the selected controls. Change only intended fields in the DC policy, preserving discharge and thermal protections rather than copying an entire AC policy. Modern Standby behavior on the target laptop must be tested. Source: [SYSTEM_POWER_POLICY](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-system_power_policy).

Non-persistent is not equivalent to process-scoped. The policy API has no override handle and does not promise to undo changes when the caller dies. Reapplying current saved battery preferences may avoid a saved-value snapshot, but release, scheme changes and crash recovery need verification.

For modern power mode, PowerSetUserConfiguredDCPowerMode changes the user's configured preference. It is not a documented separately owned temporary override. The presence of older exports or power-information enum values does not establish a supported runtime override contract. Source: [Power-mode setter](https://learn.microsoft.com/en-us/windows/win32/api/powrprof/nf-powrprof-powersetuserconfigureddcpowermode).

BrightnessOverride supports temporary brightness changes, but the per-view route requires CoreWindow. The background system route documents a systemManagement capability and Embedded mode requirement. These constraints do not establish a usable ordinary Win32 service/tray solution. Sources: [BrightnessOverride](https://learn.microsoft.com/en-us/uwp/api/windows.graphics.display.brightnessoverride?view=winrt-26100), [GetForCurrentView](https://learn.microsoft.com/en-us/uwp/api/windows.graphics.display.brightnessoverride.getforcurrentview?view=winrt-26100), [GetDefaultForSystem](https://learn.microsoft.com/en-us/uwp/api/windows.graphics.display.brightnessoverride.getdefaultforsystem?view=winrt-26100).

Power requests can inhibit idle sleep/display shutdown, but they do not simply substitute an arbitrary saved timeout or lid-close policy. Modern Standby imposes additional limits on DC system/execution requests. Source: [PowerSetRequest](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-powersetrequest).

The accepted design prefers runtime overrides but allows a saved-setting fallback with recovery. Explicit changes to battery preferences win over that fallback. The implementation must establish support, applicable user/session context, change detection and restoration separately for each control. All five remain in scope as one set; no individual toggles were accepted.

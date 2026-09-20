# Personal laptop battery charge controller

Status: specification for review and implementation planning, updated through D44 on 2026-09-21. D37, the sensor recovery rule, is explicitly deferred. The user selected GitHub Issues in `rshouker/BattaryLifeExtender` as the project tracker. The specification's triage label is `ready-for-agent`. No implementation has started. The test approach is accepted; remaining design and validation dependencies are explicit below.

An earlier revision was published as [GitHub issue #1](https://github.com/rshouker/BattaryLifeExtender/issues/1). This repository revision includes the later interview decisions; the issue was not synchronized as part of this update.

Requirements marked **agreed** come from the interview. Items marked **proposed** make the design concrete for review and are not recorded as prior user decisions. **Open** items must be resolved before the dependent work can be considered implementation-ready.

## Problem Statement

The user has an HP laptop that they report cannot enforce their preferred battery charge limit internally. They want to reduce time spent at full charge while retaining the performance and working preferences they normally use with the charger connected. They also need an easy way to prepare a full battery before leaving home without the charger.

Research supports avoiding prolonged full charge, but does not establish that deliberate 76% to 80% cycling improves this particular battery's lifespan. The user accepts this uncertainty. Success means reliable charge-range control and preservation of selected working preferences; a demonstrated increase in battery lifespan is not a first-version acceptance criterion.

## Solution

**Agreed:** Switch the original charger's AC input. An ESP32 communicates with a Windows service over BLE and controls an electromechanical relay initially, with an AC-capable SSR planned later. The ESP32 uses an independent, always-on supply that remains powered while the charger is switched off. The original charger and USB-C cable remain intact.

The device is charge-only. Automatic control must operate before Windows sign-in. On ESP32 startup or reset, charging remains permitted until the Windows service reconnects and sends explicit commands.

Normal operation restores charger power at the resume threshold, initially 76%, and interrupts it at the stop threshold, initially 80%. The tray application offers an upper threshold and a gap, and derives the resume threshold. While the laptop runs on battery during a managed charging pause, preserve Windows' plugged-in power mode, screen timeout, sleep timeout, brightness where supported, and lid-close action as one set, without individual toggles. Prefer runtime-only overrides; saved-setting changes with restoration are permitted where needed. Restore normal battery behavior when control is lost and external power is absent.

A full-charge request starts departure mode. It keeps charger power enabled until a departure is confirmed or the request is canceled, and survives sleep and restart. An unobserved departure leaves the request active. A request made without external power waits for power without expiring; if power returns after at least 30 minutes without it, notify the user when power returns and proceed with full charging. Threshold control resumes on the next connection after confirmed departure.

The ESP32 restores charger power after 60 seconds without communication. The user accepts that loss of communication during sleep, shutdown, or service failure can allow charging beyond 80%, including overnight.

The accepted AC-side architecture supersedes the earlier inline USB-C proposal. USB-C attachment and PD negotiation remain the responsibility of the original charger and laptop. Changing relay technology must preserve the charging-control behavior and BLE interface; switch-specific electrical details belong in the device driver and hardware. The initial relay uses normally closed contacts, enabling charging while its coil is unpowered. V1 includes measurement of the charger's DC output voltage through a suitably designed divider/ADC path; this measures voltage presence, not battery charging current.

## User Stories

Stories 1 through 19 express the original agreed outcomes. Stories 20 through 30 describe supporting behavior, much of which was resolved in the subsequent interview. Accepted decisions and remaining proposals are distinguished below.

1. As a laptop owner, I want the first version to work for my own laptop and device, so that the project can focus on my daily use.
2. As a laptop owner, I want the device to switch the charger's AC input, so that the original charger and USB-C connection handle power delivery.
3. As a laptop owner, I want the controller powered independently of the switched charger, so that it remains available to restore charger power.
4. As a laptop owner, I want the service to communicate with the ESP32 over Bluetooth, so that Windows can control charging without an additional control cable.
5. As a laptop owner, I want charger power interrupted at 80% by default, so that normal operation avoids charging all the way to full.
6. As a laptop owner, I want charger power restored at 76% by default, so that the laptop replenishes the charge used during the pause.
7. As a laptop owner, I want to select an upper threshold and a gap, preview the resulting resume threshold, and apply both settings together, so that I can change the charging range deliberately.
8. As a laptop owner, I want a Windows service to manage the device, so that control has a dedicated background component.
9. As a laptop owner, I want a tray application for charging controls, so that those controls are easy to reach while working.
10. As a laptop owner, I want the five agreed plugged-in preferences preserved together during managed charging pauses, so that cycling does not repeatedly change my working environment.
11. As a laptop owner, I want my usual battery preferences restored when communication is lost and external power is absent, so that genuine portable use gets normal battery behavior.
12. As a laptop owner, I want the device to restore charger power after 60 seconds without communication, so that a lost connection does not leave power cut indefinitely.
13. As a laptop owner, I want that fallback to apply when sleep, shutdown or a service failure interrupts communication, so that the device can recover without Windows issuing another command.
14. As a laptop owner, I want to request a full charge before leaving home, so that I have the available battery capacity for work away from the charger.
15. As a laptop owner, I want charger power kept enabled after reaching full charge in departure mode, so that it is still enabled if I leave later than expected.
16. As a laptop owner, I want departure mode to survive sleep and restart, so that those events do not cancel my preparation to leave.
17. As a laptop owner, I want to cancel departure mode manually, so that I can return to normal charging when my plans change.
18. As a laptop owner, I want threshold control to resume on my next connection after leaving, so that I do not have to turn it back on manually.
19. As a laptop owner, I want actual battery use recorded, so that I can understand how much cycling the controller produces without assuming a lifespan benefit.
20. As a laptop owner, I want invalid threshold combinations rejected with an explanation, so that the controller always has a usable range.
21. As a laptop owner, I want the tray to show battery level, charging mode and device connection status, so that I can understand what the controller is doing.
22. As a laptop owner, I want unavailable readings and unconfirmed device actions identified, so that missing information is not presented as a successful power change.
23. As a laptop owner, I want the device associated with my installation, so that the service controls the intended hardware.
24. As a laptop owner, I want automatic reconnection after a temporary Bluetooth interruption, so that normal operation can recover without repeated manual setup.
25. As a laptop owner, I want saved thresholds and departure mode restored after a service restart, so that routine restarts retain my choices.
26. As a laptop owner, I want temporary power-preference changes recoverable after a service failure, so that a failed session does not permanently replace my battery preferences.
27. As a laptop owner, I want intentional changes I make to Windows settings handled explicitly, so that restoring an old snapshot does not silently discard my changes.
28. As a laptop owner, I want normal low-battery and critical-battery actions preserved, so that threshold control does not disable them.
29. As a laptop owner, I want USB-C power interruption and restoration to work across reconnection and reset, so that managed charging does not break my charger connection.
30. As a laptop owner, I want unsupported settings and hardware limitations reported accurately, so that the application only promises behavior it has verified.

## Implementation Decisions

### Agreed architecture and responsibilities

- The Windows service reads battery and external-power status, controls charging through Bluetooth, persists user choices, and coordinates preservation of plugged-in preferences.
- The tray application exposes threshold configuration, a full-charge request and manual cancellation. Charging control belongs to the service; the tray is its user interface.
- The ESP32 operates the power switch and independently enforces the 60-second communication timeout. At startup or reset it permits charging until the service reconnects and explicitly commands otherwise. A valid USB-C power sequence remains required.
- The ESP32 persists normal or charge-to-full operating mode in flash. This stores intent, not an instruction to replay a previous relay state at boot. Windows retains V1 threshold decisions; RTC-based battery estimation and autonomous estimated-charge control are outside V1.
- The Windows service must support control before sign-in. A user-session-only Bluetooth process does not meet that requirement.
- Switch the charger's AC input using an electromechanical relay initially and an AC-capable SSR later. Keep the logical charging-enabled/paused interface independent of the physical switch and its GPIO polarity.
- Use normally closed relay contacts: coil unpowered means charging enabled; energizing the coil pauses charging. Preserve the charging-enabled default during controller power loss when selecting the later SSR arrangement.
- Power the ESP32 from an independent, always-on supply on the unswitched side. The selected board determines whether its regulated input is 5 V or 3.3 V. Do not derive controller power from the switched laptop charger.
- This is a personal, charge-only first version. Agreed stack: native C++ Windows service and Win32 tray/dialog, ESP-IDF C++ firmware with Arduino as a supported component, and shared portable service/ESP32 protocol definitions and codecs. BLE and separate service/UI processes communicating over restricted local named pipes are retained. The tray/service contract also uses shared C++ types with explicit binary encoding, not raw struct memory or generated RPC. Framework versions, ESP32 variant, exact messages and hardware parts are not finalized.

### Charging behavior

**Agreed:** Start with resume at 76% and stop at 80%. Departure mode permits full charge until departure or cancellation. The device restores charger power after 60 seconds without communication.

**Agreed control rules:** Fault recovery and data-validity rules constrain threshold actions. Exact event ordering and fault-clear semantics remain to be specified.

The normal-mode threshold rows below apply when automatic control is not manually suspended.

| Condition | Desired behavior |
| --- | --- |
| Normal mode; valid battery level at or above the stop threshold | Interrupt laptop power and enter a managed charging pause once the device action is confirmed. |
| Normal mode; valid battery level at or below the resume threshold | Restore laptop power. |
| Normal mode; level strictly between thresholds during an established session | Retain the last confirmed switch state. |
| New or recovered normal-mode session with automatic control active and no trustworthy previous control state; fresh level strictly between thresholds | The service commands a pause until the resume threshold is reached. The ESP32 permits charging before receiving this fresh command. |
| Departure mode | Keep power enabled regardless of the normal thresholds. |
| Manual suspension | Permit charger power, restore normal battery preferences, and suspend threshold control until Windows detects external power or the user resumes control or requests full charge. |
| Standalone cancellation of departure mode | Return to threshold control and reevaluate the current valid battery reading. Cancellation through "Use battery settings now" instead enters manual suspension. |
| Confirmed physical departure | End departure mode and temporary preference preservation; resume normal threshold control on a later connection. |
| Battery reading unavailable or stale while communication works | Retain the current relay state and retry every 10 seconds for a configurable window, default three minutes from the first failed/stale reading. Repeated failures do not restart the timer. If no valid reading arrives before expiry, permit charging until fresh readings return. |
| ESP32 has received no valid control communication for 60 seconds | Restore charger power independently of the Windows process. |
| ESP32 starts or resets | Permit charging until the service reconnects and sends explicit commands. Bluetooth reconnection alone does not resume the previous pause. |

**Agreed threshold settings:** Offer upper thresholds of 90%, 85%, 82%, 80%, 78% and 75%, and gaps of 2, 3, 4, 5, 6, 7 or 8 percentage points. Compute the resume threshold as upper threshold minus gap; it is not separately editable. Defaults remain upper 80% and gap 4 points, giving resume at 76%. Show the computed resume threshold before saving. An Apply button saves both settings together; changing either selection alone does not affect control.

Once applied, reevaluate active threshold control immediately: enable at or below the new resume threshold, pause at or above the new stop threshold, and retain the current state between the new thresholds. Departure mode and manual suspension continue to take precedence. Accept only the agreed preset values. The numerical values are software thresholds, not a promise of exact electrochemical state of charge or zero overshoot.

The battery-reading retry window is separate from the fixed 60-second communication watchdog. If communication stops during the reading retry window, the device still applies its watchdog. Exact stale-reading criteria, configurable retry-window bounds, heartbeat frequency, message deadlines and measured switching delays remain open. The device watchdog should measure valid messages from the associated controller using its own monotonic timer.

### Separate the observable states

Track Windows external-power presence, battery level, charging indication, Bluetooth availability, requested switch state, ESP32-applied output state, and measured charger-output voltage separately. The ESP32 reports its applied state and fallback reason; it must not simply echo a request as physical proof. V1 adds measured DC output voltage. Voltage presence does not establish current flow or battery charging. Missing or stale sensor data must remain distinct from measured zero voltage; exact telemetry encoding remains open.

Bluetooth loss does not prove physical departure. A battery that is not charging can still be externally powered. During a managed pause, unplugging may produce no new observation, even with voltage sensing. Delayed detection is accepted: offer a tray action, "Use battery settings now," to end the managed pause and immediately restore battery preferences. Otherwise restore preferences when communication is lost or a later power-on attempt establishes that external power is unavailable. Capacitors can delay voltage decay, so allow measured shutdown/startup settling time before classifying power state.

**Agreed unseen-departure rule:** If an unplug/replug occurs entirely during sleep or service downtime and neither endpoint captured reliable departure evidence, keep the full-charge request active. Sleep, restart and Bluetooth loss alone do not cancel it. This can leave departure mode active after an unobserved trip. The sensor may provide additional evidence, but its unplug detection behavior has not been tested.

### Manual battery-settings action

**Agreed:** Offer "Use battery settings now" only while Windows reports no external power. Open a modeless confirmation dialog. If Windows detects external power while the dialog is open, close it without applying the action. Recheck power when confirmation is submitted; a simultaneous power return takes priority.

On confirmation, restore normal battery preferences, enable charger power, suspend automatic threshold control, and cancel any active full-charge request. When such a request is active, the dialog warns that it will be canceled. After successfully canceling it, show a brief notification confirming cancellation and restoration of battery settings. If power returns before confirmation, leave the request unchanged and show no success notification. No second confirmation step is required.

Persist manual suspension through service restart and Windows reboot. Clear it whenever Windows detects external power at the laptop, even if power is already present when state is reevaluated, and resume normal charging control and the applicable plugged-in preference handling. Bluetooth reconnection alone does not clear it. Also offer "Resume automatic control" for explicit resumption.

A new "Charge to full" request clears manual suspension, enables charger power, and activates departure mode. While the laptop remains unplugged, show a waiting-for-external-power status and keep normal battery preferences.

### Full-charge requests while unplugged

**Agreed:** A request made without external power remains pending until power is available or the request is canceled. Initial absence of power is a waiting condition, not evidence that a departure has completed. The request does not expire after 30 minutes.

If power returns after at least 30 minutes without it, notify the user only when power returns, explaining that the earlier full-charge request is proceeding. Proceed automatically and offer "Cancel full charge" from the notification. Do not notify merely because the waiting period reaches 30 minutes. Exact timing/persistence mechanics for measuring the wait remain to be designed.

### Plugged-in preferences

**Agreed:** Preserve Windows power mode, screen timeout, sleep timeout, brightness where supported, and lid-close action together, without individual selection. Windows' plugged-in preferences are the source of truth, including changes made during a pause. Hibernate timeout, wake timers, Wi-Fi and video preferences are outside this initial preservation set.

Prefer temporary runtime overrides that leave saved Windows preferences untouched. Read the current saved AC preferences when applying the override and the current saved battery preferences when ending it. Avoid a saved-value backup when the chosen API does not modify stored preferences. This is a preference, not a hard ban on saved-setting changes: where runtime overrides cannot reliably preserve a setting, a saved-setting fallback with restoration is accepted.

For a saved-setting fallback, preserve an explicit user change to the battery preference and stop overriding that setting for the rest of the pause. Restore the original value only if the current value still matches the application's applied value. This conflict rule is an exception to the all-five preservation policy, not an individual settings toggle. Durable recovery records are needed for saved-setting changes. Value comparison cannot identify same-value edits; exact change detection remains a validation task.

Apply overrides only during a confirmed managed pause under active control. Automatically restart the service after failure and restore appropriate preferences when it recovers; restoration upon recovery is accepted rather than a fixed independent deadline. Runtime-only changes are not assumed to disappear on a crash. A separate recovery process is not required for V1. The actual Windows service identity and user-session execution for each API still need validation.

Microsoft documents non-persistent policy updates through CallNtPowerInformation, including fields for lid-close action and screen/sleep timeouts. Effectiveness on this Modern Standby laptop and release/recovery behavior require a prototype. Modern power mode and background brightness require separate capability checks; support for all five through runtime-only APIs is not established. See the runtime override research below.

Do not mirror every hidden AC power-plan value. Critical battery actions and low-battery protections must remain appropriate for actual battery operation. Matching selected settings does not spoof external-power status and cannot guarantee that firmware, drivers, applications or scheduled tasks behave as though the charger is connected.

### Persistence and interfaces

- Persist upper threshold, threshold gap, the configurable battery-reading retry window, departure mode and manual suspension in Windows. The ESP32 also persists normal/charge-to-full intent in flash. Windows is authoritative when saved operating modes disagree: save the user's choice before sending it and synchronize the ESP32 on reconnection, so its older record cannot resurrect a canceled request. Keep control-session state and telemetry freshness separate from intent. Exact reconciliation messages remain to be designed.
- Keep recovery records for saved-setting fallbacks separate from configuration. Do not require storing copies of Windows preferences solely to implement a supported runtime override.
- The service should provide the tray with current mode, battery reading and freshness, actual external-power status, connection state, requested/acknowledged switch state, and actionable errors.
- D4a agreed: share C++ tray/service request and response types plus explicit encode/decode functions over the restricted named pipe. Keep this contract separate from the BLE device protocol. Define versioning, framing and validation before implementation; raw struct memory is not the wire format.
- Bluetooth commands should identify the current control session and support acknowledgment, repeat-safe retries, and recovery without applying stale commands from a previous session. Exact encoding and authentication details remain open. Pairing behavior is agreed below.
- Following ESP32 reset, the service must account for the charging-enabled recovery state. Proposed reconciliation is to read fresh device and battery state, reevaluate current policy and explicitly command the resulting state, even if the service remembers issuing the same command before the reset.
- Keep 30 days of local battery-level, power-transition and fault history, remove older records automatically, and offer CSV export. Proposed additional fields include requested/applied states, voltage observations, departure changes and fallback reasons. Record energy/charge throughput and temperature only where supported and label estimates. Cloud telemetry is not required.
- Configuration and manual controls are restricted to the owner's Windows account and administrators. Charging control continues before sign-in and while the owner is logged out. The tray starts automatically when the owner's account signs in; the service operates independently of it. Closing the tray should not stop the service. Installation, account identification, startup mechanism and uninstall restoration still need detailed design.
- The service supplies settings when it connects after device startup or pairing replacement. Until fresh service commands arrive, the ESP32 permits charging. Temporary device defaults do not start independent threshold control; no separate replacement-computer configuration policy was selected under D43.

### Pairing and status

Associate the laptop with one explicitly selected ESP32 during initial setup. Reconnect automatically to that remembered device, including before sign-in. Replacing it requires an explicit "Change device" action. A physical button action on the ESP32 opens a three-minute pairing window, which closes on successful pairing. If it expires, press the button again. Ordinary reconnection to the paired laptop requires no button press. Pairing authentication, the precise button gesture and the reset protocol remain open.

The ESP32 accepts one paired computer at a time. Opening the pairing window leaves the current computer's control session running. Failed or expired pairing preserves the existing pairing. Only successful replacement removes the previous computer's access, ends its control session, enables charger power, and waits for fresh commands from the new computer. Exact protocol and bond-replacement mechanics must enforce that behavior.

Ordinary charging cycles are silent. Notify the user of persistent problems requiring attention and the explicit manual-action/full-charge events specified above. Use a low-brightness RGB status LED: steady green for charging enabled, steady amber for an intentional pause, and blinking blue only while pairing is incomplete. Add a separate low-brightness red fault LED, steadily lit while a fault is active. D36 supersedes D19's use of the main RGB LED for faults, allowing operating status and faults to be visible together. Do not blink for normal operation or faults. LED status describes controller state, not confirmed battery current.

### Unreliable voltage sensor

**Agreed:** If charger-output voltage readings become unreliable while communication and usable Windows external-power feedback remain available, continue threshold control using Windows feedback. A sensor fault alone does not suspend automatic pauses. Use external-power presence as feedback, not the battery's charging indicator. Missing or invalid sensor readings remain unknown, not zero voltage.

Show one initial notification explaining that control is continuing using Windows feedback, plus a persistent tray warning badge and details in its menu. Dismissing the notification leaves the badge active. Also light the dedicated fault LED. The sensor warning remains until the sensor recovers.

**Deferred D37:** The rule for declaring sensor recovery remains open. The proposed 30 seconds of valid readings with agreement against comparable Windows observations was not accepted; the user asked to revisit it. Sensor-unreliability detection, sensor/Windows disagreement classification, combined loss of both feedback sources, and clearing/aggregating multiple faults also require design. Do not treat recovery as specified merely because the warning is persistent until recovery.

### Voltage mismatch recovery

If valid measured voltage remains present after a charging-off command and the measured shutdown settling delay, alert the user and light the dedicated steady red fault LED. Permit charging between automatic retry attempts. Retry after 30 seconds, then 1, 2, 4, 8 and 10 minutes, and every 10 minutes thereafter. Each delay begins after the preceding failure. Reevaluate current battery policy, departure mode and manual suspension before retrying; do not blindly repeat an obsolete off command.

This policy addresses failure to remove observed charger output. Missing voltage after an on command may mean a disconnected laptop or unavailable charger and is not automatically the same fault. An unreliable sensor follows the separate Windows-feedback policy above. Voltage thresholds, settling times, fault ownership, success/clear criteria and mismatch-notification deduplication remain to be specified.

### Hardware constraints and unresolved circuit design

- Preserve the original charger and USB-C path. Verify that AC interruption removes laptop power and AC restoration reliably restarts charging, including the charger's shutdown and startup delays.
- Select the relay, driver and later SSR for the local AC supply and actual charger's repeated startup inrush. Verify control input requirements, isolation, off-state behavior, losses and heating. The recommended implementation uses enclosed, appropriately certified mains switching with an isolated low-voltage control interface; no particular part or assembly is approved.
- Keep switch polarity, drive and any switch-specific timing inside the firmware output adapter. An SSR replacement must preserve the same logical behavior but still needs validation of leakage, switching delays and startup/reset defaults. A normally closed mechanical contact cannot be assumed equivalent to an SSR without control power.
- Verify the independent controller supply remains available throughout charger interruption and restart.
- The 60-second firmware fallback assumes a powered, running ESP32. Normally closed contacts establish the unpowered-coil default; a firmware hang that keeps the coil energized still requires a watchdog/reset design. The later SSR must preserve the agreed charging-enabled default. Exact circuitry remains open.
- Include charger DC-output sensing in finished V1, deferred only from the initial BLE prototype. Choose divider scaling, ADC protection/calibration and grounding for the actual charger and ESP32. The charger remains unmodified internally; an external measurement tap must preserve the USB-C power/CC connections.
- The exact charger, board, relay/SSR, driver, independent supply, RGB LED, dedicated fault LED and sensing circuit are not specified. No particular circuit is approved by this spec.

## Testing Decisions

No source code or test framework exists in the workspace. There are no existing application test boundaries or prior tests to reuse.

### Agreed primary test boundary

Use the externally observable behavior of the Windows service as the main automated integration-test boundary. Drive it with battery/external-power observations, tray requests, device replies and elapsed time. Observe device commands, published status, persistence and requests to apply or restore Windows preferences.

Substitute controlled battery, device, clock and Windows-settings adapters at the service boundary. Keep the charging policy inside that boundary so tests do not depend on private helper structure. Exercise real service coordination and persistence with isolated test storage. Tests should cover useful user scenarios rather than duplicate implementation branches.

Build a bounded Windows-service-to-ESP32 BLE prototype and verify control before sign-in, commands, reported applied state, reconnection, reset and the 60-second fallback. Automate assertions using ESP32 telemetry; an LED is an optional visual aid and does not require a human observer for these tests. Self-reported state verifies firmware behavior, not physical relay contacts. Add voltage-feedback checks for finished V1 and separate real charger/Windows preference tests.

### Proposed scenarios

1. Reach 80%, pause, remain paused through intermediate readings, reach 76%, and restore power without repeated switching at a single threshold.
2. Offer exactly the agreed upper-threshold and gap presets, show the derived resume threshold, reject unsupported values, and change control only after Apply saves both selections together.
3. Request departure mode below, within and above the normal band; keep power enabled at full charge.
4. Preserve departure mode across sleep, service restart and operating-system restart; cancel it only through the agreed cancellation/departure rules.
5. Confirm departure and return; resume threshold control without treating the former request as permanent.
6. Lose Bluetooth while paused; end preference preservation appropriately and verify firmware restores power at the 60-second deadline.
7. Crash or stop the service; verify the device watchdog works without Windows issuing a restore command.
8. Fail battery reads while communications remain alive; retry every 10 seconds, hold state for the configured window, and permit charging after expiry. Verify repeated failures do not restart the deadline and communication loss still invokes the independent 60-second watchdog.
9. Delay, duplicate, lose or reject commands and replies; avoid falsely reporting a successful physical power change.
10. Reconnect after the device has applied its fallback; use fresh state and reject obsolete session commands.
11. Preserve the five Windows preferences together during a managed pause. Release runtime overrides to current saved battery preferences; restore saved-setting fallbacks only where an explicit user edit has not taken ownership.
12. Restart after an interrupted settings change; recover settings without corrupting configuration or silently overwriting a deliberate user edit.
13. Report battery operation accurately while overrides are active; keep critical battery actions intact.
14. Close the tray while the service runs; keep charging control active. Report a stopped service accurately when the tray remains open.
15. Restart with no trustworthy prior state and a fresh battery reading inside the range; verify the service commands pause until the resume threshold, while ESP32 startup initially permits charging.
16. Reset the ESP32 during a pause; verify charging is permitted, then verify the service reads fresh state and sends explicit commands without replaying an obsolete pause.
17. Start Windows without signing in; verify provisioned Bluetooth access and automatic charge control through the intended service identity.
18. Apply thresholds during normal control; reevaluate the new boundaries immediately, retain state inside the new band, and keep departure-mode and manual-suspension precedence.
19. Verify runtime overrides leave stored Windows preferences unchanged. For saved-setting fallbacks, preserve explicit user battery edits and restore only still-owned values. Exercise service crash/restart for both approaches.
20. Persist normal/charge-to-full mode across ESP32 reset without replaying an old pause. Cancel a request in Windows while communication fails, then reconnect and verify that Windows wins over the ESP32's older saved intent.
21. Command off while valid measured voltage stays present beyond settling time; verify notification, dedicated steady red fault LED, charging-enabled waiting state and the agreed retry intervals. Reevaluate policy before each retry, including a manual suspension begun during the retry delay.
22. Verify normally closed behavior with the controller unpowered, initial pairing requiring a button action, remembered-device reconnection and owner/administrator-only controls.
23. Verify that the RGB LED can show charging enabled or paused while the separate fault LED is lit. Verify blinking only for incomplete pairing, silent ordinary cycles, 30-day history retention and CSV export.
24. Unplug during a pause; verify accepted delayed restoration and the manual action. Unplug/replug entirely during sleep without reliable departure evidence and verify that the full-charge request survives.
25. Offer the manual action only without external power. Return power during its modeless confirmation, including at submission, and verify that the dialog closes without suspension, cancellation of a full-charge request, or a success notification.
26. Confirm the manual action without external power: restore battery preferences, enable charger power, suspend threshold control, and cancel an active full-charge request. Verify its cancellation warning before confirmation and notification after success.
27. Restart the service and reboot while manually suspended and still without external power; preserve suspension. Clear suspension on Windows-detected external power, including power already present at recovery, or explicit resume. Bluetooth reconnection alone does not clear it.
28. Request full charge while suspended and unplugged: clear suspension, enable charger power, retain battery preferences while waiting, and proceed when power returns. Verify no expiry, no notification at the 30-minute mark while still unplugged, and a notification with a cancellation action when power returns after that delay. Exercise just below and at 30 minutes.
29. Make sensor readings unreliable while Windows feedback remains usable: continue threshold control from Windows external-power status, show one initial notification, preserve the tray badge after dismissal, and light the separate fault LED. Recovery/clear assertions remain blocked on deferred D37.
30. Open pairing during active control; preserve the old session until a new pairing succeeds. Verify three-minute expiry, unchanged pairing on failure, closure on success, removal of old access only on success, and charging enabled until fresh commands from the new service.
31. Verify automatic tray startup for the owner at sign-in and independent service operation before sign-in. On device startup, permit charging until service settings and fresh commands arrive.

### Required physical and Windows verification

The service-level simulator cannot establish electrical behavior. A separate hardware-in-the-loop check must verify the ESP32 watchdog and actual power path, including loss of communication, repeated AC interruption/restoration, charger/laptop unplugging, controller reset and independent supply continuity. Measure charger shutdown/startup delays and verify fallback behavior. Repeat the relevant checks when replacing the mechanical relay with an SSR, including off-state leakage and reset defaults. These checks depend on selected hardware and a reviewed circuit.

On the target laptop, verify each selected power-setting override and its restoration through real pauses, reconnection, sleep and restart. Check unsupported settings explicitly. A saved Windows setting alone does not establish runtime behavior or performance equivalence.

Test logs may demonstrate threshold operation, power transitions and battery use. A short test cannot demonstrate years of battery-life extension.

## Out of Scope

- Demonstrating or guaranteeing longer battery lifespan, or claiming that a four-percentage-point band is universally optimal.
- A product for multiple users, multiple computers or arbitrary charger hardware.
- USB data forwarding, video and dock support in the first device.
- Custom inline USB-C power switching or PD controllers, battery-pack modification, or replacing the laptop's internal battery management system.
- Cloud control, Wi-Fi control, remote accounts or fleet management.
- RTC-based battery-level estimation or autonomous ESP32 threshold decisions based on estimated charge in V1.
- Making Windows falsely report AC power or overriding every application, firmware or driver response to battery use.
- Holding the battery at 80% while communication is unavailable. The accepted fallback can permit charging to full.
- Selecting a production enclosure, manufacturing process or commercial certification program as part of this personal software specification.
- Implementing or purchasing hardware, changing Windows settings, or generating implementation tickets as part of this specification-writing task.

## Further Notes

### Open decisions and dependencies

| Item | What is known | What it blocks |
| --- | --- | --- |
| Verification | Service integration tests and automated BLE prototype accepted | Actual before-login BLE access, runtime setting behavior and physical hardware results |
| Exact target and charger | Local system appears to be an HP OmniBook Ultra Flip 14-fh0xxx; USB-C confirmed; actual charger model and AC input/startup characteristics unknown | Switch ratings and hardware acceptance plan |
| AC power switching | Agreed: switch charger AC input; electromechanical relay first, AC-capable SSR later; original USB-C path intact | Selected switching assembly, driver and verified restart/fallback behavior |
| ESP32 and independent supply | Always-on controller supply agreed; exact parts unknown | Firmware target, Bluetooth capabilities and electrical validation |
| Controller failure defaults | Normally closed relay, charging-enabled boot and 60-second communication fallback agreed | Electrical implementation, firmware-hang watchdog and equivalent later SSR behavior |
| Preserved Windows controls | All five accepted together; runtime overrides preferred, saved-setting fallback permitted, explicit battery edits win | Per-setting capability tests, user/session context and release/recovery implementation |
| Departure and manual control | Unobserved departure preserves a full-charge request; Windows wins mode conflicts; confirmed manual suspension persists until power or a new user instruction; delayed full-charge notification agreed | Exact departure evidence, behavior with unknown power status, and timing/persistence mechanics for pending-request notifications |
| Timing and validity | Three-minute configurable battery-read window with ten-second retries; fixed 60-second device watchdog | Stale-reading definition, configuration bounds, heartbeat/message timing |
| Voltage feedback and faults | DC sensing in V1; mismatch retry backoff; Windows feedback on sensor failure; persistent tray warning and dedicated fault LED | Deferred D37 recovery rule, sensor validity/disagreement criteria, simultaneous feedback loss, actual circuit, voltage/settling thresholds, fault ownership and aggregation |
| Operational details | Three-minute pairing window; one paired computer replaced only on success; owner/admin controls; automatic owner tray startup; separate operating/fault LEDs; threshold presets and Apply; 30-day local history and CSV | Pairing authentication/button gesture and replacement protocol, installer, account/session setup, startup mechanism, uninstall and exact messages |

The open items limit implementation readiness, not the validity of the agreed product goals. The `ready-for-agent` label follows the specification workflow and does not imply that these dependencies are resolved. Proposed behavior must remain distinguishable from an accepted decision when the spec is split into tickets.

### Supporting decisions and research

- [Domain glossary](../../CONTEXT.md)
- [Discovery and agreed answers](../discovery.md)
- [External-power control decision](../adr/0001-control-external-charger-power.md)
- [Superseded inline placement decision](../adr/0002-switch-the-charger-output.md)
- [Charging-enabled recovery decision](../adr/0003-restore-charging-before-service-control.md)
- [AC-side switching decision](../adr/0004-switch-the-charger-ac-input.md)
- [Native C++ stack decision](../adr/0005-use-native-cpp-on-windows-and-esp32.md)
- [Windows runtime preference decision](../adr/0006-prefer-runtime-windows-preference-overrides.md)
- [Battery-aging evidence](../research/battery-aging-76-80-vs-100.md)
- [Windows power-settings research](../research/windows-plugged-in-settings.md)
- [Runtime Windows power override research](../research/windows-runtime-power-overrides.md)
- [Inline USB-C feasibility](../research/inline-power-switch-feasibility.md)
- [VBUS loss and PD reconnection research](../research/usb-c-vbus-interruption.md)
- [Native C++ and Arduino/ESP-IDF research](../research/native-cpp-and-esp-idf.md)
- [Tray/service contract comparison](../research/tray-service-contract.md)

The battery-longevity conclusion excludes HP advice at the user's request. Manufacturer documentation remains usable for connector and hardware specifications.

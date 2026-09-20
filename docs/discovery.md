# Project discovery

Status: synthesized into a [specification](specs/battery-charge-controller.md) at the user's request, with the interview recorded through D44 on 2026-09-21. D37 remains deferred. This document retains the interview decisions and unresolved questions. The service integration test boundary and automated BLE prototype approach are accepted. The user selected [GitHub Issues](https://github.com/rshouker/BattaryLifeExtender/issues) as the project tracker, with `ready-for-agent` as the specification workflow label.

An earlier specification revision was published as [issue #1](https://github.com/rshouker/BattaryLifeExtender/issues/1), labeled `ready-for-agent`. This repository revision includes later decisions; the issue was not synchronized as part of this update.

## Starting requirements

- The user has an HP laptop and reports that it lacks a setting to stop charging at 80%.
- An external device must interrupt power from the charger to the laptop.
- The device will use an ESP32 and communicate with the laptop over Bluetooth.
- A Windows service will communicate with the device.
- A tray application will provide a full-charge action and configurable charging thresholds.

## Decision tree

- Intended use and success criteria
  - First version is for the user's personal use, agreed
  - Behavior that will demonstrate a useful first version
- Power-control feasibility, under investigation
  - Laptop and charger compatibility
  - Running on battery during managed charging pauses is accepted
  - AC-side switch and independently powered ESP32, agreed; supersedes inline USB-C switching
  - Electromechanical relay initially, AC-capable SSR later, agreed
  - Exact switch, driver, independent supply and charger restart behavior remain open
  - Hardware construction and enclosure
- Charging policy
  - Upper-threshold and gap presets with Apply, initially upper 80% and gap 4 points, agreed
  - Full-charge request keeps power enabled until confirmed departure or cancellation, agreed
  - Departure mode survives sleep and restart, agreed
  - Unobserved departure preserves the request; a request made while unplugged waits without expiring, agreed
  - Notify on power return after at least 30 minutes without power for a pending full-charge request, agreed
  - Loss of communication during sleep or shutdown can permit charging above 80%, accepted
- Loss of control
  - Restore charger power after 60 seconds without communication, agreed
  - Confirmed manual suspension restores battery preferences and permits charging; detected external power overrides it, agreed
  - Windows wins saved operating-mode conflicts with the ESP32, agreed
  - Use Windows feedback on unreliable voltage sensing, with persistent warnings and a separate fault LED, agreed
  - Sensor recovery rule deferred under D37
- Windows and Bluetooth behavior
  - Physical action opens a three-minute pairing window; successful pairing replaces the old computer, agreed
  - Current control continues during the window; successful replacement enables charging until new service commands, agreed
  - Preserve all five agreed plugged-in settings together; runtime overrides preferred, saved-setting fallback permitted
  - Restoring normal battery-dependent settings when communication is lost and the laptop is on battery
  - Service supplies settings and controls charging independently; tray starts at owner sign-in, agreed
  - Settings ownership, status reporting, and installation

## Decisions

- Q1: Proceed with external power cycling and configurable thresholds, initially 76% to resume and 80% to stop charging. The user accepts that a battery-lifespan benefit remains uncertain. Record actual battery use.
- Q2: The first version is for the user's personal use.
- Q3a: Preserve the user's selected plugged-in performance and display settings during managed charging pauses. Restore usual battery settings when communication is lost and external power is absent. The five-setting scope is now settled under D6.
- Q3b and Q7: The ESP32 must restore charger power after 60 seconds without communication. The user accepts that this can permit charging above 80%, including during overnight sleep, shutdown or a service failure that stops communication.
- Q4: A full-charge request prepares the laptop for leaving home without a charger. Keep charger power enabled after full charge until the user unplugs to leave. Resume threshold control on the next connection. Include manual cancellation.
- Q5, superseded by D2: The original choice was an inline charger-output switch, powered from the charger before the switch. D2 now selects AC-side switching and an independent controller supply.
- Q5 follow-up, historical: The user confirmed USB-C and proposed PD communication passthrough. The original charger and USB-C path now remain intact, so custom PD passthrough is no longer part of the design.
- Q6: The user requested research into which Windows settings depend on external power. D6 selects all five controls as one set.
- Q8: A full-charge request persists through sleep and restart until departure is detected or the user cancels it.
- D1: The first device is charge-only. USB data, video and dock support are outside the initial hardware scope; required USB-C charging and PD connections remain necessary.
- D2: Switch the original charger's AC input and power the ESP32 independently from an always-on supply. Use an electromechanical relay initially, then an AC-capable SSR. Keep service, BLE and UI behavior independent of switch technology. This supersedes the inline USB-C architecture; exact parts and electrical defaults remain unverified.
- D3: Automatic control must operate before Windows sign-in. A signed-in-user Bluetooth process cannot silently replace this requirement.
- D4 agreed: native C++ Windows service and Win32 tray/dialog, ESP-IDF C++ firmware with Arduino as a supported component, and shared portable service/ESP32 protocol definitions and codecs. Retain separate service/UI processes, restricted local named pipes and BLE. D4a selects shared C++ types and explicit binary encoding for the tray/service contract.
- D5: After ESP32 startup or reset, allow charging until communication with the Windows service is restored and the service sends explicit commands. The service must account for this recovery state; reconnecting Bluetooth alone must not replay an old pause.

Recommendations made during the interview remain proposals until the user answers.

## Power-settings distinctions

The laptop should preserve selected plugged-in preferences during managed charging pauses, then return to normal battery behavior when control is lost and external power is absent. This is separate from the ESP32 restoring charger power after communication loss.

Loss of Bluetooth communication alone does not establish whether external power is present. Lack of battery charging is also distinct from lack of external power. Windows exposes these power states separately and already has AC and battery power policies. Sources: [power status](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-system_power_status), [power policies](https://learn.microsoft.com/en-us/windows/win32/power/power-policy-settings).

The [Windows power-settings research](research/windows-plugged-in-settings.md) records the read-only local inventory and Microsoft sources. The user's saved AC and battery power modes differ, as do screen/sleep timeouts and lid action. Display, device policies, Energy Saver, Modern Standby, apps and scheduled tasks can also respond to the power source. D6 selects power mode, screen timeout, sleep timeout, brightness where supported and lid-close action together. Matching selected preferences will not make Windows report AC power or guarantee identical hardware performance. Battery critical actions must not be overwritten by copying all AC values.

The [inline-switch feasibility research](research/inline-power-switch-feasibility.md) records the superseded USB-C approach and why AC-side switching was selected. Its DC-output and PD-path design constraints apply to that earlier proposal. The 60-second communication watchdog still does not establish the electrical default during total ESP32 power loss or firmware failure.

## Next decisions

- Charger AC input and startup characteristics, ESP32 board, relay/SSR, driver, independent supply and enclosure. AC-side placement and independent controller power are settled; the original USB-C connection remains intact.
- Verify runtime APIs for all five Windows controls and design fallback ownership/recovery where needed.
- Electrical implementation of the agreed charging-enabled startup state, plus behavior if controller power or firmware fails. The accepted communication watchdog assumes a functioning controller with input power.
- Design exact departure evidence and mode-synchronization messages under the agreed rules: unseen departure preserves the request, Windows wins saved-mode conflicts, and detected external power clears manual suspension. Delayed detection during a pause is accepted.
- Implement the agreed service-restart restoration and explicit user-edit precedence; validate user/session scope.
- Complete protocol/authentication, installation, timing bounds and voltage fault-clear details within the agreed pairing, status, retry and test policies. Return to explicitly deferred D37 before selecting a sensor recovery rule.

## Implementation-design interview

The user requested an engineering interview based on GitHub issue #1. The issue and comments were read; there were no comments at that read. Existing decisions remain in force. The table below records first-round answers received on 2026-09-20 and remaining proposals.

### First round decisions and open work

| Decision | Recommendation | What depends on it |
| --- | --- | --- |
| D1: USB-C scope | Agreed: charge-only controller; original USB-C path remains intact under D2 | Physical compatibility tests |
| D2: Power-path architecture | Agreed: AC-side relay initially, AC-capable SSR later, independent always-on ESP32 supply | Switch and driver selection, fallback defaults, restart validation and departure detection |
| D3: Unattended Windows operation | Agreed: automatic control before sign-in; service-account Bluetooth access still needs verification | Provisioning and service identity; no user-session-only fallback without changing the requirement |
| D4: Software stack | Agreed: native C++ Win32 tray/dialog and service, Arduino as an ESP-IDF component, and shared portable service/ESP32 protocol definitions and codecs. BLE, process separation and local named pipes retained; tray/service types and explicit binary encoding agreed | Project structure, firmware dependencies, pairing and protocol design |
| D5: ESP32 reset behavior | Agreed: permit charging until the reconnected service explicitly commands otherwise; the service must reconcile recovery state | Electrical defaults, startup sequence and service reconnect tests |

### Proposed division of responsibilities

Windows owns the charging policy, thresholds, departure-mode intent, battery observations and temporary Windows preferences. The ESP32 owns switch operation and the independent 60-second communication timeout. The original charger and laptop own USB-C attachment and PD negotiation. Firmware maps a logical charging-enabled state to the chosen relay/SSR output, keeping switch details outside the service and BLE contract.

The Windows-to-device interface should request permission to supply laptop power or a time-limited managed pause, and report observed/acknowledged device state. Wire encoding, session identity, pairing, heartbeat cadence and power feedback remain later decisions. An acknowledgment of a switch command is not proof of actual laptop power delivery.

For D5, the proposed service recovery sequence is to establish a fresh control session, read device state and a fresh Windows battery reading, reevaluate the current thresholds or departure mode, then send an explicit command even if its desired state matches what it remembers from before the reset. A bare Bluetooth reconnect or a heartbeat must not restore an obsolete pause. The exact messages remain to be designed.

### Facts supporting this round

The PD hardware comparisons below describe the investigation before the accepted AC-side change. They do not require custom PD hardware in the current design.

- .NET 10 is an LTS release supported through November 14, 2028. Microsoft documents Worker Services hosted as Windows Services and WinForms tray icons. Sources: [support policy](https://dotnet.microsoft.com/en-us/platform/support/policy), [Windows Service hosting](https://learn.microsoft.com/en-us/dotnet/core/extensions/windows-service), [NotifyIcon](https://learn.microsoft.com/en-us/dotnet/api/system.windows.forms.notifyicon?view=windowsdesktop-10.0).
- Windows services run outside the interactive desktop. A separate GUI can communicate with a service through access-controlled local IPC. Source: [interactive services](https://learn.microsoft.com/en-us/windows/win32/services/interactive-services).
- Windows BLE device acquisition may request consent and require a UI thread. Desktop GATT support does not prove unattended access under the chosen service identity. Test discovery, reads/writes, notifications and reconnection after reboot before sign-in. A user-session Bluetooth broker is a possible fallback, but would change operation while logged out and requires a separate decision. Sources: [GATT client](https://learn.microsoft.com/en-us/windows/apps/develop/devices-sensors/gatt-client), [pairing](https://learn.microsoft.com/en-us/windows/uwp/devices-sensors/pair-devices).
- Espressif supports NimBLE for BLE and documents its smaller memory footprint compared with its dual-mode stack. ESP-IDF with NimBLE is an engineering recommendation, not a requirement. Source: [Espressif Bluetooth stacks](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/bluetooth/index.html).
- A PD hard reset can remove VBUS and return to initial voltage before renegotiation. Bench validation must capture PD/CC behavior and both power rails, including the controller supply. Sources: [ST PD reset behavior](https://www.st.com/resource/en/user_manual/um2552-managing-usb-power-delivery-systems-with-stm32-microcontrollers-stmicroelectronics.pdf), [TI PD startup](https://www.ti.com/lit/ab/slvaeq7/slvaeq7.pdf).
- Separately managed charger-facing sink and laptop-facing source roles add power-budget and voltage-transition responsibilities. They are a candidate if passthrough fails, not a selected circuit. Sources: [TI power-path considerations](https://www.ti.com/lit/wp/slyy145a/slyy145a.pdf), [TI power allocation](https://www.ti.com/document-viewer/lit/html/SSZTB83).

Minimum passthrough evidence includes repeatable power interruption and restoration without manual reconnect, controller supply behavior, PD reset/recovery, laptop unplugging during a pause, connector orientation, charger restart, controller reset and rated-load switching behavior. Success on one setup establishes only that setup's demonstrated behavior. The actual charger profiles and selected hardware remain unknown.

## Research constraint

The user requested that the battery-longevity recommendation exclude HP advice and use industry guidance. The [research findings](research/battery-aging-76-80-vs-100.md) draw on original experiments, university guidance and national-laboratory engineering methods. They support reducing prolonged full charge, but do not establish that external 76%-80% cycling improves this laptop's lifespan. No universal industry guideline prescribing that band was found. The user accepted proceeding with that uncertainty; the report's original recommendation to leave Q1 open is historical.

## Feasibility findings

- Removing adapter power makes the laptop operate on battery. Repeated cutoff and restoration therefore creates discharge/recharge cycles. It does not reproduce an internal charge cap that keeps the laptop running from its adapter. Source: [HP manual](https://h10032.www1.hp.com/ctg/Manual/c02441315.pdf).
- The local machine identifies as an HP OmniBook Ultra Flip Laptop 14-fh0xxx. Whether this is the intended target remains unconfirmed. Model-specific support for an internal charge limit has not been established.
- Official specifications for the 14-fh0xxx series describe USB-C power input and a 65 W USB-C adapter. This establishes the model's supported interface, not which charger the user intends to connect. Sources: [maintenance guide](https://kaas.hpcloud.hp.com/pdf-public/pdf_10997824_en-US-1.pdf), [series specifications](https://support.hp.com/jp-ja/document/ish_11131832-11131912-16?fallbackLocale=us-en&validated=true).
- An inline USB-C switch must account for attachment detection and power negotiation; interrupting only a power conductor cannot be assumed equivalent to disconnecting and reconnecting a compliant USB-C interface. Source: [USB-IF Type-C specification, Table 4-11](https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf). Switching upstream of the existing adapter avoids adding a new inline USB-C interface, an engineering inference from these requirements.
- Windows battery reporting includes unknown values. Unknown must remain distinguishable from a valid battery percentage. Source: [Microsoft SYSTEM_POWER_STATUS](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-system_power_status).
- ESP32 variants differ in Bluetooth support. BLE is agreed; the exact board remains undecided. Source: [Espressif Bluetooth capabilities](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/bt-architecture/overview.html).
- These findings do not establish that external power cycling improves battery lifespan. Reduced time at high charge and additional battery cycling must be considered separately.

## D2 and D4 research follow-up, 2026-09-20

The D2 investigation below led to AC-side switching. Its inline hardware recommendations are historical; the D4 stack was subsequently accepted, and D4a subsequently selected explicit binary encoding for the tray/service contract.

The [VBUS interruption report](research/usb-c-vbus-interruption.md) establishes that an ordinary USB-C sink must leave its attached state on sustained VBUS loss outside the specified reset/swap exceptions. The source detects detachment through CC, so continuously wired CC does not establish that both endpoints preserve their old negotiated state. Exact HP recovery and Windows reporting remain unmeasured. Disconnecting D+/D- does not affect PD attachment; PD uses CC.

The recommended D2 direction is controlled laptop-facing CC attachment and VBUS sequencing, restoring through initial 5 V and fresh PD negotiation. Opening a transparent CC path can also remove the charger's output feeding the ESP32. Independently managed upstream sink and downstream source ports would keep the controller supplied while the laptop port is detached, but add power-budget and voltage-conversion responsibilities. This is a proposed architecture, not an accepted circuit. VBUS-only passthrough remains an experiment whose behavior must be measured before relying on it.

The [native C++ and ESP-IDF report](research/native-cpp-and-esp-idf.md) confirms that Windows SDK APIs cover the tray icon, context menu, dialog, service and native BLE access. Before-login BLE access still requires a test under the intended service account. The native C++ and Arduino/ESP-IDF stack recommendations were accepted under D4. Shared portable headers and serializers should define explicit wire bytes, versions and validation without sharing platform handles or transmitting raw struct memory. D4a subsequently accepted shared C++ types and explicit binary encoding for the tray/service contract.

Astra's official model page describes coding capability but does not disclose a Win32-specific training inventory. It is not evidence that particular native Windows applications were in its training data. Evaluate generated code by compiling, reviewing and exercising it. Source: [OpenAI model documentation](https://developers.openai.com/api/docs/models/gpt-6-astra).

### Alternative raised by the user: switch the charger's AC input

The user asked whether switching AC and powering the ESP32 separately is preferable. For a personal first version, the recommendation is yes if the mains switching is provided by an appropriately certified, enclosed unit with an isolated low-voltage control interface. Keep the existing charger and USB-C cable intact. Power the ESP32 from an isolated supply on the unswitched side, so it can restore charger power while the charger is off. The switching unit must support the charger's capacitive startup load and repeated inrush, not just its steady-state wattage. Sources: [Omron relay application guidance](https://components.omron.com/kr-en/products/basic-knowledge/relays/applications), [TI PD negotiation](https://www.ti.com/document-viewer/lit/html/SSZTD99/GUID-F71348B1-4087-40A3-8DA0-6B5EE9FDCC6C).

This reduces custom USB-C engineering by leaving power-loss and restart negotiation to the original endpoints. It remains necessary to verify reliable charger restart, power-removal delay and the agreed charging-enabled fallback. A generic smart plug or relay does not automatically provide the local BLE control or fallback behavior required here.

The user accepted this alternative, choosing an electromechanical relay until an AC-capable SSR is available. [ADR 0004](adr/0004-switch-the-charger-ac-input.md) supersedes ADR 0002, and the current spec now includes AC-side switching. The logical charging interface and all other agreed behavior remain unchanged. Switch-specific drive, polarity, leakage and reset defaults must be verified when hardware changes; an SSR does not automatically reproduce a normally closed mechanical contact's unpowered behavior.

## Decisions recorded after the one-question-at-a-time interview

The user requested one question at a time and deferred recording until explicitly authorizing it after D23. This register records that batch. Later questions should continue one at a time; this recording request is not blanket authorization to record future answers without being asked.

| Decision | Accepted outcome |
| --- | --- |
| D4a | Shared C++ tray/service types with explicit binary encoding over restricted named pipes. Separate this contract from the service/ESP32 BLE contract; do not send raw struct memory. |
| D6 | Preserve power mode, screen-off timeout, sleep timeout, brightness where supported and lid-close action as one set. No individual toggles; retain low/critical-battery protections. |
| D7 | Windows plugged-in preferences are the source of truth, including edits. Prefer runtime-only overrides that leave saved settings unchanged. Saved-setting fallback with restoration is permitted if needed; an explicit battery-preference edit wins and suspends that setting's override for the rest of the pause. |
| D8 | Service integration tests use simulated battery/device/time/settings inputs. A real BLE prototype verifies before-login access, reported ESP32 state, reset, reconnection and watchdog behavior automatically. An LED is only a visual aid, not the test oracle. Physical charger and Windows-setting tests follow. |
| D9 | Accept delayed unplug detection during a managed pause. Provide "Use battery settings now" for immediate restoration; otherwise restore on communication loss or a later failed power-restoration observation. Allow measured capacitor discharge and charger startup delays. |
| D10 discussion | Persist normal/charge-to-full mode in ESP32 flash. Windows retains V1 threshold decisions. Startup still enables charging until a fresh service command; loss of communication still enables charging after 60 seconds. RTC-based battery estimation is outside V1. Persistence alone does not settle unseen departure during sleep. |
| D11 | Automatically restart the service and restore appropriate Windows preferences on recovery. A brief recovery interval is accepted; no independent fixed-deadline settings-recovery process is required in V1. |
| D12 | On failed/stale battery readings with communication alive, retain relay state and retry every 10 seconds for a configurable window, default three minutes from the first failure. Repeated failures do not restart the timer. At expiry permit charging until fresh readings return. The separate 60-second communication watchdog remains active. |
| D13 | On a new/recovered session without trustworthy prior state, a fresh reading strictly between thresholds makes the service pause until the resume threshold. This supersedes the earlier recommendation to charge toward the stop threshold. ESP32 boot still initially enables charging. |
| D14 | Reevaluate saved threshold edits immediately: enable at/below new resume, pause at/above new stop, retain current state inside the new band. Departure mode takes precedence. D38-D39 later select upper/gap presets saved together with Apply. |
| D15 | Pair one explicitly selected ESP32 during setup and automatically reconnect afterward, including before sign-in. Device replacement requires "Change device." |
| D16 | A physical button action opens a short new-pairing window; existing paired-device reconnection needs no button press. |
| D17 | Normally closed relay contacts enable charging with an unpowered coil or ESP32 off. Energize the coil to pause. The later SSR design must deliberately preserve the agreed default. |
| D18 | Ordinary cycles are silent. Notify for persistent problems requiring attention. Include a device status light. D30 and D33 later add explicit manual-cancellation and delayed-full-charge notifications. |
| D19 | Originally one low-brightness RGB LED: green for charging enabled, amber for pause, red for fault, blinking blue only for incomplete pairing. D36 supersedes fault indication with a dedicated red LED. No blinking for faults or normal operation. |
| D20 | Retain 30 days of local battery levels, power transitions and faults; remove older records automatically and offer CSV export. |
| D21 | Include a divider/ADC measurement of the charger's USB-C DC output in finished V1, deferred only from the initial BLE prototype. Distinguish measured voltage from applied relay state and battery charging. |
| D22 | If commanded off but voltage remains after settling time, alert and show steady red. Permit charging between retries. Retry after 30 seconds, then 1, 2, 4, 8 and 10 minutes; remain at 10-minute intervals. Delays start after the preceding failure and each attempt reevaluates current policy. This supersedes the earlier manual-Retry-only proposal. |
| D23 | Restrict configuration and manual controls to the owner's Windows account and administrators. Service control continues before sign-in and while the owner is logged out. |

The [current spec](specs/battery-charge-controller.md) applies these decisions throughout the behavior, hardware and acceptance sections. [ADR 0006](adr/0006-prefer-runtime-windows-preference-overrides.md) records the preference-override tradeoff. The [runtime override research](research/windows-runtime-power-overrides.md) documents the supported non-persistent policy route and its limits; the [IPC comparison](research/tray-service-contract.md) supports D4a.

## Decisions recorded through D44, 2026-09-21

The user authorized recording this batch, committing it and pushing it after answering D44. Continue future interview questions one at a time and keep later answers in the conversation until the user again asks to record them. D37 was explicitly postponed and is not an accepted rule. The specification holds current behavior; this register preserves the decision sequence and corrections.

| Decision | Outcome |
| --- | --- |
| D24 | If neither endpoint captured reliable evidence of an unplug/replug during sleep or downtime, keep the full-charge request active until confirmed departure or cancellation. Accept that an unobserved trip can leave it active after return. |
| D25 | Windows wins when saved operating modes disagree. Persist the user's choice before sending it and synchronize the ESP32 on reconnection; its stale mode cannot resurrect a canceled request. Charging-enabled startup and the 60-second fallback still apply. |
| D26 | Retain "Use battery settings now" with confirmation. On confirmation, restore battery preferences, permit charger power and suspend automatic threshold control. No added AC sensor was selected for this action. |
| D27 | Windows-detected external power takes priority and clears manual suspension. Offer the action only without external power. Use a modeless confirmation that closes if power returns; recheck at submission so a simultaneous return wins. Manual "Resume automatic control" is also available. Bluetooth reconnection alone does not clear suspension. This supersedes the proposal to require a later disconnect/reconnect if power was already present. |
| D28 | Persist manual suspension across service restart and Windows reboot. It ends on detected external power or explicit resumption; D31 also permits a new full-charge request to clear it. |
| D29 | Confirming the manual action cancels an active full-charge request, with user notification. |
| D30 | Warn about full-charge cancellation in the existing confirmation and notify after success. No extra confirmation step. If power returns before confirmation, close the dialog without canceling the request or showing success. |
| D31 | "Charge to full" during manual suspension clears suspension, permits charger power and activates the request. While unplugged, show waiting-for-power status and retain normal battery preferences. |
| D32 | A full-charge request made while unplugged does not expire. Proceed with full charging when power returns; 30 minutes without power is the notification threshold. |
| D33 | Notify only when power returns after at least 30 minutes without it, explaining that the earlier full-charge request is proceeding. Charge automatically and offer "Cancel full charge." Do not notify simply when the waiting timer reaches 30 minutes. |
| D34 | If the voltage sensor is unreliable, continue threshold control using usable Windows external-power feedback and notify the user. This rejects the proposal to suspend pauses solely because the sensor failed. Unknown readings are not zero voltage, and battery charging indication is not external-power presence. |
| D35 | Show one initial sensor-fault notification and a persistent tray warning badge with menu details. Dismissing the notification leaves the badge active until sensor recovery. The recovery definition remains deferred under D37. |
| D36 | Add a dedicated low-brightness, steady red fault LED. Keep the main RGB LED for charging enabled in green, pause in amber and incomplete pairing in blinking blue. This replaces D19's use of the RGB red state for faults. |
| D37 | **Deferred by the user.** Return later to the sensor recovery/clear rule. The suggested 30 seconds of valid, comparable readings was not accepted. |
| D38 | Expose upper-threshold presets of 90%, 85%, 82%, 80%, 78% and 75%, and gap presets of 2 through 8 percentage points in steps of one. Derive resume as upper minus gap. Defaults remain upper 80%, gap 4, resume 76%. |
| D39 | Show the derived resume threshold and save both selections together using Apply. Only then reevaluate active control under the existing threshold rules. |
| D40 | Physical button action opens pairing for three minutes. Close on success; expiry requires another button action. Remembered-computer reconnection remains automatic. |
| D41 | Allow one paired computer. Successful new pairing removes the previous computer's access. Opening the window, failure and timeout preserve the existing pairing. |
| D42 | Continue current control while the pairing window is open. On successful replacement, end the old control session, enable charger power and wait for fresh commands from the new computer. |
| D43 | The service supplies initial settings when it connects. Until then the ESP32 permits charging; temporary defaults do not start independent cycling. The user did not select the proposed separate default/import policy for a replacement computer. |
| D44 | Start the tray automatically when the owner's Windows account signs in. The service starts before sign-in and operates independently. |

### Remaining questions and validation

- **D37: revisit sensor recovery.** No duration, matching rule or automatic fault-clear test has been accepted.
- Exact departure evidence and synchronization protocol under the accepted D24-D25 behavior; unknown Windows power status and simultaneous-event handling beyond the confirmed power-return rule.
- Timing and persistence mechanics for the delayed full-charge notification across sleep and restart.
- Before-login BLE access, Windows API/session ownership, and runtime override support and release for all five controls.
- Exact components and voltage-sensing circuit, startup/shutdown measurements, voltage thresholds, sensor validity/disagreement criteria, simultaneous loss of sensor and Windows feedback, fault ownership and aggregation.
- Pairing authentication/button gesture and replacement protocol, message fields, stale-reading definition, battery-retry-window bounds, timing and deployment/uninstall details. Threshold choices and pairing-window duration are settled.

This step authorizes documentation, commit and push. Implementation, Windows settings changes and hardware tests remain outside its scope.

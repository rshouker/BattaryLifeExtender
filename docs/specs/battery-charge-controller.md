# Personal laptop battery charge controller

Status: draft specification for test-boundary review. No implementation has started and no issue has been published. The project issue tracker is not configured in this workspace. The requested tracker label is `ready-for-agent`; it has not been applied.

Requirements marked **agreed** come from the interview. Items marked **proposed** make the design concrete for review and are not recorded as prior user decisions. **Open** items must be resolved before the dependent work can be considered implementation-ready.

## Problem Statement

The user has an HP laptop that they report cannot enforce their preferred battery charge limit internally. They want to reduce time spent at full charge while retaining the performance and working preferences they normally use with the charger connected. They also need an easy way to prepare a full battery before leaving home without the charger.

Research supports avoiding prolonged full charge, but does not establish that deliberate 76% to 80% cycling improves this particular battery's lifespan. The user accepts this uncertainty. Success means reliable charge-range control and preservation of selected working preferences; a demonstrated increase in battery lifespan is not a first-version acceptance criterion.

## Solution

**Agreed:** Build an inline USB-C device between the charger output and laptop input. An ESP32 communicates with a Windows service over Bluetooth and controls an SSR that permits or interrupts power to the laptop. The controller draws power from the charger side before the switch, regulated to the voltage required by the selected board.

Normal operation restores charger power at the resume threshold, initially 76%, and interrupts it at the stop threshold, initially 80%. The thresholds are configurable from a tray application. While the laptop runs on battery during a managed charging pause, preserve selected plugged-in preferences. Restore normal battery behavior when control is lost and external power is absent.

A full-charge request starts departure mode. It keeps charger power enabled until the user unplugs to leave or cancels the request. It survives sleep and restart. Threshold control resumes on the next connection after departure.

The ESP32 restores charger power after 60 seconds without communication. The user accepts that loss of communication during sleep, shutdown, or service failure can allow charging beyond 80%, including overnight.

The user proposes passing through USB-C Power Delivery communication. PD uses the CC connection. The feasibility of retaining that connection while switching VBUS, including controller supply continuity, remains open. The inline placement is settled; the electrical circuit is not.

## User Stories

Stories 1 through 19 express agreed outcomes. Stories 20 through 30 are proposed supporting behavior needed to operate and verify the first version; their details remain subject to the open decisions below.

1. As a laptop owner, I want the first version to work for my own laptop and device, so that the project can focus on my daily use.
2. As a laptop owner, I want the device between the USB-C charger and laptop, so that it can control external power at that connection.
3. As a laptop owner, I want the controller to draw power from the charger, so that it can operate independently of the switched laptop power path.
4. As a laptop owner, I want the service to communicate with the ESP32 over Bluetooth, so that Windows can control charging without an additional control cable.
5. As a laptop owner, I want charger power interrupted at 80% by default, so that normal operation avoids charging all the way to full.
6. As a laptop owner, I want charger power restored at 76% by default, so that the laptop replenishes the charge used during the pause.
7. As a laptop owner, I want to configure both thresholds, so that I can change the charging range.
8. As a laptop owner, I want a Windows service to manage the device, so that control has a dedicated background component.
9. As a laptop owner, I want a tray application for charging controls, so that those controls are easy to reach while working.
10. As a laptop owner, I want selected plugged-in preferences preserved during managed charging pauses, so that cycling does not repeatedly change my working environment.
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
- The ESP32 operates the power switch and independently enforces the 60-second communication timeout.
- The switching output must support DC. A DC control input on an AC-output SSR does not meet this requirement.
- Controller input power is taken upstream of the laptop power switch. The selected board determines whether its regulated input is 5 V or 3.3 V. USB-C negotiation can change or remove that upstream supply, so continuous controller operation cannot be inferred solely from tap placement.
- This is a personal first version. Language, Windows UI framework, ESP32 variant, Bluetooth mode, protocol and hardware parts have not been selected.

### Charging behavior

**Agreed:** Start with resume at 76% and stop at 80%. Departure mode permits full charge until departure or cancellation. The device restores charger power after 60 seconds without communication.

**Proposed behavior to make threshold control deterministic:**

| Condition | Desired behavior |
| --- | --- |
| Normal mode; valid battery level at or above the stop threshold | Interrupt laptop power and enter a managed charging pause once the device action is confirmed. |
| Normal mode; valid battery level at or below the resume threshold | Restore laptop power. |
| Normal mode; level strictly between thresholds during an established session | Retain the last confirmed switch state. |
| New or recovered session; level strictly between thresholds | Keep power enabled until a fresh session is established and the stop threshold is reached. |
| Departure mode | Keep power enabled regardless of the normal thresholds. |
| Manual departure-mode cancellation | Return to threshold control and reevaluate the current valid battery reading. |
| Confirmed physical departure | End departure mode and temporary preference preservation; resume normal threshold control on a later connection. |
| Battery data unknown, invalid or too old to use | Permit charger power where control is available; do not continue a pause based on stale data. |
| ESP32 has received no valid control communication for 60 seconds | Restore charger power independently of the Windows process. |

Threshold edits must be validated and reevaluated against the current reading. Proposed validation is whole percentages with 0 <= resume < stop <= 100; the product may choose a narrower configurable range before implementation. The numerical values are software thresholds, not a promise of exact electrochemical state of charge or zero overshoot.

Freshness limits, heartbeat frequency, response deadlines and allowable switching delay remain open. An alive Bluetooth connection must not indefinitely extend a pause when the service no longer has usable battery information. The device watchdog should measure valid messages from the associated controller using its own monotonic timer.

### Separate the observable states

Track Windows external-power presence, battery level, charging indication, Bluetooth availability, requested switch state, and acknowledged device state separately. Acknowledging an output command establishes controller intent or output state; it does not establish that laptop input power physically changed unless appropriate feedback exists.

Bluetooth loss does not prove physical departure. A battery that is not charging can still be externally powered. During a managed pause, Windows already reports battery operation, so physical unplugging may produce no new AC-loss transition. Departure detection requires a defined strategy before departure-mode and settings-restoration behavior are complete.

### Plugged-in preferences

**Agreed:** Preserve selected performance and display preferences during managed charging pauses. The exact set remains open.

**Proposed first-version controls:** Windows power mode, screen timeout, sleep timeout, brightness where supported, and lid-close action. Hibernate timeout, wake timers, Wi-Fi and video settings are candidates for later inclusion. This proposed list was researched but has not been accepted by the user.

Apply overrides only during a confirmed managed pause under active control. Recover the previous battery preferences when the pause ends or control is lost. A durable record of changes is needed so service restart can recover interrupted changes. Conflict handling for user edits during a pause remains open; silently replacing user edits with an obsolete snapshot is unacceptable.

Do not mirror every hidden AC power-plan value. Critical battery actions and low-battery protections must remain appropriate for actual battery operation. Matching selected settings does not spoof external-power status and cannot guarantee that firmware, drivers, applications or scheduled tasks behave as though the charger is connected.

### Persistence and interface proposals

- Persist threshold configuration and departure mode. Keep control-session state and telemetry freshness separate from persisted user intent.
- Model settings recovery records separately from normal configuration, so a crash cannot erase the values needed for restoration.
- The service should provide the tray with current mode, battery reading and freshness, actual external-power status, connection state, requested/acknowledged switch state, and actionable errors.
- Bluetooth commands should identify the current control session and support acknowledgment, repeat-safe retries, and recovery without applying stale commands from a previous session. Exact encoding, authentication and pairing are open.
- Record timestamped power-source transitions, battery levels, requested and acknowledged switch changes, departure-mode changes and fallback events. Record battery charge or energy throughput and temperature only where supported; estimates must be identified as estimates. Retention and export format remain open. Cloud telemetry is not required.
- Closing the tray should not stop charging control. Service startup, tray startup, installation, permissions, user-session handling and uninstall restoration remain to be specified.

### Hardware constraints and unresolved circuit design

- Preserve the selected inline USB-C arrangement. Validate the proposed CC passthrough with switched VBUS against charger and laptop attachment, interruption, reset, restoration and physical reconnection behavior.
- Select the SSR or semiconductor power-switch implementation from the actual charger's voltage/current profiles. Verify off-state behavior, losses, heating, inrush and reverse-current behavior for the selected topology.
- Verify regulator operation across all supported input voltages and PD transitions. Budget controller consumption within available charger output.
- The 60-second firmware fallback assumes a powered, running ESP32. It does not specify power-path behavior during ESP32 reset, firmware failure, regulator failure or loss of upstream input. Select electrical defaults and recovery behavior explicitly.
- The exact charger, board, switch and regulator are not specified. No particular circuit is approved by this spec.

## Testing Decisions

No source code or test framework exists in the workspace. There are no existing application test boundaries or prior tests to reuse.

### Proposed primary test boundary, awaiting user confirmation

Use the externally observable behavior of the Windows service as the main automated integration-test boundary. Drive it with battery/external-power observations, tray requests, device replies and elapsed time. Observe device commands, published status, persistence and requests to apply or restore Windows preferences.

Substitute controlled battery, device, clock and Windows-settings adapters at the service boundary. Keep the charging policy inside that boundary so tests do not depend on private helper structure. Exercise real service coordination and persistence with isolated test storage. Tests should cover useful user scenarios rather than duplicate implementation branches.

### Proposed scenarios

1. Reach 80%, pause, remain paused through intermediate readings, reach 76%, and restore power without repeated switching at a single threshold.
2. Reject invalid threshold edits; apply a valid edit and reevaluate current charge according to the chosen policy.
3. Request departure mode below, within and above the normal band; keep power enabled at full charge.
4. Preserve departure mode across sleep, service restart and operating-system restart; cancel it only through the agreed cancellation/departure rules.
5. Confirm departure and return; resume threshold control without treating the former request as permanent.
6. Lose Bluetooth while paused; end preference preservation appropriately and verify firmware restores power at the 60-second deadline.
7. Crash or stop the service; verify the device watchdog works without Windows issuing a restore command.
8. Receive unknown or stale battery information while communications remain alive; avoid indefinite power interruption.
9. Delay, duplicate, lose or reject commands and replies; avoid falsely reporting a successful physical power change.
10. Reconnect after the device has applied its fallback; use fresh state and reject obsolete session commands.
11. Preserve only selected Windows settings during a managed pause; restore previous preferences when preservation ends.
12. Restart after an interrupted settings change; recover settings without corrupting configuration or silently overwriting a deliberate user edit.
13. Report battery operation accurately while overrides are active; keep critical battery actions intact.
14. Close the tray while the service runs; keep charging control active. Report a stopped service accurately when the tray remains open.
15. Restart with no known switch state and a battery reading inside the range; apply the reviewed initial-state rule.

### Required physical and Windows verification

The service-level simulator cannot establish electrical behavior. A separate hardware-in-the-loop check must verify the ESP32 watchdog and actual power path, including loss of communication, switch interruption/restoration, orientation, PD reset, charger/laptop unplugging, controller reset and supply continuity. Check negotiated voltage/current behavior and switching losses with suitable equipment before unattended use. These checks depend on selected hardware and a reviewed circuit.

On the target laptop, verify each selected power-setting override and its restoration through real pauses, reconnection, sleep and restart. Check unsupported settings explicitly. A saved Windows setting alone does not establish runtime behavior or performance equivalence.

Test logs may demonstrate threshold operation, power transitions and battery use. A short test cannot demonstrate years of battery-life extension.

## Out of Scope

- Demonstrating or guaranteeing longer battery lifespan, or claiming that a four-percentage-point band is universally optimal.
- A product for multiple users, multiple computers or arbitrary charger hardware.
- Mains-side switching, battery-pack modification, or replacing the laptop's internal battery management system.
- Cloud control, Wi-Fi control, remote accounts or fleet management.
- Making Windows falsely report AC power or overriding every application, firmware or driver response to battery use.
- Holding the battery at 80% while communication is unavailable. The accepted fallback can permit charging to full.
- Selecting a production enclosure, manufacturing process or commercial certification program as part of this personal software specification.
- Implementing or purchasing hardware, changing Windows settings, or generating implementation tickets as part of this specification-writing task.

## Further Notes

### Open decisions and dependencies

| Item | What is known | What it blocks |
| --- | --- | --- |
| Test boundary | Service integration boundary plus separate physical verification proposed | Final testing-plan approval and tracker publication under the invoked skill |
| Tracker configuration | No configured project tracker is present | Publishing and applying `ready-for-agent` |
| Exact target and charger | Local system appears to be an HP OmniBook Ultra Flip 14-fh0xxx; USB-C confirmed; actual adapter profiles unknown | Component ratings and hardware acceptance plan |
| USB-C power switching | Inline architecture and communication passthrough requested | Approved circuit and dependable reconnect/supply behavior |
| ESP32, SSR, regulator | No parts chosen | Firmware target, Bluetooth capabilities and electrical validation |
| Controller failure defaults | 60-second communication fallback agreed | Boot, reset and controller-power-loss behavior |
| Preserved Windows controls | Categories researched; final control list unaccepted | Complete settings adapter and user-facing promises |
| Departure detection | End request on departure and resume thresholds on return agreed | Reliable mode cancellation, including unplugging during a pause or sleep |
| Recovery and conflicts | Restore battery preferences; preserve departure intent | Exact recovery contract when the user changes settings or departs while the service is unavailable |
| Operational details | Personal Windows service and tray required | Pairing, startup, permissions, installer, notification and logging-retention details |

The open items limit implementation readiness, not the validity of the agreed product goals. Proposed behavior must remain distinguishable from an accepted decision when the spec is split into tickets.

### Supporting decisions and research

- [Domain glossary](../../CONTEXT.md)
- [Discovery and agreed answers](../discovery.md)
- [External-power control decision](../adr/0001-control-external-charger-power.md)
- [Inline placement decision](../adr/0002-switch-the-charger-output.md)
- [Battery-aging evidence](../research/battery-aging-76-80-vs-100.md)
- [Windows power-settings research](../research/windows-plugged-in-settings.md)
- [Inline USB-C feasibility](../research/inline-power-switch-feasibility.md)

The battery-longevity conclusion excludes HP advice at the user's request. Manufacturer documentation remains usable for connector and hardware specifications.

# Project discovery

Status: synthesized into a [draft specification](specs/battery-charge-controller.md) at the user's request. This document retains the interview decisions and unresolved questions. The specification's proposed test boundary awaits user confirmation, and issue-tracker setup is required before publication.

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
  - Inline device between charger output and laptop input, powered from the charger before the switch, agreed
  - SSR and regulator selection, connector handling and power negotiation remain open
  - Hardware construction and enclosure
- Charging policy
  - Configurable thresholds, initially resume at 76% and stop at 80%, agreed
  - Full-charge request keeps power enabled until departure or manual cancellation, agreed
  - Departure mode survives sleep and restart, agreed
  - Loss of communication during sleep or shutdown can permit charging above 80%, accepted
- Loss of control
  - Restore charger power after 60 seconds without communication, agreed
  - Recovery and manual override
- Windows and Bluetooth behavior
  - Device selection and pairing
  - Preserve selected plugged-in settings during deliberate charger cutoffs, agreed; exact settings remain open
  - Restoring normal battery-dependent settings when communication is lost and the laptop is on battery
  - Service and tray responsibilities
  - Settings ownership, status reporting, and installation

## Decisions

- Q1: Proceed with external power cycling and configurable thresholds, initially 76% to resume and 80% to stop charging. The user accepts that a battery-lifespan benefit remains uncertain. Record actual battery use.
- Q2: The first version is for the user's personal use.
- Q3a: Preserve the user's selected plugged-in performance and display settings during managed charging pauses. Restore usual battery settings when communication is lost and external power is absent. The exact settings remain to be selected.
- Q3b and Q7: The ESP32 must restore charger power after 60 seconds without communication. The user accepts that this can permit charging above 80%, including during overnight sleep, shutdown or a service failure that stops communication.
- Q4: A full-charge request prepares the laptop for leaving home without a charger. Keep charger power enabled after full charge until the user unplugs to leave. Resume threshold control on the next connection. Include manual cancellation.
- Q5: Place the device between the charger output and computer power input. Power the controller from the charger side before the switching element, using regulation to 5 V or 3.3 V as required by the selected board. Use an SSR to allow or interrupt power to the laptop. Exact switch technology and ratings, regulator and USB-C behavior require validation; no component or circuit has been selected.
- Q5 follow-up: The user confirmed USB-C and proposes passing through the communication lines for USB Power Delivery. This is the requested approach to investigate, not a verified electrical design. The charger's output profiles and the ESP32 and SSR part numbers remain unknown.
- Q6: The user requested research into which Windows settings depend on external power. The specific settings to preserve are not yet selected.
- Q8: A full-charge request persists through sleep and restart until departure is detected or the user cancels it.

Recommendations made during the interview remain proposals until the user answers.

## Power-settings distinctions

The laptop should preserve selected plugged-in preferences during managed charging pauses, then return to normal battery behavior when control is lost and external power is absent. This is separate from the ESP32 restoring charger power after communication loss.

Loss of Bluetooth communication alone does not establish whether external power is present. Lack of battery charging is also distinct from lack of external power. Windows exposes these power states separately and already has AC and battery power policies. Sources: [power status](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-system_power_status), [power policies](https://learn.microsoft.com/en-us/windows/win32/power/power-policy-settings).

The [Windows power-settings research](research/windows-plugged-in-settings.md) records the read-only local inventory and Microsoft sources. The user's saved AC and battery power modes differ, as do screen/sleep timeouts and lid action. Display, device policies, Energy Saver, Modern Standby, apps and scheduled tasks can also respond to the power source. Exact first-version controls remain a user decision. Matching selected preferences will not make Windows report AC power or guarantee identical hardware performance. Battery critical actions must not be overwritten by copying all AC values.

The [inline-switch feasibility research](research/inline-power-switch-feasibility.md) documents the proposed USB-C approach. PD uses the CC connection. CC passthrough with switched VBUS remains unvalidated; attachment, power resets, reconnection and upstream controller supply continuity require verification. A DC-capable output switch is required. The 60-second communication watchdog does not establish the electrical default during ESP32 power loss or reset.

## Next decisions

- Charger output ratings, ESP32 board, switching component, regulator and enclosure. USB-C, inline placement and charger-derived controller power are settled. The proposed PD communication passthrough requires validation.
- Which performance and display settings to preserve during managed charging pauses.
- Controller boot behavior and hardware behavior if its power or firmware fails. The accepted communication watchdog assumes a functioning controller with input power.
- Departure detection when the charger is unplugged during a managed pause while Bluetooth remains connected.
- Restoration of temporary Windows settings after service failure or restart, and handling user changes while overrides are active.
- Pairing, status feedback, configuration, installation and acceptance scenarios after hardware and power-policy scope are settled.

## Research constraint

The user requested that the battery-longevity recommendation exclude HP advice and use industry guidance. The [research findings](research/battery-aging-76-80-vs-100.md) draw on original experiments, university guidance and national-laboratory engineering methods. They support reducing prolonged full charge, but do not establish that external 76%-80% cycling improves this laptop's lifespan. No universal industry guideline prescribing that band was found. The user accepted proceeding with that uncertainty; the report's original recommendation to leave Q1 open is historical.

## Feasibility findings

- Removing adapter power makes the laptop operate on battery. Repeated cutoff and restoration therefore creates discharge/recharge cycles. It does not reproduce an internal charge cap that keeps the laptop running from its adapter. Source: [HP manual](https://h10032.www1.hp.com/ctg/Manual/c02441315.pdf).
- The local machine identifies as an HP OmniBook Ultra Flip Laptop 14-fh0xxx. Whether this is the intended target remains unconfirmed. Model-specific support for an internal charge limit has not been established.
- Official specifications for the 14-fh0xxx series describe USB-C power input and a 65 W USB-C adapter. This establishes the model's supported interface, not which charger the user intends to connect. Sources: [maintenance guide](https://kaas.hpcloud.hp.com/pdf-public/pdf_10997824_en-US-1.pdf), [series specifications](https://support.hp.com/jp-ja/document/ish_11131832-11131912-16?fallbackLocale=us-en&validated=true).
- An inline USB-C switch must account for attachment detection and power negotiation; interrupting only a power conductor cannot be assumed equivalent to disconnecting and reconnecting a compliant USB-C interface. Source: [USB-IF Type-C specification, Table 4-11](https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf). Switching upstream of the existing adapter avoids adding a new inline USB-C interface, an engineering inference from these requirements.
- Windows battery reporting includes unknown values. Unknown must remain distinguishable from a valid battery percentage. Source: [Microsoft SYSTEM_POWER_STATUS](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-system_power_status).
- ESP32 variants differ in Bluetooth support. The exact board and Bluetooth mode remain undecided. Source: [Espressif Bluetooth capabilities](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/bt-architecture/overview.html).
- These findings do not establish that external power cycling improves battery lifespan. Reduced time at high charge and additional battery cycling must be considered separately.

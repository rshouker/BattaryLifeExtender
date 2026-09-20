# USB-C VBUS interruption with CC passthrough

Research date: 2026-09-20. Supports engineering decision D2. No hardware measurements have been made on the user's charger or HP laptop.

## Answer

Keeping CC connected does not guarantee that the laptop preserves its negotiated power state when VBUS disappears. The USB Type-C sink state machine treats sustained loss of VBUS as detachment. The charger detects detachment differently, through CC. Cutting only the laptop's VBUS can therefore make the two endpoints disagree about whether they are still attached.

Use a controlled Type-C disconnect/reconnect for the intended charging pause. Treat VBUS-only switching as a bench experiment until its recovery behavior is demonstrated. This recommendation concerns predictable recovery, not a claim that every VBUS-only switch is physically impossible.

## What the specification establishes

USB-IF Type-C Release 2.0, section 4.5.2.2.5.2, requires an ordinary sink to leave Attached.SNK for Unattached.SNK within tSinkDisconnect when VBUS falls below vSinkDisconnect at up to 5 V, or vSinkDisconnectPD above 5 V. Exceptions include an ongoing Hard Reset, Power Role Swap or Fast Role Swap. A laptop is not normally the special VCONN-powered-device exception. The threshold and debounce matter; a short glitch is not equivalent to a minutes-long pause. [USB-IF Type-C R2.0, page 174](https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf)

The source watches CC for removal of the sink's Rd termination. On detach it removes VBUS and VCONN and resets its interface. USB Type-C R2.4 describes this in section 4.5.1.2. [USB-IF Type-C R2.4, page 163, hosted by TI](https://e2e.ti.com/cfs-file/__key/communityserver-discussions-components-files/196/USB-Type_2D00_C-Spec-R2.4-_2D00_-October-2024.pdf)

The practical consequence is that an old 20 V agreement cannot simply be assumed to survive a sink detach and subsequent attachment. This report did not directly verify the complete USB PD contract-invalidation text from an official-hosted PD specification. It therefore does not assert the exact policy-engine path taken by this HP implementation.

## Expected behavior and what remains unknown

| Event | Standards or documented behavior | What needs measurement on this laptop |
| --- | --- | --- |
| Open only downstream VBUS | Once its voltage crosses the disconnect threshold, the ordinary sink leaves its attached state, subject to the stated exceptions. | Discharge time, Windows AC indication, CC termination changes, and whether recovery begins with a Hard Reset or another path. |
| Keep CC connected during the pause | The charger uses CC to detect detach; it may still see the laptop's Rd. | Whether the charger keeps its previous voltage, resets after PD messages, or sees a brief Rd removal. |
| Restore VBUS while upstream remains at 20 V | This is not the ordinary new-attachment sequence. TI's source example attaches at 5 V and raises voltage only after negotiation. | Acceptance, protection response, retry behavior and whether manual unplugging becomes necessary. No specific HP outcome is established. |
| Open the active CC connection too | A source that recognizes detach removes VBUS and VCONN. | Actual discharge timing and whether the ESP32 loses power before it can complete the sequence. |
| Perform a PD Hard Reset | ST documents VBUS removal, recovery to 5 V and restarted capability exchange. | Whether the upstream regulator and ESP32 ride through or reboot. |

Sources for the first two rows are above. The restore sequence is documented in [TI SLVAEQ7, section 3](https://www.ti.com/lit/ab/slvaeq7/slvaeq7.pdf). The Hard Reset behavior is documented in [ST UM2552, section 4.3.4](https://www.st.com/resource/en/user_manual/um2552-managing-usb-power-delivery-systems-with-stm32-microcontrollers-stmicroelectronics.pdf).

Three plausible outcomes deserve separate tests. The sink can retain Rd while waiting for power, leaving the source unaware of the interruption. It can change CC termination or issue recovery signaling, causing the source to drop or cycle VBUS. It can recover after restoration on this particular pair of devices. None is evidence of a universally preserved PD contract. As an implementation example, TI support describes a TPS26750 removing its CC pulldown after low-VBUS detection. That is evidence that the second outcome exists, not evidence that HP uses that controller. [TI support response](https://e2e.ti.com/support/power-management-group/power-management/f/power-management-forum/1530558/tps26750-can-i-connect-vbus_lv-to-a-3-3v-ldo-instead-of-tpd4s480)

## Which lines should be controlled?

PD messages travel on the active CC line, not USB D+/D-. Disconnecting ordinary USB data does not create the needed PD detach. For charge-only v1, USB data and video can be omitted as a product scope decision. CC orientation and VCONN still need proper handling. [ST AN5225, section 6](https://www.st.com/resource/en/application_note/dm00536349.pdf)

My engineering recommendation is to give the design explicit control of the laptop-facing attachment and power path. On pause, withdraw that source attachment and remove/discharge laptop VBUS under a Type-C controller's sequencing. On resume, begin a fresh attachment at 5 V and negotiate any higher voltage. Do not hard-wire the fallback to connect an existing upstream 20 V rail regardless of port state.

For a transparent CC arrangement, intentionally opening CC also detaches the charger from its only sink. The charger's upstream VBUS can then disappear, taking the ESP32 regulator input with it. Merely moving that regulator before the laptop SSR does not solve this. Keeping the controller alive throughout a long pause requires a deliberate supply strategy.

The cleanest fit to the stated upstream-powered ESP32 requirement is an independently managed upstream PD sink and downstream PD source. The charger can continue powering the device while the downstream port is detached. This adds voltage-generation and power-budget responsibilities, so it remains a recommendation for D2, not a selected circuit. A transparent arrangement with separate controller energy storage or supply is another candidate if the user accepts that added requirement. In either case, the inline DC-capable SSR or MOSFET power switch remains part of the architecture.

This supply concern is observable in official vendor guidance: TI documents a VBUS-powered controller disconnecting its CC terminations, losing VBUS, resetting and repeating attachment. [TI TPS25751 support response](https://e2e.ti.com/support/power-management-group/power-management/f/power-management-forum/1588718/tps25751evm-tps25751d---verify-liquid-detection-in-only-sink-mode)

## Minimum bench validation

1. Identify the charger, its supported output profiles, both cables and the proposed switch. Capture a normal negotiated connection as the baseline.
2. With a PD analyzer and appropriate voltage/current measurement, capture upstream VBUS, laptop-side VBUS, CC traffic and the ESP32 supply during cutoff. Record detachment, Hard Reset and new negotiation rather than relying on the battery icon.
3. Restore after short and extended pauses. Check the voltage actually presented before new negotiation, automatic charging recovery, current transients and repeated retries.
4. Repeat with laptop unplug/replug during a pause, both connector orientations, charger restart and ESP32 reset. Check that fallback restores protocol-managed power and does not repeatedly restart itself.

Passage of these tests establishes behavior only for the tested combination. It does not establish USB compliance or compatibility with other chargers and laptops.

## Access limits

The USB-IF R2.0 PDF exceeded the browser's full-document fetch limit in the earlier research. The exact section 4.5.2.2.5.2 was available as search-indexed primary-source text. R2.4 source-detach text was available from the specification PDF hosted on TI. Secondary mirrors appeared in searches but are not relied upon here. No exact HP PD controller, firmware behavior, actual charger model or oscilloscope trace was available.

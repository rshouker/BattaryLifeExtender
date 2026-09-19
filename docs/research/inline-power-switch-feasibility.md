# Inline power switch feasibility

Research date: 2026-09-19. This note supports the discovery interview, not a completed circuit design or component selection.

The accepted architecture places the device between the charger output and laptop input. The ESP32 takes power upstream of the laptop power switch through a regulator. The user confirmed USB-C and proposes passing through the PD communication lines while switching power. Charger output profiles and selected components remain unconfirmed.

## USB-C power requires attachment management

A USB-C charger is not an always-on fixed-voltage supply. USB Type-C attachment uses the CC pins. The source removes VBUS when the sink is removed. TI's reference design shows VBUS initially off, then 5 V following attachment, with a higher voltage supplied after PD negotiation. Sources: [USB-IF Type-C specification, attachment sequence](https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf), [TI USB-C PD source application report, section 3](https://www.ti.com/lit/ab/slvaeq7/slvaeq7.pdf).

USB-C PD messages travel on the active CC line. Ordinary USB D+/D- data-line passthrough does not pass PD communication. Cable orientation and CC/VCONN handling also matter. Source: [ST AN5225, section 6](https://www.st.com/resource/en/application_note/dm00536349.pdf).

Engineering consequences for this project:

- An upstream regulator tap does not by itself guarantee ESP32 power after laptop removal. If the laptop supplies the only sink attachment, removing it can also remove the charger's output.
- Leaving CC connected while opening only VBUS cannot be assumed to reproduce a normal disconnect. Opening CC instead affects the upstream power supplying the ESP32. The design must explicitly handle both interfaces, attachment and PD contracts.
- If the device maintains its own upstream sink connection, it must also handle the laptop-facing source behavior and available power budget. A PD trigger holding 20 V upstream does not justify applying 20 V to a newly attached laptop before negotiation.
- The regulator must tolerate every supported upstream voltage and transition. Its power draw counts toward the charger's available output. Continuous operation through resets or voltage interruptions needs verification, even if the regulator tap precedes the switch.

These are design constraints inferred from the cited attachment sequence. CC passthrough with switched VBUS is not established as impossible, but its reset, recovery and reattachment behavior cannot be assumed correct. No transparent passthrough circuit has been validated.

Two inline approaches remain to investigate. First, preserve charger-to-laptop CC communication and validate VBUS interruption and restoration against both endpoints, including PD reset and physical unplug behavior. Second, let the inline device manage an upstream sink and downstream source separately, allocating power to its own controller. The second approach makes those responsibilities explicit but adds hardware and firmware. Neither is selected by this research note.

## The switch must support DC

An SSR with a DC control input can still have an AC-only output. Omron states that triac/thyristor SSRs for AC loads cannot reset with a DC load. Select a DC-capable output technology after the charger ratings are known. Source: [Omron FAQ02101](https://www.ia.omron.com/support/faq/answer/18/faq02101/).

MOSFET power switching is a plausible implementation family. Voltage/current ratings, losses and heating, inrush, off-state leakage, reverse current and the control state during ESP32 reset need review. TI's USB-C controller provides an example of managed power paths with reverse-current protection and slew-rate control. This is evidence of relevant design concerns, not a recommendation for that part. Source: [TI TPS25750](https://www.ti.com/product/TPS25750).

## ESP32 supply and loss of control

For the original ESP32, Espressif recommends a 3.3 V supply capable of at least 500 mA. A development board may accept 5 V through an onboard regulator, but the exact board determines the permitted input. Source: [Espressif schematic checklist](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32/schematic-checklist.html).

A firmware rule that restores laptop power after 60 seconds without communication requires a powered, running controller. It does not specify what happens during regulator failure, ESP32 reset or total controller power loss. The switch's electrical default and recovery behavior need a separate decision. This follows from the proposed architecture, not from an assumed normally-on SSR.

## Facts needed next

1. Actual charger model and every output voltage/current rating on its label. USB-C is now confirmed.
2. Whether a dock or monitor also supplies laptop power.
3. Exact ESP32 board, regulator module and SSR or MOSFET module, if already chosen.
4. Required laptop-power behavior when the controller itself loses power or resets.

The local laptop model alone does not answer these questions. Keep the inline architecture and resolve these facts before selecting a circuit.

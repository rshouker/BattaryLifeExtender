# Switch the charger output

The user selected an inline device between the charger's output and the laptop's power input, using an SSR to interrupt laptop power and drawing controller power from the charger side before that switch. This fixes the control device's placement and requires a supply regulator suited to the selected ESP32 board; the exact connector, switch ratings and any USB-C power-negotiation design remain to be verified before implementation.

# Switch the charger's AC input

Follow-up decision: use normally closed contacts so an unpowered relay coil permits charging, and energize the coil to pause. The later SSR arrangement must preserve the agreed charging-enabled default during controller power loss.

The user replaced the inline USB-C design with switching the original charger's AC input and powering the ESP32 from an independent, always-on supply, avoiding custom USB-C attachment and PD management. Use an electromechanical relay initially, with an AC-capable SSR planned later; Windows and the BLE interface continue to request charging enabled or paused without depending on switch technology. This supersedes ADR 0002; each physical switch and driver must implement the agreed startup and communication-loss behavior, so an SSR substitution still requires electrical validation.

# Battery charge control

An external device controls whether a laptop receives power from its charger. The user chooses charging thresholds and can request a full charge.

## Language

**Charger**:
The external power adapter that supplies power to the laptop.
_Avoid_: Controller, switch

**Power switch**:
The external device inserted between the charger output and the laptop power input that allows or interrupts power to the laptop.
_Avoid_: Charger

**Charging enabled**:
The state in which the power switch permits charger power to reach the laptop. The battery may already be full and need no charging current.

**Battery level**:
The laptop's reported remaining battery charge, expressed as a percentage.

**Charging thresholds**:
The lower and upper battery levels that define the user's normal charging range.

**Resume threshold**:
The lower battery level at which charger power is restored during threshold control.

**Stop threshold**:
The upper battery level at which charger power is interrupted during threshold control.

**Managed charging pause**:
An intentional interruption of charger power to keep the battery within its configured range while the laptop remains under the device's control.

**Plugged-in preferences**:
The user's selected performance and display preferences for working with the charger, also preserved during a managed charging pause.

**Full-charge request**:
A user request to prepare the laptop for use away from its charger by allowing the battery to reach full charge.

**Departure mode**:
The temporary override started by a full-charge request, which keeps charger power enabled until the user unplugs to leave or cancels the request.

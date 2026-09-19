# Control external charger power

The user reports that the target laptop lacks the desired internal charge-limit control, so the project will use an external ESP32-controlled power switch with configurable battery thresholds. The user accepts deliberate battery operation between thresholds and the uncertain net lifespan benefit identified in the [battery-aging research](../research/battery-aging-76-80-vs-100.md). Selected plugged-in preferences will be preserved during managed charging pauses because repeated power transitions should not disrupt the user's working settings.

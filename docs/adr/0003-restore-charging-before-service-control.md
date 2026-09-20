# Restore charging before service control

On ESP32 startup or reset, the device permits charging until the Windows service reconnects and sends explicit commands. The user chose this behavior so a controller restart does not leave the laptop waiting for Windows to restore charger power; the service must reconcile the recovered device state before reasserting its charging policy. This requires a valid USB-C power sequence and does not authorize applying an old negotiated voltage to a newly attached laptop.

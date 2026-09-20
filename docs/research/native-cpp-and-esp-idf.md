# Native C++ and ESP-IDF options

Researched 2026-09-20. These are implementation proposals, not accepted decisions. No prototype or hardware test has established unattended Bluetooth access yet.

## Windows without a third-party GUI framework

A native C++ tray application and small settings dialog can use the Windows SDK alone. `Shell_NotifyIconW` adds, changes, and removes notification-area icons. `TrackPopupMenu` displays a context menu. `DialogBoxParamW` creates a dialog from a resource template. None of these APIs requires .NET, WinForms, Qt, or another GUI framework. The build still needs a C++ compiler and Windows SDK, with a deliberate C++ runtime deployment choice. [Shell_NotifyIconW](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shell_notifyiconw), [TrackPopupMenu](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-trackpopupmenu), [DialogBoxParamW](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-dialogboxparamw)

Proposed structure: two C++ executables, a Windows service that owns control and a user-session tray application that owns menus, configuration, and interactive setup. Microsoft documents services in Session 0 and recommends a separate GUI connected through IPC such as named pipes. The service should restrict its pipe to authorized local callers through an explicit access policy. A default named-pipe security descriptor grants broader read access than this application needs. [Interactive services](https://learn.microsoft.com/en-us/windows/win32/services/interactive-services), [Named-pipe security](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights)

This is a good fit for the small UI described. The tradeoff is more direct responsibility for message handling, resource lifetimes, layout, and error handling. That assessment is engineering judgment.

## Native Bluetooth choices and the remaining test

Windows provides native BLE client APIs in `bluetoothleapis.h`. `BluetoothGATTGetServices` enumerates primary services, `BluetoothGATTSetCharacteristicValue` writes a characteristic, and `BluetoothGATTRegisterEvent` registers for characteristic changes. These APIs use Windows device/service handles and the Windows Bluetooth library. The write API exposes encryption and authentication requirements. Their existence establishes a native route, but does not prove this application's service identity can use the actual laptop's adapter before login. [Service enumeration](https://learn.microsoft.com/en-us/windows/win32/api/bluetoothleapis/nf-bluetoothleapis-bluetoothgattgetservices), [Characteristic writes](https://learn.microsoft.com/en-us/windows/win32/api/bluetoothleapis/nf-bluetoothleapis-bluetoothgattsetcharacteristicvalue), [Notifications](https://learn.microsoft.com/en-us/windows/win32/api/bluetoothleapis/nf-bluetoothleapis-bluetoothgattregisterevent)

C++/WinRT is another native option for Windows Runtime BLE APIs. It is a standard C++ projection provided as headers in the Windows SDK, so it does not imply using .NET or a third-party UI framework. Microsoft's GATT guide warns that `BluetoothLEDevice.FromIdAsync` may request consent and must run on a UI thread. Pairing and any consent UI belong in interactive setup. Do not infer that switching languages removes service/session restrictions. [C++/WinRT](https://learn.microsoft.com/en-us/windows/apps/develop/cpp-winrt/intro-to-using-cpp-with-winrt), [GATT client](https://learn.microsoft.com/en-us/windows/apps/develop/devices-sensors/gatt-client)

Proposed first technical test: provision the ESP32 interactively, then verify discovery, authenticated writes, acknowledgements, and notifications from the intended Windows service account. Repeat after reboot before login, logout, sleep/resume, ESP32 restart, and Bluetooth off/on. Confirm failure and reconnect timing. Keep the service API choice provisional until this test passes. A Bluetooth broker in the logged-in session would change the before-login requirement and cannot silently replace it.

## Arduino alongside ESP-IDF

Espressif officially supports Arduino as an ESP-IDF component. Its current component guide names Arduino-ESP32 3.3.12 with ESP-IDF 5.5 as a compatible pair and requires a 1000 Hz FreeRTOS tick rate. Use a pinned, documented version pair when implementing. Do not independently select the newest version of each framework. [Arduino as an ESP-IDF component](https://docs.espressif.com/projects/arduino-esp32/en/latest/esp-idf_component.html)

The proposal is an ESP-IDF project with C++ application code, adding Arduino as a component if its APIs or an Arduino library are useful. ESP-IDF already supports C++, so using C++ on both sides does not require Arduino. [ESP-IDF C++ support](https://docs.espressif.com/projects/esp-idf/en/v5.5/esp32/api-guides/cplusplus.html)

Espressif supports NimBLE for BLE-only use and Bluedroid for both Classic Bluetooth and BLE on the original ESP32. NimBLE has a smaller code and memory footprint. Use one BLE host stack to own Bluetooth. Arduino integration does not establish compatibility between every Arduino BLE library and the selected ESP-IDF/NimBLE versions. Check the exact dependency and configuration before choosing an Arduino BLE wrapper. Native ESP-IDF NimBLE remains a candidate. [ESP-IDF Bluetooth stacks](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/bluetooth/index.html)

## Shared C++ protocol definitions

Recommended design: both builds consume a small portable protocol module with shared headers and, if useful, shared source files. Share message identifiers, protocol versions, fixed-width integer types, limits, explicitly sized enums, and encode/decode functions. Keep Windows, Arduino, ESP-IDF, and BLE handles out of this module.

Define the bytes transmitted explicitly, including field widths, byte order, message version, message length, and validation rules. Reject unsupported versions, malformed lengths, out-of-range values, and unknown commands according to documented rules. Give both builds the same known byte sequences as serialization test fixtures.

Do not send an in-memory struct using `sizeof` or treat `#pragma pack` as a complete protocol definition. Compilers can add padding for alignment; packing does not specify byte order or validate incoming data. Microsoft documents how alignment affects structure layout and size. The explicit serialization recommendation is a design inference from these portability constraints. [C++ alignment and structure layout](https://learn.microsoft.com/en-us/cpp/cpp/align-cpp?view=msvc-170)

Shared headers eliminate duplicate definitions. They do not eliminate the need for protocol versioning when the installed Windows service and ESP32 firmware are updated at different times.

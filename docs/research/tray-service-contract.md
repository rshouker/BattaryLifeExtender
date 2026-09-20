# Tray/service contract options

Researched 2026-09-20. The comparison below led to accepted D4a: shared C++ types and explicit binary encoding over the existing named pipe. MIDL remains a researched alternative, not the selected implementation.

Recommend shared C++ request/response definitions and shared encode/decode functions over the agreed restricted local named pipe. Keep this module separate from the service/ESP32 BLE contract because UI configuration/status and device commands have different responsibilities. Small common domain types can be shared where their meaning matches.

Sending a struct's raw memory is possible under an explicitly controlled ABI, but a shared header alone does not establish it. Padding, alignment and architecture-dependent types affect representation; pointers and C++ container internals are process-local. Define field widths, byte order, framing, protocol version, message kind and size limits explicitly. Validate every decoded request and reject incompatible versions. Sharing a header does not mean both running programs were built from its latest version. Source: [Microsoft C++ alignment](https://learn.microsoft.com/en-us/cpp/cpp/align-cpp?view=msvc-170).

Microsoft MIDL is a valid alternative supplied with Microsoft's development tools. An IDL describes the RPC operations and types, and the compiler generates a header and client/server stubs for marshaling. It reduces handwritten serialization while adding generated build outputs and RPC binding/registration. Application validation and caller authorization remain necessary. Sources: [Generated RPC files](https://learn.microsoft.com/en-us/windows/win32/midl/files-generated-for-an-rpc-interface), [RPC stub memory management](https://learn.microsoft.com/en-us/windows/win32/rpc/server-stub-memory-management).

RPC over named pipes uses ncacn_np. Microsoft's preferred local RPC protocol is ncalrpc, which would revise the agreed named-pipe transport. Neither should be confused with ordinary named pipes carrying application-defined messages. Source: [Choosing an RPC protocol sequence](https://learn.microsoft.com/en-us/windows/win32/rpc/choosing-a-protocol-sequence).

For ordinary named pipes, use explicit caller access rules and reject remote clients. Message boundaries do not remove the need to bound and correctly assemble reads. Sources: [CreateNamedPipe](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createnamedpipea), [Named-pipe modes](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-type-read-and-wait-modes).

For this small personal application, shared types and an explicit codec are the recommended tradeoff. MIDL remains an option if generated RPC interfaces are a stronger preference than retaining ordinary named-pipe messaging. No interface or codec has been implemented.

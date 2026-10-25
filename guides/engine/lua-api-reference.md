# Lua API Reference

Status: current public API reference for native ProcessEngine runnables.

Function names are case-sensitive. Bare functions use the command's default process session.

## Process Discovery

| Function | Result |
| --- | --- |
| `findProcess(name)` | First matching process ID, or `nil` |
| `findProcesses(name)` | Lua array of matching process IDs |
| `getProcessIDFromProcessName(name)` | Matching process ID |
| `findWindow(className, title)` | Top-level window handle, or `nil` |
| `getWindowProcessID(handle)` | Window owner process ID |
| `findWindowProcessID(className, title)` | Matching window owner process ID |
| `getDefaultPid()` | Default command process ID |
| `getOpenedProcessID()` | Opened/default process ID |

Process-name matching is case-insensitive on Windows. A name such as `game` may match `game.exe`.

## Process Handles

```lua
local pid = findProcess("game.exe")
local game = openProcess(pid)
local value = game:readInteger("game.exe+0x1234")
game:writeInteger("game.exe+0x1234", value + 1)
```

`openProcess()` uses the default process; `openProcess(pid)` selects the specified process. Handle
methods include `getAddress`, `getAddressSafe`, `readBytes`, `writeBytes`, `readSmallInteger`,
`writeSmallInteger`, `readWord`, `writeWord`, `readInteger`, `writeInteger`, `readQword`,
`writeQword`, `readPointer`, `readFloat`, `writeFloat`, `readDouble`, `writeDouble`,
`readMem`, `writeMem`, `registerSymbol`, and `unregisterSymbol`.

## Address and Symbol Functions

| Function | Purpose |
| --- | --- |
| `getAddress(expression)` | Resolve an address or fail |
| `getAddressSafe(expression)` | Resolve an address with safe failure |
| `registerSymbol(name, address, donotsave?)` | Publish a package-owned symbol |
| `unregisterSymbol(name)` | Remove a package-owned symbol |
| `readPointer(address)` | Read a pointer value |

Expressions may be integer addresses, module expressions such as `game.exe+0x1234`, or pointer
chains such as `[[game.exe+0x1234]+0x18]+0x30`. Bare numbers are hexadecimal as in Cheat Engine
(`[[pStaticBase]+F8]+10`). `getAddress` raises a catchable Lua error when an expression cannot be
resolved; `getAddressSafe` returns `nil`.

## Typed Memory I/O

| Read | Write |
| --- | --- |
| `readBytes(address, count?, returnAsTable?)` | `writeBytes(address, ...)` -> bytes written |
| `readByte(address, signed?)` | `writeByte(address, value)` |
| `readSmallInteger(address, signed?)` | `writeSmallInteger(address, value)` |
| `readWord(address, signed?)` | `writeWord(address, value)` |
| `readInteger(address, signed?)` | `writeInteger(address, value)` |
| `readQword(address)` | `writeQword(address, value)` |
| `readFloat(address)` | `writeFloat(address, value)` |
| `readDouble(address)` | `writeDouble(address, value)` |
| `readString(address, maxLength?, wideChar?)` | `writeString(address, text, terminate?, wideChar?)` |

Use the operation whose width and representation match the target value.

The global memory functions follow Cheat Engine: a failed read (unresolvable expression, unset
pointer level, unmapped memory) returns `nil`, a failed write returns `false` and `writeBytes`
returns `0`. Integer writes truncate a fractional value toward zero. Process-handle methods
(`game:readInteger(...)`) are engine API and still raise a Lua error on failure.

## Cheat Engine Compatibility

Converted CE `{$lua}` scripts run unchanged in a package environment. Besides the memory semantics
above, the runtime provides CE's `createTimer(owner, enabled)` / `createTimer(ms, fn, ...)` forms,
timer `get`/`set` methods with case-insensitive members, `bAnd`/`bOr`/`bXor`/`bNot`/`bShl`/`bShr`,
the byte-table conversions, `getTickCount`, `print` and `showMessage` (engine log), `enumModules`,
`getModuleSize`, `getModuleBase` and `speedhack_setSpeed`/`speedhack_getSpeed` (engine game speed).
Deliberate differences: `writeString`'s third argument is "terminate" (CE: wide), a lone boolean in
`createTimer(false)` is `Enabled` (CE: owner), and an integer write of a non-number writes nothing.
The public page "Cheat Engine Lua Compatibility" in the engine wiki has the full table.

Implementation: every Lua-callable binding is pushed through `pushGuardedClosure`
(`ProcessEngine/src/lua/lua_runtime_impl.hpp`), which converts C++ exceptions into Lua errors so
`pcall` can catch them; Lua is built as C and a C++ exception would otherwise unwind through
`lua_pcall`. CE memory functions live in `lua_runtime_memory.cpp`, timers in
`lua_runtime_timers.cpp` and CE helpers in `lua_runtime_ce_helpers.cpp`; tests in
`tests/lua_ce_compat_test.cpp`.

## Scanning and Assembly

| Function | Purpose |
| --- | --- |
| `AOBScan(pattern)` | Scan the process |
| `AOBScanUnique(pattern)` | Require one matching address |
| `AOBScanModule(module, pattern)` | Scan one module |
| `AOBScanModuleUnique(module, pattern)` | Require one match in one module |
| `autoAssemble(text, targetself?, disableInfo?)` | Execute Auto Assembler text |
| `fullAccess(address, size)` | Change access for a range |
| `targetIs64Bit()` | Check target architecture |

Use `??` for a wildcard byte in an AOB pattern. Check scan results before reading or writing them.

## Runtime Utilities

| Function | Purpose |
| --- | --- |
| `createTimer(...)` | Create a timer; callback errors are logged and the timer keeps running (CE) |
| `sleep(milliseconds)` | Pause the current script |
| `process` | Attached process main module name |
| `publishEvent(event, value)` | Publish service state to a declared feed |

Timer callbacks are serialized with script execution. Timers and published service state belong to
the package and must stop or become inactive during `disable`.

## Return Values and Errors

Runnable functions may return JSON-compatible values: `nil`, booleans, numbers, strings, and tables
containing compatible values. Raise an error when compatibility checks fail. Do not continue with a
write after an unresolved address, missing scan result, stale attachment, or invalid argument.
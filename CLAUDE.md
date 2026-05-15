# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**xiaozhi-esp32** is a voice interaction AI chatbot firmware for ESP32 chips (C3, S3, P4). It uses streaming ASR + LLM + TTS architecture with OPUS audio codec, supports WebSocket and MQTT+UDP communication protocols, and provides device control via the MCP (Model Context Protocol).

**Version**: v2.2.6 (incompatible with v1 partition table)
**ESP-IDF SDK**: >= 5.5.2
**Build System**: CMake (ESP-IDF)
**Language**: C++

## Essential Commands

### Building & Flashing

```bash
# Set target chip (first time or when switching targets)
idf.py set-target esp32s3     # or esp32c3, esp32p4, etc.

# Select board via menuconfig
idf.py menuconfig             # Navigate to Xiaozhi Assistant -> Board Type

# Build
idf.py build

# Flash and monitor
idf.py flash monitor
```

### Recommended: Using release.py

If the board directory contains a `config.json`:

```bash
python scripts/release.py [board-directory]
```

This auto-sets the target, applies sdkconfig overrides, and packages the firmware.

### Code Formatting

```bash
# Format all C++ files
find main -iname '*.h' -o -iname '*.cc' | xargs clang-format -i

# Format a single file
clang-format -i path/to/file.cc
```

Code follows Google C++ style with 4-space indent, 100-char line limit.

## High-Level Architecture

### Core Components

| Component | Directory | Description |
|-----------|-----------|-------------|
| Application | `main/application.cc` | Main event loop, state management, singleton entry point |
| Board | `main/boards/` | Hardware abstraction layer. 70+ supported boards |
| Audio | `main/audio/` | Audio service, codecs, OPUS encoding/decoding, wake word |
| Display | `main/display/` | LCD/OLED display drivers, LVGL integration, emoji support |
| Protocol | `main/protocols/` | WebSocket and MQTT+UDP communication with server |
| MCP Server | `main/mcp_server.cc` | Device-side MCP for IoT control (speaker, LED, servo, GPIO) |
| LED | `main/led/` | Single LED, circular strip, GPIO LED drivers |
| Assets | `main/assets/` | Firmware-embedded assets (sounds, fonts, etc.) |

### Entry Point & Event Loop

Entry is `app_main()` in `main/main.cc`:
1. Initializes NVS flash
2. Calls `Application::GetInstance().Initialize()` then `Run()`

`Application` (singleton) is the central orchestrator:
- Uses a **FreeRTOS EventGroup** for event-based communication between tasks
- Manages a **DeviceStateMachine** with strict transition rules (idle -> connecting -> speaking -> listening -> etc.)
- Receives events from audio service (wake word, VAD, audio ready) and network events
- Protocol selection (WebSocket vs MQTT) happens in `InitializeProtocol()` based on Kconfig setting

### Board System

All boards inherit from `Board` (base) via one of:
- `WifiBoard` - WiFi connectivity
- `Ml307Board` / `Nt26Board` - 4G modem
- `DualNetworkBoard` - WiFi/4G switchable
- `RndisBoard` - RNDIS-over-USB

Each board implements virtual overrides for: `GetAudioCodec()`, `GetDisplay()`, `GetBacklight()`, `GetNetwork()`, etc.

Board registration uses `DECLARE_BOARD(ClassName)` macro in `main/boards/`, selected via Kconfig `BOARD_TYPE`.

### Audio Pipeline

Two main data flows:
1. **MIC** -> [Processors] -> {Encode Queue} -> [Opus Encoder] -> {Send Queue} -> (Server)
2. **(Server)** -> {Decode Queue} -> [Opus Decoder] -> {Playback Queue} -> (Speaker)

Key classes:
- `AudioService` - orchestrates encoding/decoding via FreeRTOS tasks
- `AudioCodec` subclasses - hardware codec drivers (ES8311, ES8374, ES8388, etc.)
- `WakeWord` - ESP-SR wake word detection

### Communication Protocol

Base `Protocol` class with two implementations:
- `WebsocketProtocol` - WebSocket connection to xiaozhi.me server
- `MqttProtocol` - MQTT for control + UDP for audio streaming

Both handle: audio streaming, JSON messages (ASR results, TTS commands), MCP tool calls, session management.

### MCP (Model Context Protocol)

`McpServer` (singleton) provides device-side IoT control:
- Tools registered via `AddTool()` / `AddCommonTools()` / `AddUserOnlyTools()`
- Each tool has: name, description, typed properties, callback
- Tools include: speaker control, LED, brightness, battery status, GPIO
- `user_only` tools are invisible to AI, only callable by the user

## Key Files to Know

| File | Purpose |
|------|---------|
| `main/main.cc` | Entry point (app_main) |
| `main/application.h` | Application interface - understand the event bits and public API |
| `main/protocols/protocol.h` | Protocol base class - audio packet format, listening modes |
| `main/mcp_server.h` | MCP tool registration system |
| `main/boards/common/board.h` | Board base class interface |
| `main/CMakeLists.txt` | Build configuration, board selection logic |
| `main/idf_component.yml` | ESP-IDF component dependencies |
| `main/Kconfig.projbuild` | Build-time configuration options |
| `sdkconfig.defaults*` | Per-chip SDK defaults |

## Adding a New Board

1. Create directory under `main/boards/[vendor]-[model]/`
2. Create `config.h` (pin assignments) and `config.json` (build config)
3. Implement board class deriving from `WifiBoard` or `Ml307Board`
4. Register with `DECLARE_BOARD(ClassName)`
5. Add Kconfig entry in `main/Kconfig.projbuild`
6. Add `elseif` branch in `main/CMakeLists.txt` for board selection

See `docs/custom-board.md` for the full tutorial.

## Development Notes

- Linux recommended over Windows (faster compilation, fewer driver issues)
- Project uses `clang-format` based on Google style - run before committing
- ESP-IDF plugin for VSCode/Cursor, SDK version 5.4+
- Firmware connects to official `xiaozhi.me` server by default
- `main/` is the IDF project root; all source lives under it
- No formal test framework - testing is done by flashing to hardware

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a decompiled and reconstructed Tibia 7.7 game server from the leaked binary. The project is a C++ MMORPG server implementation that handles client connections, game logic, player management, and world persistence. The code is Linux-specific and uses threading for connection management.

## Build Commands

The project uses a simple Makefile build system:

```bash
make                        # build in release mode
make DEBUG=1                # build in debug mode with assertions
make clean                  # remove build directory
make clean && make          # full rebuild (recommended)
make clean && make DEBUG=1  # full debug rebuild (recommended)
```

**Requirements:**
- Linux only (uses Linux-specific features)
- OpenSSL libcrypto (`apt install libssl-dev` on Debian)
- g++ with C++11 support

## Architecture

### Core Components

- **main.cc**: Server entry point, signal handling, daemon mode setup
- **communication.cc/hh**: Network protocol handling and packet processing
- **connections.cc/hh**: Client connection management with per-connection threading
- **query.cc/hh**: Database communication with query manager service
- **map.cc/hh**: World map management and tile operations
- **objects.cc/hh**: Game object definitions and item management
- **cr*.cc**: Creature-related modules (crmain, crplayer, crnonpl, crcombat, etc.)
- **config.cc/hh**: Configuration file parsing and server settings
- **crypto.cc/hh**: RSA encryption using OpenSSL
- **threads.cc/hh**: Threading utilities and thread management

### Threading Model

The server uses a unique threading model where:
- Main thread handles core game logic and world updates
- Each client connection spawns a dedicated communication thread (64KB stack)
- Communication threads handle I/O and authentication via query manager
- This design may impact performance under high load due to context switching

### Dependencies

**Hard Dependency:**
- Query Manager service (server won't boot without it)

**Optional Services:**
- Login Server (character list management)
- Web Server (account management)

### Key Files and Directories

When setting up the server, these directories must exist:
- `usr/00` through `usr/99` (player data storage - must be created manually)
- `dat/` (game data files)
- `map/` (world map data - persistent)
- `origmap/` (clean map backup)
- `log/` (server logs)
- `save/` (save state including game.pid lockfile)

## Important Notes

- The server requires specific file permissions (especially `tibia.pem` should be 0600)
- The `save/game.pid` file prevents multiple instances and may need manual removal
- Map data in `map/` is persistent and should be backed up
- The server includes a 5-minute shutdown warning system before termination
- Debug builds enable assertions via `ENABLE_ASSERTIONS=1`

## Development Patterns

- Uses custom integer typedefs (uint8, uint16, uint32, int64, uint64)
- Extensive use of assertions (`ASSERT` macro, enabled in debug builds)
- Header-implementation split for most modules
- Linux-specific system calls and threading primitives
- Exception handling is noted as problematic in the original codebase
- String handling is noted as needing improvement
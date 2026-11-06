# RPS

RPS is a cross-platform audio infrastructure system built on a **multi-process architecture** for unconditional stability. All third-party plugin code runs in disposable worker processes — if a plugin crashes, only that worker dies.

RPS has two major domains:

1. **Plugin Scanning** — A robust, parallel scanning system that discovers and catalogs audio plugins across all major formats (VST2, VST3, CLAP, AAX, AU, LV2, LADSPA) on Windows, macOS, and Linux. Results are stored in a central SQLite database.
2. **Audio Engine & Plugin Hosting** — A graph-based, out-of-process audio engine with single- and multi-plugin hosting, native GUI embedding (SDL3 + ImGui), parallel wavefront execution, zero-allocation real-time audio path, and CPU topology-aware scheduling. Currently supports **VST3** and **CLAP** plugins.

RPS exposes a **gRPC API** so it can be driven from any language. Example clients in C++, Python and Java are included.

## Why RPS?

Audio plugins are dynamically loaded libraries that execute arbitrary third-party code. They crash during initialization, hang on iLok license checks, write to stdout, corrupt memory, and deadlock during cleanup. When a DAW or host application loads these plugins directly in its main process, a single bad plugin can crash the entire application.

RPS solves this with a **multi-process architecture** where all plugin code runs in isolated worker processes:

### Scanning
- **`rps-server`**: A gRPC server daemon that coordinates both scanning and the audio engine. Manages worker process pools, handles watchdogs/timeouts, and streams progress events to clients.
- **`rps-standalone`**: A standalone CLI wrapper around the scan engine (no server needed).
- **`rps-pluginscanner`**: The disposable scanner worker. Isolates unsafe plugin loading in its own process. Communicates via Protobuf over Boost.Interprocess message queues.

### Audio Engine & Plugin Hosting
- **`rps-pluginhost`**: An isolated plugin host worker process. Operates in two modes:
  - *Single-plugin GUI mode* — Opens a plugin's native editor in an SDL3 window with an ImGui preset sidebar.
  - *Graph mode* — Hosts multiple plugins in an arbitrary audio processing graph with wavefront-parallel execution.
- **`rps-coordinator`** (`libs/rps-coordinator/`): The graph model, parallel wavefront executor, and graph worker runtime. The core of the audio engine.
- **`rps-audio`** (`libs/rps-audio/`): Shared memory SPSC lock-free ring buffer for low-latency inter-process audio transport.

### Example Clients
- **`examples/`**: Python TUI, C++ CLI, and Java clients demonstrating the full gRPC API.

## Project Goals
1. **Robustness**: Unconditional crash and stall isolation through multi-process architecture.
2. **Performance**: Lock-free shared memory audio transport, parallel wavefront graph execution, CPU topology-aware scheduling.
3. **Accessibility**: Language-agnostic gRPC API. Central SQLite database queryable by external tools.
4. **Language**: Strictly Modern C++23 (core engine). Python for example clients.
5. **Dependencies**: STL, Boost, SQLite, gRPC/Protobuf, spdlog, SDL3, Dear ImGui, and Plugin SDKs.

## Building

### Prerequisites

| Dependency   | Minimum Version | Notes                                              |
|--------------|-----------------|----------------------------------------------------|
| CMake        | 3.22            | Build system (4.2+ for the Visual Studio 2026 generator) |
| C++ Compiler | C++23 capable   | Requires a C++23-capable standard library too; Windows uses MSVC or clang-cl |
| Ninja        | 1.11+           | Build backend (Recommended)                        |
| vcpkg        | Latest          | C/C++ package manager (Used on ALL platforms)      |
| Git          | 2.x             | For version management                             |

The CMake version range `3.22...3.25` accepts CMake 3.22 and enables policy behavior through 3.25 when available. macOS requires CMake 3.22.4 or newer for AppleClang C++23 support. A C++23-capable compiler and standard library are still required.

Third-party dependencies (`boost`, `grpc`, `protobuf`, `spdlog`, `sqlite3`, `sdl3`, and `imgui`) are managed by `vcpkg` during configuration. Linux additionally needs the system development packages listed below. Clang warning suppressions for VST3 headers are checked for compiler support in both CMake options and source pragmas, including when using Clang 18.

### Step 1: Clone the Repository

```bash
git clone https://github.com/tphzz/rps.git
cd rps
git submodule update --init --recursive
```

The submodule init will download the plugin SDK headers (CLAP, VST3).

### Step 2: Configure and Build

RPS uses static `vcpkg` libraries across Windows, macOS, and Linux. Windows builds also use the static MSVC runtime. Operating-system DLLs, macOS frameworks, and Linux system libraries remain runtime requirements; static third-party linking does not produce an executable independent of its operating system.

1. **Install vcpkg** if you haven't already:
   The manifest pins its registry baseline and dependency versions. Use a full vcpkg checkout containing that baseline. Checking out the same baseline is a reproducible local setup; a newer checkout containing it also supports manifest version resolution.
   ```bash
   git clone https://github.com/microsoft/vcpkg.git /path/to/vcpkg
   cd /path/to/vcpkg
   # Checkout the specific baseline commit required by rps/vcpkg.json
   git checkout 62159a45e18f3a9ac0548628dcaf74fcb60c6ff9
   
   # Windows:
   bootstrap-vcpkg.bat
   # Linux/macOS:
   ./bootstrap-vcpkg.sh
   ```

2. **Install Ninja** (Windows):
   The Windows builds are configured to use the Ninja generator for significantly faster and more consistent compilation.
   ```cmd
   winget install Ninja-build.Ninja
   ```

#### Windows x64 (MSVC)

Run in an x64 Developer PowerShell for Visual Studio. The local validation toolchain is Visual Studio 2026 with CMake 4.3.3; the Visual Studio 2026 generator itself requires CMake 4.2 or later. The commands below use Ninja.

```cmd
# Configure CMake to use vcpkg and static linking
# * if using the MSBuild generator (e.g. -G "Visual Studio 18 2026"), CMake 4.2+ is required
# * adapt the paths

cmake -S . -B build-vc -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=cl -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static -DRPS_MSVC_STATIC_RUNTIME=ON -DRPS_ENABLE_VST2=OFF -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      
# Build the project
cmake --build build-vc --config Release
```

#### Windows x64 (clang-cl / MSVC ABI)

This uses a custom triplet to force the Clang MSVC ABI for compilation with LLVM tools (from the llvm.org website). The triplet is based on the x64-windows-static triplet but with the Clang MSVC ABI. MSVC is still required to be installed for some dependencies. Windows SDK must be discoverable through the LIB environment variable.

Run in an x64 Developer PowerShell for Visual Studio. The local validation toolchain uses LLVM 22.1.0 and Visual Studio 2026. The custom triplet is x64-only; it must not be reused for ARM64 builds.

```cmd
# * adapt the paths

cmake -S . -B build-clang -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static-clang -DVCPKG_OVERLAY_TRIPLETS=triplets -DRPS_ENABLE_VST2=OFF -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

cmake --build build-clang --config Release
```

#### macOS (Apple Silicon and Intel)

Use the triplet and architecture matching the intended binary. Objective-C++ is enabled only on macOS, with the same C++23 requirement as C++ sources.

```bash
# Enable VST2 with custom SDK path if needed: -DRPS_ENABLE_VST2=ON -DRPS_VST2_SDK_PATH=/path/to/vstsdk2.4
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES=arm64 -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake -DVCPKG_TARGET_TRIPLET=arm64-osx-release
cmake --build build --config Release

# Intel build, in a separate directory:
cmake -G Ninja -B build_x86_64 -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES=x86_64 -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-osx-release
cmake --build build_x86_64 --config Release
```

#### Linux (Ubuntu/Debian/Fedora)

*Note: The default `x64-linux` triplet in vcpkg automatically builds static libraries.*

**Prerequisites:**
```bash
# Ubuntu/Debian:
sudo apt install build-essential cmake ninja-build pkg-config curl zip unzip tar autoconf autoconf-archive automake libtool libltdl-dev
# SDL3 default X11/Wayland/IBus features, ALSA audio, and native plugin embedding:
sudo apt install libx11-dev libxft-dev libxext-dev libwayland-dev wayland-protocols libxkbcommon-dev libegl1-mesa-dev libibus-1.0-dev libasound2-dev
# For Clang:
sudo apt install clang

# Fedora:
sudo dnf install gcc-c++ cmake ninja-build pkgconf-pkg-config curl zip unzip tar autoconf autoconf-archive automake libtool libtool-ltdl-devel
sudo dnf install libX11-devel libXft-devel libXext-devel wayland-devel wayland-protocols-devel libxkbcommon-devel mesa-libEGL-devel ibus-devel alsa-lib-devel
# For Clang:
sudo dnf install clang
```

**Build with GCC (default):**
```bash
# Enable VST2 with custom SDK path if needed: -DRPS_ENABLE_VST2=ON -DRPS_VST2_SDK_PATH=/path/to/vstsdk2.4
cmake -G Ninja -B build-gcc -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-linux
cmake --build build-gcc --config Release
```

**Build with Clang:**
```bash
cmake -G Ninja -B build-clang -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-linux
cmake --build build-clang --config Release
```

#### Vcpkg Triplet Reference

| OS          | Architecture  | Compiler            | Recommended `VCPKG_TARGET_TRIPLET` | CMake Flags Required                                        |
|-------------|---------------|---------------------|------------------------------------|-------------------------------------------------------------|
| **Windows** | x64           | MSVC                | `x64-windows-static`               | `-DRPS_MSVC_STATIC_RUNTIME=ON`                              |
| **Windows** | x64           | Clang/MSVC ABI      | `x64-windows-static-clang`         | `-DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl` |
| **Windows** | arm64         | MSVC                | `arm64-windows-static`             |                                                             |
| **macOS**   | arm64 (M1/M2) | Apple Clang         | `arm64-osx-release`                |                                                             |
| **macOS**   | x64 (Intel)   | Apple Clang         | `x64-osx-release`                  |                                                             |
| **Linux**   | x64           | GCC / Clang         | `x64-linux`                        |                                                             |
| **Linux**   | arm64         | GCC / Clang         | `arm64-linux`                      |                                                             |

The custom Windows Clang triplet also requires `-DVCPKG_OVERLAY_TRIPLETS=triplets`. For Windows ARM64, use a matching ARM64 Visual Studio developer environment. Linux ARM64 uses `arm64-linux` in a separate build directory. These target settings describe configuration; they do not imply that every platform has been validated with the same compiler versions.

### Tests and optional VST2 scanning

Coordinator tests are built by default and registered with CTest:

```bash
ctest --test-dir build-vc -C Release --output-on-failure --no-tests=error
ctest --test-dir build-clang -C Release --output-on-failure --no-tests=error
```

Parallel processing waits for every worker to finish the current wave before reusing state or buffers for the next wave or block. Coordinator tests cover repeated blocks, consecutive parallel waves, and idle workers; the test runner prints each test name before execution to identify a timeout.

Use your selected build directory on other platforms. Set `-DRPS_BUILD_COORDINATOR_TESTS=OFF` to omit the test executable and its registration. `CMAKE_EXPORT_COMPILE_COMMANDS=ON` provides a compilation database for checks such as `clang-tidy -p build-clang "--checks=clang-analyzer-*" apps/rps-server/src/main.cpp`. Select actual checks explicitly if your local clang-tidy configuration enables only compiler diagnostics.

VST2 is disabled by default. To enable it locally, keep the proprietary SDK outside the source checkout and point an existing build at it:

```bash
cmake -S . -B build-vc -DRPS_ENABLE_VST2=ON "-DRPS_VST2_SDK_PATH=C:/SDKs/vstsdk2.4"
cmake --build build-vc --config Release --target rps-pluginscanner
```

Use the corresponding external SDK path on macOS/Linux. The SDK is never part of the public source distribution.

---

### Build Output

After a successful build, you will find the following binaries in the `build/` directory:
- `build/apps/rps-standalone/rps-standalone` (or `.exe`) — standalone CLI
- `build/apps/rps-pluginscanner/rps-pluginscanner` (or `.exe`) — scanner worker
- `build/apps/rps-server/rps-server` (or `.exe`) — gRPC server
- `build/apps/rps-pluginhost/rps-pluginhost` (or `.exe`) — unified VST3/CLAP GUI and graph host worker
- `build/apps/rps-vstscannermaster/vstscannermaster` (or `.exe`) — Steinberg-compatible VST3 cache generator
- `build/examples/cpp/rps-example-client` (or `.exe`) — C++ gRPC example client

Visual Studio adds an additional layer of subdirectories "Release" or "Debug".

Replace `build` with the directory selected above. Every server build refreshes the required `rps-pluginscanner` and `rps-pluginhost` copies beside the server, including incremental builds where the server does not relink. Copies to the Python/Java example directories are convenience copies; locked destination files produce warnings. Required copies beside the server fail the build if they cannot be updated.

## Usage

RPS can be used in five ways:
1. **Standalone CLI** (`rps-standalone`) — run scans directly from the command line.
2. **gRPC Server** (`rps-server`) — a long-lived daemon that accepts scan requests and plugin GUI hosting from any language client. See [gRPC Server](#grpc-server) and [Python TUI Client](#python-tui-client) below.
3. **Plugin GUI Hosting** (`rps-pluginhost`) — an isolated worker process that selects VST3 or CLAP and opens the plugin's native GUI in an SDL3 window with an ImGui preset browser sidebar. Driven via gRPC through `rps-server`.
4. **Audio Processing** — send audio through hosted plugins via shared memory ring buffers. Use `open-gui --audio` in the Python client to enable audio, then `send-audio <file.wav>` to process a WAV file. For real-time playback through an audio device, use `open-gui --audio --audio-device sdl3`, then `play-audio <file.wav>` or `play-audio-looped <file.wav>` (shared memory path). For network-transparent gRPC playback, use `play-audio-grpc <file.wav>` or `play-audio-looped-grpc <file.wav>`.
5. **VST3 Scanner Master** (`vstscannermaster`) — drop-in replacement for Steinberg's `vstscannermaster.exe` that produces Cubase/Nuendo/Dorico-compatible XML cache files. See [VST3 Scanner Master](#vst3-scanner-master) below.

### Standalone CLI

The standalone CLI will automatically spawn the worker scanner processes as needed.

#### Command Line Arguments

```text
RPS Standalone Options:
  -h [ --help ]                         Produce help message
  -d [ --scan-dir ] arg                 Directories to recursively scan for plugins
  -s [ --scan ] arg                     Single file to scan
  -b [ --scanner-bin ] arg              Path to the scanner binary (default: rps-pluginscanner.exe)
  -t [ --timeout ] arg (=120000)        Timeout in ms per plugin (default: 2 min)
  -j [ --jobs ] arg (=6)                Number of parallel scanner workers (default: 6)
  -f [ --formats ] arg (=all)           Comma-separated list of formats to scan (e.g. vst3,clap) or 'all'
  -m [ --mode ] arg (=incremental)      Scan mode: 'full' or 'incremental' (see below)
  -r [ --retries ] arg (=3)             Number of retries for failed plugins (0 = no retries)
     --filter arg                       Only scan plugins whose filename contains this string
  -l [ --limit ] arg (=0)               Maximum number of plugins to scan (0 = unlimited)
  -v [ --verbose ]                      Enable verbose scanner output (plugin debug logs)
     --db arg (=rps-plugins.db)         Path to the output SQLite database file
```

### Examples

**1. Scan the default OS plugin directories for all formats**
If you run `rps-standalone` without any path arguments, it will automatically search the standard VST3, CLAP, AU, and AAX folders for your specific OS. VST2 is excluded from `all` and must be explicitly requested (see below).
```bash
./rps-standalone
```

**2. Scan only VST3 and CLAP formats**
```bash
./rps-standalone --formats vst3,clap
```

**3. Scan VST2 plugins (requires VST2 build — see [Optional: VST2.4 Support](#optional-vst24-support))**
```bash
# VST2 must be explicitly requested — it is never included in --formats all
./rps-standalone --formats vst2
./rps-standalone --formats vst3,clap,vst2
```

**4. Scan a specific directory**
```bash
# Windows
rps-standalone.exe --scan-dir "C:\Program Files\Common Files\VST3"

# macOS / Linux
./rps-standalone --scan-dir "/Library/Audio/Plug-Ins/VST3"
```

**5. Filter plugins by name and limit the count (useful for debugging)**
```bash
rps-standalone.exe --formats vst3 --filter "FabFilter" --limit 10
```

**6. Scan multiple directories with a specific number of workers**
```bash
rps-standalone.exe --scan-dir "C:\Folder1" "D:\Folder2" --jobs 4
```

**7. Scan a single plugin with verbose debug output**
```bash
rps-standalone.exe --scan "C:\VstPlugins\Massive.dll" --verbose
```

**8. Scan with a longer timeout (for slow iLok-protected plugins)**
```bash
rps-standalone.exe --timeout 180000
```

**9. Force a full rescan (clear and rescan all plugins of the requested formats)**
```bash
./rps-standalone --mode full
./rps-standalone --formats vst2 --mode full
```

**10. Incremental scan (default -- skip unchanged plugins)**
```bash
# This is the default. Only new or modified plugins are scanned.
./rps-standalone
```

### Default Plugin Paths

If no paths are provided, RPS will automatically search the following directories based on your operating system (32bit plugins are not supported!):

**Windows:**
- `C:\Program Files\Common Files\VST3`
- `C:\Program Files\Common Files\CLAP`
- `C:\Program Files\Common Files\Avid\Audio\Plug-Ins` (AAX)
- `C:\Program Files\Steinberg\VstPlugins` (VST2)
- `C:\Program Files\VstPlugins` (VST2)

**macOS:**
- `/Library/Audio/Plug-Ins/VST3` (and `~/Library/...`)
- `/Library/Audio/Plug-Ins/CLAP` (and `~/Library/...`)
- `/Library/Audio/Plug-Ins/Components` (AU) (and `~/Library/...`)
- `/Library/Application Support/Avid/Audio/Plug-Ins` (AAX)
- `/Library/Audio/Plug-Ins/VST` (VST2) (and `~/Library/...`)

**Linux:**
- `/usr/lib/vst3`, `/usr/local/lib/vst3`, `~/.vst3`
- `/usr/lib/clap`, `/usr/local/lib/clap`, `~/.clap`
- `/usr/lib/lv2`, `/usr/local/lib/lv2`, `~/.lv2`
- `/usr/lib/vst`, `/usr/local/lib/vst`, `~/.vst`
- `/usr/lib/ladspa`, `/usr/lib64/ladspa`, `/usr/local/lib/ladspa`, `/usr/local/lib64/ladspa`, `~/.ladspa`

## Scan Modes

RPS supports two scan modes via `--mode`:

- **`incremental`** (default): Only scans plugins that are new or have changed since the last scan. Change detection uses file modification time (`mtime`) comparison. Plugins that no longer exist on disk are automatically pruned. Previously **skipped** plugins (e.g., empty bundles) and **blocked** plugins (exhausted all retries) are also remembered and not re-scanned unless their file changes. This makes repeated scans near-instant.

- **`full`**: Clears all database entries **for the requested format(s)** and rescans everything. A `--mode full --formats vst2` will only clear VST2 entries, leaving CLAP and VST3 results intact. This also clears the skipped and blocked lists for the requested formats.

## Database Schema

Scan results are stored in a SQLite database (default: `rps-plugins.db`) with WAL journal mode for performance.

| Table | Purpose |
|---|---|
| `plugins` | One row per successfully scanned plugin: `format`, `path`, `name`, `uid`, `vendor`, `version`, `description`, `url`, `category`, `num_inputs`, `num_outputs`, `status`, `error_message`, `scan_time_ms`, `file_mtime`, `file_hash`, `last_scanned` |
| `parameters` | Plugin parameters linked by `plugin_id`: `param_index`, `name`, `default_value` |
| `aax_plugins` | AAX-specific variant data (1:N with `plugins`): manufacturer/product/plugin IDs (FourCC + numeric), effect ID, plugin type, stem formats (input/output/sidechain) |
| `plugins_skipped` | Plugins that are not scannable (e.g., empty bundles, no loadable binary): `path`, `format`, `reason`, `file_mtime` |
| `plugins_blocked` | Plugins that exhausted all retries or timed out: `path`, `format`, `reason`, `file_mtime` |
| `vst3_classes` | VST3 multi-class entries (1:N with `plugins`): `class_index`, `name`, `uid`, `category`, `class_category`, `cardinality`, `class_flags`, `sub_categories`, `sdk_version`, `vendor`, `version` |
| `vst3_compat_uids` | VST3 compatibility UIDs (1:N with `vst3_classes`): `class_id`, `new_uid`, `old_uid` |

## VST3 Scanner Master

`vstscannermaster` is an experimental drop-in replacement for Steinberg's `vstscannermaster.exe`. It uses the rps crash-isolated scanning infrastructure to produce the same XML cache files that Cubase, Nuendo, and Dorico consume. It's currently not working properly in Cubase.

### Command Line Arguments

```text
vstscannermaster -prefPath <dir> [options]

  -prefPath <dir>       Cache output directory (required)
  -licenceLevel <N>     Licence level (accepted but ignored)
  -hostName <name>      Host name
  -progress             Show progress output
  -rescan               Force full rescan (default: incremental)
  -timeout <secs>       Per-plugin timeout in seconds (default: 120)
  -recheckPath <path>   Rescan a single plugin path
  --scanner-bin <path>  Path to the scanner binary
  -j [ --jobs ] <N>     Parallel scanner workers (default: 6)
  -v [ --verbose ]      Verbose output
```

### Example

```bash
vstscannermaster -prefPath ./VST3Cache -licenceLevel 25000 -progress -rescan
```

### Output Files

| File                   | Description                                                           |
|------------------------|-----------------------------------------------------------------------|
| `vst3plugins.xml`      | All successfully scanned VST3 plugins with full class metadata        |
| `vst3blocklist.xml`    | Plugins that crashed or timed out during scanning                     |
| `vst3allowlist.xml`    | User-managed allow list (created empty, preserved on subsequent runs) |
| `cacheVersion`         | Cache format version                                                  |
| `vstscannermaster.log` | Scan log with summary statistics                                      |
| `rps-cache.db`         | Internal SQLite database (used for incremental scanning)              |

## gRPC Server

`rps-server` exposes the scan engine as a gRPC service, making RPS usable from any language.

### Server CLI

```text
RPS Server Options:
  -h [ --help ]                    Produce help message
  -p [ --port ] arg (=50051)       gRPC listen port
     --db arg (=rps-plugins.db)    Path to the SQLite database file
  -b [ --scanner-bin ] arg         Path to the scanner binary
```

### gRPC API

Defined in `proto/rps.proto`:

| RPC              | Type             | Description                                                             |
|------------------|------------------|-------------------------------------------------------------------------|
| `StartScan`      | Server streaming | Start a scan, returns a stream of `ScanEvent` messages until completion |
| `StopScan`       | Unary            | Stop a running scan gracefully                                          |
| `GetStatus`      | Unary            | Query server state (idle/scanning), uptime, db path                     |
| `ListPlugins`    | Unary            | List all scanned plugins from the database                              |
| `OpenPluginGui`  | Server streaming | Open a plugin GUI, returns `PluginEvent` stream (params, presets, audio) |
| `ClosePluginGui` | Unary            | Close a running plugin GUI                                              |
| `GetPluginState` | Unary            | Save the complete state of a running plugin as a binary blob            |
| `SetPluginState` | Unary            | Restore a plugin's state from a previously saved blob                   |
| `LoadPreset`     | Unary            | Load a preset by ID on a running plugin GUI                             |
| `StreamAudio`    | Bidi streaming   | Stream audio blocks through a hosted plugin via gRPC (networked)        |
| `Shutdown`       | Unary            | Graceful server shutdown                                                |

`OpenPluginGui` accepts optional audio parameters (`enable_audio`, `sample_rate`, `block_size`, `num_channels`). When audio is enabled, the stream emits an `AudioReady` event containing the shared memory segment name for the ring buffer.

`StreamAudio` is a bidirectional streaming RPC for network-transparent audio processing. The client sends `AudioInputBlock` messages (with raw float32 audio data), and the server returns processed `AudioOutputBlock` messages. Requires an active GUI session with audio enabled. Adds gRPC network latency compared to the local shared memory path.

The server listens on **both TCP and Unix Domain Sockets** simultaneously for zero-configuration local IPC. When the Python client spawns the server locally (managed mode), it connects via UDS (`unix://{TEMP}/rps-server-{port}.sock`) for lower-overhead IPC. Remote connections (`--server host:port`) use TCP as before. UDS is supported cross-platform (Linux, macOS, Windows 10+).

Only one scan at a time is allowed. `StartScan` returns `ALREADY_EXISTS` if a scan is in progress.

### Starting the Server

```bash
# Start with defaults (port 50051, rps-plugins.db)
./rps-server

# Custom options
./rps-server --port 50051 --db my-plugins.db --log-level debug
```

The gRPC clients spawn the server automatically.

## Example Clients

Each example client has its own README.md in its directory.

### Python TUI Client — Audio Processing

The Python client supports sending audio through plugins via shared memory or gRPC:

```bash
cd examples/python

# Open a plugin with audio processing enabled (file-based: writes output WAV)
uv run python -m rps_client open-gui --format vst3 --audio

# Inside the interactive session:
#   send-audio test.wav          — process via shared memory (low latency)
#   send-audio-grpc test.wav     — process via gRPC stream (networked)
# Output is written to test_processed.wav

# Open a plugin with real-time audio device playback
uv run python -m rps_client open-gui --format vst3 --audio --audio-device sdl3 -bs 512

# Inside the interactive session:
#   play-audio test.wav          — real-time playback via shared memory
#   play-audio-looped test.wav   — looped playback via shared memory
#   play-audio-grpc test.wav     — real-time playback via gRPC
#   play-audio-looped-grpc test.wav — looped playback via gRPC
```

Audio options for `open-gui`:
| Option | Default | Description |
|---|---|---|
| `--audio` | off | Enable shared memory audio transport |
| `--audio-device` | none | Audio output device (`sdl3` for real-time playback) |
| `--sample-rate` / `-sr` | 48000 | Audio sample rate (match your WAV) |
| `--channels` / `-ch` | 2 | Channel count |
| `--block-size` / `-bs` | 128 | Processing block size in samples |

## Logging Configuration

All RPS processes use **environment variables** for logging control. No CLI flags are needed — set the variables before launching any RPS tool, and child processes inherit them automatically.

| Variable | Default | Description |
|---|---|---|
| `RPS_SERVER_LOGGING` | `false` | Enable file logging for `rps-server` |
| `RPS_SERVER_LOGLEVEL` | `info` | Log level: `trace`, `debug`, `info`, `warn`, `error` |
| `RPS_PLUGINSCANNER_LOGGING` | `false` | Enable file logging for scanner workers |
| `RPS_PLUGINSCANNER_LOGLEVEL` | `info` | Log level for scanner workers |
| `RPS_PLUGINHOST_LOGGING` | `false` | Enable file logging for plugin host workers |
| `RPS_PLUGINHOST_LOGLEVEL` | `info` | Log level for plugin host workers |
| `RPS_LOG_DIR` | *(auto)* | Directory for log files. Set automatically by `rps-server` to its CWD so child processes (scanner, pluginhost) write logs alongside the server. Override to redirect all logs elsewhere. |

### Log File Names

All log files are written to the directory specified by `RPS_LOG_DIR`. When launched via `rps-server`, this is automatically set to the server's working directory, so all logs appear alongside `rps-server.log`.

| Process | Log File |
|---|---|
| `rps-server` | `rps-server.log` |
| `rps-pluginscanner` | `rps-pluginscanner.worker_{N}.log` (one per worker) |
| `rps-pluginhost` | `rps-pluginhost.{format}.{plugin_name}.log` |

Log messages at `info` level and above are flushed immediately; `debug`/`trace` messages are flushed every 3 seconds.

### Example: Diagnosing scanner timeouts

```bash
# Windows (PowerShell / cmd)
set RPS_PLUGINSCANNER_LOGGING=true
set RPS_PLUGINSCANNER_LOGLEVEL=debug
rps-standalone --formats vst3

# After the scan, check the per-worker log files to see where a plugin hung
```

## Documentation

- **[Architecture](ARCHITECTURE.md)** — Full system design: process architecture, IPC mechanisms, graph engine, real-time audio path, CPU topology, and design rationale.
- **[Developer Guide](DEVELOPMENT.md)** — Implementation details, component walkthroughs, and onboarding guide for contributors.

## License

MIT License

## Binary distributions

Portable packages are available for Windows x64/ARM64, Linux x64/ARM64, and macOS
Intel/ARM64. Keep the six programs together, including `rps-pluginscanner` and the
unified `rps-pluginhost`; the applications discover their worker companions beside
themselves. Apple Silicon packages also include `rps-pluginscanner_x86_64` and
`rps-pluginhost_x86_64` for Intel plugins through Rosetta.

Windows installers support per-user installation with an optional user PATH task.
The combined installer selects the native architecture; architecture-specific
installers contain their named payload. Debug installations use a separate directory.
macOS packages are signed ZIPs, with notarization recorded when requested. Linux
packages are tar.gz archives containing required non-system C++ runtimes; native
system audio/window libraries remain operating-system prerequisites.

Each archive includes licenses and `build-info.json`, recording the public source
ref/commit, package version, configuration, VST2 inclusion, notarization state, and
payload hashes. Package versions may follow a selected version tag or branch rather
than the source's CMake project version. Published release assets include checksums
and a release manifest. Proprietary SDK source and headers are excluded.

Linux archive manifests also contain `build_os` with the build distribution and
version, such as `{"id":"ubuntu","version_id":"24.04"}`. Use a system with libraries
compatible with that baseline. Packages built on Ubuntu 24.04 target a different
system-library baseline from packages built on Ubuntu 22.04.

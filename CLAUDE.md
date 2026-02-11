# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Intel(R) XPU Manager is an open-source tool for monitoring and managing Intel data center GPUs (Flex, Max, and Arc B series). It supports two modes:

- **XPU Manager** (daemon mode): Full-featured with gRPC daemon, REST API, Prometheus exporter, device grouping, and policies
- **XPU-SMI** (daemon-less mode): Subset of features, direct library calls only, included in GPU driver repo

The two modes conflict and cannot be installed on the same system.

## Common Development Commands

### Building

```bash
# Standard build (creates XPU Manager)
./build.sh

# Build XPU-SMI (daemon-less version)
./build.sh -DDAEMONLESS=ON

# Build with AMC management CLI
./build.sh -DBUILD_AMCMCLI=ON

# Build with documentation
./build.sh -DBUILD_DOC=ON
```

The build script:
1. Creates/cleans `build/` directory
2. Runs cmake with ccache (if available)
3. Builds with `make -j4`
4. Runs `cpack` to create distribution packages (.deb or .rpm)

### Docker Builds (Recommended for CI)

```bash
# Ubuntu 22.04
docker build -f builder/Dockerfile.builder-ubuntu22.04 -t intel-xpumanager .
docker run -v $PWD:/workspace intel-xpumanager bash -c "cd /workspace && ./build.sh"

# CentOS 8
docker build -f builder/Dockerfile.builder-centos8 -t intel-xpumanager .
docker run -v $PWD:/workspace intel-xpumanager bash -c "cd /workspace && ./build.sh"

# SLES
docker build -f builder/Dockerfile.builder-sles -t intel-xpumanager .
```

### Manual CMake Build

```bash
mkdir build && cd build
cmake .. [options]
make -j4
```

### Testing

The project primarily uses integration-style testing. Third-party components (PCM, hwloc, spdlog) have their own test suites in `third_party/*/tests/`. Main project tests are minimal:

```bash
# If core/test/ exists (not in main repo)
cd build
make test_xpum_api
./test_xpum_api
```

### Code Coverage

```bash
cmake -DCMAKE_BUILD_TYPE=Coverage ..
make -j4
# Generates coverage data with GCC/Clang
```

## Architecture

### Two-Mode Design

The same `cli/src/` code compiles for both modes via the `DAEMONLESS` preprocessor macro:

```
cli/src/comlet_*.cpp
       |
       ├─ DAEMONLESS=OFF → grpc_stub/ → xpumd → libxpum.so
       └─ DAEMONLESS=ON  → core_stub/ ──────────→ libxpum.so
```

### Key Components

**Core Library** (`core/`): `libxpum.so`
- Public C API in `core/include/xpum_api.h` and `xpum_structs.h`
- Internal implementation in `core/src/`
  - `device/gpu/` - GPU device abstraction
  - `monitor/` - Telemetry collection via Level Zero
  - `health/` - Health monitoring and thresholds
  - `diagnostic/` - 3-level diagnostic test suites
  - `firmware/` - Firmware updates (GFX, GFX_DATA, AMC)
  - `amc/` - AMC firmware management
  - `control/` - GPU configuration (power limits, frequency, ECC)
  - `topology/` - Hardware topology via hwloc
  - `vgpu/` - SR-IOV virtual function support
  - `redfish/` and `ipmi/` - BMC communication
  - `policy/` - Policy engine for automated actions
  - `group/` - Device grouping and batch operations

**Daemon** (`daemon/`): `xpumd`
- gRPC service defined in `core.proto`
- Service implementations in `*_service_impl.cpp`
- Manages privileged hardware access

**CLI** (`cli/`): `xpumcli` or `xpu-smi`
- Commands are "comlet" classes: `comlet_discovery`, `comlet_statistics`, `comlet_firmware`, etc.
- Uses CLI11 for argument parsing
- Stub backends in `grpc_stub/` (daemon mode) or `core_stub/` (daemon-less)

**REST API** (`rest/`): Python Flask + Gunicorn
- Endpoints mirror gRPC methods
- Prometheus exporter at `rest/prometheus_exporter/`

### Communication Flow

```
CLI/REST → gRPC (or direct) → Daemon (or direct) → Core Library → Level Zero Driver → GPU Hardware
```

### Hardware Dependencies

- **Level Zero** (oneAPI): Primary GPU driver API
- **hwloc**: Hardware topology discovery
- **PCM**: Performance counter monitoring
- **spdlog**: Logging
- **gRPC/protobuf**: Daemon communication
- **METEE**: Intel telemetry library
- **IGSC**: Intel GPU Security library for firmware updates

## CLI Commands

Each command is a comlet class in `cli/src/comlet_*.cpp`:

| Command | Purpose |
|---------|---------|
| `discovery` | GPU device info and enumeration |
| `dump` | Real-time telemetry dump |
| `statistics` | Aggregated statistics |
| `firmware` | Firmware update (GFX, AMC) |
| `config` | GPU configuration (power, frequency, ECC) |
| `diagnostic` | Run diagnostic tests |
| `health` | Health status and thresholds |
| `topology` | Hardware topology |
| `policy` | Policy management |
| `group` | Device grouping |
| `vgpu` | Virtual GPU management |
| `log` | Log collection |

## Important Design Patterns

1. **Dual compilation**: Same CLI code compiles to two modes via `DAEMONLESS` macro
2. **Stub pattern**: `grpc_stub/` vs `core_stub/` abstract the communication layer
3. **Factory pattern**: Device creation and type management
4. **Singleton pattern**: Core class for global state
5. **Command pattern**: Each CLI command is a comlet class

## Working with This Codebase

### Adding a New CLI Command

1. Create `cli/src/comlet_yourcommand.cpp` and `.h`
2. Implement `ComletBase` interface: `setupOptions()`, `getTableOptions()`, `run()`
3. Register in `cli/src/main.cpp`
4. Add stub implementations in both `grpc_stub/` and `core_stub/`
5. Add corresponding API in `core/src/api/` if needed

### Adding New Telemetry Metrics

1. Add metric enum in `core/include/xpum_structs.h`
2. Implement data collection in `core/src/monitor/`
3. Update proto in `daemon/core.proto` if exposing via daemon
4. Add CLI output formatting in relevant comlet

### Modifying Firmware Updates

Firmware updates use multiple paths:
- GPU GFX firmware: `core/src/firmware/` via IGSC library
- AMC firmware: `core/src/amc/` via IPMI or Redfish

Both have security implications and require careful error handling.

## File Structure Notes

- `third_party/` contains vendored dependencies - do not modify
- `docs/` contains generated HTML documentation (do not edit)
- `doc/` contains markdown source documentation
- `builder/` contains Dockerfiles for containerized builds

## Supported Platforms

- Ubuntu 20.04/22.04/24.04
- RHEL 8.8/9.2
- CentOS 7/8/9 Stream
- SLES 15 SP4/SP5
- Debian 10.13 (XPU-SMI only)
- Windows Server 2019/2022 (limited XPU-SMI)

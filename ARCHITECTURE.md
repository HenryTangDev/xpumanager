# Intel XPU Manager — Software Architecture

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           User Interfaces                                │
│                                                                          │
│  ┌───────────┐   ┌──────────────┐   ┌─────────────┐   ┌──────────────┐ │
│  │  xpumcli  │   │   REST API   │   │  Prometheus  │   │   xpu-smi   │ │
│  │  (CLI)    │   │  (Flask +    │   │  Exporter    │   │ (Standalone) │ │
│  │           │   │   Gunicorn)  │   │              │   │              │ │
│  └─────┬─────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘ │
│        │                │                  │                   │        │
└────────┼────────────────┼──────────────────┼───────────────────┼────────┘
         │ gRPC           │ gRPC             │ gRPC             │ direct
         ▼                ▼                  ▼                  │
┌─────────────────────────────────────────────────────┐         │
│                 Daemon (xpumd)                      │         │
│                                                     │         │
│   core.proto ──► *_service_impl.cpp                 │         │
│                                                     │         │
└─────────────────────────┬───────────────────────────┘         │
                          │                                     │
                          ▼                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    Core Library (libxpum.so)                              │
│                                                                          │
│   Public API:  xpum_api.h  /  xpum_structs.h                            │
│                                                                          │
│   ┌───────────┐  ┌──────────┐  ┌─────────────┐  ┌────────────────┐     │
│   │ monitor/  │  │ health/  │  │ diagnostic/ │  │   firmware/    │     │
│   │ telemetry │  │ checks   │  │ test suites │  │   amc/ igsc    │     │
│   └───────────┘  └──────────┘  └─────────────┘  └────────────────┘     │
│   ┌───────────┐  ┌──────────┐  ┌─────────────┐  ┌────────────────┐     │
│   │ device/   │  │ control/ │  │ topology/   │  │    policy/     │     │
│   │ gpu/      │  │ config   │  │ hwloc       │  │    group/      │     │
│   └───────────┘  └──────────┘  └─────────────┘  └────────────────┘     │
│   ┌───────────┐  ┌──────────┐  ┌─────────────┐  ┌────────────────┐     │
│   │ vgpu/     │  │ redfish/ │  │ ipmi/       │  │  data_logic/   │     │
│   └───────────┘  └──────────┘  └─────────────┘  └────────────────┘     │
│                                                                          │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      Level Zero Driver API                               │
│                      Intel GPU Hardware (Flex / Max / Arc B)             │
└──────────────────────────────────────────────────────────────────────────┘
```

## Build Variants

### XPU Manager (Daemon Mode)

```
  ┌──────────┐       ┌────────────┐       ┌──────────┐       ┌─────────────┐
  │ cli/src/ │──────►│ grpc_stub/ │─gRPC─►│  xpumd   │──────►│ libxpum.so  │
  └──────────┘       └────────────┘       └──────────┘       └─────────────┘
```

### XPU-SMI (Daemon-less Mode)

```
  ┌──────────┐       ┌────────────┐                          ┌─────────────┐
  │ cli/src/ │──────►│ core_stub/ │────────── direct ───────►│ libxpum.so  │
  └──────────┘       └────────────┘                          └─────────────┘
```

The same `cli/src/` code compiles for both modes. The `DAEMONLESS` preprocessor
macro selects which stub backend to link:

```
                          ┌─────────────────┐
                          │    cli/src/      │
                          │  comlet_*.cpp    │
                          └────────┬────────┘
                                   │
                          DAEMONLESS macro?
                          ┌────────┴────────┐
                          │                 │
                     No   ▼            Yes  ▼
               ┌──────────────┐    ┌──────────────┐
               │  grpc_stub/  │    │  core_stub/  │
               │  (RPC calls) │    │ (direct API) │
               └──────┬───────┘    └──────┬───────┘
                      │                   │
                      ▼                   │
               ┌──────────┐              │
               │  xpumd   │              │
               └─────┬────┘              │
                     │                   │
                     ▼                   ▼
               ┌────────────────────────────┐
               │       libxpum.so           │
               └────────────────────────────┘
```

## CLI Command Pattern

Each CLI command is a "comlet" class registered in the main CLI setup:

```
  main()
    │
    ├── register ──► comlet_discovery
    ├── register ──► comlet_statistics
    ├── register ──► comlet_health
    ├── register ──► comlet_diagnostic
    ├── register ──► comlet_firmware
    ├── register ──► comlet_config
    ├── register ──► comlet_topology
    ├── register ──► comlet_vgpu
    └── register ──► ...
```

## Data Flow — Telemetry Collection

```
  CLI / REST Client          xpumd              libxpum.so         Level Zero        GPU
        │                      │                     │                  │              │
        │  gRPC: GetStats()    │                     │                  │              │
        │─────────────────────►│                     │                  │              │
        │                      │  xpumGetStats()     │                  │              │
        │                      │────────────────────►│                  │              │
        │                      │                     │  zetSysman...()  │              │
        │                      │                     │─────────────────►│              │
        │                      │                     │                  │  HW read     │
        │                      │                     │                  │─────────────►│
        │                      │                     │                  │              │
        │                      │                     │                  │◄─────────────│
        │                      │                     │◄─────────────────│  raw metrics │
        │                      │◄────────────────────│  xpum_stats_t   │              │
        │◄─────────────────────│  gRPC Response      │                  │              │
        │                      │                     │                  │              │
```

## Directory Structure

```
xpumanager/
├── core/                        Core library (libxpum.so)
│   ├── include/                   Public API headers
│   │   ├── xpum_api.h               C API functions
│   │   └── xpum_structs.h           Data structures & enums
│   └── src/                       Internal implementation
│       ├── api/                     API entry points
│       ├── device/gpu/              GPU device management
│       ├── monitor/                 Telemetry collection
│       ├── health/                  Health monitoring
│       ├── diagnostic/              Diagnostic test suites
│       ├── firmware/                Firmware update (igsc)
│       ├── amc/                     AMC firmware management
│       ├── control/                 GPU configuration
│       ├── topology/                Hardware topology (hwloc)
│       ├── vgpu/                    Virtual GPU support
│       ├── redfish/                 Redfish BMC interface
│       ├── ipmi/                    IPMI interface
│       ├── policy/                  Policy engine
│       ├── group/                   Device grouping
│       ├── data_logic/              Data processing
│       └── log/                     Logging (spdlog)
├── daemon/                      gRPC daemon (xpumd)
│   ├── core.proto                 Proto3 service definition
│   └── *_service_impl.cpp         Service implementations
├── cli/                         CLI interface
│   ├── src/                       Comlet sub-commands
│   │   ├── grpc_stub/               gRPC backend (daemon mode)
│   │   └── core_stub/               Direct backend (daemon-less)
│   └── test/                      Google Test suites
├── rest/                        Python REST API
│   └── prometheus_exporter/       Prometheus metrics exporter
├── amcmcli/                     Standalone AMC firmware CLI
└── third_party/                 Vendored dependencies
    ├── CLI11/                     Argument parsing
    ├── json/                      nlohmann/json
    ├── spdlog/                    Logging
    ├── hwloc/                     HW topology
    ├── pcm/                       Performance counters
    └── googletest/                Test framework
```

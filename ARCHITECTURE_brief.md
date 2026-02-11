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
┌────────────────────────────────────────────────────────────────────────────────────┐
│                         Core Library (libxpum.so)                                  │
│                         core/                                                      │
│                                                                                    │
│    Public API:  xpum_api.h  /  xpum_structs.h                                      │
│                                                                                    │
│    ┌───────────────────────────────────────────────────────────────────────────┐   │
│    │                         API Layer (api/)                                  │   │
│    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │   │
│    │  │  xpum_api.cpp   │  │  xpum_init_*.cpp│  │  xpum_*_*.cpp   │            │   │
│    │  │  (entry points) │  │  (initializers) │  │  (operations)   │            │   │
│    │  └─────────────────┘  └─────────────────┘  └─────────────────┘            │   │
│    └───────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                          │   
│                                         ▼                                          │   
│    ┌───────────────────────────────────────────────────────────────────────────┐   │
│    │                    ORCHESTRATOR (Core Singleton)                          │   │
│    │                    core/src/core/core.cpp                                 │   │
│    │                                                                           │   │
│    │  ┌─────────────────────────────────────────────────────────────────────┐  │   │
│    │  │               Manager Initialization Order                          │  │   │
│    │  │                                                                     │  │   │
│    │  │  1. DataLogic            ──▶  (shared by all managers)             │  │   │
│    │  │  2. DeviceManager        ──▶  (GPU discovery & enumeration)        │  │   │
│    │  │  3. HealthManager        ──▶  (threshold monitoring)               │  │   │
│    │  │  4. GroupManager         ──▶  (logical device groups)              │  │   │
│    │  │  5. PolicyManager        ──▶  (automated actions)                  │  │   │
│    │  │  6. DumpRawDataManager   ──▶  (raw data export)                    │  │   │
│    │  │  7. FirmwareManager      ──▶  (firmware flash)                     │  │   │
│    │  │  8. DiagnosticManager    ──▶  (test suites)                        │  │   │
│    │  │  9. MonitorManager       ──▶  (telemetry collection)               │  │   │
│    │  │ 10. VgpuManager          ──▶  (SR-IOV virtualization)              │  │   │
│    │  └─────────────────────────────────────────────────────────────────────┘  │   │
│    └───────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                          │   
│                                         ▼                                          │   
│    ┌───────────────────────────────────────────────────────────────────────────┐   │
│    │                           Manager Layer                                   │   │
│    │                                                                           │   │
│    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐               │   │
│    │  │ DeviceManager  │  │  DataLogic     │  │ MonitorManager │               │   │
│    │  │ device/gpu/    │  │ data_logic/    │  │ monitor/       │               │   │
│    │  │ - GPU devices  │  │ - Aggregation  │  │ - Telemetry    │               │   │
│    │  │ - Enumeration  │  │ - Sessions     │  │ - Stats        │               │   │
│    │  │ - Properties   │  │ - Scale        │  │ - Sampling     │               │   │
│    │  └────────────────┘  └────────────────┘  └────────────────┘               │   │
│    │                                                                           │   │
│    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐               │   │
│    │  │ HealthManager  │  │PolicyManager   │  │GroupManager    │               │   │
│    │  │ health/        │  │ policy/        │  │ group/         │               │   │
│    │  │ - Thresholds   │  │ - Conditions   │  │ - Groups       │               │   │
│    │  │ - Alerts       │  │ - Actions      │  │ - Add/Remove   │               │   │
│    │  └────────────────┘  └────────────────┘  └────────────────┘               │   │
│    │                                                                           │   │
│    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐               │   │
│    │  │DiagnosticMgr   │  │FirmwareManager │  │VgpuManager     │               │   │
│    │  │ diagnostic/    │  │ firmware/      │  │ vgpu/          │               │   │
│    │  │ - Test suites  │  │ - AMC/IGSC     │  │ - SR-IOV       │               │   │
│    │  │ - Stress tests │  │ - Flash        │  │ - VF mgmt      │               │   │
│    │  └────────────────┘  └────────────────┘  └────────────────┘               │   │
│    │                                                                           │   │
│    │  ┌────────────────────────────────────────────────────────┐               │   │
│    │  │              DumpRawDataManager                        │               │   │
│    │  │              dump_raw_data/                            │               │   │
│    │  │              (raw metrics export to file)              │               │   │
│    │  └────────────────────────────────────────────────────────┘               │   │
│    │                                                                           │   │
│    │  Supporting Modules:                                                     │   │
│    │  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌────────────┐              │   │
│    │  │topology/ │  │ control/ │  │ redfish/   │  │ ipmi/      │              │   │
│    │  │ hwloc    │  │ config   │  │ BMC        │  │ BMC        │              │   │
│    │  └──────────┘  └──────────┘  └────────────┘  └────────────┘              │   │
│    └──────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                        │
└──────────────────────────────────────────┼────────────────────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      Level Zero Driver API                               │
│                      Intel GPU Hardware (Flex / Max / Arc B)             │
└──────────────────────────────────────────────────────────────────────────┘
```

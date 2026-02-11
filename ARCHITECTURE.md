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

## System Lifecycle

For detailed information about initialization, operations, and shutdown, see:

**[Lifecycle Documentation](docs/lifecycle-initialization-operations-shutdown.md)**

### Lifecycle Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        XPU Manager Lifecycle                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. INITIALIZATION (xpumInit)                                           │
│     ├─ Core::init()                                                     │
│     │  ├─ Configuration::init()                                         │
│     │  ├─ DataLogic::init()          (shared state)                     │
│     │  ├─ DeviceManager::init()      (GPU discovery)                   │
│     │  ├─ HealthManager::init()                                         │
│     │  ├─ GroupManager::init()                                          │
│     │  ├─ PolicyManager::init()       (1 timer thread)                  │
│     │  ├─ DumpRawDataManager::init()  (2 worker threads)                │
│     │  ├─ FirmwareManager::init()                                       │
│     │  ├─ DiagnosticManager::init()                                     │
│     │  └─ MonitorManager::init()      (16 worker threads)               │
│     │                                                                  │
│  2. OPERATIONS (running state)                                         │
│     ├─ MonitorManager: 16 threads collect metrics every 500ms          │
│     │  └─▶ GPU → Level Zero → DataLogic (shared state)                 │
│     │                                                                  │
│     ├─ PolicyManager: 1 thread evaluates policies every 500ms         │
│     │  └─▶ DataLogic → threshold checks → automated actions           │
│     │                                                                  │
│     ├─ DiagnosticManager: ephemeral threads for test runs             │
│     │  └─▶ std::thread + detach → async execution                     │
│     │                                                                  │
│     └─ RPC API: 80 gRPC endpoints handle client requests              │
│                                                                  │
│  3. SHUTDOWN (xpumShutdown)                                           │
│     └─ Core::close()                                                  │
│        ├─ Stop PolicyManager timer                                     │
│        ├─ Stop DiagnosticManager                                       │
│        ├─ Stop GroupManager                                            │
│        ├─ Stop HealthManager                                           │
│        ├─ Stop MonitorManager (16 threads)                             │
│        ├─ Stop DeviceManager                                           │
│        └─ Stop DataLogic                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Thread Summary

| Manager | Thread Type | Count | Purpose |
|---------|-------------|-------|---------|
| **MonitorManager** | ScheduledThreadPool | 16 | Collect telemetry every 500ms |
| **DumpRawDataManager** | ScheduledThreadPool | 2 | Dump raw data to files |
| **PolicyManager** | Timer | 1 | Evaluate policies every 500ms |
| **DeviceManager** | Detached thread | 1 (on-demand) | Fabric link rediscovery |
| **DiagnosticManager** | Detached thread | 1 per diagnostic run | Execute test suites |

**Total**: 19-20 persistent background threads + ephemeral diagnostic threads

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

---

## Core Library Architecture

The Core Library (`libxpum.so`) is organized into three distinct layers:

```
┌────────────────────────────────────────────────────────────┐
│  API Layer        →  Public C interface (xpum_api.h)     │
│  Orchestrator     →  Core singleton (core/src/core/)      │
│  Manager Layer    →  Functional managers (core/src/*/)    │
└────────────────────────────────────────────────────────────┘
```

### 1. API Layer (`core/src/api/`)

The public C API that external callers use:

| File | Purpose |
|------|---------|
| `xpum_api.cpp` | Main API entry points |
| `xpum_init*.cpp` | Initialization/cleanup |
| `xpum_*_*.cpp` | Domain-specific operations (stats, health, config, etc.) |

### 2. Orchestrator (`Core` Singleton)

**File:** `core/src/core/core.cpp`

The `Core` class is a singleton that:
- Initializes all managers in dependency order
- Provides access to managers via getter methods
- Manages library lifecycle (init/close)

**Manager Dependencies:**
```
DataLogic (shared state)
    ├───▶ DeviceManager (GPU discovery)
    │         ├───▶ HealthManager (threshold monitoring)
    │         ├───▶ GroupManager (logical groups)
    │         ├───▶ MonitorManager (telemetry)
    │         ├───▶ DiagnosticManager (test suites)
    │         └───▶ PolicyManager (automated actions)
    └───▶ FirmwareManager (firmware flash)
```

### 3. Manager Layer

| Manager | Directory | Purpose |
|---------|-----------|---------|
| **DeviceManager** | `device/gpu/` | GPU discovery, properties, PCIe info |
| **DataLogic** | `data_logic/` | Stats aggregation, sessions, scaling |
| **MonitorManager** | `monitor/` | Telemetry sampling, metrics collection |
| **HealthManager** | `health/` | Threshold checks, alerts |
| **PolicyManager** | `policy/` | Automated actions based on conditions |
| **GroupManager** | `group/` | Logical device grouping |
| **DiagnosticManager** | `diagnostic/` | Test suites, stress tests |
| **FirmwareManager** | `firmware/` | GPU/AMC firmware flash via IGSC |
| **VgpuManager** | `vgpu/` | SR-IOV virtualization, VF management |
| **DumpRawDataManager** | `dump_raw_data/` | Export raw metrics to files |

### 4. Supporting Modules

| Module | Directory | Purpose |
|--------|-----------|---------|
| **Topology** | `topology/` | CPU affinity, PCIe switches, NUMA |
| **Control** | `control/` | Device config (freq, power, scheduler) |
| **Redfish** | `redfish/` | BMC communication via Redfish API |
| **IPMI** | `ipmi/` | BMC IPMI interface |

---

## Threading Architecture

### Overview

The Core Library (`libxpum.so`) uses a **hybrid threading model**:

```
┌─────────────────────────────────────────────────────────────────┐
│                         xpumd Process                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Main Thread (Core::init)                      │    │
│  │     Synchronous initialization of all managers            │    │
│  └──────────────────────────────────────────────────────────┘    │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────┐              ┌──────────────────────┐  │
│  │   MonitorManager     │              │   PolicyManager      │  │
│  │   16 worker threads  │              │   1 detached thread  │  │
│  └──────────────────────┘              └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Manager Threading Models

| Manager | Threads | Implementation | Purpose |
|---------|---------|----------------|---------|
| **Core (Orchestrator)** | 0 | Synchronous only | Initializes managers sequentially |
| **MonitorManager** | 16 | `ScheduledThreadPool` | Periodic telemetry collection |
| **PolicyManager** | 1 | `Timer` | Periodic policy checking |
| **DeviceManager** | 0 | Synchronous | GPU discovery (one-time) |
| **DataLogic** | 0 | Synchronous | In-memory aggregation |
| **HealthManager** | 0 | Synchronous | Reads from DataLogic |
| **GroupManager** | 0 | Synchronous | In-memory group management |
| **DiagnosticManager** | 0 | Synchronous | On-demand tests |
| **FirmwareManager** | 0 | Synchronous | On-demand flash |
| **VgpuManager** | 0 | Synchronous | SR-IOV configuration |
| **DumpRawDataManager** | 0 | Synchronous | File I/O |

**Total background threads: 17** (16 for telemetry + 1 for policy checking)

---

### MonitorManager Threading (16 Threads)

**Why 16 threads?**

MonitorManager has the heaviest workload in the system:

```
Workload: 30+ metric types × N GPUs × Level Zero I/O every 500ms

┌─────────────────────────────────────────────────────────────────┐
│                    MonitorManager                               │
│                                                                 │
│  ScheduledThreadPool (16 worker threads)                        │
│  ┌────────────────────────────────────────────────────────┐      │
│  │  MonitorTask (POWER)      ──▶ Level Zero I/O (blocking)  │      │
│  │  MonitorTask (TEMP)       ──▶ Level Zero I/O (blocking)  │      │
│  │  MonitorTask (FREQ)       ──▶ Level Zero I/O (blocking)  │      │
│  │  MonitorTask (MEMORY)     ──▶ Level Zero I/O (blocking)  │      │
│  │  MonitorTask (UTIL)       ──▶ Level Zero I/O (blocking)  │      │
│  │  MonitorTask (RAS)        ──▶ Level Zero I/O (blocking)  │      │
│  │  ... ~30 more tasks ...                                   │      │
│  │                                                         │      │
│  │  Each task runs every 500ms, collecting data from all  │      │
│  │  GPUs via Level Zero driver calls (10-100ms per GPU)    │      │
│  │                                                         │      │
│  │  Worker threads execute tasks concurrently from        │      │
│  │  SchedulingQueue (priority-ordered by execution time)  │      │
│  └────────────────────────────────────────────────────────┘      │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │              DataLogic (shared state)                   │      │
│  │     All threads WRITE collected metrics here            │      │
│  └─────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

**Calculation:**
```
Time to collect: 30 metrics × 8 GPUs × 50ms = 240 operations

If 1 thread:  240 × 50ms = 12 seconds (too slow!)
If 16 threads: 240 ÷ 16 × 50ms = 750ms (fits in 1000ms window)
```

**MonitorTask Example** (`monitor_task.cpp`):
```cpp
void MonitorTask::start(std::shared_ptr<ScheduledThreadPool>& threadPool) {
    // Schedule this task to run periodically
    p_scheduled_task = threadPool->scheduleAtFixedRate(
        0,                    // delay
        500,                  // interval (ms)
        -1,                   // execution_times (-1 = forever)
        [this_weak_ptr]() {
            // Collect metrics from all GPUs
            Utility::parallel_in_batches(devices.size(), devices.size(), 
                [&](int start, int end){
                for (int i = start; i < end; ++i) {
                    // Blocking Level Zero API call
                    method(...);  // 10-100ms per GPU
                }
            });
        }
    );
}
```

---

### PolicyManager Threading (1 Thread)

**Why only 1 thread?**

PolicyManager does **lightweight in-memory operations**:
- Reads pre-aggregated metrics from `DataLogic` (no I/O)
- Simple threshold comparisons
- Triggers notifications

```
┌─────────────────────────────────────────────────────────────────┐
│                    PolicyManager                                │
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐     │
│   │              Timer (1 detached thread)                 │     │
│   │                                                       │     │
│   │   scheduleAtFixedRate(500ms)                          │     │
│   │         │                                             │     │
│   │         ▼                                             │     │
│   │   handleForOneCyle() {                                │     │
│   │       checkPolicy();          // Read from DataLogic  │     │
│   │       triggerAction();        // Simple comparison    │     │
│   │       triggerNotification();  // Fast execution      │     │
│   │   }                                                   │     │
│   └───────────────────────────────────────────────────────┘     │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │              DataLogic (shared state)                   │      │
│  │     Policy thread READS metrics (already collected)    │      │
│  └─────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

**PolicyManager::start()** (`policy_manager.cpp`):
```cpp
void PolicyManager::start() {
    long delay = freq - now % freq;
    this->p_timer = std::make_shared<Timer>();
    this->p_timer->scheduleAtFixedRate(delay, freq, [this_weak_ptr]() {
        p_this->handleForOneCyle();  // Runs every 500ms
    });
}
```

---

### Threading Infrastructure

**Location:** `core/src/infrastructure/`

| Component | File | Purpose |
|-----------|------|---------|
| **ScheduledThreadPool** | `scheduled_thread_pool.{h,cpp}` | Thread pool for time-scheduled tasks |
| **SchedulingQueue** | `scheduled_thread_pool.{h,cpp}` | Priority queue for timed tasks |
| **Timer** | `timer.{h,cpp}` | Single-threaded periodic timer |
| **SharedQueue** | `shared_queue.h` | Thread-safe producer-consumer queue |

**ScheduledThreadPool Implementation:**
```cpp
// scheduled_thread_pool.cpp:33-65
void ScheduledThreadPool::init(uint32_t& size) {
    for (uint32_t i = 0; i < size; ++i) {
        this->workers.emplace_back(std::thread([this]() {
            while (true) {
                auto task = this->p_taskqueue->dequeue();  // Blocks until ready
                if (this->stop) break;
                task->run();  // Execute the task
                if (task->next()) {
                    this->p_taskqueue->enqueue(task);  // Re-schedule
                }
            }
        }));
    }
}
```

**Timer Implementation:**
```cpp
// timer.cpp:28-62
void Timer::scheduleAtFixedRate(long delay, int interval, std::function<void()> task) {
    std::thread([this, delay, interval, task]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(delay));
        while (!this->to_cancel) {
            task();  // Run user function
            std::this_thread::sleep_for(std::chrono::milliseconds(interval));
        }
    }).detach();  // Detached thread runs independently
}
```

---

### Producer-Consumer Pattern

```
┌─────────────────┐         ┌─────────────────┐
│ MonitorManager  │         │ PolicyManager   │
│ (Producer)      │         │ (Consumer)      │
│                 │         │                 │
│ 16 threads @    │──────▶  │ Reads & checks  │
│ collect metrics │  Data  │ conditions      │
│ from GPUs       │  Logic  │ every 500ms     │
│ via Level Zero  │         │                 │
└─────────────────┘         └─────────────────┘
```

- **MonitorManager** (16 threads) → **PRODUCES** metrics → **DataLogic**
- **PolicyManager** (1 thread) → **CONSUMES** metrics from **DataLogic**
- **HealthManager** (main thread) → **CONSUMES** metrics from **DataLogic** when queried

---

### Configuration

**Device thread pool size** (defined but unused by MonitorManager):
```cpp
// configuration.cpp:28
int Configuration::DEVICE_THREAD_POOL_SIZE = 32;
```

**MonitorManager hardcodes 16:**
```cpp
// monitor_manager.cpp:25
p_scheduled_thread_pool = std::make_shared<ScheduledThreadPool>(16);  // Hardcoded
```

**Polling frequencies:**
```cpp
// configuration.cpp:24-27
int Configuration::TELEMETRY_DATA_MONITOR_FREQUENCE = 500;      // 500ms
int Configuration::POWER_MONITOR_INTERNAL_PERIOD = 80;          // 80ms
int Configuration::MEMORY_BANDWIDTH_MONITOR_INTERNAL_PERIOD = 80;  // 80ms
int Configuration::VF_METRICS_INTERVAL = 110;                   // 110ms
```


# Intel XPU Manager — DCGM-Aligned Modular Architecture

## Executive Summary

This document proposes restructuring the Intel XPU Manager core library (`libxpum.so`) from a monolithic design into a **modular, plugin-based architecture** aligned with NVIDIA DCGM's module system.

**The key change**: each major subsystem (monitor, health, diagnostic, firmware, policy, config, vgpu, topology) becomes a dynamically-loaded `.so` module with a standardized interface. A slimmed-down core provides shared services (device management, field cache, logging) that modules access through a `CoreProxy` pattern. Modules communicate via versioned message structs routed through the core.

### Why This Change

| Problem (Current) | Solution (Proposed) |
|---|---|
| Monolithic `libxpum.so` contains all functionality | Extract subsystems into separate `.so` modules |
| Tight coupling between subsystems | Message-based communication with versioned structs |
| Adding features requires modifying and rebuilding core | New modules can be added without touching core |
| All-or-nothing deployment | Modules can be installed selectively |
| Single large binary to test and debug | Smaller, isolated units with clear interfaces |

### Scope

- **Changes**: Core library (`core/`) internal architecture
- **Unchanged**: CLI, daemon, gRPC protocol, REST API, user experience, public C API signatures

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            User Interfaces (UNCHANGED)                               │
│                                                                                     │
│  ┌───────────┐   ┌──────────────┐   ┌─────────────┐   ┌──────────────┐            │
│  │  xpumcli  │   │   REST API   │   │  Prometheus  │   │   xpu-smi   │            │
│  │  (CLI)    │   │  (Flask +    │   │  Exporter    │   │ (Standalone) │            │
│  │           │   │   Gunicorn)  │   │              │   │              │            │
│  └─────┬─────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘            │
│        │                │                  │                   │                    │
└────────┼────────────────┼──────────────────┼───────────────────┼────────────────────┘
         │ gRPC           │ gRPC             │ gRPC             │ direct
         ▼                ▼                  ▼                  │
┌─────────────────────────────────────────────────────┐         │
│                 Daemon (xpumd) — UNCHANGED           │         │
│   core.proto ──► *_service_impl.cpp                  │         │
└─────────────────────────┬───────────────────────────┘         │
                          │                                     │
                          ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        Core Library (libxpum_core.so)                                │
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                        Public C API (xpum_api.h)                              │  │
│  │  xpumInit / xpumGetStats / xpumRunDiag / xpumSetHealth / xpumListPlugins ... │  │
│  └──────────────────────────────────┬────────────────────────────────────────────┘  │
│                                     │                                               │
│  ┌──────────────────────────────────▼────────────────────────────────────────────┐  │
│  │                       ModuleManager                                           │  │
│  │                                                                               │  │
│  │  ┌─────────────────────────────┐  ┌────────────────────────────────────────┐  │  │
│  │  │    Module Registry          │  │     Plugin Discovery                   │  │  │
│  │  │                             │  │                                        │  │  │
│  │  │  Tier 1 (built-in):        │  │  Scan /usr/lib/xpum/modules/ (Tier 1) │  │  │
│  │  │    IDs 0-31, known at      │  │  Scan /usr/lib/xpum/plugins/ (Tier 2) │  │  │
│  │  │    compile time             │  │                                        │  │  │
│  │  │                             │  │  For each .so:                         │  │  │
│  │  │  Tier 2 (extension):       │  │    1. dlopen (RTLD_LAZY)              │  │  │
│  │  │    IDs 32+, discovered     │  │    2. xpum_get_module_descriptor()    │  │  │
│  │  │    at runtime from .so     │  │    3. Check API version compat        │  │  │
│  │  │    files in plugins/       │  │    4. Check device PCI ID match       │  │  │
│  │  │                             │  │    5. Register or skip               │  │  │
│  │  └─────────────────────────────┘  │    6. dlclose (reopen on demand)     │  │  │
│  │                                    └────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────┐  ┌────────────────────────────────────────┐  │  │
│  │  │    Selective Loader         │  │     Message Router                     │  │  │
│  │  │                             │  │                                        │  │  │
│  │  │  LoadModuleWithDeps(id):    │  │  ProcessModuleCommand(cmd):            │  │  │
│  │  │    1. Check allowlist       │  │    1. Extract moduleId from header     │  │  │
│  │  │    2. Resolve dependencies  │  │    2. Load module if NOT_LOADED        │  │  │
│  │  │    3. dlopen + dlsym        │  │    3. Dispatch to ProcessMessage()     │  │  │
│  │  │    4. Alloc instance        │  │    4. Return result                    │  │  │
│  │  │    5. Status = LOADED       │  │                                        │  │  │
│  │  │                             │  │  GetPluginsByCapability(cap):           │  │  │
│  │  │  UnloadModule(id):          │  │    → returns matching Tier 2 plugins   │  │  │
│  │  │    1. Check no dependents   │  │                                        │  │  │
│  │  │    2. Free instance         │  │                                        │  │  │
│  │  │    3. dlclose               │  │                                        │  │  │
│  │  └─────────────────────────────┘  └────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Core Services (always resident)                       │  │
│  │                                                                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │  │
│  │  │ DeviceManager│  │  FieldCache  │  │ Configuration│  │ScheduledThread   │  │  │
│  │  │              │  │              │  │              │  │ Pool             │  │  │
│  │  │ - device list│  │ - time-series│  │ - global cfg │  │                  │  │  │
│  │  │ - ze handles │  │   storage    │  │ - env vars   │  │ - periodic tasks │  │  │
│  │  │ - properties │  │ - watch list │  │ - module     │  │ - background ops │  │  │
│  │  │ - discovery  │  │ - eviction   │  │   allowlist  │  │ - timer mgmt    │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘  │  │
│  └──────────────────────────────────┬────────────────────────────────────────────┘  │
│                                     │                                               │
│  ┌──────────────────────────────────▼────────────────────────────────────────────┐  │
│  │                    CoreCallbacks (passed to every module)                      │  │
│  │                                                                               │  │
│  │  .postFunc    → route requests to core (device, field, inter-module)          │  │
│  │  .loggerFunc  → forward logs to spdlog                                        │  │
│  │  .coreContext → opaque Core* pointer                                          │  │
│  │  .version     → API version (for plugin compatibility check)                  │  │
│  └──────────────────────────────────┬────────────────────────────────────────────┘  │
│                                     │                                               │
└─────────────────────────────────────┼───────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────────────┐
          │ dlopen on demand          │                                   │
          │                           │                                   │
          ▼                           ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Tier 1: Built-in Modules  (/usr/lib/xpum/modules/)                                 │
│  IDs 0-31 — shipped with package, loaded ONLY when needed                           │
│                                                                                     │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │
│  │ monitor.so │ │ health.so  │ │  diag.so   │ │firmware.so │ │ policy.so  │       │
│  │  ID=1      │ │  ID=2      │ │  ID=3      │ │  ID=4      │ │  ID=5      │       │
│  │            │ │            │ │            │ │            │ │            │       │
│  │ Collects   │ │ Threshold  │ │ L1/L2/L3   │ │ GFX/AMC    │ │ Automated  │       │
│  │ metrics    │ │ health     │ │ diagnostic │ │ firmware   │ │ policy     │       │
│  │ → Field   │ │ checks     │ │ tests      │ │ updates    │ │ engine     │       │
│  │   Cache    │ │            │ │            │ │            │ │            │       │
│  │            │ │ deps:      │ │ deps:      │ │ deps:      │ │ deps:      │       │
│  │ deps: none│ │ [MONITOR]  │ │ [MONITOR]  │ │ none       │ │ [MONITOR,  │       │
│  │            │ │            │ │            │ │            │ │  GROUP]    │       │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │
│  │ config.so  │ │topology.so │ │  vgpu.so   │ │ group.so   │ │  dump.so   │       │
│  │  ID=6      │ │  ID=7      │ │  ID=8      │ │  ID=9      │ │  ID=10     │       │
│  │            │ │            │ │            │ │            │ │            │       │
│  │ Power/Freq │ │ hwloc      │ │ SR-IOV     │ │ Device     │ │ Raw data   │       │
│  │ ECC control│ │ topology   │ │ vGPU mgmt  │ │ grouping   │ │ export     │       │
│  │            │ │            │ │            │ │            │ │            │       │
│  │ deps: none│ │ deps: none │ │ deps: none │ │ deps: none │ │ deps:      │       │
│  │            │ │            │ │            │ │            │ │ [MONITOR]  │       │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘       │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Tier 2: Extension Plugins  (/usr/lib/xpum/plugins/)                                │
│  IDs 32+ — drop-in, auto-discovered, NO core changes needed                        │
│                                                                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐         │
│  │ xe2_perf.so         │  │ custom_health.so    │  │ oam_monitor.so     │  ...    │
│  │  ID=32 (dynamic)    │  │  ID=33 (dynamic)    │  │  ID=34 (dynamic)   │         │
│  │                     │  │                     │  │                     │         │
│  │ cap: DIAGNOSTIC     │  │ cap: HEALTH         │  │ cap: MONITORING    │         │
│  │ devices: Xe2 only   │  │ devices: all        │  │ devices: OAM boards│         │
│  │ apiVer: 100         │  │ apiVer: 100         │  │ apiVer: 110        │         │
│  │                     │  │                     │  │                     │         │
│  │ Self-describes via  │  │ Self-describes via  │  │ Self-describes via │         │
│  │ xpum_get_module_    │  │ xpum_get_module_    │  │ xpum_get_module_   │         │
│  │ descriptor()        │  │ descriptor()        │  │ descriptor()       │         │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘         │
│                                                                                     │
│  Capability-based routing: Diag module queries all CAP_DIAGNOSTIC plugins           │
│  and aggregates their test results alongside built-in tests.                        │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
          │                           │                                   │
          ▼                           ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    Level Zero Driver API / Intel GPU Hardware                         │
│                    (Flex / Max / Arc B Series)                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### DCGM Comparison

```
DCGM:                                      XPU Manager (Proposed):
┌────────────────────────┐                  ┌────────────────────────┐
│  nv-hostengine         │                  │  xpumd (daemon)        │
│  DcgmHostEngineHandler │                  │  gRPC services         │
└──────────┬─────────────┘                  └──────────┬─────────────┘
           │                                           │
┌──────────▼─────────────┐                  ┌──────────▼─────────────┐
│  dcgmlib (core)        │                  │  libxpum_core.so       │
│  - CacheManager        │        ≈         │  - FieldCache          │
│  - GroupManager        │                  │  - DeviceManager       │
│  - ClientHandler       │                  │  - ModuleManager       │
│  - Module loader       │                  │  - Plugin discovery    │
└──────────┬─────────────┘                  └──────────┬─────────────┘
           │ dlopen                                    │ dlopen
           │ (hardcoded IDs)                           │ (Tier1: hardcoded
           │                                           │  Tier2: discovered)
    ┌──────┼──────┐                          ┌─────────┼──────────┐
    ▼      ▼      ▼                          ▼         ▼          ▼
 health  diag  policy                     Tier 1    Tier 1     Tier 2
 module  module module                    health    diag       xe2_perf
                                          module    module     plugin
  (all hardcoded,                         (selective load +
   no extensibility)                       drop-in plugins)
```

### Detailed Core Library Internals

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          Core Library (libxpum_core.so)                               │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           Public C API Layer                                   │  │
│  │                                                                                │  │
│  │  xpum_api.cpp ─────────────────────────────────────────────────────────────── │  │
│  │  │                                                                             │  │
│  │  │  xpumInit()            → Core::init()                                      │  │
│  │  │                          → DeviceManager::init()                            │  │
│  │  │                          → FieldCache::init()                               │  │
│  │  │                          → ModuleManager::init() ← registers names ONLY    │  │
│  │  │                          → ModuleManager::DiscoverPlugins() ← scans dirs   │  │
│  │  │                          (NO modules loaded yet — zero dlopen at startup)   │  │
│  │  │                                                                             │  │
│  │  │  xpumGetDeviceList()   → DeviceManager::getDeviceList() [core-direct]      │  │
│  │  │                          (no module needed — always available)              │  │
│  │  │                                                                             │  │
│  │  │  xpumGetStats()        → build msg(MONITOR, GET_STATS)                     │  │
│  │  │                          → ModuleManager::ProcessModuleCommand()            │  │
│  │  │                          → triggers: LoadModule(MONITOR) if first call      │  │
│  │  │                                                                             │  │
│  │  │  xpumRunDiag()         → build msg(DIAG, RUN_DIAG)                         │  │
│  │  │                          → ModuleManager::ProcessModuleCommand()            │  │
│  │  │                          → triggers: LoadModule(MONITOR) ← dependency      │  │
│  │  │                          → triggers: LoadModule(DIAG)                       │  │
│  │  │                          → DIAG queries Tier 2 CAP_DIAGNOSTIC plugins      │  │
│  │  │                          → triggers: LoadPlugin("xe2_perf") if matched     │  │
│  │  │                                                                             │  │
│  │  │  xpumSetHealth()       → build msg(HEALTH, SET_CONFIG)                     │  │
│  │  │                          → triggers: LoadModule(MONITOR) ← dependency      │  │
│  │  │                          → triggers: LoadModule(HEALTH)                     │  │
│  │  │                                                                             │  │
│  │  │  xpumFirmwareFlash()   → build msg(FIRMWARE, FLASH)                        │  │
│  │  │                          → triggers: LoadModule(FIRMWARE) (no deps)         │  │
│  │  │                                                                             │  │
│  │  │  xpumListPlugins()     → ModuleManager::GetRegisteredPlugins()             │  │
│  │  │                          (returns descriptors, no loading needed)           │  │
│  │  │                                                                             │  │
│  └──┼─────────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                                │
│     ▼                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Core Singleton                                         │  │
│  │                                                                                │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────────┐ │  │
│  │  │   DeviceManager  │  │    FieldCache    │  │  ModuleManager              │ │  │
│  │  │   (always init)  │  │  (always init)   │  │  (always init)              │ │  │
│  │  │                  │  │                  │  │                              │ │  │
│  │  │  - device list   │  │  - time-series   │  │  ┌────────────────────────┐ │ │  │
│  │  │  - ze handles    │  │    storage       │  │  │  Tier 1 Registry       │ │ │  │
│  │  │  - properties    │  │  - watch list    │  │  │  [0] CORE: resident    │ │ │  │
│  │  │  - tile handles  │  │  - eviction      │  │  │  [1] MONITOR: not_loaded│ │ │  │
│  │  │  - PCI info      │  │  - aggregation   │  │  │  [2] HEALTH: not_loaded│ │ │  │
│  │  │                  │  │  - per-device     │  │  │  [3] DIAG: not_loaded  │ │ │  │
│  │  │  Provides:       │  │    per-field     │  │  │  [4] FIRMWARE: not_loaded│ │ │  │
│  │  │  - GetDeviceList │  │    buckets       │  │  │  ...                    │ │ │  │
│  │  │  - GetHandle     │  │                  │  │  │  [10] DUMP: not_loaded  │ │ │  │
│  │  │  - GetProperties │  │  Provides:       │  │  └────────────────────────┘ │ │  │
│  │  │                  │  │  - AddFieldWatch │  │  ┌────────────────────────┐ │ │  │
│  │  │                  │  │  - StoreSample   │  │  │  Tier 2 Registry       │ │ │  │
│  │  │                  │  │  - GetLatest     │  │  │  "xe2_perf": not_loaded│ │ │  │
│  │  │                  │  │  - GetStats      │  │  │    cap=DIAGNOSTIC      │ │ │  │
│  │  │                  │  │  - GetActive     │  │  │    dev=Xe2 PCI IDs     │ │ │  │
│  │  │                  │  │    Watches       │  │  │  "oam_mon": not_loaded │ │ │  │
│  │  │                  │  │                  │  │  │    cap=MONITORING      │ │ │  │
│  │  │                  │  │                  │  │  │  (discovered from .so  │ │ │  │
│  │  │                  │  │                  │  │  │   files at init time)  │ │ │  │
│  │  │                  │  │                  │  │  └────────────────────────┘ │ │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────────────────┘ │  │
│  │                                                                                │  │
│  │  ┌──────────────────┐  ┌──────────────────────────────────────────────────┐   │  │
│  │  │ScheduledThread   │  │  CoreCallbacks (constructed once, shared)        │   │  │
│  │  │ Pool             │  │                                                  │   │  │
│  │  │ - periodic tasks │  │  .postFunc    → Core::handleModuleRequest()      │   │  │
│  │  │ - background ops │  │  .loggerFunc  → spdlog::get("xpum")->log()      │   │  │
│  │  │ - timer mgmt    │  │  .coreContext → this (Core singleton pointer)    │   │  │
│  │  │                  │  │  .version     → XPUM_MODULE_API_VERSION (100)   │   │  │
│  │  └──────────────────┘  └──────────────────────────────────────────────────┘   │  │
│  │                                                                                │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### Detailed Module Internals (Tier 1: Health Module)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    libxpum_mod_health.so  (Tier 1, ID=2)                      │
│                                                                              │
│  Exported C Functions (4 standard exports):                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  xpum_get_module_descriptor()                                        │   │
│  │    → { name="health", cap=HEALTH, apiVer=100, devices=NULL(all) }   │   │
│  │                                                                      │   │
│  │  xpum_alloc_module_instance(&coreCallbacks) → new XpumModuleHealth  │   │
│  │  xpum_free_module_instance(module)          → delete module         │   │
│  │  xpum_module_process_message(module, cmd)   → module->ProcessMsg()  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  class XpumModuleHealth : XpumModuleWithCoreProxy<MODULE_ID_HEALTH> │   │
│  │                                                                      │   │
│  │  GetDependencies() → [XPUM_MODULE_ID_MONITOR]                       │   │
│  │                                                                      │   │
│  │  ProcessMessage(cmd)                                                 │   │
│  │    │                                                                 │   │
│  │    ├── subCommand == SET_CONFIG  → ProcessSetConfig(cmd)             │   │
│  │    │     └── m_coreProxy.AddFieldWatch(...)                          │   │
│  │    │     └── m_healthWatch->setThreshold(...)                        │   │
│  │    │                                                                 │   │
│  │    ├── subCommand == GET_CONFIG  → ProcessGetConfig(cmd)             │   │
│  │    │     └── m_healthWatch->getThreshold(...)                        │   │
│  │    │                                                                 │   │
│  │    ├── subCommand == GET_STATUS  → ProcessGetStatus(cmd)             │   │
│  │    │     └── m_coreProxy.GetLatestFieldValue(...)                    │   │
│  │    │     └── m_healthWatch->evaluateHealth(value, threshold)         │   │
│  │    │     └── Query Tier 2 CAP_HEALTH plugins for extra checks       │   │
│  │    │     └── cmd->healthData = aggregated result                     │   │
│  │    │                                                                 │   │
│  │    └── unknown subCommand → return XPUM_GENERIC_ERROR                │   │
│  │                                                                      │   │
│  │  Private Members:                                                    │   │
│  │    XpumCoreProxy m_coreProxy          (inherited from template)      │   │
│  │    std::unique_ptr<HealthWatch> m_healthWatch   (health logic)       │   │
│  │    std::mutex m_mutex                  (thread safety)               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Detailed Plugin Internals (Tier 2: Extension Plugin)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│              libxpum_plugin_xe2_perf.so  (Tier 2, ID=dynamic)                │
│                                                                              │
│  Exported C Functions (same 4 exports as Tier 1):                            │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  xpum_get_module_descriptor()                                        │   │
│  │    → { name="xe2-performance",                                      │   │
│  │         description="Extended perf diagnostics for Xe2 GPUs",       │   │
│  │         cap=DIAGNOSTIC,                                              │   │
│  │         apiVer=100,                                                  │   │
│  │         devices=["0x56c0","0x56c1",...] (Xe2 PCI IDs only) }        │   │
│  │                                                                      │   │
│  │  xpum_alloc_module_instance(&coreCallbacks)  → new Xe2PerfPlugin    │   │
│  │  xpum_free_module_instance(module)           → delete module        │   │
│  │  xpum_module_process_message(module, cmd)    → module->ProcessMsg() │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  class Xe2PerfPlugin : XpumModuleWithCoreProxy<32>                   │   │
│  │                                                                      │   │
│  │  GetDependencies() → [XPUM_MODULE_ID_MONITOR]                       │   │
│  │                                                                      │   │
│  │  ProcessMessage(cmd)                                                 │   │
│  │    │                                                                 │   │
│  │    ├── subCommand == DIAG_RUN → RunXe2Tests(cmd)                    │   │
│  │    │     ├── m_coreProxy.GetDeviceHandle(devId, &zeDev, &zesDev)    │   │
│  │    │     ├── Run Xe2 compute kernel benchmark                        │   │
│  │    │     ├── Run Xe2 memory bandwidth test                           │   │
│  │    │     ├── Run Xe2 EU occupancy test                               │   │
│  │    │     ├── Compare results against Xe2 reference values            │   │
│  │    │     └── Write results into cmd response struct                  │   │
│  │    │                                                                 │   │
│  │    ├── subCommand == DIAG_STATUS → GetXe2TestStatus(cmd)            │   │
│  │    │     └── Return in-progress / completed / failed                 │   │
│  │    │                                                                 │   │
│  │    ├── subCommand == DIAG_LIST_TESTS → ListAvailableTests(cmd)      │   │
│  │    │     └── Return ["xe2_compute", "xe2_membw", "xe2_eu_occ"]      │   │
│  │    │                                                                 │   │
│  │    └── unknown subCommand → return XPUM_GENERIC_ERROR                │   │
│  │                                                                      │   │
│  │  Private Members:                                                    │   │
│  │    XpumCoreProxy m_coreProxy                                         │   │
│  │    Xe2ReferenceData m_refData     (Xe2-specific benchmark targets)   │   │
│  │    std::atomic<bool> m_running    (test execution state)             │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Note: This plugin was NEVER compiled into core. It was:                    │
│    1. Built separately (cmake --build . --target xpum_plugin_xe2_perf)      │
│    2. Dropped into /usr/lib/xpum/plugins/                                    │
│    3. Auto-discovered by ModuleManager::DiscoverPlugins()                    │
│    4. Loaded on-demand when Diag module asked for CAP_DIAGNOSTIC plugins    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### ModuleManager Routing Detail (Tier 1 — Built-in Module)

```
                    API call: xpumGetHealth(deviceId, type, &data)
                              │
                              ▼
                    ┌──────────────────────────────────────────────┐
                    │  Build Message Struct                         │
                    │  xpum_health_msg_get_status_v1 msg;          │
                    │  msg.header.moduleId   = HEALTH (2)          │
                    │  msg.header.subCommand = GET_STATUS           │
                    │  msg.header.version    = v1                   │
                    │  msg.deviceId = deviceId                      │
                    │  msg.healthType = type                        │
                    └──────────────────┬───────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ModuleManager::ProcessModuleCommand(&msg.header)                            │
│                                                                              │
│  1. Extract moduleId from header → XPUM_MODULE_ID_HEALTH (2)                │
│     Is Tier 1 (ID < 32)? → Yes, look up in m_modules[]                      │
│                                                                              │
│  2. Check allowlist:                                                         │
│     ┌────────────────────────────────────────────────────────┐              │
│     │  m_config.enabledModules[HEALTH] == true?              │              │
│     │  No  → return XPUM_MODULE_NOT_LOADED (blocked by cfg)  │              │
│     │  Yes → continue                                         │              │
│     └────────────────────────────────────────────────────────┘              │
│                                                                              │
│  3. Check module status + load with dependencies:                            │
│     ┌────────────────────────────────────────────────────────┐              │
│     │  m_modules[2].status == ?                               │              │
│     │                                                         │              │
│     │  NOT_LOADED ─► LoadModuleWithDependencies(HEALTH)       │              │
│     │                  │                                      │              │
│     │                  ├─ Health declares deps: [MONITOR]     │              │
│     │                  │                                      │              │
│     │                  ├─ m_modules[MONITOR].status?           │              │
│     │                  │   NOT_LOADED → LoadModule(MONITOR)   │              │
│     │                  │     ├─ dlopen("libxpum_mod_monitor") │              │
│     │                  │     ├─ dlsym (alloc/free/process)    │              │
│     │                  │     ├─ descriptor check (API ver)    │              │
│     │                  │     ├─ allocFunc(&coreCallbacks)     │              │
│     │                  │     └─ status = LOADED ✓             │              │
│     │                  │   LOADED → skip (already loaded)     │              │
│     │                  │                                      │              │
│     │                  ├─ Now load HEALTH itself:             │              │
│     │                  │   ├─ dlopen("libxpum_mod_health.so") │              │
│     │                  │   ├─ dlsym (alloc/free/process)      │              │
│     │                  │   ├─ allocFunc(&coreCallbacks)       │              │
│     │                  │   └─ status = LOADED ✓               │              │
│     │                  │                                      │              │
│     │  LOADED ─► proceed to step 4                            │              │
│     │  FAILED ─► return XPUM_MODULE_NOT_LOADED                │              │
│     └────────────────────────────────────────────────────────┘              │
│                                                                              │
│  4. Dispatch: m_modules[2].processFunc(module, &msg.header)                  │
│                          │                                                   │
│                          ▼                                                   │
│              XpumModuleHealth::ProcessMessage(&msg.header)                    │
│                          │                                                   │
│                          ▼                                                   │
│              result written into msg.healthData                              │
│                                                                              │
│  5. Return msg.result to caller                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### ModuleManager Routing Detail (Tier 2 — Extension Plugin Discovery)

```
                    API call: xpumRunDiagnostics(deviceId, level=3)
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ModuleManager routes to DIAG module (Tier 1, loaded with deps)              │
│  → XpumModuleDiag::ProcessMessage(RUN_DIAG)                                 │
│                                                                              │
│  Inside Diag Module:                                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Run built-in diagnostic tests (L1/L2/L3)                          │  │
│  │     - Software env checks                                             │  │
│  │     - Hardware validation                                              │  │
│  │     - Performance benchmarks                                           │  │
│  │     - Memory tests                                                     │  │
│  │     → results[] ← built-in results                                    │  │
│  │                                                                        │  │
│  │  2. Query for extension plugins:                                       │  │
│  │     m_coreProxy.GetPluginsByCapability(XPUM_MODULE_CAP_DIAGNOSTIC)    │  │
│  │                          │                                             │  │
│  │                          ▼                                             │  │
│  │     ModuleManager scans Tier 2 registry:                               │  │
│  │     ┌──────────────────────────────────────────────────────────┐      │  │
│  │     │  "xe2_perf"  cap=DIAGNOSTIC, dev=Xe2 → match? ✓         │      │  │
│  │     │  "oam_mon"   cap=MONITORING          → match? ✗ (wrong) │      │  │
│  │     │  "custom_h"  cap=HEALTH              → match? ✗ (wrong) │      │  │
│  │     └──────────────────────────────────────────────────────────┘      │  │
│  │     Returns: ["xe2_perf"]                                             │  │
│  │                                                                        │  │
│  │  3. For each matching plugin:                                          │  │
│  │     ┌──────────────────────────────────────────────────────────┐      │  │
│  │     │  "xe2_perf" status == NOT_LOADED                         │      │  │
│  │     │  → ModuleManager::LoadPlugin("xe2_perf")                 │      │  │
│  │     │    ├─ dlopen("/usr/lib/xpum/plugins/xe2_perf.so")       │      │  │
│  │     │    ├─ xpum_get_module_descriptor()                       │      │  │
│  │     │    │   apiVersion=100 ≤ XPUM_MODULE_API_VERSION? ✓      │      │  │
│  │     │    │   supportedDevices matches current GPU PCI? ✓       │      │  │
│  │     │    ├─ dlsym (alloc/free/process)                         │      │  │
│  │     │    ├─ allocFunc(&coreCallbacks) → Xe2PerfPlugin*         │      │  │
│  │     │    └─ status = LOADED ✓                                  │      │  │
│  │     │                                                          │      │  │
│  │     │  Send DIAG_RUN command to xe2_perf plugin:               │      │  │
│  │     │  → Xe2PerfPlugin::ProcessMessage(DIAG_RUN)               │      │  │
│  │     │    ├─ m_coreProxy.GetDeviceHandle(deviceId) → ze_dev     │      │  │
│  │     │    ├─ Run Xe2-specific compute kernel tests              │      │  │
│  │     │    ├─ Run Xe2-specific memory bandwidth tests            │      │  │
│  │     │    └─ Return test results via message struct              │      │  │
│  │     │                                                          │      │  │
│  │     │  results[] ← append xe2_perf results                     │      │  │
│  │     └──────────────────────────────────────────────────────────┘      │  │
│  │                                                                        │  │
│  │  4. Aggregate all results:                                             │  │
│  │     results = built-in results + xe2_perf results                     │  │
│  │     Write aggregated results into diagnostic message struct            │  │
│  │     Return to caller                                                   │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### FieldCache Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FieldCache Data Flow                                    │
│                                                                                     │
│  CONSUMERS (modules that read data)          PRODUCER (monitor module)               │
│                                                                                     │
│  ┌──────────┐  ┌──────────┐                 ┌──────────────────────────────┐        │
│  │ Health   │  │ Policy   │                 │  Monitor Module              │        │
│  │ Module   │  │ Module   │                 │                              │        │
│  │          │  │          │                 │  Periodic loop:              │        │
│  │ AddField │  │ AddField │                 │  1. GetActiveWatches()       │        │
│  │ Watch()  │  │ Watch()  │                 │  2. For each watch:          │        │
│  └────┬─────┘  └────┬─────┘                 │     zetSysmanGet*()          │        │
│       │              │                      │  3. StoreSample(deviceId,    │        │
│       ▼              ▼                      │     fieldId, timestamp, val) │        │
│  ┌──────────────────────────────────────┐   └──────────────┬───────────────┘        │
│  │           FieldCache                  │                  │                        │
│  │                                       │                  │                        │
│  │  Watch Registry:                      │                  │                        │
│  │  ┌────────────────────────────────┐  │                  │                        │
│  │  │ deviceId=0, field=GPU_TEMP     │  │   StoreSample()  │                        │
│  │  │   watchers: [HEALTH, POLICY]   │◄─┼──────────────────┘                        │
│  │  │   interval: 1000ms (min)       │  │                                           │
│  │  │   maxAge: 600s, maxSamples: 100│  │                                           │
│  │  ├────────────────────────────────┤  │                                           │
│  │  │ deviceId=0, field=GPU_POWER    │  │                                           │
│  │  │   watchers: [POLICY]           │  │                                           │
│  │  │   interval: 5000ms             │  │                                           │
│  │  │   maxAge: 300s, maxSamples: 60 │  │                                           │
│  │  └────────────────────────────────┘  │                                           │
│  │                                       │                                           │
│  │  Time-Series Storage:                 │                                           │
│  │  ┌────────────────────────────────┐  │  GetLatestSample()                        │
│  │  │ {dev=0, field=GPU_TEMP}        │  │  GetStatistics()                          │
│  │  │   [t1: 72°C] [t2: 73°C] ...   │──┼──────────────────►  Health Module          │
│  │  ├────────────────────────────────┤  │                     Policy Module          │
│  │  │ {dev=0, field=GPU_POWER}       │  │                     Dump Module            │
│  │  │   [t1: 150W] [t2: 148W] ...   │──┼──────────────────►  REST / Prometheus      │
│  │  └────────────────────────────────┘  │                                           │
│  │                                       │                                           │
│  │  Eviction: samples older than maxAge  │                                           │
│  │  or exceeding maxSamples are dropped  │                                           │
│  └───────────────────────────────────────┘                                           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Inter-Module Communication

```
Example: Policy module needs firmware version from Firmware module

  Policy Module                    Core                        Firmware Module
       │                            │                                │
       │  m_coreProxy.SendModule    │                                │
       │  Command(&fwVersionMsg)    │                                │
       │───────────────────────────►│                                │
       │                            │  ModuleManager routes to       │
       │                            │  XPUM_MODULE_ID_FIRMWARE       │
       │                            │───────────────────────────────►│
       │                            │                                │
       │                            │                ProcessMessage()│
       │                            │                  │             │
       │                            │                  ▼             │
       │                            │              query FW version  │
       │                            │              via IGSC library  │
       │                            │                  │             │
       │                            │                  ▼             │
       │                            │          write result into msg │
       │                            │◄───────────────────────────────│
       │                            │  return XPUM_OK                │
       │◄───────────────────────────│                                │
       │  read fwVersionMsg.version │                                │
       │                            │                                │
```

### Complete End-to-End Flow: `xpumcli diagnostic -d 0 -l 3` (with Plugin Discovery)

This is the most complex flow — shows Tier 1 module loading with dependencies AND Tier 2 plugin discovery.

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────┐
│  User    │  │  CLI     │  │  Daemon  │  │  Core Lib    │  │  Diag      │  │ xe2_perf │
│  Terminal│  │ xpumcli  │  │  xpumd   │  │ ModuleMgr    │  │  Module    │  │ Plugin   │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  └─────┬──────┘  └────┬─────┘
     │              │              │               │                │              │
     │ xpumcli      │              │               │                │              │
     │ diag -d 0    │              │               │                │              │
     │ -l 3         │              │               │                │              │
     │─────────────►│              │               │                │              │
     │              │              │               │                │              │
     │              │ CLI11 parse  │               │                │              │
     │              │ ComletDiag   │               │                │              │
     │              │              │               │                │              │
     │              │ ┌──────────────────────────────────────────────────────────┐ │
     │              │ │ DAEMONLESS=OFF: grpc_stub → gRPC → xpumd → core API    │ │
     │              │ │ DAEMONLESS=ON:  core_stub → core API directly           │ │
     │              │ └──────────────────────────────────────────────────────────┘ │
     │              │              │               │                │              │
     │              │ gRPC call    │               │                │              │
     │              │─────────────►│               │                │              │
     │              │              │               │                │              │
     │              │              │ xpumRunDiag() │                │              │
     │              │              │──────────────►│                │              │
     │              │              │               │                │              │
     │              │              │               │ Build msg:     │              │
     │              │              │               │ mod=DIAG(3)    │              │
     │              │              │               │ sub=RUN_DIAG   │              │
     │              │              │               │                │              │
     │              │              │               │ ProcessModule  │              │
     │              │              │               │ Command()      │              │
     │              │              │               │                │              │
     │              │              │               │ ┌────────────────────────┐   │
     │              │              │               │ │ SELECTIVE LOADING:     │   │
     │              │              │               │ │                        │   │
     │              │              │               │ │ DIAG deps=[MONITOR]   │   │
     │              │              │               │ │ MONITOR not loaded    │   │
     │              │              │               │ │ → dlopen(monitor.so)  │   │
     │              │              │               │ │ → alloc MonitorModule │   │
     │              │              │               │ │ → status=LOADED ✓    │   │
     │              │              │               │ │                        │   │
     │              │              │               │ │ Now load DIAG:        │   │
     │              │              │               │ │ → dlopen(diag.so)     │   │
     │              │              │               │ │ → alloc DiagModule    │   │
     │              │              │               │ │ → status=LOADED ✓    │   │
     │              │              │               │ │                        │   │
     │              │              │               │ │ Loaded: 2 of 10       │   │
     │              │              │               │ │ NOT loaded: 8 others  │   │
     │              │              │               │ └────────────────────────┘   │
     │              │              │               │                │              │
     │              │              │               │ Dispatch ─────►│              │
     │              │              │               │                │              │
     │              │              │               │                │ Run built-in │
     │              │              │               │                │ L1/L2/L3     │
     │              │              │               │                │ tests...     │
     │              │              │               │                │              │
     │              │              │               │                │ Now query    │
     │              │              │               │                │ extensions:  │
     │              │              │               │◄───────────────│              │
     │              │              │               │ GetPluginsByCap│              │
     │              │              │               │ (DIAGNOSTIC)   │              │
     │              │              │               │                │              │
     │              │              │               │ ┌────────────────────────┐   │
     │              │              │               │ │ PLUGIN DISCOVERY:      │   │
     │              │              │               │ │                        │   │
     │              │              │               │ │ Tier 2 registry:       │   │
     │              │              │               │ │ xe2_perf: DIAGNOSTIC ✓│   │
     │              │              │               │ │ oam_mon:  MONITORING ✗│   │
     │              │              │               │ │                        │   │
     │              │              │               │ │ Check device match:    │   │
     │              │              │               │ │ GPU PCI ID ∈ Xe2? ✓  │   │
     │              │              │               │ │                        │   │
     │              │              │               │ │ Load xe2_perf:         │   │
     │              │              │               │ │ → dlopen(xe2_perf.so) │   │
     │              │              │               │ │ → descriptor check ✓  │   │
     │              │              │               │ │ → alloc Xe2PerfPlugin │   │
     │              │              │               │ │ → status=LOADED ✓    │   │
     │              │              │               │ └────────────────────────┘   │
     │              │              │               │                │              │
     │              │              │               │ Return plugin ►│              │
     │              │              │               │ handle         │              │
     │              │              │               │                │              │
     │              │              │               │                │ Send DIAG_RUN│
     │              │              │               │                │─────────────►│
     │              │              │               │                │              │
     │              │              │               │                │  CoreProxy   │
     │              │              │               │◄───────────────│◄──GetDevice  │
     │              │              │               │                │   Handle()   │
     │              │              │               │───────────────►│──►ze_dev     │
     │              │              │               │                │              │
     │              │              │               │                │  Run Xe2     │
     │              │              │               │                │  specific    │
     │              │              │               │                │  benchmarks  │
     │              │              │               │                │              │
     │              │              │               │                │◄─────────────│
     │              │              │               │                │ xe2 results  │
     │              │              │               │                │              │
     │              │              │               │                │ Aggregate:   │
     │              │              │               │                │ built-in +   │
     │              │              │               │                │ xe2_perf     │
     │              │              │               │                │ results      │
     │              │              │               │                │              │
     │              │              │               │◄───────────────│              │
     │              │              │               │ diag_result_t  │              │
     │              │              │               │                │              │
     │              │              │◄──────────────│                │              │
     │              │              │ result        │                │              │
     │              │◄─────────────│               │                │              │
     │              │ gRPC resp    │               │                │              │
     │              │              │               │                │              │
     │  ┌─────────────────────────────────────────────────────────────────────┐  │
     │  │ Device 0 — Diagnostic Level 3                                       │  │
     │  │ +──────────────────────────────+──────────+──────────────────────+  │  │
     │  │ | Test                         | Status   | Details              |  │  │
     │  │ +──────────────────────────────+──────────+──────────────────────+  │  │
     │  │ | Software Environment         | PASS     | All libs present     |  │  │
     │  │ | Hardware Validation          | PASS     | PCIe Gen4 x16       |  │  │
     │  │ | Computation (built-in)       | PASS     | 95.2% of reference  |  │  │
     │  │ | Memory Bandwidth (built-in)  | PASS     | 512 GB/s            |  │  │
     │  │ | Xe2 Compute (plugin)         | PASS     | 98.1% of Xe2 ref   |  │  │
     │  │ | Xe2 EU Occupancy (plugin)    | PASS     | 94.3% occupancy     |  │  │
     │  │ +──────────────────────────────+──────────+──────────────────────+  │  │
     │  │ Overall: PASS (6/6 tests passed — 4 built-in + 2 from xe2-perf)   │  │
     │  └─────────────────────────────────────────────────────────────────────┘  │
     │◄─────────────│              │               │                │              │
     │  table output│              │               │                │              │
```

### Complete End-to-End Flow: `xpumcli firmware -u -d 0 -f image.bin` (Minimal Loading)

This shows the opposite extreme — a command that needs only 1 module with no dependencies.

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────┐
│  User    │  │  CLI     │  │  Daemon  │  │  Core Lib    │  │ Firmware   │
│  Terminal│  │ xpumcli  │  │  xpumd   │  │ ModuleMgr    │  │ Module     │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  └─────┬──────┘
     │              │              │               │                │
     │ xpumcli      │              │               │                │
     │ firmware -u   │              │               │                │
     │ -d 0 -f img   │              │               │                │
     │──────────────►│              │               │                │
     │              │              │               │                │
     │              │ gRPC call    │               │                │
     │              │─────────────►│               │                │
     │              │              │               │                │
     │              │              │ xpumFirmware  │                │
     │              │              │ Flash()       │                │
     │              │              │──────────────►│                │
     │              │              │               │                │
     │              │              │               │ Build msg:     │
     │              │              │               │ mod=FIRMWARE(4)│
     │              │              │               │                │
     │              │              │               │ FIRMWARE deps=[]│
     │              │              │               │ (no deps!)     │
     │              │              │               │                │
     │              │              │               │ dlopen ONLY    │
     │              │              │               │ firmware.so    │
     │              │              │               │                │
     │              │              │               │ Loaded: 1 of 10│
     │              │              │               │                │
     │              │              │               │───────────────►│
     │              │              │               │                │
     │              │              │               │      IGSC flash│
     │              │              │               │      FW image  │
     │              │              │               │         ...    │
     │              │              │               │                │
     │              │              │               │◄───────────────│
     │              │              │◄──────────────│ result         │
     │              │◄─────────────│               │                │
     │◄─────────────│              │               │                │
     │ "FW update   │              │               │                │
     │  success"    │              │               │                │
     │              │              │               │                │

     Memory: Core (~3MB) + Firmware module (~1MB) = ~4MB total
     NOT loaded: Monitor, Health, Diag, Policy, Config, Topology,
                 vGPU, Group, Dump, ALL Tier 2 plugins
```

### Module Dependency Graph (with Selective Loading + Two Tiers)

```
                              ┌────────────────────────────────────────┐
                              │         Core Services                   │
                              │         (ALWAYS resident in memory)     │
                              │                                        │
                              │  DeviceManager   FieldCache            │
                              │  ModuleManager   ScheduledThreadPool   │
                              │  Configuration                         │
                              └──────────────────┬─────────────────────┘
                                                 │
                  ┌──────────────────────────────┼──────────────────────────────┐
                  │                              │                              │
                  │  TIER 1: Built-in Modules    │  (loaded selectively)        │
                  │                              │                              │
     ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ Layer 0: No deps ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
                  │                              │                              │
          ┌───────▼──────┐ ┌──────▼───────┐ ┌───▼────────┐ ┌──────────┐ ┌──────▼──────┐
          │   Monitor    │ │    Group      │ │   Config   │ │ Firmware │ │  Topology   │
          │   (ID=1)     │ │    (ID=9)     │ │   (ID=6)   │ │ (ID=4)   │ │  (ID=7)     │
          │              │ │              │ │            │ │          │ │             │
          │ deps: []     │ │ deps: []     │ │ deps: []   │ │ deps: [] │ │ deps: []    │
          │              │ │              │ │            │ │          │ │             │
          │ Feeds        │ │ Device       │ │ Power/Freq │ │ IGSC/    │ │ hwloc       │
          │ FieldCache   │ │ grouping     │ │ ECC ctl    │ │ IPMI/    │ │ topology    │
          │ via L0 API   │ │              │ │            │ │ Redfish  │ │             │
          └──────┬───────┘ └──────┬───────┘ └────────────┘ └──────────┘ └─────────────┘
                 │                │
                 │                │ ┌────────────┐
                 │                │ │   vGPU     │
                 │                │ │   (ID=8)   │
                 │                │ │ deps: []   │
                 │                │ │ SR-IOV     │
                 │                │ └────────────┘
                 │                │
     ─ ─ ─ ─ ─ ─┼─ ─ ─ Layer 1: ┼Depend on Monitor/Group ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
                 │                │                                              │
          ┌──────▼──────┐  ┌──────▼──────┐  ┌──────────────┐  ┌──────────────┐ │
          │   Health    │  │  Policy     │  │   Dump       │  │  Diagnostic  │ │
          │   (ID=2)    │  │  (ID=5)     │  │   (ID=10)    │  │  (ID=3)      │ │
          │             │  │             │  │              │  │              │ │
          │ deps:       │  │ deps:       │  │ deps:        │  │ deps:        │ │
          │ [MONITOR]   │  │ [MONITOR,   │  │ [MONITOR]    │  │ [MONITOR]    │ │
          │             │  │  GROUP]     │  │              │  │              │ │
          │ Reads       │  │ Reads       │  │ Exports raw  │  │ L1/L2/L3     │ │
          │ FieldCache  │  │ FieldCache  │  │ FieldCache   │  │ tests        │ │
          │ + thresholds│  │ + group info│  │ data         │  │              │ │
          └─────────────┘  └─────────────┘  └──────────────┘  └──────┬───────┘ │
                                                                      │         │
     ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ┤
                                                                      │         │
                  │  TIER 2: Extension Plugins  (drop-in discovery)   │         │
                  │                                                    │         │
          ┌───────────────────┐  ┌───────────────────┐  ┌─────────────▼───────┐ │
          │  oam_monitor      │  │  custom_health    │  │  xe2_perf           │ │
          │  (ID=34, dynamic) │  │  (ID=33, dynamic) │  │  (ID=32, dynamic)   │ │
          │                   │  │                   │  │                     │ │
          │  cap: MONITORING  │  │  cap: HEALTH      │  │  cap: DIAGNOSTIC    │ │
          │  dev: OAM boards  │  │  dev: all         │  │  dev: Xe2 PCI IDs   │ │
          │  apiVer: 110      │  │  apiVer: 100      │  │  apiVer: 100        │ │
          │                   │  │                   │  │                     │ │
          │  Loaded by:       │  │  Loaded by:       │  │  Loaded by:         │ │
          │  Monitor module   │  │  Health module    │  │  Diag module        │ │
          │  via GetPlugins   │  │  via GetPlugins   │  │  via GetPlugins     │ │
          │  ByCap(MONITOR)   │  │  ByCap(HEALTH)    │  │  ByCap(DIAGNOSTIC)  │ │
          └───────────────────┘  └───────────────────┘  └─────────────────────┘ │
                  │                                                              │
                  └──────────────────────────────────────────────────────────────┘

Legend:
  ───► hard dependency (auto-loaded by ModuleManager)
  - - - layer boundary (modules above have no module deps)
  Tier 1 modules: known IDs, loaded on first API call
  Tier 2 plugins: dynamic IDs, loaded when parent module queries by capability
  Core Services: always in memory, never unloaded
  Everything else: loaded ONLY when needed, can be unloaded
```

### Selective Loading Visualization Over Time

```
Time    Event                         Modules in Memory
─────   ──────────────────────────    ─────────────────────────────────────────
t=0     xpumInit()                    [Core Services only]
        DiscoverPlugins()             (scanned 3 plugins, registered, NOT loaded)

t=1     xpumcli discovery             [Core Services]
        xpumGetDeviceList()           (handled by DeviceManager, no module needed)

t=2     xpumcli stats -d 0            [Core Services] + [MONITOR]
        xpumGetStats()                (first call → load MONITOR, no deps)

t=3     xpumcli health -d 0           [Core Services] + [MONITOR] + [HEALTH]
        xpumGetHealth()               (MONITOR already loaded, load HEALTH)

t=4     xpumcli diag -d 0 -l 3        [Core + MONITOR + HEALTH] + [DIAG] + [xe2_perf]
        xpumRunDiagnostics()          (MONITOR already loaded, load DIAG,
                                       DIAG queries CAP_DIAGNOSTIC → xe2_perf loaded)

t=5     xpumcli firmware -u -f img    [Core + MONITOR + HEALTH + DIAG + xe2_perf]
        xpumFirmwareFlash()                                       + [FIRMWARE]
                                      (no deps, just load FIRMWARE)

        NEVER loaded: POLICY, CONFIG, TOPOLOGY, VGPU, GROUP, DUMP,
                      custom_health, oam_monitor
        → Memory savings: only 5 of 10 built-in + 1 of 3 plugins loaded
```

---

## Module Interface Definition

### Module Base Class

```cpp
// File: core/include/xpum_module.h

namespace xpum {

/**
 * Base class for all XPU Manager modules.
 * Each module is a dynamically-loaded .so that processes versioned messages.
 *
 * Lifecycle:
 *   1. dlopen("libxpum_mod_xxx.so")
 *   2. xpum_alloc_module_instance(&coreCallbacks) → XpumModule*
 *   3. ProcessMessage(cmd) called for each request
 *   4. xpum_free_module_instance(ptr) on shutdown
 *   5. dlclose()
 */
class XpumModule {
public:
    virtual ~XpumModule() = default;

    /**
     * Process a module command.
     *   - header.moduleId  → which module
     *   - header.subCommand → which operation
     *   - header.version   → struct version for compatibility
     *   - payload          → specific to the subCommand
     */
    virtual xpum_result_t ProcessMessage(
        xpum_module_command_header_t* moduleCommand) = 0;

    /** Version check helper */
    static xpum_result_t CheckVersion(
        const xpum_module_command_header_t* cmd,
        unsigned int expectedVersion);
};

/**
 * Template base providing CoreProxy access.
 * Parameterized on moduleId for scoped logging.
 */
template <unsigned int moduleId>
class XpumModuleWithCoreProxy : public XpumModule {
public:
    explicit XpumModuleWithCoreProxy(const xpum_core_callbacks_t& cb)
        : m_coreCallbacks(cb), m_coreProxy(cb) {}

protected:
    xpum_core_callbacks_t m_coreCallbacks;
    XpumCoreProxy m_coreProxy;
};

} // namespace xpum
```

### C Factory Functions (Module Exports)

Every module `.so` exports three standard C functions:

```cpp
extern "C" {
    // Create module instance, receiving core callbacks
    xpum::XpumModule* xpum_alloc_module_instance(
        xpum::xpum_core_callbacks_t* coreCallbacks);

    // Destroy module instance
    void xpum_free_module_instance(xpum::XpumModule* module);

    // Process a message (convenience wrapper)
    xpum_result_t xpum_module_process_message(
        xpum::XpumModule* module,
        xpum::xpum_module_command_header_t* cmd);
}
```

### Export Macro

```cpp
#define XPUM_EXPORT_MODULE(ModuleClass, ModuleIdEnum)                         \
extern "C" {                                                                   \
    xpum::XpumModule* xpum_alloc_module_instance(                             \
            xpum::xpum_core_callbacks_t* coreCallbacks) {                     \
        try { return new ModuleClass(*coreCallbacks); }                       \
        catch (...) { return nullptr; }                                        \
    }                                                                          \
    void xpum_free_module_instance(xpum::XpumModule* module) {                \
        delete module;                                                         \
    }                                                                          \
    xpum_result_t xpum_module_process_message(                                \
            xpum::XpumModule* module,                                         \
            xpum::xpum_module_command_header_t* cmd) {                        \
        if (!module || !cmd) return XPUM_GENERIC_ERROR;                       \
        try { return module->ProcessMessage(cmd); }                           \
        catch (...) { return XPUM_GENERIC_ERROR; }                            \
    }                                                                          \
}
```

---

## Message-Based Communication

### Module Command Header

```cpp
// File: core/include/xpum_module_structs.h

namespace xpum {

/** Module identifiers */
typedef enum {
    XPUM_MODULE_ID_CORE      = 0,   // Always resident in core
    XPUM_MODULE_ID_MONITOR   = 1,   // Telemetry collection
    XPUM_MODULE_ID_HEALTH    = 2,   // Health monitoring
    XPUM_MODULE_ID_DIAG      = 3,   // Diagnostics
    XPUM_MODULE_ID_FIRMWARE  = 4,   // Firmware update
    XPUM_MODULE_ID_POLICY    = 5,   // Policy engine
    XPUM_MODULE_ID_CONFIG    = 6,   // GPU configuration
    XPUM_MODULE_ID_TOPOLOGY  = 7,   // Hardware topology
    XPUM_MODULE_ID_VGPU      = 8,   // Virtual GPU / SR-IOV
    XPUM_MODULE_ID_GROUP     = 9,   // Device grouping
    XPUM_MODULE_ID_DUMP      = 10,  // Raw data dump
    XPUM_MODULE_ID_COUNT
} xpum_module_id_t;

/**
 * Module command header. Every module message starts with this.
 * Versioned for forward/backward compatibility.
 */
typedef struct {
    unsigned int length;          // Total length of enclosing struct
    xpum_module_id_t moduleId;   // Target module
    unsigned int subCommand;      // Module-specific operation
    unsigned int version;         // Version of this sub-command struct
    unsigned int requestId;       // Optional request tracking ID
} xpum_module_command_header_t;

/** Version macro: encodes struct size + version number */
#define XPUM_MAKE_VERSION(structType, ver) \
    ((unsigned int)(sizeof(structType) | ((unsigned int)(ver) << 24)))

} // namespace xpum
```

### Example: Health Module Messages

```cpp
// File: modules/health/xpum_health_structs.h

// Sub-commands
#define XPUM_HEALTH_SR_SET_CONFIG    1
#define XPUM_HEALTH_SR_GET_CONFIG    2
#define XPUM_HEALTH_SR_GET_STATUS    3

// Set health config message
typedef struct {
    xpum_module_command_header_t header;
    xpum_device_id_t deviceId;              // IN
    xpum_health_config_type_t configKey;    // IN
    uint64_t configValue;                   // IN
    xpum_result_t result;                   // OUT
} xpum_health_msg_set_config_v1;

#define xpum_health_msg_set_config_version1 \
    XPUM_MAKE_VERSION(xpum_health_msg_set_config_v1, 1)

// Get health status message
typedef struct {
    xpum_module_command_header_t header;
    xpum_device_id_t deviceId;              // IN
    xpum_health_type_t healthType;          // IN
    xpum_health_data_t healthData;          // OUT
    xpum_result_t result;                   // OUT
} xpum_health_msg_get_status_v1;

#define xpum_health_msg_get_status_version1 \
    XPUM_MAKE_VERSION(xpum_health_msg_get_status_v1, 1)
```

---

## CoreProxy / CoreServices

### Core Callbacks (Passed to Modules)

```cpp
// File: core/include/xpum_core_communication.h

namespace xpum {

/** Post a request from module to core */
typedef xpum_result_t (*xpum_core_post_request_f)(
    xpum_module_command_header_t* request, void* coreContext);

/** Logging from module to core's spdlog */
typedef void (*xpum_core_logger_f)(
    int level, const char* message, void* coreContext);

/** Callbacks passed from core to each module at construction time */
typedef struct {
    unsigned int version;
    xpum_core_post_request_f postFunc;
    void* coreContext;
    xpum_core_logger_f loggerFunc;
} xpum_core_callbacks_t;

/** Core request IDs (operations modules can ask core to perform) */
typedef enum {
    XPUM_CORE_REQ_GET_DEVICE_LIST          = 0,
    XPUM_CORE_REQ_GET_DEVICE_PROPERTIES    = 1,
    XPUM_CORE_REQ_GET_DEVICE_HANDLE        = 2,
    XPUM_CORE_REQ_ADD_FIELD_WATCH          = 3,
    XPUM_CORE_REQ_REMOVE_FIELD_WATCH       = 4,
    XPUM_CORE_REQ_GET_LATEST_FIELD_VALUE   = 5,
    XPUM_CORE_REQ_GET_FIELD_STATISTICS     = 6,
    XPUM_CORE_REQ_STORE_FIELD_VALUE        = 7,
    XPUM_CORE_REQ_SEND_MODULE_COMMAND      = 8,
    XPUM_CORE_REQ_GET_GROUP_DEVICES        = 9,
    XPUM_CORE_REQ_GET_DRIVER_HANDLE        = 10,
    XPUM_CORE_REQ_COUNT
} xpum_core_req_id_t;

} // namespace xpum
```

### CoreProxy Class

```cpp
// File: modules/common/xpum_core_proxy.h

namespace xpum {

/**
 * Proxy that modules use to access core services.
 * All calls routed through core callbacks (function pointers),
 * so modules never directly link against core internals.
 *
 * Mirrors DCGM's DcgmCoreProxy pattern.
 */
class XpumCoreProxy {
public:
    explicit XpumCoreProxy(xpum_core_callbacks_t coreCallbacks);

    // --- Device Management ---
    xpum_result_t GetDeviceList(std::vector<xpum_device_id_t>& deviceIds);
    xpum_result_t GetDeviceProperties(xpum_device_id_t deviceId,
                                       xpum_device_properties_t* props);
    xpum_result_t GetDeviceHandle(xpum_device_id_t deviceId,
                                   ze_device_handle_t* zeDevice,
                                   zes_device_handle_t* zesDevice);
    ze_driver_handle_t GetDriverHandle();

    // --- Field Cache (Metric Data) ---
    xpum_result_t AddFieldWatch(xpum_device_id_t deviceId,
                                 MeasurementType fieldId,
                                 uint64_t monitorIntervalUs,
                                 double maxSampleAge,
                                 int maxKeepSamples);
    xpum_result_t RemoveFieldWatch(xpum_device_id_t deviceId,
                                    MeasurementType fieldId);
    xpum_result_t GetLatestFieldValue(xpum_device_id_t deviceId,
                                       MeasurementType fieldId,
                                       xpum_device_stats_data_t* value);
    xpum_result_t GetFieldStatistics(xpum_device_id_t deviceId,
                                      MeasurementType fieldId,
                                      uint64_t startTime, uint64_t endTime,
                                      xpum_device_stats_data_t* stats);

    // --- Inter-Module Communication ---
    xpum_result_t SendModuleCommand(xpum_module_command_header_t* cmd);

    // --- Logging ---
    void LogInfo(const char* fmt, ...);
    void LogWarning(const char* fmt, ...);
    void LogError(const char* fmt, ...);

    // --- Utility ---
    bool IsDaemonMode();
    const char* GetModulePath();

private:
    xpum_core_callbacks_t m_coreCallbacks;
};

} // namespace xpum
```

---

## Module Lifecycle

```
                    ModuleManager                          Module .so
                         │                                     │
    ─── Core::init() ──►│                                     │
                         │  1. Register module filenames       │
                         │     m_modules[HEALTH].filename =   │
                         │       "libxpum_mod_health.so"       │
                         │                                     │
    ─── API call ──────►│                                     │
                         │  2. Is module loaded?               │
                         │     No → LoadModule(moduleId)       │
                         │                                     │
                         │  3. dlopen(filename)     ─────────►│
                         │                                     │
                         │  4. dlsym("xpum_alloc_module_      │
                         │           instance")                │
                         │     dlsym("xpum_free_module_       │
                         │           instance")                │
                         │     dlsym("xpum_module_process_    │
                         │           message")                 │
                         │                                     │
                         │  5. allocCB(&coreCallbacks) ──────►│ constructor
                         │                             ◄──────│ XpumModule*
                         │                                     │
                         │  6. Status = LOADED                 │
                         │                                     │
                         │  7. Route message ────────────────►│ ProcessMessage()
                         │                             ◄──────│ xpum_result_t
                         │                                     │
    ─── Core::close() ─►│                                     │
                         │  8. freeCB(module)    ─────────────►│ destructor
                         │  9. dlclose(handle)                 │
                         │                                     │
```

Key design decisions:
- **On-demand loading only**: Modules are loaded only when explicitly needed — never all at once
- **Thread-safe loading**: Double-checked locking prevents concurrent dlopen of the same module
- **Fail-safe**: If a module `.so` is missing, API calls return `XPUM_MODULE_NOT_LOADED`
- **Unloadable**: Modules that are no longer needed can be unloaded to reclaim memory

---

## Selective Module Loading

Like DCGM, the framework **only loads the modules actually needed** for the current operation. This is a core design principle — not all modules are loaded at startup.

### Loading Strategy

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Module Loading Strategy                                   │
│                                                                              │
│  At startup (Core::init):                                                    │
│    - NO modules are loaded                                                   │
│    - Only Core Services are initialized (DeviceManager, FieldCache, etc.)    │
│    - ModuleManager registers module filenames but does NOT dlopen anything   │
│                                                                              │
│  On first API call to a feature:                                             │
│    xpumGetHealth() → needs HEALTH module → load ONLY health module           │
│    xpumRunDiag()   → needs DIAG module   → load ONLY diag module            │
│    xpumGetStats()  → needs MONITOR module → load ONLY monitor module         │
│                                                                              │
│  Dependency-driven loading:                                                  │
│    Health module needs FieldCache data → triggers MONITOR module load         │
│    Policy module needs group info    → triggers GROUP module load             │
│    Diagnostic needs device handles   → uses Core DeviceManager (no module)   │
│                                                                              │
│  Result: Only the modules required for the user's actual commands            │
│  are ever loaded into memory.                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

### ModuleManager with Selective Loading

```cpp
// File: core/src/core/module_manager.h

class ModuleManager {
public:
    ModuleManager();
    ~ModuleManager();

    /** Initialize — registers module metadata but loads NOTHING */
    xpum_result_t Init(const xpum_core_callbacks_t& coreCallbacks);

    /** Route a command to the appropriate module, loading it if needed */
    xpum_result_t ProcessModuleCommand(xpum_module_command_header_t* cmd);

    /** Explicitly load a specific module (called on-demand) */
    xpum_result_t LoadModule(xpum_module_id_t moduleId);

    /** Unload a specific module to free resources */
    xpum_result_t UnloadModule(xpum_module_id_t moduleId);

    /** Check if a module is currently loaded */
    bool IsModuleLoaded(xpum_module_id_t moduleId) const;

    /** Get list of currently loaded modules (for diagnostics/logging) */
    std::vector<xpum_module_id_t> GetLoadedModules() const;

    /** Shutdown — unloads all loaded modules */
    void Shutdown();

private:
    enum class ModuleStatus {
        NOT_LOADED,     // Registered but not loaded
        LOADING,        // Currently being loaded (thread safety)
        LOADED,         // Successfully loaded and ready
        FAILED,         // Load attempt failed (will not retry)
        UNLOADED        // Was loaded, now unloaded
    };

    struct ModuleEntry {
        std::string filename;               // e.g., "libxpum_mod_health.so"
        ModuleStatus status = ModuleStatus::NOT_LOADED;
        void* dlHandle = nullptr;           // dlopen handle
        XpumModule* instance = nullptr;     // Module instance
        xpum_module_alloc_f allocFunc = nullptr;
        xpum_module_free_f freeFunc = nullptr;
        xpum_module_process_message_f processFunc = nullptr;
        std::vector<xpum_module_id_t> dependencies;  // Modules this one needs
        mutable std::mutex mutex;           // Per-module lock
    };

    ModuleEntry m_modules[XPUM_MODULE_ID_COUNT];
    xpum_core_callbacks_t m_coreCallbacks;

    /** Load a module and all its dependencies recursively */
    xpum_result_t LoadModuleWithDependencies(xpum_module_id_t moduleId);
};
```

### Module Dependency Declaration

Each module declares its dependencies so the framework can auto-load required modules:

```cpp
// File: core/include/xpum_module.h (addition to base class)

class XpumModule {
public:
    // ...existing methods...

    /**
     * Return the list of module IDs this module depends on.
     * Called after construction. The framework will ensure
     * these modules are loaded before routing messages.
     */
    virtual std::vector<xpum_module_id_t> GetDependencies() const {
        return {};  // Default: no dependencies
    }
};
```

```cpp
// Example: Health module declares dependency on Monitor
class XpumModuleHealth : public XpumModuleWithCoreProxy<XPUM_MODULE_ID_HEALTH> {
public:
    std::vector<xpum_module_id_t> GetDependencies() const override {
        return { XPUM_MODULE_ID_MONITOR };  // Needs FieldCache data from Monitor
    }
    // ...
};

// Example: Firmware module has no module dependencies
class XpumModuleFirmware : public XpumModuleWithCoreProxy<XPUM_MODULE_ID_FIRMWARE> {
public:
    std::vector<xpum_module_id_t> GetDependencies() const override {
        return {};  // Only uses Core DeviceManager (always available)
    }
    // ...
};
```

### Loading Scenarios

```
Scenario 1: User runs "xpumcli health -d 0"
─────────────────────────────────────────────
  Startup:  0 modules loaded
  API call: xpumGetHealth()
    → ModuleManager: need HEALTH module
    → LoadModule(HEALTH)
      → Health declares dependency: [MONITOR]
      → LoadModule(MONITOR)   ← auto-loaded because Health needs it
      → LoadModule(HEALTH)    ← now load Health
  Result:   2 modules loaded (Monitor + Health)
  NOT loaded: Diag, Firmware, Policy, Config, Topology, vGPU, Group, Dump


Scenario 2: User runs "xpumcli firmware -d 0 -t GFX -f image.bin"
───────────────────────────────────────────────────────────────────
  Startup:  0 modules loaded
  API call: xpumFirmwareFlash()
    → ModuleManager: need FIRMWARE module
    → LoadModule(FIRMWARE)
      → Firmware declares dependency: []  ← no module dependencies
      → LoadModule(FIRMWARE)
  Result:   1 module loaded (Firmware)
  NOT loaded: Monitor, Health, Diag, Policy, Config, Topology, vGPU, Group, Dump


Scenario 3: User runs "xpumcli diagnostic -d 0 -l 3"
─────────────────────────────────────────────────────
  Startup:  0 modules loaded
  API call: xpumRunDiagnostics()
    → ModuleManager: need DIAG module
    → LoadModule(DIAG)
      → Diag declares dependency: [MONITOR]
      → LoadModule(MONITOR)
      → LoadModule(DIAG)
  Result:   2 modules loaded (Monitor + Diag)
  NOT loaded: Health, Firmware, Policy, Config, Topology, vGPU, Group, Dump


Scenario 4: Full daemon mode (xpumd) with all features active
──────────────────────────────────────────────────────────────
  Over time, as different CLI/REST requests arrive, modules
  are loaded incrementally:

  t=0   [startup]     0 modules loaded
  t=1   GET /stats  → Monitor loaded                    (1 total)
  t=2   GET /health → Health loaded (Monitor already)   (2 total)
  t=3   POST /diag  → Diag loaded (Monitor already)     (3 total)
  t=4   POST /fw    → Firmware loaded                    (4 total)
  ...

  Modules that are never requested are never loaded.
```

### Configuration: Module Allowlist/Blocklist

For deployments that want to restrict which modules can be loaded:

```cpp
// File: core/include/xpum_core_communication.h (addition)

/** Optional module loading configuration */
typedef struct {
    bool enabledModules[XPUM_MODULE_ID_COUNT];  // true = allowed to load
} xpum_module_config_t;

// Default: all modules enabled
// Can be configured via:
//   - Environment variable: XPUM_ENABLED_MODULES="monitor,health,diag"
//   - Config file: /etc/xpum/modules.conf
//   - API call: xpumSetModuleConfig()
```

```
Example: Monitoring-only deployment
  XPUM_ENABLED_MODULES="monitor,health,dump"

  → Only Monitor, Health, and Dump modules can be loaded
  → xpumRunDiag() returns XPUM_MODULE_NOT_LOADED
  → xpumFirmwareFlash() returns XPUM_MODULE_NOT_LOADED
  → Smaller memory footprint, reduced attack surface
```

### Module Unloading

Unlike DCGM (which keeps modules loaded for the process lifetime), XPU Manager supports explicit module unloading for long-running daemon scenarios:

```cpp
// Unload a module that is no longer needed
xpum_result_t ModuleManager::UnloadModule(xpum_module_id_t moduleId) {
    // 1. Check no other loaded module depends on this one
    // 2. Call freeFunc(instance) → destructor
    // 3. dlclose(dlHandle)
    // 4. Set status = UNLOADED
    // Can be re-loaded later if needed (status goes back to NOT_LOADED)
}

// Use case: After a firmware update completes, unload firmware module
// to free IGSC library resources
```

### Memory Comparison: Monolithic vs Selective Loading

```
Current (monolithic libxpum.so):
  ┌──────────────────────────────────────────────────────┐
  │ libxpum.so loaded entirely at startup                 │
  │ ~15 MB resident: ALL managers, ALL diagnostic code,   │
  │ ALL firmware handlers, ALL policy logic, etc.         │
  │                                                       │
  │ Even "xpumcli discovery" loads everything.            │
  └──────────────────────────────────────────────────────┘

Proposed (selective module loading):
  "xpumcli discovery" → loads Core only (~3 MB)
  ┌──────────────┐
  │ libxpum_core │
  └──────────────┘

  "xpumcli health -d 0" → loads Core + Monitor + Health (~5 MB)
  ┌──────────────┐ ┌─────────┐ ┌────────┐
  │ libxpum_core │ │ monitor │ │ health │
  └──────────────┘ └─────────┘ └────────┘

  "xpumcli diagnostic -d 0 -l 3" → loads Core + Monitor + Diag (~8 MB)
  ┌──────────────┐ ┌─────────┐ ┌──────┐
  │ libxpum_core │ │ monitor │ │ diag │
  └──────────────┘ └─────────┘ └──────┘

  Full daemon (all features over time) → same as monolithic but incremental
  ┌──────────────┐ ┌─────────┐ ┌────────┐ ┌──────┐ ┌──────────┐ ...
  │ libxpum_core │ │ monitor │ │ health │ │ diag │ │ firmware │
  └──────────────┘ └─────────┘ └────────┘ └──────┘ └──────────┘
```

---

## Complete Functionality Mapping

### All XPU Manager Functions → Module Mapping

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              API Layer (xpum_api.h)                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ xpumInit()           xpumGetDeviceList()     xpumGetStats()        xpumGetHealth()     │
│  │   ↓                    ↓                     ↓                    ↓                    │  │
│  │ Core::init()        DeviceManager        MONITOR (1)          HEALTH (2)         │  │
│  │   ↓                   ↓                    ↓                    ↓                    │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │  │                          Core Services                                │  │
│  │  │  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  │  │  DeviceManager    FieldCache    Configuration      ScheduledThread   │  │
│  │  │  │  (always resident)  (time-series)   (global cfg)     Pool             │  │
│  │  │  │  ┌──────┬───────┐   ┌─────┬──────┐   ┌──────┬──────┐  │  │
│  │  │  │  │Device handles│   │Metrics     │   │Env vars  │   │Timer/      │  │
│  │  │  │  │  & PCI info   │   │storage    │   │  │         │   │Background │
│  │  │  │  └───┬───────┘   └─────┬──────┘   └─────┬──────┘   │  │
│  │  │  │      │              │              │              │   │         │   │
│  │  └──┬───▼──────┴─────────▼───────▼───────▼───────▼───────┘   │  │
│  │     │              │              │              │              │   │         │   │
│  │     ↓              ↓              ↓              ↓              ↓   ↓         │   ↓
│  │  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  │              Tier 1: Built-in Modules              │
│  │  │  MONITOR(0) HEALTH(1) DIAG(2) FIRMWARE(3) CONFIG(4) TOPOLOGY(5) GROUP(6) POLICY(7) DUMP(8) VGPU(9) │  │
│  │  │  └─────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬──│  │
│  │     │     │    │    │    │    │    │    │    │    │    │    │    │    │    │
│  │     │     ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    │  │
│  │     │Collects Thresholds  Runs  Updates  Power/ Config  Hardware  Device   Auto   Raw    SR-IOV │  │
│  │     │metrics →   health →   tests   →   GFX/AMC →   ECC    →   topology →   rules   →   data   →   func │  │
│  │     │→       → checks →   →      →   →       →      →   →      →   →      →   │  │
│  │     │FieldCache│ FieldCache│ FieldCache│ FieldCache│ Config  │ FieldCache│ FieldCache│ FieldCache│ FieldCache│ FieldCache│ FieldCache│ FieldCache│ │  │
│  │  └─────┬───────┴───────┴───────┴───────┴───────┴───────┴───────┴───────┴─────┘  │  │
│  │                                                                   │  │
│  │  ┌───────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │              Tier 2: Extension Plugins (optional, drop-in)       │  │
│  │  │                                                               │  │
│  │  │  Potential extensions (auto-discovered):                         │  │
│  │  │  • amc-perf.so — Advanced AMC monitoring beyond firmware module │  │
│  │  │  • oam-monitor.so — OEM OAM telemetry                        │  │
│  │  │  • fabric-topology.so — Custom fabric analysis                    │  │
│  │  │  • power-model.so — Device-specific power modeling               │  │
│  │  │  • custom-health.so — Vendor-specific health rules               │  │
│  │  │  • ddrt-diag.so — DDRT library diagnostic tests                │  │
│  │  │  • rapl-plugin.so — RAPL power measurement                   │  │
│  │  │  • profiler.so — GPU sampling/profiling                      │  │
│  │  │  • telemetry-exporter.so — Custom metric export formats           │  │
│  │  └───────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                   │  │
└───────────────────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ModuleManager                                            │
│                                                                       │
│  Routes API calls to correct module based on function:           │
│  xpumGetStats()          → MONITOR module (ID=1)                         │
│  xpumGetHealth()          → HEALTH module (ID=2)                         │
│  xpumRunDiagnostics()   → DIAG module (ID=3) + discover Tier 2 plugins      │
│  xpumFlashFirmware()     → FIRMWARE module (ID=4)                        │
│  xpumSetPowerLimit()     → CONFIG module (ID=5)                        │
│  xpumSetFrequency()      → CONFIG module (ID=5)                        │
│  xpumSetECCState()       → CONFIG module (ID=5)                        │
│  xpumGetTopology()       → TOPOLOGY module (ID=6)                       │
│  xpumGroupCreate()        → GROUP module (ID=7)                         │
│  xpumPolicyCreate()      → POLICY module (ID=8)                         │
│  xpumDumpRawData()       → DUMP module (ID=9)                         │
│  xpumCreateVirtualGPU()  → VGPU module (ID=10)                        │
│  xpumListPlugins()       → ModuleManager::GetRegisteredPlugins() (metadata)   │
│                                                                       │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Per-Functionality Flow Examples

#### Device Discovery (No Module)
```
xpumcli discovery -d 0
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumGetDeviceList()      → Direct to DeviceManager                │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  DeviceManager::getDeviceList()                │  │
│  │    → zesDeviceEnum() via Level Zero               │  │
│  │    → Returns: std::vector<std::shared_ptr<Device>>   │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Statistics/Telemetry (Monitor Module)
```
xpumcli stats -d 0 -i 1000 -m GPU_UTILIZATION,POWER
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumGetStats() / xpumGetMetricsTimestamp()      │
│  ──────────────────────► ModuleManager                        │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to MONITOR module (ID=1)               │  │
│  │                                        │  │
│  │  Check: m_modules[1].status?                │  │
│  │    NOT_LOADED → LoadModule(MONITOR)         │  │
│  │                                        │  │
│  │  Dispatch: monitorModule->ProcessMessage()      │  │
│  │    → Monitor reads from FieldCache (metrics     │  │
│  │       collected by Monitor's background tasks) │  │
│  │    → Returns aggregated stats                │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Health Monitoring (Health Module)
```
xpumcli health -d 0
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumGetHealth()       ───────────────────► ModuleManager                │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to HEALTH module (ID=2)               │  │
│  │                                        │  │
│  │  Check: m_modules[2].status?                │  │
│  │    NOT_LOADED → LoadModuleWithDependencies()   │  │
│  │    → Resolve HEALTH's deps: [MONITOR]        │  │
│  │    → Load MONITOR if not loaded              │  │
│  │    → Load HEALTH                           │  │
│  │                                        │  │
│  │  Dispatch: healthModule->ProcessMessage()       │  │
│  │    → Health reads from FieldCache              │  │
│  │       (GPU_TEMP, POWER, etc.)                 │  │
│  │    → Compares vs thresholds                  │  │
│  │    → Returns: PASS/FAIL per metric           │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Firmware Updates (Firmware Module)
```
xpumcli firmware -d 0 -t GFX -f image.bin
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumFirmwareFlash() ─────────────────────► ModuleManager                  │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to FIRMWARE module (ID=4)             │  │
│  │                                        │  │
│  │  Check: m_modules[4].status?                │  │
│  │    NOT_LOADED → LoadModule(FIRMWARE)           │  │
│  │    → FIRMWARE has no module deps              │  │
│  │    → dlopen("libxpum_mod_firmware.so")       │  │
│  │                                        │  │
│  │  Dispatch: firmwareModule->ProcessMessage()     │  │
│  │    → Firmware uses IGSC library                 │  │
│  │       to flash GFX firmware image               │  │
│  │    → Returns: progress, completion status       │  │
│  │                                        │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### GPU Configuration (Config Module — Power, Frequency, ECC)
```
xpumcli config -d 0 --set-power-limit 250
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumSetPowerLimit() ───────────────────────► ModuleManager                │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to CONFIG module (ID=5)              │  │
│  │                                        │  │
│  │  Check: m_modules[5].status?                │  │
│  │    NOT_LOADED → LoadModule(CONFIG)              │  │
│  │    → CONFIG has no module deps                │  │
│  │    → dlopen("libxpum_mod_config.so")          │  │
│  │                                        │  │
│  │  Dispatch: configModule->ProcessMessage()       │  │
│  │    → Config uses Sysman API                 │  │
│  │       zesDeviceSetPowerLimit()                  │  │
│  │       zesDeviceSetFrequencyRange()              │  │
│  │       zesDeviceSetECCState()                    │  │
│  │    → Returns: success/failure                  │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Topology (Topology Module)
```
xpumcli topology -d 0
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumGetTopology() ───────────────────────► ModuleManager                │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to TOPOLOGY module (ID=6)             │  │
│  │                                        │  │
│  │  Check: m_modules[6].status?                │  │
│  │    NOT_LOADED → LoadModule(TOPOLOGY)           │  │
│  │    → TOPOLOGY uses hwloc library               │  │
│  │       to build CPU/GPU/PCI tree                │  │
│  │    → Returns: topology as JSON tree            │  │
│  │                                        │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Device Groups (Group Module)
```
xpumcli group create -d 0,1,2
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumGroupCreate() ───────────────────────► ModuleManager                 │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to GROUP module (ID=7)              │  │
│  │                                        │  │
│  │  Check: m_modules[7].status?                │  │
│  │    NOT_LOADED → LoadModule(GROUP)               │  │
│  │    → GROUP has no module deps                │  │
│  │    → dlopen("libxpum_mod_group.so")            │  │
│  │                                        │  │
│  │  Dispatch: groupModule->ProcessMessage()        │  │
│  │    → Group manages device sets               │  │
│  │       Stores: group ID → device ID list          │  │
│  │    → Returns: success/failure                  │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Policies (Policy Module)
```
xpumcli policy create -d 0 -t power -c 250 --gt 200
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumPolicyCreate() ───────────────────────► ModuleManager                 │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to POLICY module (ID=8)              │  │
│  │                                        │  │
│  │  Check: m_modules[8].status?                │  │
│  │    NOT_LOADED → LoadModule(POLICY)              │  │
│  │    → POLICY depends on:                     │  │
│  │       [MONITOR, GROUP]                       │  │
│  │    → Load dependencies if needed                 │  │
│  │    → dlopen("libxpum_mod_policy.so")           │  │
│  │                                        │  │
│  │  Dispatch: policyModule->ProcessMessage()       │  │
│  │    → Policy monitors FieldCache                 │  │
│  │       (power, temperature, etc.)                 │  │
│  │    → Evaluates rules                          │  │
│  │    → Triggers actions when thresholds crossed     │  │
│  │    → Returns: policy execution result             │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Raw Data Export (Dump Module)
```
xpumcli dump -d 0 -f json -o output.json
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumDumpRawData() ───────────────────────► ModuleManager                 │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to DUMP module (ID=9)              │  │
│  │                                        │  │
│  │  Check: m_modules[9].status?                │  │
│  │    NOT_LOADED → LoadModule(DUMP)               │  │
│  │    → DUMP depends on [MONITOR]                 │  │
│  │    → Load MONITOR if not loaded                │  │
│  │    → Load DUMP                              │  │
│  │                                        │  │
│  │  Dispatch: dumpModule->ProcessMessage()        │  │
│  │    → Dump reads from FieldCache              │  │
│  │       (all metrics time-series)                 │  │
│  │    → Formats as JSON/CSV                  │  │
│  │    → Returns: export data/file                  │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### vGPU / SR-IOV (VGPU Module)
```
xpumcli vgpu create -d 0 -n 4 -c 0,1
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  xpumCreateVirtualGPU() ───────────────────────► ModuleManager                 │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Route to VGPU module (ID=10)             │  │
│  │                                        │  │
│  │  Check: m_modules[10].status?               │  │
│  │    NOT_LOADED → LoadModule(VGPU)               │  │
│  │    → VGPU depends on: [none]                 │  │
│  │    → dlopen("libxpum_mod_vgpu.so")             │  │
│  │                                        │  │
│  │  Dispatch: vgpuModule->ProcessMessage()        │  │
│  │    → VGPU uses Sysman Vf API                 │  │
│  │       to manage SR-IOV virtual functions           │  │
│  │    → Returns: VF creation result               │  │
│                          │                                  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                          │                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Extension Plugin Examples

#### Example: GPU-Family Specific Performance Plugin
```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  libxpum_plugin_xe3_perf.so (Tier 2, discovered at runtime)                    │
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  xpum_get_module_descriptor()                                          │   │
│  │    → { name: "xe3-performance",                                      │   │
│  │         description: "Extended perf diagnostics for Xe3 architecture",         │   │
│  │         capabilities: DIAGNOSTIC,                                  │   │
│  │         apiVersion: 100,                                            │   │
│  │         supportedDevices: ["0x56c0","0x56c1","0x56c2"]}            │   │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  Plugin loaded on-demand when:                                        │   │
│  │    1. Diag module runs level 3 test                              │   │
│  │    2. Diag queries GetPluginsByCapability(DIAGNOSTIC)              │   │
│  │    3. xe3-perf descriptor returned in discovery                        │   │
│  │    4. If device PCI ID matches Xe3, plugin is loaded              │   │
│  │                                                                       │   │
│  │  Dispatch via xe3-perf ProcessMessage(DIAG_RUN)                    │   │
│  │    ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  │  Plugin uses CoreProxy to:                                 │   │
│  │  │    GetDeviceHandle(deviceId, &zeDevice, nullptr)               │   │
│  │  │       → Filter: Is this Xe3 device?                           │   │
│  │  │       → Check PCI ID: deviceId in ["0x56c0","0x56c1","0x56c2"] │   │
│  │  │       → NO match: Skip test, return UNSUPPORTED              │   │
│  │  │       → YES match: Continue                                      │   │
│  │  │                                                                  │   │
│  │  │  5. Run Xe3-specific compute test                       │   │
│  │  │    → Use Xe3 instruction extensions for optimal performance     │   │
│  │  │    → Dispatch compute kernels                                 │   │
│  │  │    → Measure execution time, throughput                    │   │
│  │  │                                                                  │   │
│  │  │  6. Return result via message                            │   │
│  │  │    → Contains: execution time, GFLOPS, % of Xe3 reference   │   │
│  │  │                                                                  │   │
│  └───────────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  Results aggregated by Diag module with built-in tests                 │   │
│  User sees unified diagnostic output with custom plugin metrics            │   │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### Example: Custom Metric Exporter Plugin
```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  libxpum_plugin_telemetry_exporter.so (Tier 2)                            │
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  xpum_get_module_descriptor()                                          │   │
│  │    → { name: "telemetry-exporter",                               │   │
│  │         description: "Export metrics in custom formats",                   │   │
│  │         capabilities: MONITORING,                                   │   │
│  │         apiVersion: 100                                            │   │
│  │         supportedDevices: NULL (all)                                   │   │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  Plugin hooks into FieldCache for raw metric access:                   │   │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  When Monitor module collects a sample:                           │   │
│  │    → FieldCache::StoreSample() called                             │   │
│  │  → Plugin's CoreProxy intercepts via SEND_MODULE_COMMAND       │   │
│  │    → Asks Monitor module to route custom sub-command to         │   │
│  │       telemetry_exporter module                               │   │
│  │         → Export metrics to file, database, or network           │   │
│  │                                                                       │   │
│  └───────────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Dependency Graph and Tier Definitions

**Design Heritage**: This modular architecture is inspired by NVIDIA DCGM's module system, with adaptations for XPU Manager's existing codebase. The `new_architecture.md` document covered test plugin extraction; this document extends modularization to ALL subsystems.

**Tier 1** = Built-in Modules (shipped with package, IDs 0-31)
- Source code is part of `libxpum_core.so`
- Always available (no separate `.so` installation needed)
- Loaded on-demand by ModuleManager
- Known to ModuleManager at compile time
- Version: 1.0 (API version tied to core build)

**Tier 2** = Extension Plugins (discovered at runtime, IDs 32+)
- Separate `.so` files in `/usr/lib/xpum/plugins/`
- Auto-discovered by ModuleManager scanning plugin directory
- IDs 32+ assigned dynamically when registered
- Enabled via allowlist (optional)
- Loaded on-demand when first referenced
- Version info from descriptor (API compatibility check)
- Device filtering (PCI IDs) - can limit to specific GPUs
- Capability flags (DIAGNOSTIC, MONITORING, HEALTH, etc.)

**Diagnostic Levels** = Test Duration Categories (NOT module tiers)
- Level 1 (Quick): ~seconds to minutes
- Level 2 (Medium): minutes
- Level 3 (Long-running): minutes to hours

---

## Adding New Plugins Without Modifying Core

A key limitation of DCGM's approach is that module IDs are hardcoded — adding a new module requires editing `DcgmModuleIdEnum` in core headers and recompiling. The XPU Manager architecture improves on this by supporting **two tiers of modules**:

### Two-Tier Module System

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  Tier 1: Built-in Modules (known at compile time)                                │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Module IDs 0-31 (reserved range)                                          │  │
│  │                                                                            │  │
│  │  CORE(0), MONITOR(1), HEALTH(2), DIAG(3), FIRMWARE(4), POLICY(5),         │  │
│  │  CONFIG(6), TOPOLOGY(7), VGPU(8), GROUP(9), DUMP(10)                      │  │
│  │                                                                            │  │
│  │  - Registered at startup by ModuleManager                                  │  │
│  │  - Loaded on-demand (lazy)                                                 │  │
│  │  - API layer knows how to construct messages for these                     │  │
│  │  - Tight integration with Core Services                                    │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  Tier 2: Extension Plugins (discovered at runtime, NO core changes needed)       │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Module IDs 32+ (dynamic range)                                            │  │
│  │                                                                            │  │
│  │  Drop a .so file into /usr/lib/xpum/plugins/ and it is auto-discovered.   │  │
│  │  No core code changes. No recompilation. No new enum values.              │  │
│  │                                                                            │  │
│  │  Examples:                                                                 │  │
│  │  - libxpum_plugin_xe2_perf.so    (new GPU-gen-specific perf tests)        │  │
│  │  - libxpum_plugin_custom_diag.so (customer-specific diagnostics)          │  │
│  │  - libxpum_plugin_fabric.so      (XeLink fabric monitoring)              │  │
│  │  - libxpum_plugin_telemetry_export.so (custom metric exporters)           │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Plugin Self-Description

To support Tier 2 plugins that core knows nothing about, each module `.so` exports an additional **descriptor function** that tells the framework what the plugin provides:

```cpp
// File: core/include/xpum_module.h (additions)

/** Plugin capability flags */
typedef enum {
    XPUM_MODULE_CAP_DIAGNOSTIC    = (1 << 0),  // Provides diagnostic tests
    XPUM_MODULE_CAP_MONITORING    = (1 << 1),  // Provides metric collection
    XPUM_MODULE_CAP_HEALTH        = (1 << 2),  // Provides health checks
    XPUM_MODULE_CAP_CONFIG        = (1 << 3),  // Provides GPU configuration
    XPUM_MODULE_CAP_FIRMWARE      = (1 << 4),  // Provides firmware operations
    XPUM_MODULE_CAP_EXPORT        = (1 << 5),  // Provides data export
    XPUM_MODULE_CAP_CUSTOM        = (1 << 6),  // Custom/vendor-specific
} xpum_module_capability_t;

/** Plugin descriptor — self-reported metadata */
typedef struct {
    unsigned int version;               // Descriptor struct version
    char name[64];                      // Human-readable name
    char description[256];              // What this plugin does
    unsigned int apiVersion;            // Required core API version
    unsigned int capabilities;          // Bitmask of xpum_module_capability_t
    unsigned int numSubCommands;        // Number of sub-commands supported
    const char** supportedDevices;      // NULL-terminated list of PCI IDs, or NULL for all
} xpum_module_descriptor_t;

// Every module .so exports this (in addition to alloc/free/process):
extern "C" {
    /** Return plugin descriptor. Called BEFORE alloc to check compatibility. */
    const xpum_module_descriptor_t* xpum_get_module_descriptor();
}
```

### Updated Export Macro (with Descriptor)

```cpp
#define XPUM_EXPORT_MODULE(ModuleClass, Name, Desc, Caps)                      \
                                                                                \
static const xpum_module_descriptor_t s_descriptor = {                         \
    .version = 1,                                                               \
    .name = Name,                                                               \
    .description = Desc,                                                        \
    .apiVersion = XPUM_MODULE_API_VERSION,                                     \
    .capabilities = Caps,                                                       \
    .numSubCommands = 0,                                                        \
    .supportedDevices = nullptr,                                                \
};                                                                              \
                                                                                \
extern "C" {                                                                    \
    const xpum_module_descriptor_t* xpum_get_module_descriptor() {             \
        return &s_descriptor;                                                   \
    }                                                                           \
    xpum::XpumModule* xpum_alloc_module_instance(                              \
            xpum::xpum_core_callbacks_t* cb) {                                 \
        try { return new ModuleClass(*cb); }                                   \
        catch (...) { return nullptr; }                                         \
    }                                                                           \
    void xpum_free_module_instance(xpum::XpumModule* m) { delete m; }          \
    xpum_result_t xpum_module_process_message(                                 \
            xpum::XpumModule* m,                                               \
            xpum::xpum_module_command_header_t* cmd) {                         \
        if (!m || !cmd) return XPUM_GENERIC_ERROR;                             \
        try { return m->ProcessMessage(cmd); }                                 \
        catch (...) { return XPUM_GENERIC_ERROR; }                             \
    }                                                                           \
}

// Usage in a new plugin:
XPUM_EXPORT_MODULE(
    Xe2PerfPlugin,
    "xe2-performance",
    "Performance diagnostics for Intel Xe2 architecture GPUs",
    XPUM_MODULE_CAP_DIAGNOSTIC
)
```

### Plugin Discovery Flow

```
ModuleManager::DiscoverPlugins()
         │
         │  1. Scan Tier 1 path: /usr/lib/xpum/modules/
         │     (built-in modules with known IDs)
         │
         │  2. Scan Tier 2 path: /usr/lib/xpum/plugins/
         │     (extension plugins, auto-discovered)
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  For each .so in /usr/lib/xpum/plugins/:                             │
│                                                                      │
│  1. dlopen(path, RTLD_LAZY | RTLD_LOCAL)                             │
│                                                                      │
│  2. dlsym("xpum_get_module_descriptor")                              │
│     │                                                                │
│     ▼                                                                │
│  3. Call descriptor = xpum_get_module_descriptor()                   │
│     │                                                                │
│     ├─ Check API version compatibility:                              │
│     │    descriptor->apiVersion <= XPUM_MODULE_API_VERSION ?         │
│     │    No  → log warning, dlclose, skip this plugin                │
│     │    Yes → continue                                              │
│     │                                                                │
│     ├─ Check for duplicate names:                                    │
│     │    descriptor->name already registered?                        │
│     │    Yes → log warning, dlclose, skip (first-loaded wins)        │
│     │    No  → continue                                              │
│     │                                                                │
│     ├─ Check device compatibility (optional):                        │
│     │    descriptor->supportedDevices != NULL ?                      │
│     │    Yes → check if any current GPU matches PCI ID list          │
│     │          No match → dlclose, skip (not relevant to this HW)    │
│     │    No  → plugin works with all devices                         │
│     │                                                                │
│     └─ Register:                                                     │
│          Assign dynamic moduleId (32, 33, 34, ...)                   │
│          Store in m_dynamicModules[name] → {descriptor, dlHandle}    │
│          Do NOT allocate instance yet (lazy loading)                  │
│          dlclose() for now — will re-open when needed                │
│                                                                      │
│  4. Log: "Discovered plugin: xe2-performance (diagnostic) v1"        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### How Existing Features Use Extension Plugins

Extension plugins integrate with the existing system through the **capability-based routing** pattern:

```
Example: DiagnosticManager gathers tests from all diagnostic-capable plugins

  xpumRunDiagnostics(deviceId, level)
       │
       ▼
  ModuleManager::ProcessModuleCommand(DIAG, RUN_DIAG)
       │
       ▼
  DiagModule::ProcessMessage(RUN_DIAG)
       │
       ├─ Run built-in diagnostic tests (Tier 1)
       │
       └─ Query ModuleManager for all plugins with CAP_DIAGNOSTIC
            │
            ▼
          ModuleManager::GetPluginsByCapability(XPUM_MODULE_CAP_DIAGNOSTIC)
            │
            ├─ Returns: ["xe2-performance", "custom-diag"]
            │
            ▼
          For each diagnostic plugin:
            ├─ Load plugin if not loaded (lazy)
            ├─ Send DIAG_RUN sub-command to plugin
            ├─ Plugin runs its tests using CoreProxy
            ├─ Plugin returns results via message struct
            └─ DiagModule aggregates all results
```

### Adding a New Plugin: Step-by-Step

```
Adding "libxpum_plugin_xe2_perf.so" — a new diagnostic plugin for Xe2 GPUs:

Step 1: Create the plugin source (single directory)
  plugins/xe2_perf/
  ├── CMakeLists.txt
  ├── xe2_perf_plugin.h
  ├── xe2_perf_plugin.cpp          ← implements XpumModule
  └── xe2_perf_tests.cpp           ← plugin-specific test kernels

Step 2: Implement the module interface
  class Xe2PerfPlugin : public XpumModuleWithCoreProxy<32> {
  public:
      xpum_result_t ProcessMessage(xpum_module_command_header_t* cmd) override {
          switch (cmd->subCommand) {
              case DIAG_RUN:    return RunTests(cmd);
              case DIAG_STATUS: return GetStatus(cmd);
              default:          return XPUM_GENERIC_ERROR;
          }
      }

      std::vector<xpum_module_id_t> GetDependencies() const override {
          return { XPUM_MODULE_ID_MONITOR };  // Needs metric data
      }

  private:
      xpum_result_t RunTests(xpum_module_command_header_t* cmd) {
          // Use CoreProxy to get device handles
          ze_device_handle_t dev;
          m_coreProxy.GetDeviceHandle(deviceId, &dev, nullptr);

          // Run Xe2-specific compute/memory tests
          // ...

          // Store results via CoreProxy
          return XPUM_OK;
      }
  };

  XPUM_EXPORT_MODULE(
      Xe2PerfPlugin,
      "xe2-performance",
      "Extended performance diagnostics for Xe2 architecture",
      XPUM_MODULE_CAP_DIAGNOSTIC
  )

Step 3: Build
  cmake --build . --target xpum_plugin_xe2_perf

Step 4: Deploy (no core changes, no recompilation of core)
  cp libxpum_plugin_xe2_perf.so /usr/lib/xpum/plugins/

Step 5: Verify
  $ xpumcli diagnostic --list-plugins
  Built-in modules:
    monitor (loaded), health (not loaded), diag (loaded), ...
  Extension plugins:
    xe2-performance v1 [diagnostic] (not loaded)

Step 6: Use
  $ xpumcli diagnostic -d 0 -l 3
  ...runs built-in tests...
  ...auto-discovers and runs xe2-performance tests...
  ...aggregated results displayed...

  Total changes to core: ZERO
  Total files modified: ZERO
  Only new files: plugins/xe2_perf/* and the deployed .so
```

### Plugin API Versioning

```
Core API versions ensure forward compatibility:

  Core v1.0 (initial release)
    XPUM_MODULE_API_VERSION = 100
    Provides: GetDeviceList, GetDeviceHandle, GetLatestFieldValue, ...

  Core v1.1 (adds topology queries)
    XPUM_MODULE_API_VERSION = 110
    Adds: GetDeviceTopology, GetPCIeInfo
    All v1.0 functions still work

  Plugin compiled against v1.0:
    descriptor->apiVersion = 100
    Core v1.1 loads it fine (100 <= 110) ✓

  Plugin compiled against v1.2 (future):
    descriptor->apiVersion = 120
    Core v1.1 rejects it (120 > 110) ✗
    Log: "Plugin xe3-perf requires API v1.2 but core is v1.1"

  This ensures plugins never call CoreProxy functions that don't exist.
```

### Directory Layout with Extension Plugins

```
/usr/lib/xpum/
├── modules/                    Tier 1: Built-in modules (shipped with package)
│   ├── libxpum_mod_monitor.so
│   ├── libxpum_mod_health.so
│   ├── libxpum_mod_diag.so
│   ├── libxpum_mod_firmware.so
│   ├── libxpum_mod_policy.so
│   ├── libxpum_mod_config.so
│   ├── libxpum_mod_topology.so
│   ├── libxpum_mod_vgpu.so
│   ├── libxpum_mod_group.so
│   └── libxpum_mod_dump.so
│
├── plugins/                    Tier 2: Extension plugins (drop-in)
│   ├── libxpum_plugin_xe2_perf.so       ← new Xe2 diagnostics
│   ├── libxpum_plugin_custom_health.so  ← vendor health checks
│   └── libxpum_plugin_oam_monitor.so    ← OAM board monitoring
│
└── plugins.d/                  Tier 2: Plugin config overrides (optional)
    ├── xe2_perf.conf           ← per-plugin configuration
    └── custom_health.conf
```

---

## Manager-to-Module Mapping

### What Becomes a Module

| Current Manager | Module ID | Filename | Core Services Used |
|---|---|---|---|
| `MonitorManager` | `XPUM_MODULE_ID_MONITOR` | `libxpum_mod_monitor.so` | DeviceManager, FieldCache, ScheduledThreadPool |
| `HealthManager` | `XPUM_MODULE_ID_HEALTH` | `libxpum_mod_health.so` | DeviceManager, FieldCache (via CoreProxy) |
| `DiagnosticManager` | `XPUM_MODULE_ID_DIAG` | `libxpum_mod_diag.so` | DeviceManager, FieldCache |
| `FirmwareManager` | `XPUM_MODULE_ID_FIRMWARE` | `libxpum_mod_firmware.so` | DeviceManager |
| `PolicyManager` | `XPUM_MODULE_ID_POLICY` | `libxpum_mod_policy.so` | DeviceManager, FieldCache, Group (inter-module) |
| DeviceManager (config) | `XPUM_MODULE_ID_CONFIG` | `libxpum_mod_config.so` | DeviceManager (handles only) |
| Topology | `XPUM_MODULE_ID_TOPOLOGY` | `libxpum_mod_topology.so` | DeviceManager |
| `VgpuManager` | `XPUM_MODULE_ID_VGPU` | `libxpum_mod_vgpu.so` | DeviceManager |
| `GroupManager` | `XPUM_MODULE_ID_GROUP` | `libxpum_mod_group.so` | DeviceManager |
| `DumpRawDataManager` | `XPUM_MODULE_ID_DUMP` | `libxpum_mod_dump.so` | DeviceManager, FieldCache |

### What Stays in Core

| Component | Reason |
|---|---|
| `DeviceManager` | Required by all modules for device discovery and handles |
| `FieldCache` (new, replaces `DataLogic`) | Shared time-series storage used by all metric-related modules |
| `Configuration` | Global configuration singleton |
| `ScheduledThreadPool` | Shared thread pool infrastructure |
| `ModuleManager` | Module loading and message routing |
| Public C API (`xpum_api.h`) | External interface, routes to modules internally |

---

## Generalized Field/Metric System

The current `DataLogic` + `DataHandlerManager` is replaced by a unified `FieldCache` with watchable fields, configurable intervals, and time-series retention — aligned with DCGM's `CacheManager` and field watch pattern.

```cpp
// File: core/src/field_cache/field_cache.h
// Replaces: data_logic/data_logic.h + data_handler_manager.h

class FieldCache {
public:
    /**
     * Add a watch on a field for a device.
     * Multiple watchers can watch the same field; shortest interval wins.
     */
    xpum_result_t AddFieldWatch(
        xpum_device_id_t deviceId,
        MeasurementType fieldId,
        uint64_t monitorIntervalUs,     // Polling interval in microseconds
        double maxSampleAge,             // Max age before eviction (seconds)
        int maxKeepSamples,              // Max samples to retain
        xpum_module_id_t watcherModule); // Who is watching

    xpum_result_t RemoveFieldWatch(
        xpum_device_id_t deviceId,
        MeasurementType fieldId,
        xpum_module_id_t watcherModule);

    /** Store a new sample (called by monitor module after collecting) */
    xpum_result_t StoreSample(
        xpum_device_id_t deviceId,
        MeasurementType fieldId,
        uint64_t timestamp,
        const MeasurementData& data);

    /** Get the latest sample for a field */
    xpum_result_t GetLatestSample(
        xpum_device_id_t deviceId,
        MeasurementType fieldId,
        MeasurementData* data);

    /** Get statistics over a time range */
    xpum_result_t GetStatistics(
        xpum_device_id_t deviceId,
        MeasurementType fieldId,
        uint64_t startTime, uint64_t endTime,
        xpum_device_stats_data_t* stats);

    /** Get all active field watches (monitor module uses this) */
    std::vector<FieldWatchInfo> GetActiveWatches();
};
```

**Data flow**: Monitor module calls `GetActiveWatches()` → collects data from Level Zero → calls `StoreSample()`. Other modules (Health, Policy) call `AddFieldWatch()` to express interest, `GetLatestSample()` / `GetStatistics()` to read data.

---

## API Layer Changes

The public C API functions in `core/src/api/` currently call directly into manager interfaces. With modules, they construct a message struct and route it through the `ModuleManager`.

### Before (Current)

```cpp
xpum_result_t xpumGetHealth(...) {
    auto& core = Core::instance();
    return core.getHealthManager()->getHealth(deviceId, type, data);
}
```

### After (Proposed)

```cpp
xpum_result_t xpumGetHealth(xpum_device_id_t deviceId,
                             xpum_health_type_t type,
                             xpum_health_data_t* data) {
    xpum_health_msg_get_status_v1 msg = {};
    msg.header.length     = sizeof(msg);
    msg.header.moduleId   = XPUM_MODULE_ID_HEALTH;
    msg.header.subCommand = XPUM_HEALTH_SR_GET_STATUS;
    msg.header.version    = xpum_health_msg_get_status_version1;
    msg.deviceId   = deviceId;
    msg.healthType = type;

    xpum_result_t ret = Core::instance()
        .getModuleManager()
        ->ProcessModuleCommand(&msg.header);

    if (ret == XPUM_OK) {
        *data = msg.healthData;
    }
    return ret;
}
```

This matches the pattern DCGM uses in `dcgmlib/src/` where API functions construct module command structs and route them through the host engine handler.

---

## Proposed Directory Structure

```
xpumanager/
├── core/                              Core library (libxpum_core.so)
│   ├── include/                         Public API headers
│   │   ├── xpum_api.h                    C API functions (UNCHANGED)
│   │   ├── xpum_structs.h                Data structures (UNCHANGED)
│   │   ├── xpum_module.h                 NEW: Module base class
│   │   ├── xpum_module_structs.h         NEW: Module command header
│   │   └── xpum_core_communication.h     NEW: Core callback types
│   └── src/
│       ├── api/                          API entry points (MODIFIED: route via modules)
│       ├── core/
│       │   ├── core.h                     MODIFIED: owns ModuleManager + FieldCache
│       │   ├── core.cpp
│       │   ├── module_manager.h           NEW: Module loading & routing
│       │   └── module_manager.cpp         NEW
│       ├── device/                        STAYS: Device discovery & handles
│       │   └── gpu/                       STAYS: GPU device abstraction
│       ├── field_cache/                   NEW: Replaces data_logic/
│       │   ├── field_cache.h
│       │   ├── field_cache.cpp
│       │   └── data_handler.h             MOVED from data_logic/
│       ├── infrastructure/                STAYS: Logging, config, threading
│       └── control/                       STAYS: DeviceManagerInterface
│
├── modules/                            NEW: Top-level modules directory
│   ├── common/                           Shared module utilities
│   │   ├── CMakeLists.txt
│   │   ├── xpum_core_proxy.h
│   │   └── xpum_core_proxy.cpp
│   │
│   ├── monitor/                          libxpum_mod_monitor.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_monitor_structs.h
│   │   ├── xpum_module_monitor.h
│   │   ├── xpum_module_monitor.cpp
│   │   ├── monitor_task.h                MOVED from core/src/monitor/
│   │   └── monitor_task.cpp
│   │
│   ├── health/                           libxpum_mod_health.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_health_structs.h
│   │   ├── xpum_module_health.h
│   │   ├── xpum_module_health.cpp
│   │   ├── health_manager.h              MOVED from core/src/health/
│   │   └── health_manager.cpp
│   │
│   ├── diag/                             libxpum_mod_diag.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_diag_structs.h
│   │   ├── xpum_module_diag.h
│   │   ├── xpum_module_diag.cpp
│   │   ├── diagnostic_manager.h          MOVED from core/src/diagnostic/
│   │   ├── diagnostic_manager.cpp
│   │   └── precheck/                     MOVED
│   │
│   ├── firmware/                         libxpum_mod_firmware.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_firmware_structs.h
│   │   ├── xpum_module_firmware.h
│   │   └── xpum_module_firmware.cpp
│   │
│   ├── policy/                           libxpum_mod_policy.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_policy_structs.h
│   │   ├── xpum_module_policy.h
│   │   └── xpum_module_policy.cpp
│   │
│   ├── config/                           libxpum_mod_config.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_config_structs.h
│   │   ├── xpum_module_config.h
│   │   └── xpum_module_config.cpp
│   │
│   ├── topology/                         libxpum_mod_topology.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_topology_structs.h
│   │   ├── xpum_module_topology.h
│   │   └── xpum_module_topology.cpp
│   │
│   ├── vgpu/                             libxpum_mod_vgpu.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_vgpu_structs.h
│   │   ├── xpum_module_vgpu.h
│   │   └── xpum_module_vgpu.cpp
│   │
│   ├── group/                            libxpum_mod_group.so
│   │   ├── CMakeLists.txt
│   │   ├── xpum_group_structs.h
│   │   ├── xpum_module_group.h
│   │   └── xpum_module_group.cpp
│   │
│   └── dump/                             libxpum_mod_dump.so
│       ├── CMakeLists.txt
│       ├── xpum_dump_structs.h
│       ├── xpum_module_dump.h
│       └── xpum_module_dump.cpp
│
├── daemon/                             UNCHANGED
├── cli/                                UNCHANGED
├── rest/                               UNCHANGED
└── third_party/                        UNCHANGED
```

---

## Build System Changes

### Top-Level CMakeLists.txt

```cmake
# After existing core build
add_subdirectory(modules/common)
add_subdirectory(modules/monitor)
add_subdirectory(modules/health)
add_subdirectory(modules/diag)
add_subdirectory(modules/firmware)
add_subdirectory(modules/policy)
add_subdirectory(modules/config)
add_subdirectory(modules/topology)
add_subdirectory(modules/vgpu)
add_subdirectory(modules/group)
add_subdirectory(modules/dump)
```

### Example Module CMakeLists.txt (Health)

```cmake
# modules/health/CMakeLists.txt
add_library(xpum_mod_health SHARED
    xpum_module_health.cpp
    health_manager.cpp       # Moved from core/src/health/
)

target_include_directories(xpum_mod_health PRIVATE
    ${CMAKE_SOURCE_DIR}/core/include
    ${CMAKE_SOURCE_DIR}/core/src
    ${CMAKE_SOURCE_DIR}/modules/common
)

target_link_libraries(xpum_mod_health PRIVATE
    xpum_module_common       # CoreProxy, utilities
    ze_loader
    ${CMAKE_DL_LIBS}
)

set_target_properties(xpum_mod_health PROPERTIES
    OUTPUT_NAME xpum_mod_health
    PREFIX lib
)

install(TARGETS xpum_mod_health
    LIBRARY DESTINATION lib/xpum/modules
)
```

### Core Library Changes

The `core/CMakeLists.txt` would:
1. **Remove** sources moved to modules (health/, diagnostic/, firmware/, policy/, vgpu/, topology/)
2. **Add** new sources: `module_manager.cpp`, `field_cache.cpp`
3. **Keep** device/, infrastructure/, api/, control/, field_cache/

### DAEMONLESS Mode

Both modes work identically — modules are loaded the same way whether called from xpumd (daemon) or xpu-smi (direct). `CoreProxy::IsDaemonMode()` lets modules adjust behavior if needed.

The CLI stubs (`grpc_stub/` and `core_stub/`) remain **UNCHANGED**. They call the same `xpum_api.h` functions. The modularization is entirely internal to the core library.

---

## Migration Path

### Phase 1: Framework (2-3 weeks)

1. Create `xpum_module.h`, `xpum_module_structs.h`, `xpum_core_communication.h`
2. Implement `ModuleManager` in `core/src/core/module_manager.{h,cpp}`
3. Implement `XpumCoreProxy` in `modules/common/`
4. Create `FieldCache` as a refactored `DataLogic` with watch semantics
5. Add `modules/` directory and build infrastructure

Core still works monolithically at this point — no modules extracted yet.

### Phase 2: First Module — Health (1-2 weeks)

Health is the simplest module (3 methods, small state):
1. Create `modules/health/` with `XpumModuleHealth`
2. Define health message structs
3. Move `HealthManager` code into the module
4. Modify `api/xpum_health.cpp` to route through `ModuleManager`
5. **Verify**: all health CLI commands work identically

### Phase 3: Extract Remaining Modules (1-2 weeks each, parallelizable)

Order by complexity (simplest first):
1. **Group** — small, few dependencies
2. **Topology** — small, mostly hwloc wrapper
3. **Config** — medium, power/frequency/ECC controls
4. **Dump** — small, raw data export
5. **Policy** — medium, depends on FieldCache + Group
6. **Firmware** — medium, IGSC/IPMI dependencies
7. **vGPU** — medium, SR-IOV complexity
8. **Diagnostic** — large, many test types (leverages existing `new_architecture.md` plugin spec within this module)
9. **Monitor** — last, feeds FieldCache; all other modules depend on it

### Phase 4: Cleanup (1 week)

1. Remove dead code from core
2. Update all documentation
3. Verify packaging (.deb, .rpm) includes modules in `/usr/lib/xpum/modules/`
4. Performance testing (module loading overhead)
5. CI/CD pipeline integration

---

## Backward Compatibility

| Layer | Impact |
|---|---|
| **Public C API** (`xpum_api.h`) | Unchanged — same function signatures |
| **gRPC protocol** (`core.proto`) | Unchanged — daemon dispatches to same API functions |
| **CLI** (comlet classes) | Unchanged — same stub interface |
| **REST API** (Flask) | Unchanged — same gRPC methods |
| **Packaging** | Modules installed to `/usr/lib/xpum/modules/` |
| **Graceful degradation** | Missing module `.so` → `XPUM_MODULE_NOT_LOADED` (no crash) |

---

## Key Differences from DCGM

| Aspect | DCGM | XPU Manager (Proposed) | Rationale |
|---|---|---|---|
| Language | C-style structs throughout | C++ with C factory exports | Match existing XPU Manager C++ codebase |
| Connection tracking | `connectionId` in every message | Omitted | XPU Manager has simpler single-process client model |
| Entity system | Multi-type (GPU, GPU_I, GPU_CI, NVSwitch) | Device-only (`xpum_device_id_t`) | Intel GPU topology is flatter |
| Module discovery | Hardcoded filename table | Hardcoded filename table | Predictable, secure loading (same as DCGM) |
| Field IDs | Numeric field IDs (1-1000+) | `MeasurementType` enum | Keep existing metric system |
| Cache manager | Centralized `CacheManager` | `FieldCache` (evolved `DataLogic`) | Natural evolution of existing code |
| Embedded mode | `dcgmStartEmbedded()` | `DAEMONLESS` compile flag | Already exists, different mechanism but same goal |

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| ABI breakage between core and modules | Module crashes on version mismatch | Version checking on every message (`CheckVersion` pattern) |
| Performance overhead of message routing | Microsecond latency per call | Negligible vs. Level Zero call latency (ms); benchmark to confirm |
| dlopen overhead at first call | Delay on first command | Pre-load critical modules (monitor, health) during init |
| Complex debugging across .so boundaries | Harder to trace issues | Shared logging via CoreProxy; `-DXPUM_MODULES_STATIC` build option for debug builds |
| Thread safety across modules | Race conditions | Each module manages own locking; CoreProxy methods are thread-safe; FieldCache uses read-write locks |
| Build complexity increases | More CMake targets to manage | Clear per-module CMakeLists.txt template; CI validates all modules build |

---

## Design Patterns Summary

| Pattern | DCGM Implementation | XPU Manager Implementation |
|---|---|---|
| **Plugin/Factory** | `dcgm_alloc_module_instance()` | `xpum_alloc_module_instance()` |
| **Proxy** | `DcgmCoreProxy` | `XpumCoreProxy` |
| **Command** | `dcgm_module_command_header_t` | `xpum_module_command_header_t` |
| **Template Method** | `DcgmModuleWithCoreProxy<moduleId>` | `XpumModuleWithCoreProxy<moduleId>` |
| **Lazy Loading** | Module loaded on first command | Module loaded on first command |
| **Observer** | Field watching / notifications | `FieldCache::AddFieldWatch()` |
| **Singleton** | `DcgmHostEngineHandler` | `Core::instance()` (existing) |

---

## Critical Files Reference

### Files to Create
- `core/include/xpum_module.h` — Module base class
- `core/include/xpum_module_structs.h` — Module command header
- `core/include/xpum_core_communication.h` — Core callback types
- `core/src/core/module_manager.h` / `.cpp` — Module loading and routing
- `core/src/field_cache/field_cache.h` / `.cpp` — Unified field cache
- `modules/common/xpum_core_proxy.h` / `.cpp` — CoreProxy
- `modules/*/` — Each module directory

### Files to Modify
- `core/src/core/core.h` / `.cpp` — Add ModuleManager, replace individual manager pointers
- `core/src/api/*.cpp` — Route API calls through ModuleManager
- `core/CMakeLists.txt` — Remove extracted sources, add new sources
- `CMakeLists.txt` (root) — Add `modules/` subdirectories

### Files to Move (into modules/)
- `core/src/health/` → `modules/health/`
- `core/src/diagnostic/` → `modules/diag/`
- `core/src/firmware/` → `modules/firmware/`
- `core/src/policy/` → `modules/policy/`
- `core/src/vgpu/` → `modules/vgpu/`
- `core/src/topology/` → `modules/topology/`
- `core/src/monitor/` → `modules/monitor/`
- `core/src/group/` → `modules/group/`
- `core/src/dump_raw_data/` → `modules/dump/`

### DCGM Reference Files
- `DCGM/modules/DcgmModule.h` — Module base class pattern
- `DCGM/modules/health/DcgmModuleHealth.cpp` — Example module implementation
- `DCGM/dcgmlib/src/DcgmHostEngineHandler.cpp` — Module loading pattern
- `DCGM/common/DcgmCoreProxy.h` — CoreProxy pattern

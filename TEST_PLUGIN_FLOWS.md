# XPU Manager Test Execution Flows — New Plugin Architecture

This document illustrates how diagnostic tests are executed in the new modular architecture, showing:
- Built-in tests (Tier 1 module)
- Plugin tests (Tier 2 dynamic modules)
- Memory test detail
- All other test types

---

## Overview

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                     User Request                                           │
│                     xpumcli diagnostic -d 0 -l 3 -t memory       │
└───────────────────────────────────────────────────┬────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module (Tier 1 — libxpum_mod_diag.so)                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  Built-in Tests (hardcoded in module)                       │   │
│  │  SOFTWARE_ENV_VARS      → Check environment, libraries, permissions      │   │
│  │  HARDWARE_PCIE          → Validate PCIe via Sysman                       │   │
│  │  PERF_COMPUTE           → Run compute kernel benchmark                │   │
│  │  PERF_MEMORY_ALLOC      → Run memory allocation bandwidth test           │   │
│  │  PERF_MEMORY_BANDWIDTH → Run memory read/write bandwidth test          │   │
│  │  PERF_POWER            → Run power measurement test                    │   │
│  │  MEDIA_CODEC            → Run media codec encode/decode tests            │   │
│  │  XELINK_THROUGHPUT      → Run XeLink P2P and all-to-all throughput      │   │
│  │  LIGHT_CODEC            → Run codec validation test                   │   │
│  │  STRESS                → Run extended stress test                        │   │
│  │  PRE_CHECK             → Run pre-diagnostic validation                 │   │
│  │                                                                      │   │
│  │  Plugin Tests (discovered at runtime)                      │   │
│  │                                                              │   │
│  │  → GetPluginsByCapability(XPUM_MODULE_CAP_DIAGNOSTIC)  │   │
│  │    → Returns: ["xe2-performance", "custom-gpgpu"]       │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ProcessMessage(DISPATCH_RUN_DIAGS)                              │   │
│                                                                      │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Memory Test Flow (MEMORY_ERROR — Level 3)

```
User Command:
$ xpumcli diagnostic -d 0 -t memory -l 3

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CLI: comlet_diagnostic                           │
│                                                                  │
│  Parse args:                                           │
│  deviceId = 0                                          │
│  level = XPUM_DIAG_LEVEL_3 (long-running)                   │
│  types = [XPUM_DIAG_MEMORY_ERROR]                        │
│                                                                  │
└───────────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  grpc_stub / core_stub                           │
│                                                  │
│  xpumRunDiagnostics(deviceId=0,                    │
│                    level=LEVEL_3,                          │
│                    types=[MEMORY_ERROR],                       │
│                    count=1)                                │
└────────────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Daemon / API Layer                              │
│                                                                   │
│  xpumRunDiagnostics() {                            │
│                                                                   │
│    Build message:                                │
│    msg.header.moduleId = DIAG (3)                        │
│    msg.header.subCommand = RUN_DIAGS                      │
│    msg.level = LEVEL_3                                 │
│    msg.types[] = [MEMORY_ERROR]                          │
│    msg.count = 1                                       │
│                                                                   │
│    Call ModuleManager::ProcessModuleCommand()                 │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  ModuleManager:                                │   │
│  │    Check: m_modules[DIAG].status?                     │   │
│  │    NOT_LOADED → LoadModuleWithDependencies()           │   │
│  │                                            │   │
│  │    → Diag declares deps: [MONITOR]            │   │
│  │    → LoadModule(MONITOR) if needed                 │   │
│  │    → LoadModule(DIAG)                             │   │
│  │       → dlopen("libxpum_mod_diag.so")              │   │
│  │       → dlsym (alloc, free, process)              │   │
│  │       → Create XpumModuleDiag instance             │   │
│  │                                            │   │
│  │    Status = LOADED ✓                           │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                   │
│    Dispatch: m_modules[DIAG].processFunc(module, msg)        │
│                                                                   │
│  ▼                                                       │   │
│  XpumModuleDiag::ProcessMessage(RUN_DIAGS)            │   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dispatch built-in tests:                        │   │
│  │                                                │   │
│  │  for (type in msg.types[]) {                    │   │
│  │    switch (type) {                               │   │
│  │       case MEMORY_ERROR:                         │   │
│  │          → doDiagnosticMemoryError()                  │   │
│  │                                                │   │
│  │  Query for extension plugins:                   │   │
│  │    m_coreProxy.GetPluginsByCapability(              │   │
│  │      XPUM_MODULE_CAP_DIAGNOSTIC)                │   │
│  │                                                │   │
│  │  Returns: ["xe2-perf"] (Tier 2)            │   │
│  │                                                │   │
│  │  For each plugin:                             │   │
│  │    → LoadPlugin if not loaded                      │   │
│  │    → Send DIAG_RUN sub-command                      │   │
│  │    → Plugin returns results via message             │   │
│  │    → Aggregate with built-in results                │   │
│  │                                                │   │
│  │  Return aggregated results[]                       │   │
│  │                                                │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                   │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  XpumModuleDiag (Tier 1) Internal Execution   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Built-in Memory Error Test Execution          │   │
│  │                                                    │   │
│  │  1. Get device handle from CoreProxy          │   │
│  │    m_coreProxy.GetDeviceHandle(                          │   │
│  │      deviceId, &ze_device, &zes_device)                │   │
│  │                                                    │   │
│  │  2. Allocate test buffers (pattern to check)     │   │
│  │    → Allocate memory with specific bit pattern       │   │
│  │    → GPU writes to it, reads back               │   │
│  │    → If bits flipped → memory error detected          │   │
│  │                                                    │   │
│  │  3. Execute test via Level Zero                   │   │
│  │    → Submit GPU kernel or commands                │   │
│  │    → Wait for completion                        │   │
│  │                                                    │   │
│  │  4. Validate result                             │   │
│  │    → Check error counters via Sysman               │   │
│  │    → If errors exceed threshold → test FAIL           │   │
│  │    → Else → test PASS                           │   │
│  │                                                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  xe2-perf Plugin (Tier 2) Memory Test Execution │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Plugin receives DIAG_RUN sub-command          │   │
│  │  → msg.header contains deviceId, level, type            │   │
│  │                                                    │   │
│  │  1. Get Xe2 device handle from CoreProxy       │   │
│  │    m_coreProxy.GetDeviceHandle(                       │   │
│  │      deviceId, &ze_device, nullptr)                 │   │
│  │    → Filter: Is this Xe2 device?                    │   │
│  │       → Check PCI ID against supported list:           │   │
│  │          ["0x56c0","0x56c1"] (Xe2)              │   │
│  │       → If NOT match, skip test, return UNKNOWN       │   │
│  │                                                    │   │
│  │  2. Run Xe2-specific memory test                 │   │
│  │    → Allocate test buffers (different pattern)         │   │
│  │    → Use Xe2 compute instruction extension          │   │
│  │    → Dispatch Xe2 kernel                         │   │
│  │    → Measure bandwidth, compare to reference          │   │
│  │                                                    │   │
│  │  3. Return result via message struct            │   │
│  │    → Write: result, bandwidth, reference, etc.    │   │
│  │                                                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CoreProxy / FieldCache                       │
│  → Device handle acquired from DeviceManager         │
│  → Test results stored in FieldCache             │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                      Result displayed to user
```

### Memory Test Key Data

Built-in test (`XPUM_DIAG_MEMORY_ERROR`):
- **Target**: Detect uncorrectable memory errors
- **Method**: Bit pattern test (allocate memory with 0xAA/0x55 pattern, read back and verify)
- **Duration**: Minutes (Level 3 test)
- **Data sources**:
  - `zesDeviceGetRegisters` for register access
  - `zesDeviceGetState` for ECC error counters
- **Level Zero APIs used**:
  - `zeMemAllocBuffer`
  - `zeMemFree`
  - `zesDeviceGetRegisters`

Xe2 plugin test (`xe2-perf` Tier 2):
- **Target**: Validate Xe2-specific memory behavior
- **Method**: Xe2 compute instruction + bandwidth measurement
- **Device filter**: Only runs on Xe2 GPUs (PCI IDs 0x56c0/c1)
- **Reference values**: Built-in Xe2 performance baselines
- **Extension point**: Shows how new GPU families can add custom tests

---

## Software Environment Test (SOFTWARE_ENV_VARIABLES — Level 1)

```
User Command:
$ xpumcli diagnostic -d 0 -l 1 -t software

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CLI: comlet_diagnostic                           │
│                                                                  │
│  Parse args:                                           │
│  deviceId = 0                                          │
│  level = XPUM_DIAG_LEVEL_1 (quick)                      │
│  types = [XPUM_DIAG_SOFTWARE_ENV_VARIABLES]               │
│                                                                  │
└───────────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module ProcessMessage(SOFTWARE_ENV_VARS)   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  doDiagnosticEnvironmentVariables()          │   │
│  │                                            │   │
│  │  Checks:                                    │   │
│  │  1. Environment variables set?                     │   │
│  │     → Check: XPUM_DISABLE_PERIODIC_METRIC_MONITOR     │   │
│  │     → Check: Level Zero loader path                │   │
│  │                                            │   │
│  │  2. Required libraries present?                    │   │
│  │     → Check: libze_loader.so exists                  │   │
│  │     → Try dlopen, report if fails                    │   │
│  │                                            │   │
│  │  3. Device permissions                             │   │
│  │     → Check: read access to GPU devices              │   │
│  │     → Verify: Can query device info                 │   │
│  │                                            │   │
│  │  4. GPU in exclusive mode?                      │   │
│  │     → Check: GPU not used by other process          │   │
│  │     → Query: Is XPU Manager running?            │   │
│  │                                            │   │
│  │  Return: PASS/FAIL for each check via result[]   │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                      Result displayed to user

Note: This test does NOT use CoreProxy — direct checks only.
No plugin extension point for software tests (simple environment validation).
```

---

## Hardware PCIe Test (HARDWARE_PCIE — Level 2)

```
User Command:
$ xpumcli diagnostic -d 0 -l 2 -t hardware

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module ProcessMessage(HARDWARE_PCIE)   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  doDiagnosticHardwareSysman()              │   │
│  │                                            │   │
│  │  Uses CoreProxy to access:                   │   │
│  │                                            │   │
│  │  Checks:                                    │   │
│  │  1. Sysman device properties                    │   │
│  │     → zesDeviceGetProperties() via DeviceManager   │   │
│  │     → Validate: PCIe link width, speed, max payload  │   │
│  │                                            │   │
│  │  2. PCIe domain validation                     │   │
│  │     → zesDeviceGetPCIeProperties()              │   │
│  │     → Check: max payload size supported             │   │
│  │     → Validate: device can handle size             │   │
│  │                                            │   │
│  │  3. Gen-specific verification                   │   │
│  │     → Check device Gen against capabilities           │   │
│  │     → Query: Is PCIe FLR supported?              │   │
│  │                                            │   │
│  │  Return: PASS/FAIL for each check via result[]   │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                      Result displayed to user

Note: Uses Sysman APIs via CoreProxy device handles.
No plugin extension point (hardware tests are device-specific).
```

---

## Performance Computation Test (PERF_COMPUTE — Level 1/2)

```
User Command:
$ xpumcli diagnostic -d 0 -l 1 -t compute

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module ProcessMessage(PERF_COMPUTE)   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  doDiagnosticPeformanceComputation()         │   │
│  │                                            │   │
│  │  1. Setup via CoreProxy:                     │   │
│  │     m_coreProxy.GetDeviceHandle(deviceId, ...)     │   │
│  │                                            │   │
│  │  2. Load and dispatch compute kernel          │   │
│  │     → Load test kernel binary                     │   │
│  │     → zeModuleGetFunctionPointer()                │   │
│  │     → zeKernelCreate()                           │   │
│  │     → zeCommandListAppendCommand()                │   │
│  │     → zeCommandListSynchronize()                 │   │
│  │     → zeCommandQueueExecuteCommand()              │   │
│  │                                            │   │
│  │  3. Run test, measure time                     │   │
│  │     → Dispatch kernel, time execution               │   │
│  │     → Measure start/end with high-resolution timer   │   │
│  │                                            │   │
│  │  4. Calculate performance vs reference             │   │
│  │     → Count compute units (GFLOPS equivalent)       │   │
│  │     → Compare against device reference values         │   │
│  │                                            │   │
│  │  5. Return result via message struct           │   │
│  │     → Write: execution time, GFLOPS, % of ref    │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Tier 2 Plugins (extension opportunity)                │
│                                                                  │
│  Could extend with:                                    │   │
│  • GPU-family-specific compute tests                 │   │
│  • New benchmark suites                           │   │
│  • Custom reference values                         │   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                      Result displayed to user
```

---

## Memory Bandwidth Test (PERF_MEMORY_BANDWIDTH — Level 1/2)

```
User Command:
$ xpumcli diagnostic -d 0 -l 2 -t bandwidth

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module ProcessMessage(PERF_MEMORY_BANDWIDTH) │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  doDiagnosticPeformanceMemoryBandwidth()      │   │
│  │                                            │   │
│  │  1. Setup via CoreProxy:                     │   │
│  │     m_coreProxy.GetDeviceHandle(deviceId, ...)     │   │
│  │                                            │   │
│  │  2. Allocate source/destination buffers         │   │
│  │     → Use zeMemAllocBuffer() for test buffers     │   │
│  │     → Lock memory with zeVirtualMemReserve       │   │
│  │                                            │   │
│  │  3. Run bandwidth test (via Level Zero copy)     │   │
│  │     → Use command queue for async copy             │   │
│  │     → zeCommandListAppendMemoryCopy()               │   │
│  │     → Measure transfer time and size              │   │
│  │     → Calculate bandwidth in GB/s                  │   │
│  │                                            │   │
│  │  4. Store result in FieldCache              │   │
│  │     → m_coreProxy.StoreFieldValue(                 │   │
│  │         deviceId, MEMORY_BANDWIDTH, timestamp, value)    │   │
│  │                                            │   │
│  │  5. Return result via message struct           │   │
│  │     → Write: bandwidth, latency values             │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FieldCache                                                │
│  → Monitors bandwidth metric via Monitor module     │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                      Result displayed to user
```

---

## XeLink Throughput Test (XELINK_THROUGHPUT — Level 3)

```
User Command:
$ xpumcli diagnostic -d 0 -l 3 -t xelink

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module ProcessMessage(XELINK_THROUGHPUT)  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  doDiagnosticXeLinkThroughput()              │   │
│  │                                            │   │
│  │  1. Setup via CoreProxy:                     │   │
│  │     m_coreProxy.GetDeviceHandle(deviceId, ...)     │   │
│  │     → Get fabric port info via Sysman               │   │
│  │                                            │   │
│  │  2. Run P2P throughput test                    │   │
│  │     → Query XeLink counters                      │   │
│  │     → zesDeviceGetFabricPortProperties()            │   │
│  │     → Get TX counter per port                    │   │
│  │     → Measure bytes transmitted over time            │   │
│  │     → Calculate throughput (MB/s)                │   │
│  │                                            │   │
│  │  3. Run all-to-all throughput test             │   │
│  │     → Sum all TX counters, get total bytes          │   │
│  │     → Get all RX counters from neighbor devices     │   │
│  │     → Calculate all-to-all bandwidth             │   │
│  │                                            │   │
│  │  4. Store results in FieldCache              │   │
│  │     → P2P and all-to-all metrics stored           │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Extension Opportunity                               │
│                                                                  │
│  Tier 2 plugins could add:                              │
│  • GPU-family-specific XeLink optimizations            │
│  • Custom latency measurements                        │
│  • Multi-topology validation tests                  │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                      Result displayed to user
```

---

## Media Codec Test (MEDIA_CODEC — Level 2)

```
User Command:
$ xpumcli diagnostic -d 0 -l 2 -t media

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module ProcessMessage(MEDIA_CODEC)   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  doDiagnosticMediaCodec()                   │   │
│  │                                            │   │
│  │  1. Check file paths for codec tools            │   │
│  │     → Test: /usr/bin/intel_gpu_top requested │   │
│  │     → Load kernel, run codec test                  │   │
│  │                                            │   │
│  │  2. Run H.264 encode/decode                     │   │
│  │     → Execute tool with test stream                  │   │
│  │     → Parse output: pass/fail, FPS achieved      │   │
│  │                                            │   │
│  │  3. Run H.265 encode/decode                     │   │
│  │     → Execute: test_stream_4k script                 │   │
│  │     → Parse output: resolution, FPS                  │   │
│  │                                            │   │
│  │  4. Return aggregated results                 │   │
│  │     → Combine results from both codecs             │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Extension Opportunity                               │
│                                                                  │
│  Tier 2 plugins could add:                              │
│  • AV1 codec support tests                       │
│  • New codec standard tests (AV1, VP9, etc.)         │
│  • Vendor-specific media validation                │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                      Result displayed to user
```

---

## Complete Diagnostic Execution — All Tests

```
User Command:
$ xpumcli diagnostic -d 0 -l 3

                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CLI parses:                                                │
│  deviceId = 0, level = 3, no specific type (-t all)    │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  xpumRunDiagnostics() → API Layer                      │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Build message:                                            │
│  msg.header.moduleId = DIAG (3)                          │
│  msg.header.subCommand = RUN_DIAGS                      │
│  msg.level = LEVEL_3                                     │
│  msg.types = ALL (run all built-in + query plugins)       │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ModuleManager::ProcessModuleCommand()                  │
│                                                                  │
│  1. Check DIAG module loaded                            │
│     → Load diag.so if NOT_LOADED (depends on Monitor)       │
│  → Load monitor.so if needed for dependencies             │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Diagnostic Module (Tier 1)                         │
│                                                                  │
│  ProcessMessage(RUN_DIAGS)                              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Run built-in tests (dispatch by type):           │   │
│  │    SOFTWARE_ENV_VARS → doDiagnosticEnvironmentVariables()   │   │
│  │    HARDWARE_PCIE → doDiagnosticHardwareSysman()         │   │
│  │    PERF_COMPUTE → doDiagnosticPeformanceComputation() │   │
│  │    PERF_MEMORY_ALLOC → doDiagnosticPeformanceMemoryAllocation() │
│  │    PERF_MEMORY_BANDWIDTH → doDiagnosticPeformanceMemoryBandwidth() │
│  │    PERF_POWER → doDiagnosticPeformancePower()         │   │
│  │    MEDIA_CODEC → doDiagnosticMediaCodec()             │   │
│  │    XELINK_THROUGHPUT → doDiagnosticXeLinkThroughput() │
│  │    LIGHT_CODEC → doDiagnosticLightCodec()           │   │
│  │    STRESS → doDiagnosticStress()                   │   │
│  │    PRE_CHECK → doDiagnosticPreCheck()               │   │
│  │                                            │   │
│  │  Query for Tier 2 diagnostic plugins:             │   │
│  │  m_coreProxy.GetPluginsByCapability(                   │
│  │    XPUM_MODULE_CAP_DIAGNOSTIC)                       │
│  │                                            │   │
│  │  Returns: ["xe2-perf"] if installed             │   │
│  │                                            │   │
│  │  For each Tier 2 plugin:                         │   │
│  │    Load plugin if not already loaded                  │   │
│  │    Send DIAG_RUN sub-command                         │   │
│  │    Plugin runs its tests using CoreProxy             │   │
│  │    Plugin returns results via message struct           │   │
│  │                                            │   │
│  │  Aggregate plugin results with built-in results       │   │
│  │                                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Return to CLI                                            │
│  results[] contains:                                      │
│  [0] SOFTWARE_ENV_VARS: PASS/FAIL                       │
│  [1] HARDWARE_PCIE: PASS/FAIL                            │
│  [2] PERF_COMPUTE: GFLOPS, % of reference                │
│  [3] PERF_MEMORY_ALLOC: Bandwidth, % of reference          │
│  [4] PERF_MEMORY_BANDWIDTH: Bandwidth, % of reference       │
│  [5] PERF_POWER: Watts, % of reference                     │
│  [6] MEDIA_CODEC: H.264 pass/fail, FPS                   │
│  [7] XELINK_THROUGHPUT: P2P, A2A bandwidth              │
│  [8] LIGHT_CODEC: Supported/Not supported                    │
│  [9] STRESS: Elapsed time, score                          │
│  [10] PRE_CHECK: All validations passed                    │
│  [11+] xe2-perf plugin tests: Custom metrics from plugin    │
│                                                                  │
└───────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                      CLI formats and displays results
```

### Key Extension Points Illustrated

1. **Memory Test**: `xe2-perf` plugin shows how GPU-family-specific tests can be added
   - Plugin checks device PCI ID (Xe2 only)
   - Runs Xe2-optimized memory test with different patterns
   - Returns Xe2-specific performance metrics
   - Built-in test runs generic pattern for all GPUs

2. **Compute Test**: Could add `amx-perf` plugin for AMX extensions
   - Tests AMX-specific instruction sets
   - Returns AMX GFLOPS metrics
   - Diag module aggregates seamlessly: built-in + AMX plugin

3. **Bandwidth Test**: Could add `ddr-bandwidth` plugin
   - Tests memory controller-specific timing
   - Returns detailed DRAM timings
   - Useful for memory tuning/validation

4. **Media Test**: Could add `avc-codec` plugin
   - Tests AV1/AVC2/VP9 codec support
   - Returns codec encode/decode quality metrics
   - Extends beyond built-in H.264/H.265

5. **XeLink Test**: Could add `xelink-latency` plugin
   - Measures point-to-point latency between GPUs
   - Returns detailed latency histograms
   - Complements built-in throughput tests

---

## Summary

| Test Type | Level | Built-In Module | Extension Point | Who Loads Plugin |
|---|---|---|---|---|
| SOFTWARE_ENV_VARS | L1 | Diagnostic (doDiagnosticEnvironmentVariables) | None | Diag module only |
| HARDWARE_PCIE | L2 | Diagnostic (doDiagnosticHardwareSysman) | None | Diag module only |
| PERF_COMPUTE | L1/2 | Diagnostic (doDiagnosticPeformanceComputation) | Tier 2 GPU-family perf plugins | Diag module via GetPluginsByCapability |
| PERF_MEMORY_ALLOC | L1/2 | Diagnostic (doDiagnosticPeformanceMemoryAllocation) | Tier 2 memory micro-benchmark plugins | Diag module via GetPluginsByCapability |
| PERF_MEMORY_BANDWIDTH | L1/2 | Diagnostic (doDiagnosticPeformanceMemoryBandwidth) | Tier 2 custom bandwidth measurement plugins | Diag module via GetPluginsByCapability |
| MEDIA_CODEC | L2 | Diagnostic (doDiagnosticMediaCodec) | Tier 2 codec validation plugins (AV1, AV1, etc.) | Diag module via GetPluginsByCapability |
| XELINK_THROUGHPUT | L3 | Diagnostic (doDiagnosticXeLinkThroughput) | Tier 2 fabric topology plugins | Diag module via GetPluginsByCapability |
| MEMORY_ERROR | L3 | Diagnostic (doDiagnosticMemoryError) | Tier 2 memory error pattern tests | Diag module via GetPluginsByCapability |
| STRESS | L3 | Diagnostic (doDiagnosticStress) | Tier 2 endurance test plugins | Diag module via GetPluginsByCapability |
| PRE_CHECK | L1 | Diagnostic (doDiagnosticPreCheck) | None | Diag module only |

**Key Points:**
- Diag module (Tier 1) queries for Tier 2 plugins via `GetPluginsByCapability(CAP_DIAGNOSTIC)`
- Plugins are loaded lazily when Diag module first requests them by capability
- Each plugin runs tests using `CoreProxy` for device handles, field cache access
- Results aggregated by Diag module — CLI sees unified result set

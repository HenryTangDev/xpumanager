# gRPC and Protocol Buffers Analysis - XPU Manager

## Overview

| Attribute | Value |
|-----------|-------|
| Proto File | `daemon/core.proto` |
| Syntax | Proto3 |
| Lines | 1277 |
| Messages | ~95 |
| Enums | 21 |
| Service | 1 (`XpumCoreService`) |
| Total RPCs | 80 |
| RPC Type | **Synchronous Unary** (79) + **Server Streaming** (1) |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         User Interfaces                              │
│                                                                      │
│  ┌──────────┐   ┌──────────────┐   ┌────────────┐  ┌─────────────┐  │
│  │ xpumcli  │   │  REST API    │   │ Prometheus │  │   xpu-smi   │  │
│  │ cli/     │   │  rest/       │   │ Exporter   │  │   cli/      │  │
│  │          │   │  Flask +     │   │ rest/      │  │             │  │
│  │ grpc_stub│   │  Gunicorn    │   │ prometheus_│  │  core_stub  │  │
│  └────┬─────┘   └──────┬───────┘   │ exporter/  │  └──────┬──────┘  │
│       │                │           └─────┬──────┘         │         │
└───────┼────────────────┼─────────────────┼────────────────┼─────────┘
        │ gRPC           │ gRPC            │ gRPC           │ direct
        │ (Unix Socket)  │ (Unix Socket)   │ (Unix Socket)  │
        ▼                ▼                 ▼                │
┌───────────────────────────────────────────────┐           │
│              Daemon (xpumd)                   │           │
│              daemon/                          │           │
│                                               │           │
│  core.proto ──▶ *_service_impl.cpp            │           │
│                                               │           │
│  Sockets:                                     │           │
│    /tmp/xpum_p.sock  (privileged)            │           │
│    /tmp/xpum_up.sock (unprivileged)          │           │
└───────────────────────┬───────────────────────┘           │
                        │                                   │
                        ▼                                   ▼
┌───────────────────────────────────────────────────────────────────┐
│                  Core Library (libxpum.so)                        │
│                  core/                                            │
│                                                                   │
│  Public API: include/xpum_api.h, include/xpum_structs.h          │
└───────────────────────────────────────────────────────────────────┘
```

---

## Communication Types

### 1. Synchronous Unary RPC (79 of 80 RPCs)

Standard request-response pattern. Client blocks until server responds.

```protobuf
rpc getVersion(google.protobuf.Empty) returns (XpumVersionInfoArray);
rpc getDeviceList(google.protobuf.Empty) returns (XpumDeviceBasicInfoArray);
rpc runDiagnostics(RunDiagnosticsRequest) returns (DiagnosticsTaskInfo);
```

### 2. Server Streaming RPC (1 of 80 RPCs)

Single request, multiple streamed responses over time.

```protobuf
rpc readPolicyNotifyData(google.protobuf.Empty) returns (stream ReadPolicyNotifyDataResponse);
```

Used for real-time policy violation notifications (e.g., "GPU temperature exceeded threshold").

### 3. Not Used

| Type | Syntax | Used? |
|------|--------|-------|
| Client Streaming | `rpc Method(stream Request) returns (Response)` | No |
| Bidirectional Streaming | `rpc Method(stream Request) returns (stream Response)` | No |

---

## Why Synchronous (Not Async)?

### Server Evidence (`daemon/daemon.cpp`)

```cpp
// Sync server setup (lines 127-132)
unique_ptr<grpc::Server> buildAndStartRPCServer(...) {
    grpc::ServerBuilder builder;                              // Sync builder
    builder.AddListeningPort(unixSockAddr, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);                        // Sync service (NOT AsyncService)
    return builder.BuildAndStart();
}

// Blocking wait (lines 186-190)
std::thread grpc_server_thread(
    [](::grpc::Server* grpc_server_ptr) {
        grpc_server_ptr->Wait();                              // Blocking
    },
    privServer.get());
```

**If async**, would use: `AsyncService`, `ServerCompletionQueue`, `cq->Next()` polling.

### Client Evidence (`cli/src/grpc_stub/grpc_core_stub.cpp`)

```cpp
// Sync blocking call
grpc::ClientContext context;
GroupInfo response;
grpc::Status status = stub->groupCreate(&context, name, &response);  // BLOCKS
if (status.ok()) { ... }
```

**If async**, would use: `stub->AsyncGroupCreate()`, `CompletionQueue`, callbacks/futures.

### Rationale

1. **Simpler code** - No completion queues, callbacks, or state machines
2. **Sufficient throughput** - GPU management isn't ultra-high volume
3. **Core library is sync** - `xpumGetDeviceList()` etc. are blocking C calls
4. **Unix socket transport** - Local IPC already has low latency

---

## Code Generation (CMake)

### `.cmake/grpc_common.cmake`

```cmake
find_package(Protobuf CONFIG)
find_package(gRPC CONFIG)
find_program(_PROTOBUF_PROTOC protoc)
find_program(_GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin)
```

### `daemon/CMakeLists.txt`

```cmake
get_filename_component(core_proto "${CMAKE_CURRENT_LIST_DIR}/core.proto" ABSOLUTE)

# Generated output files
set(core_proto_srcs "${CMAKE_CURRENT_BINARY_DIR}/core.pb.cc")      # Protobuf messages
set(core_proto_hdrs "${CMAKE_CURRENT_BINARY_DIR}/core.pb.h")
set(core_grpc_srcs "${CMAKE_CURRENT_BINARY_DIR}/core.grpc.pb.cc")  # gRPC service stubs
set(core_grpc_hdrs "${CMAKE_CURRENT_BINARY_DIR}/core.grpc.pb.h")

# protoc invocation
add_custom_command(
  OUTPUT "${core_proto_srcs}" "${core_proto_hdrs}" "${core_grpc_srcs}" "${core_grpc_hdrs}"
  COMMAND ${_PROTOBUF_PROTOC}
  ARGS --grpc_out "${CMAKE_CURRENT_BINARY_DIR}"
       --cpp_out "${CMAKE_CURRENT_BINARY_DIR}"
       -I "${core_proto_path}"
       --plugin=protoc-gen-grpc="${_GRPC_CPP_PLUGIN_EXECUTABLE}"
       "${core_proto}"
  DEPENDS "${core_proto}")
```

---

## Proto File Structure (`daemon/core.proto`)

### Imports

```protobuf
syntax = "proto3";
import "google/protobuf/empty.proto";
```

### Foundational Messages

```protobuf
message GeneralEnum {
    uint32 value = 1;  // Generic enum wrapper for C enums
}

message DeviceId {
    uint32 id = 1;
    string errorMsg = 2;
    int32 errorNo = 3;
}

message DeviceBDF {
    string bdf = 1;  // PCI Bus:Device.Function (e.g., "0000:03:00.0")
}
```

### Enums by Category

#### Diagnostics (16 test types)

```protobuf
enum DiagnosticsTaskResult { DIAG_RESULT_UNKNOWN=0; DIAG_RESULT_PASS=1; DIAG_RESULT_FAIL=2; }

message DiagnosticsComponentInfo {
  enum Type {
    DIAG_SOFTWARE_ENV_VARIABLES = 0;
    DIAG_SOFTWARE_LIBRARY = 1;
    DIAG_SOFTWARE_PERMISSION = 2;
    DIAG_SOFTWARE_EXCLUSIVE = 3;
    DIAG_LIGHT_COMPUTATION = 4;
    DIAG_HARDWARE_SYSMAN = 5;
    DIAG_INTEGRATION_PCIE = 6;
    DIAG_MEDIA_CODEC = 7;
    DIAG_PERFORMANCE_COMPUTATION = 8;
    DIAG_PERFORMANCE_POWER = 9;
    DIAG_PERFORMANCE_MEMORY_BANDWIDTH = 10;
    DIAG_PERFORMANCE_MEMORY_ALLOCATION = 11;
    DIAG_MEMORY_ERROR = 12;
    DIAG_LIGHT_CODEC = 13;
    DIAG_XE_LINK_THROUGHPUT = 14;
    DIAG_XE_LINK_ALL_TO_ALL_THROUGHPUT = 15;
  }
}

enum DiagnosticsMediaCodecResolution { DIAG_MEDIA_1080p=0; DIAG_MEDIA_4K=1; }
enum DiagnosticsMediaCodecFormat { DIAG_MEDIA_H265=0; DIAG_MEDIA_H264=1; DIAG_MEDIA_AV1=2; }
```

#### Precheck Errors (19 error types)

```protobuf
enum PrecheckErrorType {
    GUC_NOT_RUNNING = 0;
    GUC_ERROR = 1;
    GUC_INITIALIZATION_FAILED = 2;
    IOMMU_CATASTROPHIC_ERROR = 3;
    LMEM_NOT_INITIALIZED_BY_FIRMWARE = 4;
    PCIE_ERROR = 5;
    DRM_ERROR = 6;
    GPU_HANG = 7;
    I915_ERROR = 8;
    I915_NOT_LOADED = 9;
    LEVEL_ZERO_INIT_ERROR = 10;
    HUC_DISABLED = 11;
    HUC_NOT_RUNNING = 12;
    LEVEL_ZERO_METRICS_INIT_ERROR = 13;
    MEMORY_ERROR = 14;
    GPU_INITIALIZATION_FAILED = 15;
    MEI_ERROR = 16;
    XE_ERROR = 17;
    XE_NOT_LOADED = 18;
}

enum PrecheckErrorCategory { PRECHECK_ERROR_CATEGORY_HARDWARE=0; PRECHECK_ERROR_CATEGORY_KMD=1; PRECHECK_ERROR_CATEGORY_UMD=2; }
enum PrecheckErrorSeverity { CRITICAL=0; HIGH=1; MEDIUM=2; LOW=3; }
enum PrecheckComponentType { DRIVER=0; CPU=1; GPU=2; }
enum PrecheckComponentStatus { UNKNOWN=0; PASS=1; FAIL=2; }
```

#### Health Monitoring

```protobuf
enum HealthType {
    HEALTH_CORE_THERMAL = 0;
    HEALTH_MEMORY_THERMAL = 1;
    HEALTH_POWER = 2;
    HEALTH_MEMORY = 3;
    HEALTH_FABRIC_PORT = 4;
    HEALTH_FREQUENCY = 5;
}

enum HealthStatusType { HEALTH_STATUS_UNKNOWN=0; HEALTH_STATUS_OK=1; HEALTH_STATUS_WARNING=2; HEALTH_STATUS_CRITICAL=3; }
enum HealthConfigType { HEALTH_CORE_THERMAL_LIMIT=0; HEALTH_MEMORY_THERMAL_LIMIT=1; HEALTH_POWER_LIMIT=2; }
```

#### Configuration

```protobuf
enum XpumPolicyConditionType { POLICY_CONDITION_TYPE_GREATER=0; POLICY_CONDITION_TYPE_LESS=1; POLICY_CONDITION_TYPE_WHEN_INCREASE=2; }
enum XpumPolicyActionType { POLICY_ACTION_TYPE_NULL=0; POLICY_ACTION_TYPE_THROTTLE_DEVICE=1; }
enum XpumStandbyMode { STANDBY_DEFAULT=0; STANDBY_NEVER=1; STANDBY_NULL=2; }
enum XpumSchedulerMode { SCHEDULER_TIMEOUT=0; SCHEDULER_TIMESLICE=1; SCHEDULER_EXCLUSIVE=2; SCHEDULER_DEBUG=3; }
enum XpumDeviceFunctionType { VIRTUAL=0; PHYSICAL=1; UNKNOWN=2; }
```

#### Policy Types (10 types)

```protobuf
enum XpumPolicyType {
    POLICY_TYPE_GPU_TEMPERATURE = 0;
    POLICY_TYPE_GPU_MEMORY_TEMPERATURE = 1;
    POLICY_TYPE_GPU_POWER = 2;
    POLICY_TYPE_RAS_ERROR_CAT_RESET = 3;
    POLICY_TYPE_RAS_ERROR_CAT_PROGRAMMING_ERRORS = 4;
    POLICY_TYPE_RAS_ERROR_CAT_DRIVER_ERRORS = 5;
    POLICY_TYPE_RAS_ERROR_CAT_CACHE_ERRORS_CORRECTABLE = 6;
    POLICY_TYPE_RAS_ERROR_CAT_CACHE_ERRORS_UNCORRECTABLE = 7;
    POLICY_TYPE_GPU_MISSING = 8;
    POLICY_TYPE_GPU_THROTTLE = 9;
    POLICY_TYPE_MAX = 10;
}
```

---

## Message Groups by Domain

### Device Discovery

```protobuf
message XpumDeviceBasicInfoArray {
    message XpumDeviceBasicInfo {
        DeviceId id = 1;
        GeneralEnum type = 2;
        string uuid = 3;
        string deviceName = 4;        // e.g., "Intel Data Center GPU Flex 170"
        string pcieDeviceId = 5;      // e.g., "0x56c0"
        string pciBdfAddress = 6;     // e.g., "0000:03:00.0"
        string vendorName = 7;
        string drmDevice = 8;         // e.g., "/dev/dri/card0"
        XpumDeviceFunctionType deviceFunctionType = 9;
    }
    repeated XpumDeviceBasicInfo info = 1;
    string errorMsg = 2;
    int32 errorNo = 3;
}

message XpumTopologyInfo {
    DeviceId id = 1;
    message XpumCpuAffinity {
        string localCpuList = 1;  // e.g., "0-7"
        string localCpus = 2;     // bitmask
    }
    XpumCpuAffinity cpuAffinity = 2;
    uint32 switchCount = 3;
    repeated XpumSwitchInfo switchInfo = 4;
}
```

### Diagnostics

```protobuf
message DiagnosticsTaskInfo {
    int32 deviceId = 1;
    uint32 level = 2;           // 1=quick, 2=medium, 3=full
    uint32 targetType = 3;
    bool finished = 4;
    DiagnosticsTaskResult result = 5;
    repeated DiagnosticsComponentInfo componentInfo = 6;
    string message = 7;
    uint32 count = 8;
    uint64 startTime = 9;
    uint64 endTime = 10;
    string errorMsg = 11;
    int32 errorNo = 12;
}
```

### Statistics/Telemetry

```protobuf
message DeviceStatsData {
    GeneralEnum metricsType = 1;  // GPU_UTILIZATION, MEMORY_USED, POWER, etc.
    bool isCounter = 2;           // Cumulative vs. gauge
    uint64 value = 3;
    uint64 min = 4;
    uint64 avg = 5;
    uint64 max = 6;
    uint64 accumulated = 7;
    uint32 scale = 8;             // Decimal places (100 = divide by 100)
}

message DeviceStatsInfo {
    uint32 deviceId = 1;
    bool isTileData = 2;          // Multi-tile GPUs
    int32 tileId = 3;
    int32 count = 4;
    repeated DeviceStatsData dataList = 5;
}
```

### Configuration

```protobuf
message ConfigTileData {
    string tileId = 1;
    uint32 minFreq = 2;
    uint32 maxFreq = 3;
    XpumStandbyMode standby = 5;
    XpumSchedulerMode scheduler = 7;
    uint32 schedulerTimeout = 8;
    double computePerformanceFactor = 12;
    double mediaPerformanceFactor = 14;
    bool memoryEccAvailable = 21;
    string memoryEccState = 23;
}
```

### Fabric/XeLink Topology

```protobuf
message XpumXelinkTopoInfoArray {
    message XelinkTopoUnit {
        uint32 deviceId = 1;
        bool onSubdevice = 2;
        uint32 subdeviceId = 3;
        uint32 numaIndex = 4;
        string cpuAffinity = 5;
    }
    message XelinkTopoInfo {
        XelinkTopoUnit localDevice = 1;
        XelinkTopoUnit remoteDevice = 2;
        string linkType = 3;       // "XE_LINK"
        repeated uint32 linkPortList = 4;
        int64 maxBitRate = 5;      // Gbps
    }
    repeated XelinkTopoInfo topoInfo = 1;
}
```

### vGPU/SR-IOV

```protobuf
message VgpuPrecheckResponse {
    bool vmxFlag = 1;       // VMX CPU extension
    bool iommuStatus = 3;   // IOMMU enabled
    bool sriovStatus = 5;   // SR-IOV capable
}

message VgpuCreateVfRequest {
    uint32 numVfs = 1;      // Number of Virtual Functions
    uint64 lmemPerVf = 2;   // Local memory per VF (bytes)
    int32 deviceId = 3;
}

message VfInfo {
    uint64 lmemSize = 1;
    XpumDeviceFunctionType deviceFunctionType = 2;
    string bdfAddress = 3;
    int32 deviceId = 4;
}
```

### Special Protobuf Features

**Oneof (Union Type)**:
```protobuf
message FlexTypeValue {
    oneof value {
        int64 intValue = 1;
        double floatValue = 2;
        string stringValue = 3;
    }
}
```

**Nested Messages**:
```protobuf
message XpumDeviceBasicInfoArray {
    message XpumDeviceBasicInfo { ... }  // Scoped type
    repeated XpumDeviceBasicInfo info = 1;
}
```

---

## Service Definition (80 RPCs)

### RPCs by Category

| Category | Count | RPCs |
|----------|-------|------|
| **Discovery** | 5 | `getVersion`, `getDeviceList`, `getDeviceProperties`, `getDeviceIdByBDF`, `getTopology` |
| **Groups** | 6 | `groupCreate`, `groupDestory`, `groupAddDevice`, `groupRemoveDevice`, `groupGetInfo`, `getAllGroups` |
| **Diagnostics** | 11 | `runDiagnostics*`, `getDiagnosticsResult*`, `runStress`, `checkStress`, `precheck`, `getPrecheckErrorList` |
| **Health** | 6 | `getHealth*`, `getHealthConfig*`, `setHealthConfig*` |
| **Statistics** | 8 | `getMetrics*`, `getStatistics*`, `getEngineStatistics`, `getEngineCount` |
| **Firmware** | 4 | `runFirmwareFlash`, `getFirmwareFlashResult`, `getAMCFirmwareVersions`, `getRedfishAmcWarnMsg` |
| **Configuration** | 11 | `setDevice*`, `getDeviceConfig`, `setAgentConfig`, `getAgentConfig` |
| **Policy** | 3 | `getPolicy`, `setPolicy`, `readPolicyNotifyData` (streaming) |
| **Process** | 4 | `getDeviceProcessState`, `getDeviceUtilizationByProcess`, `getAllDeviceUtilizationByProcess`, `getDeviceComponentOccupancyRatio` |
| **Topology** | 5 | `getTopoXMLBuffer`, `getXelinkTopology`, `getFabricStatistics*`, `getFabricCount` |
| **vGPU** | 5 | `doVgpuPrecheck`, `createVf`, `getDeviceFunction`, `removeAllVf`, `getVfMetrics` |
| **Misc** | 4 | `resetDevice`, `applyPPR`, `genDebugLog`, `getDeviceSerialNumberAndAmcFwVersion` |
| **Dump** | 3 | `startDumpRawDataTask`, `stopDumpRawDataTask`, `listDumpRawDataTasks` |
| **Sensor** | 1 | `getAMCSensorReading` |

### Full Service Definition

```protobuf
service XpumCoreService {
    // Discovery
    rpc getVersion(google.protobuf.Empty) returns (XpumVersionInfoArray);
    rpc getDeviceList(google.protobuf.Empty) returns (XpumDeviceBasicInfoArray);
    rpc getDeviceProperties(DeviceId) returns (XpumDeviceProperties);
    rpc getDeviceIdByBDF(DeviceBDF) returns (DeviceId);
    rpc getTopology(DeviceId) returns (XpumTopologyInfo);

    // Groups
    rpc groupCreate(GroupName) returns (GroupInfo);
    rpc groupDestory(GroupId) returns (GroupInfo);
    rpc groupAddDevice(GroupAddRemoveDevice) returns (GroupInfo);
    rpc groupRemoveDevice(GroupAddRemoveDevice) returns (GroupInfo);
    rpc groupGetInfo(GroupId) returns (GroupInfo);
    rpc getAllGroups(google.protobuf.Empty) returns (GroupArray);

    // Diagnostics
    rpc runDiagnostics(RunDiagnosticsRequest) returns (DiagnosticsTaskInfo);
    rpc runDiagnosticsByGroup(RunDiagnosticsByGroupRequest) returns (DiagnosticsGroupTaskInfo);
    rpc runMultipleSpecificDiagnostics(RunMultipleSpecificDiagnosticsRequest) returns (DiagnosticsTaskInfo);
    rpc runMultipleSpecificDiagnosticsByGroup(RunMultipleSpecificDiagnosticsByGroupRequest) returns (DiagnosticsGroupTaskInfo);
    rpc runStress(RunStressRequest) returns (DiagnosticsTaskInfo);
    rpc getDiagnosticsResult(DeviceId) returns (DiagnosticsTaskInfo);
    rpc getDiagnosticsMediaCodecResult(DeviceId) returns (DiagnosticsMediaCodecInfoArray);
    rpc getDiagnosticsXeLinkThroughputResult(DeviceId) returns (DiagnosticsXeLinkThroughputInfoArray);
    rpc getDiagnosticsResultByGroup(GroupId) returns (DiagnosticsGroupTaskInfo);
    rpc checkStress(CheckStressRequest) returns (CheckStressResponse);
    rpc precheck(PrecheckOptionsRequest) returns (PrecheckComponentInfoListResponse);
    rpc getPrecheckErrorList(google.protobuf.Empty) returns (PrecheckErrorListResponse);

    // Health
    rpc getHealth(HealthDataRequest) returns (HealthData);
    rpc getHealthByGroup(HealthDataByGroupRequest) returns (HealthDataByGroup);
    rpc getHealthConfig(HealthConfigRequest) returns (HealthConfigInfo);
    rpc getHealthConfigByGroup(HealthConfigByGroupRequest) returns (HealthConfigByGroupInfo);
    rpc setHealthConfig(HealthConfigRequest) returns (HealthConfigInfo);
    rpc setHealthConfigByGroup(HealthConfigByGroupRequest) returns (HealthConfigByGroupInfo);

    // Statistics
    rpc getMetrics(DeviceId) returns (DeviceStatsInfoArray);
    rpc getMetricsByGroup(GroupId) returns (DeviceStatsInfoArray);
    rpc getStatistics(XpumGetStatsRequest) returns (XpumGetStatsResponse);
    rpc getEngineStatistics(XpumGetEngineStatsRequest) returns (XpumGetEngineStatsResponse);
    rpc getEngineCount(GetEngineCountRequest) returns (GetEngineCountResponse);
    rpc getStatisticsByGroup(XpumGetStatsByGroupRequest) returns (XpumGetStatsResponse);
    rpc getStatisticsNotForPrometheus(XpumGetStatsRequest) returns (XpumGetStatsResponse);
    rpc getStatisticsByGroupNotForPrometheus(XpumGetStatsByGroupRequest) returns (XpumGetStatsResponse);

    // Firmware
    rpc runFirmwareFlash(XpumFirmwareFlashJob) returns (XpumFirmwareFlashJobResponse);
    rpc getFirmwareFlashResult(XpumFirmwareFlashTaskRequest) returns (XpumFirmwareFlashTaskResult);
    rpc getAMCFirmwareVersions(GetAMCFirmwareVersionsRequest) returns (GetAMCFirmwareVersionsResponse);
    rpc getRedfishAmcWarnMsg(google.protobuf.Empty) returns (GetRedfishAmcWarnMsgResponse);

    // Policy (includes streaming RPC)
    rpc getPolicy(GetPolicyRequest) returns (GetPolicyResponse);
    rpc setPolicy(SetPolicyRequest) returns (SetPolicyResponse);
    rpc readPolicyNotifyData(google.protobuf.Empty) returns (stream ReadPolicyNotifyDataResponse);

    // Configuration
    rpc getDeviceConfig(ConfigDeviceDataRequest) returns (ConfigDeviceData);
    rpc setDeviceSchedulerMode(ConfigDeviceSchdeulerModeRequest) returns (ConfigDeviceResultData);
    rpc setDevicePowerLimit(ConfigDevicePowerLimitRequest) returns (ConfigDeviceResultData);
    rpc setDevicePowerLimitExt(ConfigDevicePowerLimitExtRequest) returns (ConfigDeviceResultData);
    rpc setDeviceFrequencyRange(ConfigDeviceFrequencyRangeRequest) returns (ConfigDeviceResultData);
    rpc setDeviceStandbyMode(ConfigDeviceStandbyRequest) returns (ConfigDeviceResultData);
    rpc setDeviceFabricPortEnabled(ConfigDeviceFabricPortEnabledRequest) returns (ConfigDeviceResultData);
    rpc setDeviceFabricPortBeaconing(ConfigDeviceFabricPortBeconingRequest) returns (ConfigDeviceResultData);
    rpc setDeviceMemoryEccState(ConfigDeviceMemoryEccStateRequest) returns (ConfigDeviceMemoryEccStateResultData);
    rpc setAgentConfig(SetAgentConfigRequest) returns (SetAgentConfigResponse);
    rpc getAgentConfig(google.protobuf.Empty) returns (GetAgentConfigResponse);

    // Process
    rpc getDeviceProcessState(DeviceId) returns (DeviceProcessStateResponse);
    rpc getDeviceComponentOccupancyRatio(DeviceComponentOccupancyRatioRequest) returns (DeviceComponentOccupancyRatioResponse);
    rpc getDeviceUtilizationByProcess(DeviceUtilizationByProcessRequest) returns (DeviceUtilizationByProcessResponse);
    rpc getAllDeviceUtilizationByProcess(UtilizationInterval) returns (DeviceUtilizationByProcessResponse);
    rpc getPerformanceFactor(DeviceDataRequest) returns (DevicePerformanceFactorResponse);
    rpc setPerformanceFactor(PerformanceFactor) returns (DevicePerformanceFactorSettingResponse);

    // Device Control
    rpc resetDevice(ResetDeviceRequest) returns (ResetDeviceResponse);
    rpc applyPPR(ApplyPprRequest) returns (ApplyPprResponse);

    // Dump
    rpc startDumpRawDataTask(StartDumpRawDataTaskRequest) returns (StartDumpRawDataTaskResponse);
    rpc stopDumpRawDataTask(StopDumpRawDataTaskRequest) returns (StopDumpRawDataTaskReponse);
    rpc listDumpRawDataTasks(google.protobuf.Empty) returns (ListDumpRawDataTaskResponse);

    // Topology
    rpc getTopoXMLBuffer(google.protobuf.Empty) returns (TopoXMLResponse);
    rpc getXelinkTopology(google.protobuf.Empty) returns (XpumXelinkTopoInfoArray);
    rpc getFabricStatistics(GetFabricStatsRequest) returns (GetFabricStatsResponse);
    rpc getFabricStatisticsEx(GetFabricStatsExRequest) returns (GetFabricStatsResponse);
    rpc getFabricCount(GetFabricCountRequest) returns (GetFabricCountResponse);

    // Sensor
    rpc getAMCSensorReading(google.protobuf.Empty) returns (GetAMCSensorReadingResponse);
    rpc getDeviceSerialNumberAndAmcFwVersion(GetDeviceSerialNumberRequest) returns (GetDeviceSerialNumberResponse);

    // Debug
    rpc genDebugLog(FileName) returns (GenDebugLogResponse);

    // vGPU
    rpc doVgpuPrecheck(VgpuPrecheckRequest) returns (VgpuPrecheckResponse);
    rpc createVf(VgpuCreateVfRequest) returns (VgpuCreateVfResponse);
    rpc getDeviceFunction(VgpuGetDeviceFunctionRequest) returns (VgpuGetDeviceFunctionResponse);
    rpc removeAllVf(VgpuRemoveAllVfRequest) returns (VgpuRemoveAllVfResponse);
    rpc getVfMetrics(GetVfMetricsRequest) returns (GetVfMetricsResponse);
}
```

---

## Implementation Files

### Server Side

| File | Purpose |
|------|---------|
| `daemon/daemon.cpp` | Main entry, gRPC server setup |
| `daemon/xpum_core_service_impl.h` | Service class declaration |
| `daemon/xpum_core_service_impl.cpp` | Most RPC implementations |
| `daemon/statistics_core_service_impl.cpp` | Statistics RPCs |
| `daemon/firmware_update_service_impl.cpp` | Firmware RPCs |
| `daemon/dump_raw_data_core_service_impl.cpp` | Dump RPCs |
| `daemon/agent_config_core_service_impl.cpp` | Agent config RPCs |

### Client Side (C++)

| File | Purpose |
|------|---------|
| `cli/src/grpc_stub/grpc_core_stub.cpp` | Main gRPC client |
| `cli/src/grpc_stub/devices_stub.cpp` | Device operations |
| `cli/src/grpc_stub/statistics_stub.cpp` | Statistics operations |
| `cli/src/grpc_stub/firmware_stub.cpp` | Firmware operations |
| `cli/src/grpc_stub/dump_stub.cpp` | Dump operations |
| `cli/src/grpc_stub/agentset_stub.cpp` | Agent config operations |

### Client Side (Python)

| File | Purpose |
|------|---------|
| `rest/stub/grpc_stub.py` | Python gRPC client setup |
| `rest/stub/devices.py` | Device operations |
| `rest/stub/statistics.py` | Statistics operations |
| `rest/stub/firmwares.py` | Firmware operations |

---

## Design Patterns

### Error Handling Pattern

Every response includes embedded error fields:

```protobuf
message SomeResponse {
    // ... actual data fields ...
    string errorMsg = N;
    int32 errorNo = N+1;
}
```

### Tile-Aware Pattern

Multi-tile GPUs (Intel Max Series) use:

```protobuf
bool isTileData = X;
int32 tileId = Y;
```

### Array/List Pattern

```protobuf
message FooArray {
    repeated Foo dataList = 1;
    uint32 count = 2;
    string errorMsg = 3;
    int32 errorNo = 4;
}
```

### Request/Response Naming

- `*Request` → Input message
- `*Response` / `*Info` / `*Data` → Output message

---

## Threading Model

```
┌─────────────────────────────────────────────────────┐
│                    Daemon (xpumd)                   │
│                                                     │
│  Main Thread              gRPC Thread Pool          │
│  ┌─────────┐              ┌─────────────────┐       │
│  │ Wait on │              │ Thread per RPC  │       │
│  │ signal  │              │ (sync handlers) │       │
│  └─────────┘              └─────────────────┘       │
│       │                          │                  │
│       │ cv.wait()                │ server->Wait()   │
│       ▼                          ▼                  │
│   [SIGTERM] ──────────────▶ Shutdown()              │
└─────────────────────────────────────────────────────┘
```

gRPC's sync server internally uses a thread pool. Each incoming RPC gets a thread, executes the handler synchronously, then returns the thread.

---

## Adding New Functionality

### Decision Tree

```
Do you want the new functionality accessible via xpumcli/REST API?
│
├── YES → Need to modify core.proto + daemon service
│
└── NO (xpu-smi only) → Just add to C API, no proto change needed
```

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Adding New Functionality                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Daemon Mode (xpumcli)                                      │
│  ┌─────────┐    gRPC     ┌─────────┐    libxpum.so         │
│  │ Client  │ ───────────▶ │ xpumd   │ ───────────────────▶  │
│  │         │◀─────────── │         │   (C API)             │
│  └─────────┘             └─────────┘                       │
│                                                            │
│  Daemon-less Mode (xpu-smi)                                │
│  ┌─────────┐    direct    ┌───────────────────┐            │
│  │ xpu-smi │ ───────────▶ │   libxpum.so      │            │
│  │         │              │   (C API only)    │            │
│  └─────────┘              └───────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Scenario 1: Daemon Mode (Requires Proto Change)

**Use case:** New functionality should be accessible via `xpumcli`, REST API, or Prometheus exporter

| Step | File | Change |
|------|------|--------|
| 1 | `core/include/xpum_api.h` | Add C function declaration |
| 2 | `core/include/xpum_structs.h` | Add structs/enums if needed |
| 3 | `core/src/...` | Implement the C function |
| 4 | **`daemon/core.proto`** | **Add new RPC + request/response messages** |
| 5 | `daemon/xpum_core_service_impl.h` | Add RPC method declaration |
| 6 | `daemon/xpum_core_service_impl.cpp` | Implement RPC (calls C API) |
| 7 | `cli/src/grpc_stub/*.cpp` | Add client stub call |
| 8 | `cli/src/comlet_*.cpp` | Add CLI command |

#### Example: Adding a Simple Query RPC

**1. Proto (`daemon/core.proto`):**
```protobuf
// Add message for new data type
message XpumNewFeatureData {
    uint32 deviceId = 1;
    string someValue = 2;
    uint32 count = 3;
    string errorMsg = 4;
    int32 errorNo = 5;
}

// Add request message
message XpumNewFeatureRequest {
    uint32 deviceId = 1;
    bool someFlag = 2;
}

// Add to service
service XpumCoreService {
    // ... existing RPCs ...
    rpc getNewFeature(XpumNewFeatureRequest) returns (XpumNewFeatureData);
}
```

**2. Daemon Header (`daemon/xpum_core_service_impl.h`):**
```cpp
virtual grpc::Status getNewFeature(
    grpc::ServerContext* context,
    const XpumNewFeatureRequest* request,
    XpumNewFeatureData* response) override;
```

**3. Daemon Impl (`daemon/xpum_core_service_impl.cpp`):**
```cpp
grpc::Status XpumCoreServiceImpl::getNewFeature(
    grpc::ServerContext* context,
    const XpumNewFeatureRequest* request,
    XpumNewFeatureData* response)
{
    xpum_new_feature_data_t data;
    xpum_result_t res = xpumGetNewFeature(request->deviceid(), &data);

    if (res == XPUM_OK) {
        response->set_deviceid(data.deviceId);
        response->set_somevalue(data.someValue);
        response->set_count(data.count);
    } else {
        response->set_errormsg("Error");
    }
    response->set_errorno(res);
    return grpc::Status::OK;
}
```

**4. Client Stub (`cli/src/grpc_stub/...`):**
```cpp
std::unique_ptr<nlohmann::json> GrpcCoreStub::getNewFeature(uint32 deviceId) {
    grpc::ClientContext context;
    XpumNewFeatureRequest request;
    request.set_deviceid(deviceId);
    XpumNewFeatureData response;

    grpc::Status status = stub->getNewFeature(&context, request, &response);

    auto json = std::unique_ptr<nlohmann::json>(new nlohmann::json());
    if (status.ok()) {
        (*json)["device_id"] = response.deviceid();
        (*json)["some_value"] = response.somevalue();
        (*json)["count"] = response.count();
    } else {
        (*json)["error"] = status.error_message();
    }
    return json;
}
```

---

### Scenario 2: Daemon-less Only (No Proto Change)

**Use case:** New functionality is only for `xpu-smi` (daemon-less mode)

| Step | File | Change |
|------|------|--------|
| 1 | `core/include/xpum_api.h` | Add C function declaration |
| 2 | `core/include/xpum_structs.h` | Add structs/enums |
| 3 | `core/src/...` | Implement the C function |
| 4 | `cli/src/core_stub/...` | Add direct C API call |
| 5 | `cli/src/comlet_*.cpp` | Add CLI command |

#### Example: Direct C API Call

**CLI Stub (`cli/src/core_stub/...`):**
```cpp
std::unique_ptr<nlohmann::json> CoreStub::getNewFeature(uint32 deviceId) {
    auto json = std::unique_ptr<nlohmann::json>(new nlohmann::json());

    xpum_new_feature_data_t data;
    xpum_result_t res = xpumGetNewFeature(deviceId, &data);

    if (res == XPUM_OK) {
        (*json)["device_id"] = data.deviceId;
        (*json)["some_value"] = data.someValue;
        (*json)["count"] = data.count;
    } else {
        (*json)["error"] = "Error";
    }
    (*json)["errno"] = res;
    return json;
}
```

---

### Scenario 3: Both Modes (Recommended)

Most features support **both** modes:

1. Implement the C API in `core/` (shared by both)
2. Add proto RPC + daemon wrapper (for xpumcli)
3. Add core_stub call (for xpu-smi)

The CLI switches between stubs using the `DAEMONLESS` macro:

```cpp
#ifdef DAEMONLESS
    // Direct calls to libxpum
    return core_stub->getNewFeature(deviceId);
#else
    // gRPC calls to xpumd
    return grpc_stub->getNewFeature(deviceId);
#endif
```

---

### Quick Reference

| Question | Answer |
|----------|--------|
| New internal helper in `core/` only? | No proto change |
| New xpu-smi command only? | No proto change |
| New xpumcli command? | **Yes, proto change needed** |
| New REST API endpoint? | **Yes, proto change needed** |
| New Prometheus metric? | **Yes, proto change needed** |

---

## Key Technical Details

| Feature | Details |
|---------|---------|
| **Transport** | Unix domain sockets (`/tmp/xpum_p.sock`, `/tmp/xpum_up.sock`) |
| **Numeric IDs** | Devices use `uint32 deviceId`, not BDF strings |
| **Timestamps** | `uint64` Unix epoch (milliseconds) |
| **Scales** | Floating metrics use `uint64 value` + `uint32 scale` |
| **Sessions** | Statistics use `sessionId` for time-windowed aggregation |
| **Auth** | AMC/Redfish RPCs pass `username`/`password` in requests |

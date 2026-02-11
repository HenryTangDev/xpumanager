# XPU Manager Test Plugin Architecture

## Document Version

- **Version**: 1.0
- **Date**: 2025-02-11
- **Status**: Finalized

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Plugin Interface](#plugin-interface)
4. [Plugin Context](#plugin-context)
5. [Plugin Registration & Discovery](#plugin-registration--discovery)
6. [Test Execution Flow](#test-execution-flow)
7. [Error Handling](#error-handling)
8. [Configuration Management](#configuration-management)
9. [Dependency Declaration](#dependency-declaration)
10. [Thread Safety](#thread-safety)
11. [Backward Compatibility](#backward-compatibility)
12. [Build System](#build-system)
13. [Adding New Plugins](#adding-new-plugins)
14. [Plugin Specifications](#plugin-specifications)
15. [Design Decisions](#design-decisions)

---

## Overview

### Problem Statement

The XPU Manager core library (`libxpum.so`) currently contains all diagnostic and test implementations as static methods within `DiagnosticManager`. This creates several issues:

- **Monolithic design**: All tests baked into core library
- **Tight coupling**: Test code directly references core internals
- **Extensibility**: Adding new tests requires modifying core
- **Deployment**: All-or-nothing - cannot add tests without rebuilding core

### Solution: Plugin Architecture

Extract all test implementations into separate plugin `.so` files that are dynamically loaded at runtime. The core library becomes a thin coordinator that discovers and executes plugins.

### Goals

1. **Modularity**: Each test category is a separate plugin
2. **Extensibility**: New plugins can be added without modifying core
3. **Isolation**: Plugin crashes don't crash core
4. **Maintainability**: Tests are self-contained and easier to understand
5. **Zero UI changes**: CLI, daemon, gRPC remain unchanged

### Scope

- **Changes**: Core library (`core/`) only
- **Unchanged**: CLI, daemon, gRPC interface, user experience

---

## Architecture

### High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         User Layer                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ xpu-smi   │  │ xpumcli   │  │ REST API  │              │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────────┘
          │                │                │
          ↓                ↓                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      Service Layer                         │
│  ┌──────────────────────────────────────────────────────┐      │
│  │              xpumd (Daemon)                      │      │
│  │  - gRPC server                                    │      │
│  │  - gRPC service handlers                          │      │
│  │  - Plugin loader coordination                       │      │
│  └──────────────────────┬───────────────────────────────┘      │
│                         │                                   │
└─────────────────────────┼───────────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Core Library (libxpum.so)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │            DiagnosticManager (Coordinator)              │   │
│  │  - Loads plugins from /usr/lib/xpum/tests/        │   │
│  │  - Orchestrates test execution by level             │   │
│  │  - Aggregates results from plugins                  │   │
│  │  - Manages threads & device handles                 │   │
│  │  - Provides PluginContext to plugins                │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │                                   │
│  ┌────────────────────┴──────────────────────────────────────┐   │
│  │              PluginLoader                                │   │
│  │  - Scans plugin directory                            │   │
│  │  - dlopen() shared libraries                          │   │
│  │  - Calls plugin factories                              │   │
│  │  - Builds test type registry                          │   │
│  │  - Manages plugin lifecycle                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                               │
│  Other Core Components (unchanged):                           │
│  - DeviceManager, DataLogic, FirmwareManager, etc.             │
└───────────────────────────────────────────────────────────────────────┘
                          │
                          │ loads
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      Test Plugins                            │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │ libxpum_test_   │  │ libxpum_test_   │                  │
│  │ software.so      │  │ hardware.so      │                  │
│  │ - env vars       │  │ - sysman        │                  │
│  │ - libraries       │  │ - pcie          │                  │
│  │ - permissions     │  └──────────────────┘                  │
│  │ - exclusive      │                                        │
│  └──────────────────┘                                        │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │ libxpum_test_   │  │ libxpum_test_   │                  │
│  │ performance.so   │  │ memory.so       │                  │
│  │ - computation    │  │ - memory error  │                  │
│  │ - power          │  └──────────────────┘                  │
│  │ - memory bw/alloc│                                         │
│  └──────────────────┘                                        │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │ libxpum_test_   │  │ libxpum_test_   │                  │
│  │ media.so        │  │ xelink.so       │                  │
│  │ - codec tests    │  │ - throughput    │                  │
│  └──────────────────┘  │ - all-to-all    │                  │
│  ┌──────────────────┘  └──────────────────┘                  │
│  │ libxpum_test_   │  ┌──────────────────┐                  │
│  │ precheck.so      │  │ libxpum_test_   │                  │
│  │ - kernel logs    │  │ vgpu.so         │                  │
│  │ - gpu status     │  │ - iommu         │                  │
│  │ - pcie val       │  │ - sriov         │                  │
│  └──────────────────┘  └──────────────────┘                  │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │ libxpum_test_   │  │ libxpum_test_   │                  │
│  │ health.so        │  │ stress.so        │                  │
│  │ - thresholds     │  │ - extended       │                  │
│  │ - validation     │  │ - multi-device   │                  │
│  └──────────────────┘  └──────────────────┘                  │
│                                                              │
│  Each plugin:                                               │
│  - Implements TestPlugin interface                             │
│  - Declares supported test types                               │
│  - Independent build (.so)                                    │
│  - Loaded at runtime via dlopen()                              │
└───────────────────────────────────────────────────────────────────────┘
```

### Component Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                    Dependency Graph                         │
│                                                              │
│   CLI ─────► Daemon ─────► DiagnosticManager             │
│                                      │                    │
│                                      ▼                    │
│                              PluginLoader ──────┐        │
│                                      │          │         │        │
│                                      ▼          ▼         ▼        │
│                              TestPlugin ← PluginContext    │        │
│                                      │          │         │        │
│                                      ▼          ▼         │        │
│                              DeviceManager/DataLogic   │        │
│                                                              │
└───────────────────────────────────────────────────────────────────┘
```

---

## Plugin Interface

### Core Interface Definition

**File**: `core/include/xpum_test_plugin.h`

```cpp
#pragma once

#include "xpum_structs.h"
#include <vector>
#include <memory>

namespace xpum {

// Forward declarations
class PluginContext;

/**
 * Test category for plugin organization
 */
enum class TestCategory {
    SOFTWARE,       // Environment, libraries, permissions
    HARDWARE,       // Sysman, PCIe
    PERFORMANCE,     // Computation, power, memory
    MEMORY,         // Memory error testing
    MEDIA,          // Codec testing
    XELINK,         // XeLink throughput
    PRECHECK,       // Pre-diagnostic validation
    VGPU,          // vGPU validation
    HEALTH,         // Health threshold validation
    STRESS,         // Stress testing
    CUSTOM          // User-defined tests
};

/**
 * Plugin interface that all test plugins must implement
 *
 * Lifecycle:
 * 1. Plugin loaded via dlopen()
 * 2. xpum_create_plugin() called
 * 3. init(context) called
 * 4. runTest() called for each test
 * 5. cleanup() called
 * 6. xpum_destroy_plugin() called
 */
class TestPlugin {
public:
    virtual ~TestPlugin() = default;

    /**
     * Get plugin name (for logging/debugging)
     * @return Plugin name string
     */
    virtual const char* getName() const = 0;

    /**
     * Get plugin category
     * @return TestCategory enum value
     */
    virtual TestCategory getCategory() const = 0;

    /**
     * Get test types this plugin supports
     * @return Vector of xpum_diag_task_type_t values
     */
    virtual std::vector<xpum_diag_task_type_t> getTestTypes() const = 0;

    /**
     * Get diagnostic level for a test type
     * @param testType The test type
     * @return XPUM_DIAG_LEVEL_1, LEVEL_2, or LEVEL_3
     */
    virtual xpum_diag_level_t getTestLevel(xpum_diag_task_type_t testType) const = 0;

    /**
     * Initialize plugin (called once after loading)
     * @param context Plugin context providing access to core services
     * @return XPUM_RESULT_SUCCESS on success
     */
    virtual xpum_result_t init(PluginContext* context) = 0;

    /**
     * Run a specific test
     * @param deviceId Target device ID
     * @param testType Which test to run
     * @param result Output structure for test results
     * @return XPUM_RESULT_SUCCESS on success
     */
    virtual xpum_result_t runTest(
        xpum_device_id_t deviceId,
        xpum_diag_task_type_t testType,
        xpum_diag_component_info_t* result
    ) = 0;

    /**
     * Cleanup plugin resources (called before unload)
     */
    virtual void cleanup() = 0;

    /**
     * Check if plugin supports a specific test type
     * @param testType Test type to check
     * @return true if plugin handles this test type
     */
    bool supportsTest(xpum_diag_task_type_t testType) const {
        auto types = getTestTypes();
        return std::find(types.begin(), types.end(), testType) != types.end();
    }
};

/**
 * Factory function signature
 */
typedef TestPlugin* (*CreatePluginFunc)();

/**
 * Destructor function signature
 */
typedef void (*DestroyPluginFunc)(TestPlugin*);

/**
 * Macro to export plugin factory functions
 * Usage: XPUM_EXPORT_PLUGIN(MyPluginClass)
 */
#define XPUM_EXPORT_PLUGIN(PluginClass) \
    extern "C" TestPlugin* xpum_create_plugin() { \
        return new PluginClass(); \
    } \
    extern "C" void xpum_destroy_plugin(TestPlugin* plugin) { \
        delete plugin; \
    }

} // namespace xpum
```

---

## Plugin Context

### Context Interface

**File**: `core/include/xpum_plugin_context.h`

```cpp
#pragma once

#include "xpum_structs.h"
#include <memory>
#include "device/device_manager_interface.h"
#include "data_logic/data_logic_interface.h"

namespace xpum {

/**
 * Configuration value types
 */
enum class ConfigType {
    INTEGER,
    FLOAT,
    STRING,
    BOOLEAN
};

/**
 * Configuration value wrapper
 */
struct ConfigValue {
    ConfigType type;
    union {
        int intValue;
        float floatValue;
        const char* stringValue;
        bool boolValue;
    };
};

/**
 * Context object passed to plugins during initialization
 * Provides controlled access to core services
 */
class PluginContext {
public:
    virtual ~PluginContext() = default;

    // Device Access
    virtual xpum_result_t getDeviceHandles(
        xpum_device_id_t deviceId,
        ze_device_handle_t* zeDevice,
        zes_device_handle_t* zesDevice
    ) = 0;

    virtual ze_driver_handle_t getDriverHandle(xpum_device_id_t deviceId) = 0;

    // Core Service Access
    virtual std::shared_ptr<DeviceManagerInterface> getDeviceManager() = 0;
    virtual std::shared_ptr<DataLogicInterface> getDataLogic() = 0;

    // Resources
    virtual const char* getResourcePath() = 0;
    virtual const char* getPluginPath() = 0;

    // Configuration
    virtual xpum_result_t getConfig(
        const char* key,
        ConfigValue* value
    ) = 0;

    virtual xpum_result_t setConfig(
        const char* key,
        const ConfigValue* value
    ) = 0;

    // Dependency Declaration
    virtual xpum_result_t declareDependency(
        const char* resourceType,
        const char* resourcePath
    ) = 0;

    virtual xpum_result_t checkDependency(
        const char* resourceType
    ) = 0;

    // Logging (uses core's spdlog)
    virtual void log(
        const char* level,
        const char* format, ...
    ) = 0;

    // Plugin Info
    virtual const char* getVersion() = 0;
    virtual bool isDaemonMode() = 0;
};

} // namespace xpum
```

---

## Plugin Registration & Discovery

### Factory Pattern

Each plugin exports two C functions:

```cpp
extern "C" {
    // Create plugin instance
    TestPlugin* xpum_create_plugin();

    // Destroy plugin instance
    void xpum_destroy_plugin(TestPlugin* plugin);
}
```

### Discovery Process

```
┌─────────────────────────────────────────────────────────────┐
│                    PluginLoader                         │
│                                                          │
│  1. Scan plugin directory                                     │
│     scandir("/usr/lib/xpum/tests/")                         │
│     For each: libxpum_test_*.so                            │
│                                                          │
│  2. Load shared library                                     │
│     handle = dlopen(path, RTLD_LAZY | RTLD_LOCAL)          │
│     On error: log warning, continue to next                    │
│                                                          │
│  3. Find factory function                                    │
│     createFunc = dlsym(handle, "xpum_create_plugin")          │
│     If not found: dlclose(), log warning, continue           │
│                                                          │
│  4. Create plugin instance                                   │
│     plugin = createFunc()                                     │
│     If fails: log error, continue                            │
│                                                          │
│  5. Get supported test types                                 │
│     testTypes = plugin->getTestTypes()                        │
│                                                          │
│  6. Register in registry                                    │
│     For each testType:                                      │
│         registry[testType] = plugin                            │
│     Check for duplicates (warn if duplicate)                    │
│                                                          │
│  7. Initialize plugin                                       │
│     result = plugin->init(this)                               │
│     If fails: log error, don't register                      │
│                                                          │
│  8. Store plugin                                            │
│     plugins.push_back(std::unique_ptr<TestPlugin>(plugin))        │
│                                                          │
└──────────────────────────────────────────────────────────────────┘
```

### Registry Structure

```cpp
class PluginLoader {
private:
    // Map each test type to its plugin
    std::map<xpum_diag_task_type_t, TestPlugin*> testTypeRegistry;

    // All loaded plugins (owned by PluginLoader)
    std::vector<std::unique_ptr<TestPlugin>> plugins;

    // Plugin context (passed to all plugins)
    std::unique_ptr<PluginContext> context;

public:
    // Load all plugins from directory
    xpum_result_t loadPlugins(const char* pluginDir);

    // Find plugin for a test type
    TestPlugin* getPluginForTest(xpum_diag_task_type_t testType);

    // Get all plugins
    const std::vector<TestPlugin*>& getPlugins() const;

    // Unload all plugins
    void unloadAll();
};
```

---

## Test Execution Flow

### Diagnostic Manager Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              User runs: xpu-smi diagnostic -l 2          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│              CLI (comlet_diagnostic.cpp)                   │
│  - Parse level argument (2)                               │
│  - Map level to test types                                  │
│  - Call coreStub->runDiagnostics()                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓ (gRPC if daemon mode)
┌─────────────────────────────────────────────────────────────────┐
│              DiagnosticManager                              │
│                                                          │
│  runLevelDiagnostics(deviceId, level=2)                     │
│  {                                                        │
│      for (plugin : plugins) {                            │
│          testTypes = plugin->getTestTypes();               │
│                                                            │
│          for (testType : testTypes) {                     │
│              if (plugin->getTestLevel(testType) == 2) {    │
│                                                            │
│                  // Create result structure                   │
│                  result = new xpum_diag_component_info_t;   │
│                                                            │
│                  // Run test in thread                  │
│                  thread = std::thread([=] {               │
│                      plugin->runTest(deviceId,                  │
│                                        testType,                  │
│                                        result);                │
│                  });                                          │
│                                                            │
│                  threads.push_back(std::move(thread));        │
│              }                                               │
│          }                                                   │
│      }                                                       │
│                                                            │
│      // Wait for all threads                                 │
│      for (thread : threads) thread.join();                  │
│  }                                                          │
│                                                              │
│  // Aggregate results                                      │
│  taskInfo.componentList[i] = *result;                       │
│                                                              │
└───────────────────────────────────────────────────────────────────┘
```

### Plugin Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              TestPlugin::runTest()                          │
│                                                          │
│  1. Get device handles                                    │
│     context->getDeviceHandles(deviceId, &zeDevice, &zesDevice) │
│     if (failed) return XPUM_RESULT_ERROR;               │
│                                                          │
│  2. Get configuration values (thresholds, timeouts)          │
│     context->getConfig("power_threshold", &config);           │
│                                                          │
│  3. Check dependencies                                     │
│     if (!context->checkDependency("kernel_headers"))          │
│         return XPUM_RESULT_ERROR;                           │
│                                                          │
│  4. Run test logic                                       │
│     passed = runMyTestLogic(zeDevice, config);             │
│                                                          │
│  5. Fill result structure                                   │
│     result->type = testType;                                │
│     result->finished = true;                                 │
│     result->result = passed ? XPUM_DIAG_RESULT_PASS           │
│                                : XPUM_DIAG_RESULT_FAIL;      │
│     strncpy(result->message,                                  │
│             passed ? "Test passed" : "Test failed",          │
│             sizeof(result->message));                           │
│                                                          │
│  6. Log completion                                        │
│     context->log("info", "Test completed: %s",            │
│                result->message);                               │
│                                                          │
│  7. Return                                                  │
│     return XPUM_RESULT_SUCCESS;                              │
│                                                              │
└───────────────────────────────────────────────────────────────────┘
```

---

## Error Handling

### Error Scenarios & Behaviors

| Scenario | Detection | Behavior | User Impact |
|----------|------------|------------|-------------|
| Plugin `.so` missing | `scandir()` returns NULL | Warning logged, continue with other plugins | Degrade gracefully |
| Plugin `dlopen()` fails | `dlopen()` returns NULL | Warning logged, continue | Missing tests won't run |
| Missing `xpum_create_plugin` | `dlsym()` returns NULL | Warning logged, `dlclose()`, continue | Plugin not loaded |
| `init()` fails | Returns non-SUCCESS | Error logged, plugin not registered | Plugin unavailable |
| Duplicate test type | Registry finds existing entry | Error logged, first wins, second warned | Only first runs |
| `runTest()` throws | Try-catch in DiagnosticManager | Exception caught, return ERROR | Test marked failed |
| `runTest()` hangs | Thread timeout (future) | Thread terminated, test marked failed | Test timeout |
| Plugin crash during cleanup | Segfault in destructor | Caught by signal handler, logged | Plugin unloaded uncleanly |
| Missing dependency | `checkDependency()` fails | Test returns ERROR, dependency missing logged | Clear error message |

### Error Handling Implementation

```cpp
// In DiagnosticManager::runTest()

xpum_result_t DiagnosticManager::runTest(
    xpum_device_id_t deviceId,
    xpum_diag_task_type_t testType
) {
    try {
        auto plugin = pluginLoader.getPluginForTest(testType);
        if (!plugin) {
            log("error", "No plugin found for test type %d", testType);
            return XPUM_RESULT_ERROR;
        }

        xpum_diag_component_info_t result = {};

        // Run test with exception handling
        xpum_result_t ret = plugin->runTest(deviceId, testType, &result);

        if (ret != XPUM_RESULT_SUCCESS) {
            log("error", "Test %d failed with code %d", testType, ret);
        }

        return ret;

    } catch (const std::exception& e) {
        log("error", "Exception in test %d: %s", testType, e.what());
        return XPUM_RESULT_ERROR;

    } catch (...) {
        log("error", "Unknown exception in test %d", testType);
        return XPUM_RESULT_ERROR;
    }
}
```

---

## Configuration Management

### Configuration Sources (Priority Order)

1. **Plugin-specific config**: `/usr/lib/xpum/tests/<plugin>/config.json`
2. **Global test config**: `/etc/xpum/tests/config.json`
3. **Environment variables**: `XPUM_TEST_<KEY>=<value>`
4. **Default values**: Hardcoded in plugin

### Configuration File Format

**Global** (`/etc/xpum/tests/config.json`):
```json
{
    "version": "1.0",
    "defaults": {
        "power_threshold": 300000,
        "temperature_threshold": 85,
        "memory_threshold": 0.9,
        "timeout_seconds": 300
    },
    "plugins": {
        "media": {
            "test_file_1080p": "/usr/lib/xpum/tests/resources/media/test_1080p.265",
            "test_file_4k": "/usr/lib/xpum/tests/resources/media/test_4k.265"
        },
        "performance": {
            "memory_allocation_size_mb": 1024,
            "computation_iterations": 1000
        }
    }
}
```

**Plugin-specific** (`/usr/lib/xpum/tests/media/config.json`):
```json
{
    "timeout_seconds": 600,
    "requires": ["media_files", "hardware_acceleration"]
}
```

### Configuration Access in Plugins

```cpp
xpum_result_t MyTestPlugin::init(PluginContext* context) {
    // Read configuration
    ConfigValue config;
    xpum_result_t ret = context->getConfig("power_threshold", &config);
    if (ret == XPUM_RESULT_SUCCESS) {
        this->powerThreshold = config.floatValue;
        context->log("info", "Power threshold: %.2f mW", this->powerThreshold);
    } else {
        // Use default
        this->powerThreshold = DEFAULT_POWER_THRESHOLD;
        context->log("info", "Using default power threshold");
    }

    return XPUM_RESULT_SUCCESS;
}
```

---

## Dependency Declaration

### Dependency Types

| Type | Description | Example |
|-------|-------------|---------|
| `media_files` | Test media files required | media plugin |
| `kernel_headers` | Kernel headers needed for compilation | precheck plugin |
| `hardware_acceleration` | Hardware encode/decode support | media plugin |
| `test_resources` | Generic test resources | performance plugin |
| `gpu_device` | Specific GPU type required | xelink plugin |

### Declaration API

```cpp
// In plugin init()

xpum_result_t MyTestPlugin::init(PluginContext* context) {
    xpum_result_t ret;

    // Declare file dependency
    ret = context->declareDependency(
        "media_files",
        "/usr/lib/xpum/tests/resources/media"
    );
    if (ret != XPUM_RESULT_SUCCESS) {
        context->log("error", "Failed to declare media_files dependency");
        return ret;
    }

    // Declare capability dependency
    ret = context->declareDependency(
        "hardware_acceleration",
        "required"
    );
    if (ret != XPUM_RESULT_SUCCESS) {
        context->log("warning", "Hardware acceleration not available");
        // Plugin can still work, with reduced functionality
    }

    return XPUM_RESULT_SUCCESS;
}
```

### Dependency Checking

```cpp
// In PluginContext implementation

xpum_result_t PluginContextImpl::checkDependency(const char* resourceType) {
    if (strcmp(resourceType, "media_files") == 0) {
        // Check if media files exist
        return fileExists("/usr/lib/xpum/tests/resources/media")
            ? XPUM_RESULT_SUCCESS
            : XPUM_RESULT_ERROR;
    }

    if (strcmp(resourceType, "hardware_acceleration") == 0) {
        // Query system for hardware support
        return hasHardwareAcceleration()
            ? XPUM_RESULT_SUCCESS
            : XPUM_RESULT_ERROR;
    }

    return XPUM_RESULT_ERROR;
}
```

---

## Thread Safety

### Threading Model

| Component | Threading | Thread-Safety Required |
|-----------|------------|----------------------|
| **Plugin Loading** | Single-threaded (init only) | No |
| **Plugin Discovery** | Single-threaded | No |
| **Test Execution** | Multi-threaded (one thread per test) | Yes, for plugin |
| **Result Aggregation** | Single-threaded (after all threads join) | No |
| **Plugin Unloading** | Single-threaded (shutdown only) | No |

### Thread Safety Requirements for Plugins

```cpp
class MyTestPlugin : public TestPlugin {
private:
    // Shared state requiring protection
    std::mutex configMutex;
    std::mutex cacheMutex;

    // Thread-local state (no protection needed)
    thread_local xpum_device_id_t currentDeviceId;

public:
    xpum_result_t runTest(
        xpum_device_id_t deviceId,
        xpum_diag_task_type_t testType,
        xpum_diag_component_info_t* result
    ) override {
        // Each call gets its own result struct
        // No need to protect result

        // Protect shared config access
        {
            std::lock_guard<std::mutex> lock(configMutex);
            // Read config
        }

        // Run test logic
        runMyTest(deviceId);

        return XPUM_RESULT_SUCCESS;
    }
};
```

### Thread Creation in DiagnosticManager

```cpp
// Spawn thread for each test
std::vector<std::thread> threads;

for (auto& plugin : plugins) {
    for (auto testType : plugin->getTestTypes()) {
        if (plugin->getTestLevel(testType) == requestedLevel) {

            // Create result struct (per-thread)
            auto result = std::make_unique<xpum_diag_component_info_t>();

            // Spawn thread
            threads.emplace_back([&, plugin, testType, resultPtr=result.get()]() {
                xpum_result_t ret = plugin->runTest(deviceId, testType, resultPtr);
                if (ret != XPUM_RESULT_SUCCESS) {
                    // Log error
                }
            });
        }
    }
}

// Join all threads
for (auto& thread : threads) {
    thread.join();
}
```

---

## Backward Compatibility

### Required vs Optional Plugins

| Plugin | Required | Rationale |
|--------|-----------|-----------|
| software | **Yes** | Basic diagnostics |
| hardware | **Yes** | Hardware validation |
| performance | **Yes** | Performance testing |
| memory | **Yes** | Memory testing |
| media | **No** | Media tools may not be installed |
| xelink | **No** | XeLink may not exist |
| precheck | **No** | Pre-check may not be needed |
| vgpu | **No** | vGPU not always used |
| health | **No** | Threshold validation optional |
| stress | **No** | Stress testing optional |

### Fail-Fast Behavior

```cpp
xpum_result_t PluginLoader::loadPlugins(const char* pluginDir) {
    std::set<std::string> requiredPlugins = {
        "software", "hardware", "performance", "memory"
    };

    bool allRequiredLoaded = true;

    for (auto& pluginName : requiredPlugins) {
        if (!isPluginLoaded(pluginName)) {
            log("error", "Required plugin %s not found", pluginName.c_str());
            allRequiredLoaded = false;
        }
    }

    if (!allRequiredLoaded) {
        return XPUM_RESULT_ERROR_NOT_SUPPORTED;
    }

    return XPUM_RESULT_SUCCESS;
}
```

### Graceful Degradation for Optional Plugins

```cpp
// In DiagnosticManager

xpum_result_t DiagnosticManager::runDiagnostics(...) {
    xpum_result_t ret = XPUM_RESULT_SUCCESS;

    // Try optional plugin
    if (auto plugin = pluginLoader.getPlugin("media")) {
        ret = plugin->runTest(...);
    } else {
        // Log warning, continue
        log("warning", "Media plugin not available, skipping codec tests");
    }

    return ret;
}
```

---

## Build System

### Directory Structure

```
core/
├── include/
│   ├── xpum_test_plugin.h           # Plugin interface
│   └── xpum_plugin_context.h        # Plugin context
├── src/
│   ├── diagnostic/
│   │   ├── diagnostic_manager.cpp    # Modified: uses plugins
│   │   ├── plugin_loader.cpp        # Plugin loading
│   │   ├── plugin_loader.h
│   │   ├── plugin_context.cpp       # Context implementation
│   │   └── plugin_context.h
│   └── tests/                       # Test plugins
│       ├── software/
│       │   ├── software_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── hardware/
│       │   ├── hardware_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── performance/
│       │   ├── performance_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── memory/
│       │   ├── memory_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── media/
│       │   ├── media_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── xelink/
│       │   ├── xelink_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── precheck/
│       │   ├── precheck_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── vgpu/
│       │   ├── vgpu_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── health/
│       │   ├── health_plugin.cpp
│       │   └── CMakeLists.txt
│       ├── stress/
│       │   ├── stress_plugin.cpp
│       │   └── CMakeLists.txt
│       └── CMakeLists.txt            # Plugin build orchestration
└── CMakeLists.txt                    # Modified: builds plugins
```

### CMakeLists.txt Files

**core/src/tests/CMakeLists.txt**:
```cmake
# Test plugin directory
set(XPUM_TEST_PLUGIN_DIR ${CMAKE_INSTALL_PREFIX}/lib/xpum/tests)

# Add each test plugin
add_subdirectory(software)
add_subdirectory(hardware)
add_subdirectory(performance)
add_subdirectory(memory)
add_subdirectory(media)
add_subdirectory(xelink)
add_subdirectory(precheck)
add_subdirectory(vgpu)
add_subdirectory(health)
add_subdirectory(stress)
```

**core/src/tests/software/CMakeLists.txt** (example):
```cmake
# Software test plugin
add_library(xpum_test_software SHARED software_plugin.cpp)

# Set library properties
set_target_properties(xpum_test_software PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
    OUTPUT_NAME xpum_test_software
    PREFIX lib
)

# Include directories
target_include_directories(xpum_test_software
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/../../../include
        ${CMAKE_CURRENT_SOURCE_DIR}/../../include
)

# Link libraries
target_link_libraries(xpum_test_software
    PRIVATE
        xpum                    # Core library (for PluginContext)
        ze_loader               # Level Zero loader
        ${CMAKE_DL_LIBS}       # dlopen
)

# Install
install(TARGETS xpum_test_software
    LIBRARY DESTINATION ${XPUM_TEST_PLUGIN_DIR}
)
```

**core/CMakeLists.txt** (add at end):
```cmake
# Build test plugins
if(NOT DAEMONLESS)
    add_subdirectory(src/tests)
endif()
```

---

## Adding New Plugins

### Step-by-Step Example: Adding "Temperature Test" Plugin

#### Step 1: Create Plugin Directory

```bash
cd core/src/tests/
mkdir temperature
cd temperature
```

#### Step 2: Implement Plugin

**temperature/temperature_plugin.cpp**:
```cpp
#include "xpum_test_plugin.h"
#include "xpum_plugin_context.h"

using namespace xpum;

class TemperatureTestPlugin : public TestPlugin {
private:
    PluginContext* context;
    float temperatureThreshold;

public:
    const char* getName() const override {
        return "TemperatureTest";
    }

    TestCategory getCategory() const override {
        return TestCategory::CUSTOM;
    }

    std::vector<xpum_diag_task_type_t> getTestTypes() const override {
        return {XPUM_DIAG_CUSTOM_TEMPERATURE};
    }

    xpum_diag_level_t getTestLevel(xpum_diag_task_type_t testType) const override {
        (void)testType;
        return XPUM_DIAG_LEVEL_1;  // Quick test
    }

    xpum_result_t init(PluginContext* ctx) override {
        this->context = ctx;

        // Get configuration
        ConfigValue config;
        if (context->getConfig("temp_threshold", &config) == XPUM_RESULT_SUCCESS) {
            this->temperatureThreshold = config.floatValue;
        } else {
            this->temperatureThreshold = 85.0f;  // Default
        }

        return XPUM_RESULT_SUCCESS;
    }

    xpum_result_t runTest(
        xpum_device_id_t deviceId,
        xpum_diag_task_type_t testType,
        xpum_diag_component_info_t* result
    ) override {
        // Get device
        ze_device_handle_t zeDevice;
        zes_device_handle_t zesDevice;
        xpum_result_t ret = context->getDeviceHandles(deviceId, &zeDevice, &zesDevice);
        if (ret != XPUM_RESULT_SUCCESS) {
            strcpy(result->message, "Failed to get device handles");
            result->result = XPUM_DIAG_RESULT_FAIL;
            return ret;
        }

        // Run temperature test
        float currentTemp = getGPUTemperature(zesDevice);

        bool passed = (currentTemp < temperatureThreshold);

        result->type = testType;
        result->finished = true;
        result->result = passed ? XPUM_DIAG_RESULT_PASS : XPUM_DIAG_RESULT_FAIL;
        snprintf(result->message, XPUM_MAX_STR_LENGTH,
                 "Temperature: %.1f°C (threshold: %.1f°C) - %s",
                 currentTemp, temperatureThreshold,
                 passed ? "PASS" : "FAIL");

        return XPUM_RESULT_SUCCESS;
    }

    void cleanup() override {
        // Nothing to clean up
    }

private:
    float getGPUTemperature(zes_device_handle_t device) {
        // Implementation details...
        return 42.0f;
    }
};

XPUM_EXPORT_PLUGIN(TemperatureTestPlugin)
```

#### Step 3: Create Build Config

**temperature/CMakeLists.txt**:
```cmake
add_library(xpum_test_temperature SHARED temperature_plugin.cpp)

set_target_properties(xpum_test_temperature PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
    OUTPUT_NAME xpum_test_temperature
    PREFIX lib
)

target_include_directories(xpum_test_temperature
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/../../../include
        ${CMAKE_CURRENT_SOURCE_DIR}/../../include
)

target_link_libraries(xpum_test_temperature
    PRIVATE
        xpum
        ze_loader
        ${CMAKE_DL_LIBS}
)

install(TARGETS xpum_test_temperature
    LIBRARY DESTINATION ${XPUM_TEST_PLUGIN_DIR}
)
```

#### Step 4: Add to Build

**core/src/tests/CMakeLists.txt** - add line:
```cmake
add_subdirectory(temperature)
```

#### Step 5: Define Test Type (if new)

**core/include/xpum_structs.h**:
```cpp
typedef enum xpum_diag_task_type_enum {
    // ... existing types ...
    XPUM_DIAG_TASK_TYPE_MAX = 16,
    XPUM_DIAG_CUSTOM_TEMPERATURE = 100,  // Custom range
    XPUM_DIAG_CUSTOM_MAX
} xpum_diag_task_type_t;
```

#### Step 6: Build & Install

```bash
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
sudo make install

# Plugin installs to:
# /usr/lib/xpum/tests/libxpum_test_temperature.so
```

#### Step 7: Run Test

```bash
# Plugin is auto-discovered
xpu-smi diagnostic --singletest 100

# Or define in config for by-name lookup
```

---

## Plugin Specifications

### 1. Software Test Plugin

**Library**: `libxpum_test_software.so`

**Tests**:
- `XPUM_DIAG_SOFTWARE_ENV_VARIABLES` - Environment variables validation
- `XPUM_DIAG_SOFTWARE_LIBRARY` - Library dependency checks
- `XPUM_DIAG_SOFTWARE_PERMISSION` - File/device permissions
- `XPUM_DIAG_SOFTWARE_EXCLUSIVE` - Exclusive process checks

**Level**: 1 (quick)

**Source**: `diagnostic/`

### 2. Hardware Test Plugin

**Library**: `libxpum_test_hardware.so`

**Tests**:
- `XPUM_DIAG_HARDWARE_SYSMAN` - Sysman validation
- `XPUM_DIAG_INTEGRATION_PCIE` - PCIe integration tests

**Level**: 2 (medium)

**Source**: `diagnostic/`

### 3. Performance Test Plugin

**Library**: `libxpum_test_performance.so`

**Tests**:
- `XPUM_DIAG_LIGHT_COMPUTATION` - Quick computation test
- `XPUM_DIAG_PERFORMANCE_COMPUTATION` - Full computation test
- `XPUM_DIAG_PERFORMANCE_POWER` - Power/performance test
- `XPUM_DIAG_PERFORMANCE_MEMORY_BANDWIDTH` - Memory bandwidth test
- `XPUM_DIAG_PERFORMANCE_MEMORY_ALLOCATION` - Memory allocation test

**Level**: 1, 3 (light=1, full=3)

**Source**: `diagnostic/`

### 4. Memory Test Plugin

**Library**: `libxpum_test_memory.so`

**Tests**:
- `XPUM_DIAG_MEMORY_ERROR` - Memory error detection test

**Level**: 3 (long)

**Source**: `diagnostic/`

### 5. Media Test Plugin

**Library**: `libxpum_test_media.so`

**Tests**:
- `XPUM_DIAG_LIGHT_CODEC` - Light codec test
- `XPUM_DIAG_MEDIA_CODEC` - Full codec test

**Level**: 1, 2 (light=1, full=2)

**Source**: `diagnostic/`

**Dependencies**: Media test files

### 6. XeLink Test Plugin

**Library**: `libxpum_test_xelink.so`

**Tests**:
- `XPUM_DIAG_XE_LINK_THROUGHPUT` - Point-to-point throughput
- `XPUM_DIAG_XE_LINK_ALL_TO_ALL_THROUGHPUT` - All-to-all throughput

**Level**: 3 (long)

**Source**: `diagnostic/`

### 7. Precheck Test Plugin

**Library**: `libxpum_test_precheck.so`

**Tests** (custom types):
- Kernel log scanning
- GPU firmware status (GuC, HuC)
- PCIe device validation
- GPU wedged state detection
- Memory initialization check
- CPU temperature check
- Driver loading validation

**Level**: Pre-diagnostic (before levels)

**Source**: `diagnostic/precheck/`

### 8. vGPU Test Plugin

**Library**: `libxpum_test_vgpu.so`

**Tests** (custom types):
- IOMMU enablement check
- SR-IOV configuration validation
- vGPU environment validation

**Level**: Pre-diagnostic

**Source**: `vgpu/precheck/`

### 9. Health Test Plugin

**Library**: `libxpum_test_health.so`

**Tests** (custom types):
- Power threshold validation
- Temperature threshold validation
- Memory health checks
- Fabric port health checks

**Level**: Validation (independent)

**Source**: `health/`

### 10. Stress Test Plugin

**Library**: `libxpum_test_stress.so`

**Tests** (custom types):
- Extended stress testing
- GPU stress computation
- Memory stress
- Power stress
- Multi-device coordination

**Level**: Extended (user-defined duration)

**Source**: `diagnostic/`

---

## Design Decisions

### Decision 1: Memory Plugin Scope

**Question**: Should memory plugin (1 test) be merged with performance (5 tests)?

**Decision**: **Keep separate**

**Rationale**:
- Memory error testing is distinct from performance benchmarks
- Memory error detects hardware faults
- Performance tests measure capabilities
- Different purposes, different audiences

### Decision 2: Health Plugin Necessity

**Question**: Is health threshold validation really a "test"?

**Decision**: **Keep health plugin**

**Rationale**:
- Threshold validation is test-like activity
- Validates configuration against actual hardware
- Separate from runtime monitoring (which stays in core)
- Useful for deployment verification

### Decision 3: Stress Test Location

**Question**: Should stress test be separate plugin or part of performance?

**Decision**: **Separate stress plugin**

**Rationale**:
- Stress testing is distinct from benchmarking
- Focus on endurance vs peak performance
- Longer duration, different resource patterns
- Different use case (burn-in vs characterization)

### Decision 4: Custom Test Types

**Question**: Should precheck/vgpu/health/stress types be added to main enum?

**Decision**: **Keep custom types separate**

**Rationale**:
- Main enum (`xpum_diag_task_type_t`) is public API
- Don't want to bloat public API with implementation details
- Plugins define their own custom types
- Cleaner separation of concerns
- Custom types use separate enum range (100+)

### Decision 5: Plugin Dependencies

**Question**: Should plugins declare dependencies explicitly?

**Decision**: **Yes, declare dependencies**

**Rationale**:
- Clear dependency documentation
- Better error messages
- Allows graceful degradation
- Helps with packaging/deployment
- Supports validation before test execution

### Decision 6: Configuration Management

**Question**: How are configuration values passed to plugins?

**Decision**: **Keep current approach (centralized)**

**Rationale**:
- Consistent with current architecture
- Thresholds managed centrally
- Easier for operators to configure
- PluginContext provides `getConfig()` interface
- Configuration from files + environment variables

### Decision 7: Backward Compatibility

**Question**: Fail-fast or degrade gracefully for missing plugins?

**Decision**: **Fail-fast for required, degrade for optional**

**Rationale**:
- Required plugins (software, hardware, performance, memory) must be present
- Clear error if missing
- Optional plugins (media, xelink, precheck, vgpu, health, stress) can be missing
- Warning logged for missing optional plugins
- Tests continue without optional plugins

---

## Summary

### Key Benefits

| Benefit | Description |
|----------|-------------|
| **Modularity** | Each test category is isolated |
| **Extensibility** | New tests added without modifying core |
| **Maintainability** | Tests are self-contained |
| **Deployment** | Plugins can be updated independently |
| **Development** | Plugins can be developed in separate repos |
| **Testing** | Each plugin can be tested independently |
| **Zero UI changes** | CLI, daemon, gRPC unchanged |

### Migration Path

1. **Phase 1**: Implement plugin interface and loader
2. **Phase 2**: Migrate existing tests to plugins
3. **Phase 3**: Deploy and validate
4. **Phase 4**: Add new tests as plugins directly

### Files Changed

| File | Change Type | Lines Added |
|-------|--------------|--------------|
| `core/include/xpum_test_plugin.h` | New | ~200 |
| `core/include/xpum_plugin_context.h` | New | ~150 |
| `core/src/diagnostic/plugin_loader.cpp` | New | ~400 |
| `core/src/diagnostic/plugin_loader.h` | New | ~80 |
| `core/src/diagnostic/plugin_context.cpp` | New | ~200 |
| `core/src/diagnostic/plugin_context.h` | New | ~60 |
| `core/src/diagnostic/diagnostic_manager.cpp` | Modified | -2000, +200 |
| `core/src/tests/*/CMakeLists.txt` | New (11 files) | ~50 each |
| `core/src/tests/*/*_plugin.cpp` | New (11 files) | ~500 each |
| `core/src/tests/CMakeLists.txt` | New | ~20 |
| `core/CMakeLists.txt` | Modified | +5 |

### Testing Strategy

1. **Unit tests**: Plugin loader, context
2. **Integration tests**: Plugin discovery, registration
3. **Functional tests**: Each plugin independently
4. **End-to-end tests**: Full diagnostic flow with plugins
5. **Performance tests**: Plugin overhead measurement

---

**End of Architecture Document**

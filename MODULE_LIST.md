# XPU Manager Module Files — Complete List

## Tier 1: Built-in Modules (10 modules)

Installed to `/usr/lib/xpum/modules/`

| # | Module Name | .so Filename | Module ID | Purpose |
|---|---|---|---|---|
| 1 | Monitor | `libxpum_mod_monitor.so` | 1 | Telemetry collection and metric aggregation |
| 2 | Health | `libxpum_mod_health.so` | 2 | Health monitoring with threshold checking |
| 3 | Diagnostic | `libxpum_mod_diag.so` | 3 | Diagnostic test orchestration |
| 4 | Firmware | `libxpum_mod_firmware.so` | 4 | Firmware flash operations (GFX, AMC) |
| 5 | Configuration | `libxpum_mod_config.so` | 5 | GPU power, frequency, ECC configuration |
| 6 | Topology | `libxpum_mod_topology.so` | 6 | Hardware topology via hwloc |
| 7 | Group | `libxpum_mod_group.so` | 7 | Device grouping for batch operations |
| 8 | Policy | `libxpum_mod_policy.so` | 8 | Policy engine for automated actions |
| 9 | Dump | `libxpum_mod_dump.so` | 9 | Raw data export to file/network |
| 10 | vGPU | `libxpum_mod_vgpu.so` | 10 | SR-IOV virtual GPU management |

**Total Tier 1 modules: 10**

---

## Tier 2: Extension Plugins (11 plugins)

Installed to `/usr/lib/xpum/plugins/` — discovered at runtime

| # | Plugin Name | .so Filename | Module ID | Capability Flags | Description |
|---|---|---|---|---|---|---|---|---|
| 1 | Xe2 Performance | `libxpum_plugin_xe2_perf.so` | Dynamic (32+) | DIAGNOSTIC | Xe2 GPU family-specific performance tests, bandwidth measurement |
| 2 | AMC Advanced | `libxpum_plugin_amc.so` | Dynamic (33+) | DIAGNOSTIC | Enhanced AMC monitoring beyond firmware module |
| 3 | OAM Monitor | `libxpum_plugin_oam_monitor.so` | Dynamic (34+) | MONITORING | OEM OAM telemetry collection |
| 4 | Fabric Topology | `libxpum_plugin_fabric_topology.so` | Dynamic (35+) | MONITORING | Custom fabric interconnect analysis |
| 5 | Power Model | `libxpum_plugin_power_model.so` | Dynamic (36+) | MONITORING | Device-specific power modeling |
| 6 | Custom Health | `libxpum_plugin_custom_health.so` | Dynamic (37+) | HEALTH | Vendor-specific health rules beyond generic module |
| 7 | DDRT Diagnostics | `libxpum_plugin_ddrt_diag.so` | Dynamic (38+) | DIAGNOSTIC | Diagnostic tests using DDRT library |
| 8 | RAPL Plugin | `libxpum_plugin_rapl_plugin.so` | Dynamic (39+) | MONITORING | Running Average Power Limit accuracy measurements |
| 9 | Profiler | `libxpum_plugin_profiler.so` | Dynamic (40+) | MONITORING | GPU sampling, profiling, and performance analysis |
| 10 | Workload Optimizer | `libxpum_plugin_workload_optimizer.so` | Dynamic (41+) | POLICY | Workload recommendation engine |
| 11 | Telemetry Exporter | `libxpum_plugin_telemetry_exporter.so` | Dynamic (42+) | MONITORING | Export metrics in custom formats (Prometheus, InfluxDB, etc.) |

**Total Tier 2 plugins: 11**

---

## Installation Layout

```
/usr/lib/xpum/
├── modules/                    # Tier 1: Built-in modules (10 .so files)
│   ├── libxpum_mod_monitor.so
│   ├── libxpum_mod_health.so
│   ├── libxpum_mod_diag.so
│   ├── libxpum_mod_firmware.so
│   ├── libxpum_mod_config.so
│   ├── libxpum_mod_topology.so
│   ├── libxpum_mod_group.so
│   ├── libxpum_mod_policy.so
│   ├── libxpum_mod_dump.so
│   └── libxpum_mod_vgpu.so
│
└── plugins/                    # Tier 2: Extension plugins (11 .so files)
    ├── libxpum_plugin_xe2_perf.so
    ├── libxpum_plugin_amc.so
    ├── libxpum_plugin_oam_monitor.so
    ├── libxpum_plugin_fabric_topology.so
    ├── libxpum_plugin_power_model.so
    ├── libxpum_plugin_custom_health.so
    ├── libxpum_plugin_ddrt_diag.so
    ├── libxpum_plugin_rapl_plugin.so
    ├── libxpum_plugin_profiler.so
    ├── libxpum_plugin_workload_optimizer.so
    └── libxpum_plugin_telemetry_exporter.so
```

## Key Design Principles

- **Built-in modules** are statically linked into `libxpum_core.so` at compile time
- **Extension plugins** are dynamically loaded via `dlopen()` at runtime
- All modules communicate via **message-based API** through `ModuleManager`
- All modules use **CoreProxy** to access shared services (DeviceManager, FieldCache, logging)
- **Module IDs 0-31** reserved for built-in modules
- **Module IDs 32+** assigned dynamically to Tier 2 plugins
- **Capability flags** determine which modules can handle which requests

## Migration Summary

**Current State**: All functionality is in `libxpum.so` as manager classes
**Proposed State**: Extract into 21 separate `.so` files (10 built-in + 11 plugins)

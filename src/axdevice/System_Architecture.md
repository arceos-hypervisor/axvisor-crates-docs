# System Architecture

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/Cargo.toml)
> * [src/config.rs](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/config.rs)
> * [src/device.rs](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs)
> * [src/lib.rs](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/lib.rs)

## Purpose and Scope

This document provides a comprehensive view of the `axdevice` system architecture, detailing how the configuration and device management components work together to provide virtual machine device emulation within the ArceOS hypervisor ecosystem. The architecture encompasses MMIO operation handling, device lifecycle management, and integration with external ArceOS components.

For detailed information about individual core components, see [Core Components](/arceos-hypervisor/axdevice/3-core-components). For ArceOS ecosystem integration details, see [ArceOS Ecosystem Integration](/arceos-hypervisor/axdevice/4-arceos-ecosystem-integration).

## High-Level System Architecture

The `axdevice` crate implements a two-layer architecture consisting of configuration management and device emulation orchestration:

### Core Architecture Overview

```mermaid
flowchart TD
subgraph subGraph2["Guest VM Interface"]
    MMIO_Read["handle_mmio_read()"]
    MMIO_Write["handle_mmio_write()"]
    find_dev["find_dev()"]
end
subgraph subGraph1["External Dependencies"]
    EmulatedDeviceConfig["EmulatedDeviceConfig(from axvmconfig)"]
    BaseDeviceOps["BaseDeviceOps(from axdevice_base)"]
    GuestPhysAddr["GuestPhysAddr(from axaddrspace)"]
end
subgraph subGraph0["axdevice Core"]
    AxVmDeviceConfig["AxVmDeviceConfigConfiguration Container"]
    AxVmDevices["AxVmDevicesDevice Orchestrator"]
    emu_devices["emu_devices: Vec<Arc<dyn BaseDeviceOps>>Device Collection"]
end

AxVmDeviceConfig --> AxVmDevices
AxVmDevices --> MMIO_Read
AxVmDevices --> MMIO_Write
AxVmDevices --> emu_devices
AxVmDevices --> find_dev
BaseDeviceOps --> emu_devices
EmulatedDeviceConfig --> AxVmDeviceConfig
GuestPhysAddr --> MMIO_Read
GuestPhysAddr --> MMIO_Write
GuestPhysAddr --> find_dev
```

Sources: [src/config.rs(L5 - L8)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/config.rs#L5-L8) [src/device.rs(L12 - L16)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L12-L16) [src/device.rs(L57 - L62)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L57-L62) [src/device.rs(L65 - L75)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L65-L75) [src/device.rs(L78 - L92)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L78-L92)

## Component Interaction Flow

The system follows a clear initialization and runtime operation pattern:

### Device Initialization and MMIO Handling Flow

```mermaid
sequenceDiagram
    participant AxVmDeviceConfig as "AxVmDeviceConfig"
    participant AxVmDevices as "AxVmDevices"
    participant emu_devices as "emu_devices"
    participant GuestVM as "Guest VM"
    participant BaseDeviceOps as "BaseDeviceOps"

    Note over AxVmDeviceConfig,BaseDeviceOps: Initialization Phase
    AxVmDeviceConfig ->> AxVmDevices: "new(config)"
    AxVmDevices ->> AxVmDevices: "init(&mut self, &emu_configs)"
    Note over AxVmDevices: "Currently stub implementation"
    Note over AxVmDeviceConfig,BaseDeviceOps: Runtime MMIO Operations
    GuestVM ->> AxVmDevices: "handle_mmio_read(addr, width)"
    AxVmDevices ->> AxVmDevices: "find_dev(addr)"
    AxVmDevices ->> emu_devices: "iter().find(|dev| dev.address_range().contains(addr))"
    emu_devices -->> AxVmDevices: "Some(Arc<dyn BaseDeviceOps>)"
    AxVmDevices ->> BaseDeviceOps: "handle_read(addr, width)"
    BaseDeviceOps -->> AxVmDevices: "AxResult<usize>"
    AxVmDevices -->> GuestVM: "Return value"
    GuestVM ->> AxVmDevices: "handle_mmio_write(addr, width, val)"
    AxVmDevices ->> AxVmDevices: "find_dev(addr)"
    AxVmDevices ->> BaseDeviceOps: "handle_write(addr, width, val)"
    BaseDeviceOps -->> AxVmDevices: "Operation complete"
    AxVmDevices -->> GuestVM: "Write acknowledged"
```

Sources: [src/device.rs(L20 - L28)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L20-L28) [src/device.rs(L30 - L54)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L30-L54) [src/device.rs(L57 - L62)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L57-L62) [src/device.rs(L65 - L75)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L65-L75) [src/device.rs(L78 - L92)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L78-L92)

## Core Data Structures and Relationships

### Configuration and Device Management Structure

```mermaid
classDiagram
class AxVmDeviceConfig {
    +emu_configs: Vec~EmulatedDeviceConfig~
    +new(emu_configs: Vec~EmulatedDeviceConfig~) Self
}

class AxVmDevices {
    -emu_devices: Vec~Arc~dyn BaseDeviceOps~~
    +new(config: AxVmDeviceConfig) Self
    -init(&mut self, emu_configs: &Vec~EmulatedDeviceConfig~)
    +find_dev(ipa: GuestPhysAddr) Option~Arc~dyn BaseDeviceOps~~
    +handle_mmio_read(addr: GuestPhysAddr, width: usize) AxResult~usize~
    +handle_mmio_write(addr: GuestPhysAddr, width: usize, val: usize)
}

class EmulatedDeviceConfig {
    <<external>>
    +emu_type: field
    
}

class BaseDeviceOps {
    <<trait>>
    
    +address_range() Range
    +handle_read(addr: GuestPhysAddr, width: usize) AxResult~usize~
    +handle_write(addr: GuestPhysAddr, width: usize, val: usize)
}

AxVmDeviceConfig  *--  EmulatedDeviceConfig : "contains"
AxVmDevices  o--  AxVmDeviceConfig : "initialized from"
AxVmDevices  *--  BaseDeviceOps : "manages collection"
```

Sources: [src/config.rs(L5 - L16)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/config.rs#L5-L16) [src/device.rs(L12 - L16)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L12-L16) [src/device.rs(L20 - L28)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L20-L28) [src/device.rs(L57 - L62)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L57-L62)

## MMIO Operation Handling Architecture

The system's core functionality centers around Memory-Mapped I/O (MMIO) operation routing:

### MMIO Address Resolution and Device Routing

|Operation|Method|Address Resolution|Device Interaction|
| --- | --- | --- | --- |
|Read|handle_mmio_read|find_dev(addr)→address_range().contains(addr)|device.handle_read(addr, width)|
|Write|handle_mmio_write|find_dev(addr)→address_range().contains(addr)|device.handle_write(addr, width, val)|

The address resolution process follows this pattern:

```mermaid
flowchart TD
MMIO_Request["MMIO Request(addr: GuestPhysAddr)"]
find_dev["find_dev(addr)"]
device_iter["emu_devices.iter()"]
address_check["dev.address_range().contains(addr)"]
device_found["Device FoundArc<dyn BaseDeviceOps>"]
device_missing["No Device Foundpanic!"]
handle_operation["handle_read() orhandle_write()"]

MMIO_Request --> find_dev
address_check --> device_found
address_check --> device_missing
device_found --> handle_operation
device_iter --> address_check
find_dev --> device_iter
```

Sources: [src/device.rs(L57 - L62)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L57-L62) [src/device.rs(L65 - L75)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L65-L75) [src/device.rs(L78 - L92)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L78-L92)

## Dependency Architecture and External Integration

The system integrates with multiple ArceOS ecosystem components:

### External Dependencies and Integration Points

```mermaid
flowchart TD
subgraph subGraph3["Standard Dependencies"]
    log_crate["logLogging Macros"]
    cfg_if["cfg-ifConditional Compilation"]
end
subgraph subGraph2["ArceOS System Components"]
    axerrno["axerrnoAxResult"]
    memory_addr["memory_addrMemory Addressing"]
end
subgraph subGraph1["ArceOS Hypervisor Components"]
    axvmconfig["axvmconfigEmulatedDeviceConfig"]
    axaddrspace["axaddrspaceGuestPhysAddr"]
    axdevice_base["axdevice_baseBaseDeviceOps"]
end
subgraph subGraph0["axdevice Implementation"]
    Config["AxVmDeviceConfig"]
    Devices["AxVmDevices"]
end

axaddrspace --> Devices
axdevice_base --> Devices
axerrno --> Devices
axvmconfig --> Config
cfg_if --> Config
log_crate --> Devices
memory_addr --> Devices
```

### Dependency Details

|Component|Source|Purpose|Usage|
| --- | --- | --- | --- |
|EmulatedDeviceConfig|axvmconfig|Device configuration data|Stored inAxVmDeviceConfig.emu_configs|
|GuestPhysAddr|axaddrspace|Guest physical addressing|MMIO operation parameters|
|BaseDeviceOps|axdevice_base|Device operation interface|Stored asArc<dyn BaseDeviceOps>|
|AxResult|axerrno|Error handling|Return type forhandle_mmio_read|

Sources: [Cargo.toml(L8 - L18)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/Cargo.toml#L8-L18) [src/config.rs(L2)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/config.rs#L2-L2) [src/device.rs(L6 - L9)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L6-L9) [src/lib.rs(L11 - L13)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/lib.rs#L11-L13)

## Current Implementation Status

The system architecture includes placeholder implementations for future device types:

### Device Type Support (Planned Implementation)

The `init` method in `AxVmDevices` contains commented code indicating planned support for multiple device types:

* Console devices (`EmuDeviceTConsole`)
* VirtIO devices (`EmuDeviceTVirtioBlk`, `EmuDeviceTVirtioNet`, `EmuDeviceTVirtioConsole`)
* Interrupt controllers (`EmuDeviceTGicdV2`, `EmuDeviceTGICR`)
* IOMMU (`EmuDeviceTIOMMU`)
* Other specialized devices (`EmuDeviceTGPPT`, `EmuDeviceTICCSRE`, `EmuDeviceTSGIR`, `EmuDeviceTMeta`)

Sources: [src/device.rs(L30 - L54)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L30-L54)
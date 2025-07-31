# Device Emulation

> **Relevant source files**
> * [src/device.rs](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs)
> * [src/lib.rs](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/lib.rs)

## Purpose and Scope

This document covers the device emulation system implemented by the `AxVmDevices` struct, which serves as the central orchestrator for all emulated devices within a virtual machine. The system handles Memory-Mapped I/O (MMIO) operations from guest VMs, manages device lifecycles, and routes operations to appropriate device implementations through the `BaseDeviceOps` trait interface.

For information about device configuration management, see [Configuration Management](/arceos-hypervisor/axdevice/3.1-configuration-management). For broader system integration details, see [ArceOS Ecosystem Integration](/arceos-hypervisor/axdevice/4-arceos-ecosystem-integration).

## AxVmDevices Architecture

The `AxVmDevices` struct acts as the primary interface between guest virtual machines and emulated devices. It maintains a collection of device implementations and provides unified access to MMIO operations.

### Core Structure

```mermaid
flowchart TD
subgraph subGraph2["Configuration Input"]
    H["AxVmDeviceConfig"]
    I["emu_configs: Vec"]
end
subgraph Dependencies["Dependencies"]
    D["axaddrspace::GuestPhysAddr"]
    E["axdevice_base::BaseDeviceOps"]
    F["axerrno::AxResult"]
    G["axvmconfig::EmulatedDeviceConfig"]
end
subgraph subGraph0["AxVmDevices Structure"]
    A["AxVmDevices"]
    B["emu_devices: Vec>"]
    C["TODO: passthrough devices"]
end

A --> B
A --> C
A --> D
A --> E
A --> F
H --> A
H --> I
I --> A
```

Sources: [src/device.rs(L1 - L16)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L1-L16)

The `AxVmDevices` struct currently focuses on emulated devices through the `emu_devices` field, with placeholders for future passthrough device support.

### Device Collection Management

```mermaid
flowchart TD
subgraph subGraph1["Trait Interface"]
    E["BaseDeviceOps"]
    F["address_range()"]
    G["handle_read()"]
    H["handle_write()"]
end
subgraph subGraph0["Device Storage"]
    A["Vec>"]
    B["Arc"]
    C["Arc"]
    D["Arc"]
end

A --> B
A --> C
A --> D
B --> E
C --> E
D --> E
E --> F
E --> G
E --> H
```

Sources: [src/device.rs(L12 - L16)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L12-L16)

## Device Lifecycle Management

### Initialization Process

The `AxVmDevices` initialization follows a two-phase approach: structure creation and device initialization.

```mermaid
sequenceDiagram
    participant AxVmDeviceConfig as "AxVmDeviceConfig"
    participant AxVmDevices as "AxVmDevices"
    participant EmulatedDeviceConfig as "EmulatedDeviceConfig"
    participant BaseDeviceOps as "BaseDeviceOps"

    Note over AxVmDeviceConfig,BaseDeviceOps: Initialization Phase
    AxVmDeviceConfig ->> AxVmDevices: "new(config)"
    AxVmDevices ->> AxVmDevices: "Create empty emu_devices Vec"
    AxVmDevices ->> AxVmDevices: "init(&mut self, &config.emu_configs)"
    Note over AxVmDevices,BaseDeviceOps: Device Creation (TODO)
    loop "For each EmulatedDeviceConfig"
        AxVmDevices ->> EmulatedDeviceConfig: "Read device configuration"
        EmulatedDeviceConfig ->> BaseDeviceOps: "Create device instance"
        BaseDeviceOps ->> AxVmDevices: "Add to emu_devices"
    end
```

Sources: [src/device.rs(L19 - L28)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L19-L28) [src/device.rs(L30 - L54)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L30-L54)

The `new()` method creates an empty device collection and delegates to the `init()` method for device-specific initialization. Currently, the `init()` method contains placeholder code with TODO comments indicating future device type handling.

### Device Type Enumeration

The commented code in `init()` reveals the planned device type architecture:

|Device Type|Description|
| --- | --- |
|EmuDeviceTConsole|Console device emulation|
|EmuDeviceTGicdV2|GIC distributor v2|
|EmuDeviceTGPPT|General Purpose Physical Timer|
|EmuDeviceTVirtioBlk|VirtIO block device|
|EmuDeviceTVirtioNet|VirtIO network device|
|EmuDeviceTVirtioConsole|VirtIO console device|
|EmuDeviceTIOMMU|I/O Memory Management Unit|
|EmuDeviceTICCSRE|Interrupt Controller System Register Enable|
|EmuDeviceTSGIR|Software Generated Interrupt Register|
|EmuDeviceTGICR|GIC redistributor|
|EmuDeviceTMeta|Metadata device|

Sources: [src/device.rs(L34 - L46)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L34-L46)

## MMIO Operation Handling

### Device Lookup Mechanism

The `find_dev()` method provides address-based device resolution using the `BaseDeviceOps` trait's `address_range()` method.

```mermaid
flowchart TD
subgraph subGraph0["Device Lookup Process"]
    A["find_dev(ipa: GuestPhysAddr)"]
    B["Iterate emu_devices"]
    C["dev.address_range().contains(ipa)"]
    D["Address Match?"]
    E["Return Some(Arc)"]
    F["Continue iteration"]
    G["More devices?"]
    H["Return None"]
end

A --> B
B --> C
C --> D
D --> E
D --> F
F --> G
G --> B
G --> H
```

Sources: [src/device.rs(L56 - L62)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L56-L62)

### MMIO Read Operations

The `handle_mmio_read()` method processes guest VM read requests and returns data from the appropriate device.

```mermaid
sequenceDiagram
    participant GuestVM as "Guest VM"
    participant AxVmDevices as "AxVmDevices"
    participant Device as "Device"

    GuestVM ->> AxVmDevices: "handle_mmio_read(addr, width)"
    AxVmDevices ->> AxVmDevices: "find_dev(addr)"
    alt "Device found"
        AxVmDevices ->> AxVmDevices: "Log operation info"
        AxVmDevices ->> Device: "handle_read(addr, width)"
        Device ->> AxVmDevices: "Return AxResult<usize>"
        AxVmDevices ->> GuestVM: "Return value"
    else "No device found"
        AxVmDevices ->> AxVmDevices: "panic!(no emul handler)"
    end
```

Sources: [src/device.rs(L64 - L75)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L64-L75)

The method includes comprehensive logging that outputs the device's address range and the specific IPA (Intermediate Physical Address) being accessed.

### MMIO Write Operations

The `handle_mmio_write()` method processes guest VM write requests to device registers.

```mermaid
sequenceDiagram
    participant GuestVM as "Guest VM"
    participant AxVmDevices as "AxVmDevices"
    participant Device as "Device"

    GuestVM ->> AxVmDevices: "handle_mmio_write(addr, width, val)"
    AxVmDevices ->> AxVmDevices: "find_dev(addr)"
    alt "Device found"
        AxVmDevices ->> AxVmDevices: "Log operation info"
        AxVmDevices ->> Device: "handle_write(addr, width, val)"
        Device ->> AxVmDevices: "Operation complete"
        AxVmDevices ->> GuestVM: "Return"
    else "No device found"
        AxVmDevices ->> AxVmDevices: "panic!(no emul handler)"
    end
```

Sources: [src/device.rs(L77 - L93)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L77-L93)

Both MMIO operations follow a similar pattern: device lookup, logging, delegation to the device implementation, and error handling via panic for unmapped addresses.

## Address Space Integration

### Guest Physical Address Handling

The system uses `GuestPhysAddr` from the `axaddrspace` crate to represent guest physical memory addresses, ensuring type safety in address operations.

```mermaid
flowchart TD
subgraph subGraph1["Device Address Ranges"]
    H["Device 1 Range"]
    I["Device 2 Range"]
    J["Device N Range"]
end
subgraph subGraph0["Address Resolution"]
    A["GuestPhysAddr"]
    B["Device Lookup"]
    C["BaseDeviceOps::address_range()"]
    D["AddressRange::contains()"]
    E["Match Found?"]
    F["Route to Device"]
    G["Panic: No Handler"]
end

A --> B
B --> C
C --> D
C --> H
C --> I
C --> J
D --> E
E --> F
E --> G
```

Sources: [src/device.rs(L6)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L6-L6) [src/device.rs(L57 - L62)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L57-L62)

### Error Handling Strategy

The current implementation uses panic-based error handling for unmapped addresses, which provides immediate feedback during development but may require more graceful handling in production environments.

```mermaid
flowchart TD
A["MMIO Request"]
B["find_dev(addr)"]
C["Device Found?"]
D["Execute Operation"]
E["panic!(no emul handler)"]
F["Log Operation"]
G["Call BaseDeviceOps Method"]
H["Return Result"]

A --> B
B --> C
C --> D
C --> E
D --> F
F --> G
G --> H
```

Sources: [src/device.rs(L74)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L74-L74) [src/device.rs(L88 - L91)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L88-L91)

## Integration with BaseDeviceOps

The `AxVmDevices` system relies entirely on the `BaseDeviceOps` trait for device-specific functionality, creating a clean separation between device management and device implementation.

### Trait Method Usage

|Method|Purpose|Usage in AxVmDevices|
| --- | --- | --- |
|address_range()|Define device memory mapping|Used infind_dev()for address resolution|
|handle_read()|Process read operations|Called fromhandle_mmio_read()|
|handle_write()|Process write operations|Called fromhandle_mmio_write()|

Sources: [src/device.rs(L60)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L60-L60) [src/device.rs(L72)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L72-L72) [src/device.rs(L85)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L85-L85)

### Device Abstraction Benefits

```mermaid
flowchart TD
subgraph subGraph2["Device Implementations"]
    G["Console Device"]
    H["VirtIO Devices"]
    I["Interrupt Controllers"]
    J["Custom Devices"]
end
subgraph subGraph1["BaseDeviceOps (Device Abstraction)"]
    D["address_range()"]
    E["handle_read()"]
    F["handle_write()"]
end
subgraph subGraph0["AxVmDevices (Device Manager)"]
    A["Unified MMIO Interface"]
    B["Address-based Routing"]
    C["Lifecycle Management"]
end

A --> D
B --> D
C --> D
D --> G
D --> H
D --> I
D --> J
E --> G
E --> H
E --> I
E --> J
F --> G
F --> H
F --> I
F --> J
```

Sources: [src/device.rs(L7)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L7-L7) [src/device.rs(L14)&emsp;](https://github.com/arceos-hypervisor/axdevice/blob/8652ce80/src/device.rs#L14-L14)

This architecture enables the `AxVmDevices` system to remain device-agnostic while providing consistent MMIO handling across all emulated device types.
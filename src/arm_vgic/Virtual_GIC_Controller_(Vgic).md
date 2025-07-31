# Virtual GIC Controller (Vgic)

> **Relevant source files**
> * [src/consts.rs](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/consts.rs)
> * [src/vgic.rs](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs)

This document covers the `Vgic` struct and its associated components, which form the core of the virtual interrupt controller implementation in the arm_vgic crate. The `Vgic` provides virtualized access to ARM Generic Interrupt Controller (GIC) functionality for guest virtual machines within the ArceOS hypervisor ecosystem.

For information about the CPU-specific interface components, see [CPU Interface (Vgicc)](/arceos-hypervisor/arm_vgic/3.2-cpu-interface-(vgicc)). For details about the device framework integration, see [Device Operations Interface](/arceos-hypervisor/arm_vgic/3.3-device-operations-interface).

## Core Structure and Components

The Virtual GIC Controller is implemented through two primary structures that work together to manage interrupt virtualization state and provide thread-safe access to interrupt controller resources.

### Primary Data Structures

The `Vgic` implementation follows a layered approach with an outer wrapper providing thread-safe access to the internal state:

```mermaid
flowchart TD
subgraph subGraph4["Control State"]
    ctrlr["ctrlr: u32"]
    typer["typer: u32"]
    iidr["iidr: u32"]
end
subgraph subGraph3["GIC Registers"]
    gicd_igroupr["gicd_igroupr[SPI_ID_MAX/32]"]
    gicd_isenabler["gicd_isenabler[SPI_ID_MAX/32]"]
    gicd_ipriorityr["gicd_ipriorityr[SPI_ID_MAX]"]
    gicd_itargetsr["gicd_itargetsr[SPI_ID_MAX]"]
    gicd_icfgr["gicd_icfgr[SPI_ID_MAX/16]"]
end
subgraph subGraph2["State Arrays"]
    used_irq["used_irq[SPI_ID_MAX/32]"]
    ptov["ptov[SPI_ID_MAX]"]
    vtop["vtop[SPI_ID_MAX]"]
    gicc_vec["gicc: Vec"]
end
subgraph subGraph1["Protected State"]
    VgicInner["VgicInner"]
    Mutex["Mutex"]
end
subgraph subGraph0["Public Interface"]
    Vgic["Vgic"]
end

Mutex --> VgicInner
Vgic --> Mutex
VgicInner --> ctrlr
VgicInner --> gicc_vec
VgicInner --> gicd_icfgr
VgicInner --> gicd_igroupr
VgicInner --> gicd_ipriorityr
VgicInner --> gicd_isenabler
VgicInner --> gicd_itargetsr
VgicInner --> iidr
VgicInner --> ptov
VgicInner --> typer
VgicInner --> used_irq
VgicInner --> vtop
```

Sources: [src/vgic.rs(L15 - L34)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L15-L34)

### State Management Arrays

The `VgicInner` struct maintains several key data structures for interrupt management:

|Array|Type|Size|Purpose|
| --- | --- | --- | --- |
|used_irq|[u32; SPI_ID_MAX/32]|16 words|Tracks which interrupt IDs are allocated|
|ptov|[u32; SPI_ID_MAX]|512 entries|Physical-to-virtual interrupt ID mapping|
|vtop|[u32; SPI_ID_MAX]|512 entries|Virtual-to-physical interrupt ID mapping|
|gicd_igroupr|[u32; SPI_ID_MAX/32]|16 words|Interrupt group register state|
|gicd_isenabler|[u32; SPI_ID_MAX/32]|16 words|Interrupt enable register state|
|gicd_ipriorityr|[u8; SPI_ID_MAX]|512 bytes|Interrupt priority values|
|gicd_itargetsr|[u8; SPI_ID_MAX]|512 bytes|Interrupt target CPU mapping|
|gicd_icfgr|[u32; SPI_ID_MAX/16]|32 words|Interrupt configuration register state|

Sources: [src/vgic.rs(L15 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L15-L30) [src/consts.rs(L1 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/consts.rs#L1-L4)

## Memory Access Handling

The `Vgic` provides width-specific handlers for processing virtual machine memory accesses to the interrupt controller's memory-mapped registers. These handlers implement the virtualization layer between guest VM register accesses and the underlying interrupt controller state.

### Handler Method Dispatch

```mermaid
flowchart TD
subgraph subGraph2["Register Processing"]
    addr_decode["Address Decoding"]
    register_logic["Register-Specific Logic"]
    state_update["VgicInner State Update"]
end
subgraph subGraph1["Handler Methods"]
    handle_read8["handle_read8()"]
    handle_read16["handle_read16()"]
    handle_read32["handle_read32()"]
    handle_write8["handle_write8()"]
    handle_write16["handle_write16()"]
    handle_write32["handle_write32()"]
end
subgraph subGraph0["VM Memory Access"]
    vm_read8["VM 8-bit Read"]
    vm_read16["VM 16-bit Read"]
    vm_read32["VM 32-bit Read"]
    vm_write8["VM 8-bit Write"]
    vm_write16["VM 16-bit Write"]
    vm_write32["VM 32-bit Write"]
end

addr_decode --> register_logic
handle_read16 --> addr_decode
handle_read32 --> addr_decode
handle_read8 --> addr_decode
handle_write16 --> addr_decode
handle_write32 --> addr_decode
handle_write8 --> addr_decode
register_logic --> state_update
vm_read16 --> handle_read16
vm_read32 --> handle_read32
vm_read8 --> handle_read8
vm_write16 --> handle_write16
vm_write32 --> handle_write32
vm_write8 --> handle_write8
```

Sources: [src/vgic.rs(L56 - L133)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L56-L133)

### Read Handler Implementation

The read handlers currently provide basic functionality with placeholder implementations:

* `handle_read8()`: Returns 0 for all 8-bit register reads
* `handle_read16()`: Returns 0 for all 16-bit register reads
* `handle_read32()`: Returns 0 for all 32-bit register reads

These methods use the signature `fn handle_readX(&self, addr: usize) -> AxResult<usize>` and are marked as `pub(crate)` for internal crate access.

Sources: [src/vgic.rs(L56 - L66)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L56-L66)

## Register Handling and Interrupt Control

The write handlers implement the core interrupt virtualization logic by processing writes to specific GIC distributor registers and coordinating with the physical interrupt controller.

### Control Register Handling

The `VGICD_CTLR` register controls the overall enable/disable state of the interrupt distributor:

```mermaid
flowchart TD
subgraph subGraph2["Physical GIC Operations"]
    scan_used["Scan used_irq array"]
    set_enable["GicInterface::set_enable()"]
    set_priority["GicInterface::set_priority()"]
end
subgraph subGraph1["Conditional Processing"]
    check_enable["ctrlr > 0?"]
    enable_path["Enable Path"]
    disable_path["Disable Path"]
end
subgraph subGraph0["Control Register Write Flow"]
    write_ctlr["Write to VGICD_CTLR"]
    extract_bits["Extract bits [1:0]"]
    lock_state["Lock VgicInner"]
    update_ctrlr["Update ctrlr field"]
end

check_enable --> disable_path
check_enable --> enable_path
disable_path --> scan_used
enable_path --> scan_used
enable_path --> set_priority
extract_bits --> lock_state
lock_state --> update_ctrlr
scan_used --> set_enable
update_ctrlr --> check_enable
write_ctlr --> extract_bits
```

Sources: [src/vgic.rs(L70 - L95)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L70-L95)

### Enable Register Handling

The interrupt enable registers (`VGICD_ISENABLER_SGI_PPI` and `VGICD_ISENABLER_SPI`) are handled through delegation to wider access methods:

* 8-bit writes to enable registers delegate to `handle_write32()`
* 16-bit writes to enable registers delegate to `handle_write32()`
* 32-bit writes to enable registers are handled directly but currently contain placeholder logic

Sources: [src/vgic.rs(L96 - L132)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L96-L132)

## Physical Hardware Integration

The `Vgic` coordinates with the physical ARM GIC hardware through the `arm_gicv2::GicInterface` to ensure that virtual interrupt state changes are reflected in the underlying hardware configuration.

### Hardware Interface Operations

|Operation|Method|Purpose|
| --- | --- | --- |
|Enable/Disable|GicInterface::set_enable(irq_id, enabled)|Control interrupt enable state|
|Priority Setting|GicInterface::set_priority(irq_id, priority)|Set interrupt priority level|

The integration logic scans the `used_irq` bitmap to identify allocated interrupts and applies configuration changes only to interrupts that are actively managed by the virtual controller.

Sources: [src/vgic.rs(L5)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L5-L5) [src/vgic.rs(L82 - L92)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L82-L92)

## Initialization and Construction

The `Vgic::new()` constructor initializes all internal state arrays to zero values and creates an empty vector for CPU interface objects:

```mermaid
flowchart TD
subgraph subGraph1["Default Values"]
    zero_ctrlr["ctrlr = 0"]
    zero_arrays["All arrays = [0; SIZE]"]
    empty_gicc["gicc = Vec::new()"]
end
subgraph subGraph0["Constructor Process"]
    new_call["Vgic::new()"]
    create_mutex["Create Mutex"]
    init_arrays["Initialize State Arrays"]
    empty_vec["Empty gicc Vec"]
end

create_mutex --> init_arrays
empty_vec --> empty_gicc
init_arrays --> empty_vec
init_arrays --> zero_arrays
init_arrays --> zero_ctrlr
new_call --> create_mutex
```

The constructor ensures that the virtual interrupt controller starts in a clean, disabled state with no allocated interrupts or configured CPU interfaces.

Sources: [src/vgic.rs(L37 - L54)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgic.rs#L37-L54)
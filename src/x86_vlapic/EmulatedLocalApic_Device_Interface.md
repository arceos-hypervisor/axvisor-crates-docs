# EmulatedLocalApic Device Interface

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs)

## Purpose and Scope

This document covers the `EmulatedLocalApic` struct, which serves as the primary device interface for virtual Local APIC emulation in the x86_vlapic crate. The `EmulatedLocalApic` implements the `BaseDeviceOps` trait twice to provide unified handling for both xAPIC MMIO accesses and x2APIC MSR accesses, while delegating actual register operations to the underlying virtual register management system.

For details on the virtual register management system that `EmulatedLocalApic` delegates to, see [Virtual Register Management](/arceos-hypervisor/x86_vlapic/2.2-virtual-register-management). For information about address translation mechanisms, see [Register Address Translation](/arceos-hypervisor/x86_vlapic/2.3-register-address-translation).

## Core EmulatedLocalApic Structure

The `EmulatedLocalApic` struct is a thin wrapper around the virtual register management system that provides the device interface layer required by the ArceOS hypervisor framework.

```mermaid
flowchart TD
subgraph subGraph3["Address Translation"]
    XAPIC_XLATE["xapic_mmio_access_reg_offset()"]
    X2APIC_XLATE["x2apic_msr_access_reg()"]
end
subgraph subGraph2["Device Framework Integration"]
    BDO_MMIO["BaseDeviceOps<AddrRange<GuestPhysAddr>>"]
    BDO_MSR["BaseDeviceOps<SysRegAddrRange>"]
end
subgraph subGraph0["EmulatedLocalApic Device Interface"]
    ELA["EmulatedLocalApic<H: AxMmHal>"]
    VAR["vlapic_regs: VirtualApicRegs<H>"]
end
subgraph subGraph1["Memory Management"]
    AAP["VIRTUAL_APIC_ACCESS_PAGE: APICAccessPage"]
    STATIC["Static 4KB aligned page"]
end

AAP --> STATIC
BDO_MMIO --> XAPIC_XLATE
BDO_MSR --> X2APIC_XLATE
ELA --> BDO_MMIO
ELA --> BDO_MSR
ELA --> VAR
```

**EmulatedLocalApic Structure Components**

|Component|Type|Purpose|
| --- | --- | --- |
|vlapic_regs|VirtualApicRegs<H>|Manages actual APIC register virtualization and storage|

The struct uses a generic type parameter `H: AxMmHal` to integrate with the ArceOS memory management abstraction layer.

**Sources:** [src/lib.rs(L33 - L35)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L33-L35) [src/lib.rs(L39 - L43)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L39-L43)

## Device Interface Implementation

The `EmulatedLocalApic` implements `BaseDeviceOps` twice to handle the two distinct access methods for Local APIC registers: xAPIC memory-mapped I/O and x2APIC model-specific registers.

### Dual Interface Architecture

```mermaid
flowchart TD
subgraph subGraph4["Register Operations"]
    VLAPIC_REGS["VirtualApicRegs<H>"]
    READ_OP["handle_read()"]
    WRITE_OP["handle_write()"]
end
subgraph subGraph3["Address Translation Layer"]
    XAPIC_FUNC["xapic_mmio_access_reg_offset()"]
    X2APIC_FUNC["x2apic_msr_access_reg()"]
end
subgraph subGraph2["EmulatedLocalApic Interface Layer"]
    ELA["EmulatedLocalApic<H>"]
    subgraph subGraph1["BaseDeviceOps Implementations"]
        MMIO_IMPL["BaseDeviceOps<AddrRange<GuestPhysAddr>>"]
        MSR_IMPL["BaseDeviceOps<SysRegAddrRange>"]
    end
end
subgraph subGraph0["Guest Access Methods"]
    GUEST_MMIO["Guest MMIO Access0xFEE00000-0xFEE00FFF"]
    GUEST_MSR["Guest MSR Access0x800-0x8FF"]
end

ELA --> MMIO_IMPL
ELA --> MSR_IMPL
GUEST_MMIO --> MMIO_IMPL
GUEST_MSR --> MSR_IMPL
MMIO_IMPL --> XAPIC_FUNC
MSR_IMPL --> X2APIC_FUNC
READ_OP --> VLAPIC_REGS
WRITE_OP --> VLAPIC_REGS
X2APIC_FUNC --> READ_OP
X2APIC_FUNC --> WRITE_OP
XAPIC_FUNC --> READ_OP
XAPIC_FUNC --> WRITE_OP
```

**Sources:** [src/lib.rs(L67 - L112)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L67-L112) [src/lib.rs(L114 - L159)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L114-L159)

### xAPIC MMIO Interface

The xAPIC interface handles memory-mapped I/O accesses in the traditional Local APIC address range.

**Interface Configuration**

|Property|Value|Description|
| --- | --- | --- |
|Address Range|0xFEE00000 - 0xFEE00FFF|4KB MMIO region for xAPIC registers|
|Device Type|EmuDeviceTInterruptController|Identifies as interrupt controller device|
|Access Translation|xapic_mmio_access_reg_offset()|ConvertsGuestPhysAddrtoApicRegOffset|

**Access Flow**

1. Guest performs MMIO read/write to xAPIC address range
2. `handle_read()` or `handle_write()` called with `GuestPhysAddr`
3. Address translated via `xapic_mmio_access_reg_offset()`
4. Operation delegated to `VirtualApicRegs::handle_read()/handle_write()`

**Sources:** [src/lib.rs(L67 - L112)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L67-L112) [src/lib.rs(L90 - L91)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L90-L91) [src/lib.rs(L105 - L106)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L105-L106)

### x2APIC MSR Interface

The x2APIC interface handles model-specific register accesses for the extended Local APIC mode.

**Interface Configuration**

|Property|Value|Description|
| --- | --- | --- |
|Address Range|0x800 - 0x8FF|MSR range for x2APIC registers|
|Device Type|EmuDeviceTInterruptController|Identifies as interrupt controller device|
|Access Translation|x2apic_msr_access_reg()|ConvertsSysRegAddrtoApicRegOffset|

**Access Flow**

1. Guest performs MSR read/write to x2APIC address range
2. `handle_read()` or `handle_write()` called with `SysRegAddr`
3. Address translated via `x2apic_msr_access_reg()`
4. Operation delegated to `VirtualApicRegs::handle_read()/handle_write()`

**Sources:** [src/lib.rs(L114 - L159)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L114-L159) [src/lib.rs(L137 - L138)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L137-L138) [src/lib.rs(L152 - L153)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L152-L153)

## Memory Management

The `EmulatedLocalApic` manages two distinct 4KB memory pages for virtual Local APIC functionality, each serving different purposes in the virtualization architecture.

### APIC Access Page

```mermaid
flowchart TD
subgraph subGraph2["Access Methods"]
    ACCESSOR["virtual_apic_access_addr()"]
    HAL_CONVERT["H::virt_to_phys()"]
end
subgraph subGraph1["Hypervisor Integration"]
    VMCS["VM-execution control'virtualize APIC accesses'"]
    VM_EXIT["VM exits or processor virtualization"]
end
subgraph subGraph0["APIC Access Page Management"]
    STATIC_PAGE["VIRTUAL_APIC_ACCESS_PAGEAPICAccessPage"]
    ALIGNED_ARRAY["[u8; PAGE_SIZE_4K]4KB aligned static array"]
    VIRT_ADDR["HostVirtAddr"]
    PHYS_ADDR["HostPhysAddr"]
end

ACCESSOR --> HAL_CONVERT
ALIGNED_ARRAY --> VIRT_ADDR
HAL_CONVERT --> PHYS_ADDR
PHYS_ADDR --> VMCS
STATIC_PAGE --> ALIGNED_ARRAY
VIRT_ADDR --> PHYS_ADDR
VMCS --> VM_EXIT
```

The APIC access page is a static 4KB page used by hardware virtualization features to control guest access to the APIC MMIO region.

**APIC Access Page Characteristics**

|Property|Value|Purpose|
| --- | --- | --- |
|Type|APICAccessPage|4KB aligned wrapper around byte array|
|Alignment|4096 bytes|Required by hardware virtualization|
|Initialization|Zero-filled|Safe default state|
|Scope|Static global|Shared across all EmulatedLocalApic instances|

**Sources:** [src/lib.rs(L27 - L30)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L27-L30) [src/lib.rs(L47 - L56)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L47-L56)

### Virtual APIC Page

The virtual APIC page contains the actual register storage and is managed by the `VirtualApicRegs` component.

```mermaid
flowchart TD
subgraph subGraph0["Virtual APIC Page Access"]
    REG_STORAGE["VirtualApicRegs register storage"]
    subgraph subGraph1["Register Layout"]
        PHYS_PAGE["4KB Physical Page"]
        LAPIC_REGS["LocalAPICRegs structure"]
        REG_FIELDS["Individual APIC registers"]
        ELA_METHOD["virtual_apic_page_addr()"]
        DELEGATE["self.vlapic_regs.virtual_apic_page_addr()"]
    end
end

DELEGATE --> REG_STORAGE
ELA_METHOD --> DELEGATE
LAPIC_REGS --> REG_FIELDS
PHYS_PAGE --> LAPIC_REGS
```

The virtual APIC page provides the actual register storage that guests read from and write to during APIC operations.

**Sources:** [src/lib.rs(L58 - L64)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L58-L64)

## Address Translation and Delegation

The `EmulatedLocalApic` acts as an address translation layer, converting guest addresses to register offsets before delegating operations to the virtual register system.

### Translation Architecture

```mermaid
sequenceDiagram
    participant GuestVM as "Guest VM"
    participant EmulatedLocalApic as "EmulatedLocalApic"
    participant AddressTranslator as "Address Translator"
    participant VirtualApicRegs as "VirtualApicRegs"

    Note over GuestVM,VirtualApicRegs: xAPIC MMIO Access Example
    GuestVM ->> EmulatedLocalApic: "handle_read(GuestPhysAddr, width, context)"
    EmulatedLocalApic ->> AddressTranslator: "xapic_mmio_access_reg_offset(addr)"
    AddressTranslator -->> EmulatedLocalApic: "ApicRegOffset"
    EmulatedLocalApic ->> VirtualApicRegs: "handle_read(reg_off, width, context)"
    VirtualApicRegs -->> EmulatedLocalApic: "AxResult<usize>"
    EmulatedLocalApic -->> GuestVM: "Register value"
    Note over GuestVM,VirtualApicRegs: x2APIC MSR Access Example
    GuestVM ->> EmulatedLocalApic: "handle_write(SysRegAddr, width, val, context)"
    EmulatedLocalApic ->> AddressTranslator: "x2apic_msr_access_reg(addr)"
    AddressTranslator -->> EmulatedLocalApic: "ApicRegOffset"
    EmulatedLocalApic ->> VirtualApicRegs: "handle_write(reg_off, width, context)"
    VirtualApicRegs -->> EmulatedLocalApic: "AxResult"
    EmulatedLocalApic -->> GuestVM: "Write complete"
```

**Translation Functions**

|Function|Input Type|Output Type|Purpose|
| --- | --- | --- | --- |
|xapic_mmio_access_reg_offset()|GuestPhysAddr|ApicRegOffset|Convert xAPIC MMIO address to register offset|
|x2apic_msr_access_reg()|SysRegAddr|ApicRegOffset|Convert x2APIC MSR address to register offset|

Both translation functions convert different address formats to the unified `ApicRegOffset` enum used by the virtual register system.

**Sources:** [src/lib.rs(L23 - L24)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L23-L24) [src/lib.rs(L90 - L91)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L90-L91) [src/lib.rs(L105 - L106)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L105-L106) [src/lib.rs(L137 - L138)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L137-L138) [src/lib.rs(L152 - L153)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L152-L153)
# Error Handling LVT Registers

> **Relevant source files**
> * [src/regs/lvt/cmci.rs](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs)
> * [src/regs/lvt/error.rs](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs)

This document covers the two Local Vector Table (LVT) registers responsible for error handling in the virtual LAPIC implementation: the LVT Error Register and the LVT CMCI (Corrected Machine Check Interrupt) Register. These registers configure interrupt delivery for APIC internal errors and corrected machine check errors respectively.

For information about other LVT registers such as timer and external interrupt pins, see [Timer LVT Register](/arceos-hypervisor/x86_vlapic/3.2.1-timer-lvt-register) and [External Interrupt Pin Registers](/arceos-hypervisor/x86_vlapic/3.2.2-external-interrupt-pin-registers). For system monitoring LVT registers, see [System Monitoring LVT Registers](/arceos-hypervisor/x86_vlapic/3.2.3-system-monitoring-lvt-registers).

## Error Handling LVT Architecture

The error handling LVT registers are part of the broader LVT subsystem and provide interrupt configuration for different types of error conditions detected by the APIC.

### Error Handling LVT Position in Register Space

```mermaid
flowchart TD
subgraph subGraph2["Virtual Register Space (4KB Page)"]
    subgraph subGraph1["Error_Sources[Error Detection Sources]"]
        APIC_INTERNAL["APIC InternalError Conditions"]
        MCE_CORRECTED["Machine CheckCorrected Errors"]
    end
    subgraph subGraph0["LVT_Registers[Local Vector Table Registers]"]
        LVT_ERROR["LVT_ERROR0x370APIC Internal Errors"]
        LVT_CMCI["LVT_CMCI0x2F0Corrected Machine Check"]
        LVT_TIMER["LVT_TIMER0x320Timer Interrupts"]
        LVT_THERMAL["LVT_THERMAL0x330Thermal Monitor"]
        LVT_PMC["LVT_PMC0x340Performance Counter"]
        LVT_LINT0["LVT_LINT00x350External Pin 0"]
        LVT_LINT1["LVT_LINT10x360External Pin 1"]
    end
end

APIC_INTERNAL --> LVT_ERROR
MCE_CORRECTED --> LVT_CMCI
```

**Sources:** [src/regs/lvt/error.rs(L35 - L36)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L35-L36) [src/regs/lvt/cmci.rs(L62 - L65)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L62-L65)

### Code Entity Mapping

```mermaid
flowchart TD
subgraph subGraph1["Register_Access_Patterns[Register Access Patterns]"]
    DIRECT_MMIO["Direct MMIO AccessVolatile Hardware Read/Write"]
    CACHED_LOCAL["Cached Local CopyNon-volatile Memory Access"]
end
subgraph subGraph0["Rust_Types[Rust Type Definitions]"]
    LVT_ERROR_BITFIELD["LVT_ERRORregister_bitfields!"]
    LVT_CMCI_BITFIELD["LVT_CMCIregister_bitfields!"]
    ERROR_MMIO["LvtErrorRegisterMmioReadWrite"]
    ERROR_LOCAL["LvtErrorRegisterLocalLocalRegisterCopy"]
    CMCI_MMIO["LvtCmciRegisterMmioReadWrite"]
    CMCI_LOCAL["LvtCmciRegisterLocalLocalRegisterCopy"]
end

CMCI_LOCAL --> CACHED_LOCAL
CMCI_MMIO --> DIRECT_MMIO
ERROR_LOCAL --> CACHED_LOCAL
ERROR_MMIO --> DIRECT_MMIO
LVT_CMCI_BITFIELD --> CMCI_LOCAL
LVT_CMCI_BITFIELD --> CMCI_MMIO
LVT_ERROR_BITFIELD --> ERROR_LOCAL
LVT_ERROR_BITFIELD --> ERROR_MMIO
```

**Sources:** [src/regs/lvt/error.rs(L37 - L44)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L37-L44) [src/regs/lvt/cmci.rs(L66 - L73)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L66-L73)

## LVT Error Register

The LVT Error Register handles interrupts generated when the APIC detects internal error conditions. It is located at offset `0x370` in the APIC register space.

### Register Structure

|Bits|Field|Description|
| --- | --- | --- |
|31-17|Reserved2|Reserved bits|
|16|Mask|Interrupt mask (0=NotMasked, 1=Masked)|
|15-13|Reserved1|Reserved bits|
|12|DeliveryStatus|Read-only delivery status (0=Idle, 1=SendPending)|
|11-8|Reserved0|Reserved bits|
|7-0|Vector|Interrupt vector number|

The LVT Error Register has a simplified structure compared to other LVT registers, lacking delivery mode configuration since error interrupts are always delivered using Fixed mode.

### Key Characteristics

* **Fixed Delivery Mode Only**: Unlike the CMCI register, the Error register implicitly uses Fixed delivery mode
* **Simple Configuration**: Only requires vector number and mask bit configuration
* **Internal Error Detection**: Triggers on APIC hardware-detected error conditions
* **Read-Only Status**: The `DeliveryStatus` field provides read-only interrupt delivery state

**Sources:** [src/regs/lvt/error.rs(L5 - L33)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L5-L33) [src/regs/lvt/error.rs(L35 - L36)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L35-L36)

## LVT CMCI Register

The LVT CMCI (Corrected Machine Check Interrupt) Register handles interrupts for corrected machine check errors when error counts reach threshold values. It is located at offset `0x2F0` in the APIC register space.

### Register Structure

|Bits|Field|Description|
| --- | --- | --- |
|31-17|Reserved2|Reserved bits|
|16|Mask|Interrupt mask (0=NotMasked, 1=Masked)|
|15-13|Reserved1|Reserved bits|
|12|DeliveryStatus|Read-only delivery status (0=Idle, 1=SendPending)|
|11|Reserved0|Reserved bit|
|10-8|DeliveryMode|Interrupt delivery mode|
|7-0|Vector|Interrupt vector number|

### Delivery Modes

The CMCI register supports multiple delivery modes for flexible error handling:

|Mode|Binary|Description|Supported|
| --- | --- | --- | --- |
|Fixed|000|Delivers interrupt specified in vector field|✓|
|SMI|010|System Management Interrupt|✓|
|NMI|100|Non-Maskable Interrupt|✓|
|INIT|101|Initialization request|✗|
|Reserved|110|Not supported|✗|
|ExtINT|111|External interrupt controller|✗|

**Note**: INIT and ExtINT delivery modes are explicitly not supported for the CMCI register.

**Sources:** [src/regs/lvt/cmci.rs(L5 - L60)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L5-L60) [src/regs/lvt/cmci.rs(L44 - L45)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L44-L45) [src/regs/lvt/cmci.rs(L54 - L55)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L54-L55)

## Register Field Comparison

```mermaid
flowchart TD
subgraph subGraph2["CMCI_Specific[LVT CMCI Specific]"]
    CMCI_RESERVED["Reserved0 (Bit 11)1 Reserved Bit"]
    CMCI_DELIVERY["DeliveryMode (Bits 10-8)Fixed/SMI/NMI Support"]
end
subgraph subGraph1["Error_Specific[LVT Error Specific]"]
    ERROR_RESERVED["Reserved0 (Bits 11-8)4 Reserved Bits"]
    ERROR_IMPLICIT["Implicit Fixed ModeNo DeliveryMode Field"]
end
subgraph subGraph0["Common_Fields[Common LVT Fields]"]
    MASK_FIELD["Mask (Bit 16)0=NotMasked, 1=Masked"]
    DELIVERY_STATUS["DeliveryStatus (Bit 12)0=Idle, 1=SendPending"]
    VECTOR_FIELD["Vector (Bits 7-0)Interrupt Vector Number"]
end

DELIVERY_STATUS --> CMCI_RESERVED
DELIVERY_STATUS --> ERROR_RESERVED
MASK_FIELD --> CMCI_RESERVED
MASK_FIELD --> ERROR_RESERVED
VECTOR_FIELD --> CMCI_DELIVERY
VECTOR_FIELD --> ERROR_IMPLICIT
```

**Sources:** [src/regs/lvt/error.rs(L10 - L31)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L10-L31) [src/regs/lvt/cmci.rs(L10 - L58)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L10-L58)

## Error Handling Workflow

The error handling LVT registers integrate into the broader APIC interrupt delivery system to handle different categories of errors:

```mermaid
sequenceDiagram
    participant APICHardware as "APIC Hardware"
    participant LVT_ERRORRegister as "LVT_ERROR Register"
    participant LVT_CMCIRegister as "LVT_CMCI Register"
    participant ProcessorCore as "Processor Core"

    Note over APICHardware,ProcessorCore: Internal Error Detection
    APICHardware ->> LVT_ERRORRegister: Internal error detected
    LVT_ERRORRegister ->> LVT_ERRORRegister: Check Mask bit
    alt Error not masked
        LVT_ERRORRegister ->> ProcessorCore: Generate Fixed mode interrupt
        LVT_ERRORRegister ->> LVT_ERRORRegister: Set DeliveryStatus=SendPending
        ProcessorCore ->> LVT_ERRORRegister: Accept interrupt
        LVT_ERRORRegister ->> LVT_ERRORRegister: Set DeliveryStatus=Idle
    else Error masked
        LVT_ERRORRegister ->> LVT_ERRORRegister: Suppress interrupt
    end
    Note over APICHardware,ProcessorCore: Corrected Machine Check
    APICHardware ->> LVT_CMCIRegister: Corrected error threshold reached
    LVT_CMCIRegister ->> LVT_CMCIRegister: Check Mask bit and DeliveryMode
    alt CMCI not masked
    alt DeliveryMode=Fixed
        LVT_CMCIRegister ->> ProcessorCore: Generate vector interrupt
    else DeliveryMode=SMI
        LVT_CMCIRegister ->> ProcessorCore: Generate SMI
    else DeliveryMode=NMI
        LVT_CMCIRegister ->> ProcessorCore: Generate NMI
    end
    LVT_CMCIRegister ->> LVT_CMCIRegister: Set DeliveryStatus=SendPending
    ProcessorCore ->> LVT_CMCIRegister: Accept interrupt
    LVT_CMCIRegister ->> LVT_CMCIRegister: Set DeliveryStatus=Idle
    else CMCI masked
        LVT_CMCIRegister ->> LVT_CMCIRegister: Suppress interrupt
    end
```

**Sources:** [src/regs/lvt/error.rs(L18 - L27)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L18-L27) [src/regs/lvt/cmci.rs(L18 - L27)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L18-L27) [src/regs/lvt/cmci.rs(L32 - L56)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L32-L56)

## Register Type Definitions

Both registers provide dual access patterns through the tock-registers framework:

* **MMIO Types**: `LvtErrorRegisterMmio` and `LvtCmciRegisterMmio` for direct hardware access
* **Local Copy Types**: `LvtErrorRegisterLocal` and `LvtCmciRegisterLocal` for cached register copies

The local copy types enable efficient register manipulation without volatile MMIO operations for each access, which is particularly useful in virtualization scenarios where register state is maintained in memory.

**Sources:** [src/regs/lvt/error.rs(L37 - L44)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/error.rs#L37-L44) [src/regs/lvt/cmci.rs(L66 - L73)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/regs/lvt/cmci.rs#L66-L73)
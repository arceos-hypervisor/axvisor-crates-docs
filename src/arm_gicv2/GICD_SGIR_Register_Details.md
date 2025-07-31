# GICD_SGIR Register Details

> **Relevant source files**
> * [src/regs/gicd_sgir.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs)

This document provides comprehensive technical documentation for the GICD_SGIR (Software Generated Interrupt Register) implementation in the arm_gicv2 crate. The register controls the generation and routing of Software Generated Interrupts (SGIs) within the ARM GICv2 interrupt controller system.

For broader context on SGI interrupt types and their role in the interrupt system, see [Software Generated Interrupts](/arceos-hypervisor/arm_gicv2/3.2-software-generated-interrupts). For general register interface organization, see [Register Module Organization](/arceos-hypervisor/arm_gicv2/4.1-register-module-organization).

## Register Overview and Purpose

The GICD_SGIR register serves as the primary control interface for generating Software Generated Interrupts in the GIC Distributor. SGIs enable inter-processor communication in multi-core ARM systems by allowing one CPU to trigger interrupts on other CPUs or itself.

## GICD_SGIR in GIC Architecture

### GICD_SGIR Location in Interrupt Processing

```mermaid
flowchart TD
CPU0["CPU 0"]
DISTRIBUTOR["GicDistributor"]
CPU1["CPU 1"]
CPU2["CPU 2"]
CPU3["CPU 3"]
GICD_SGIR["GICD_SGIR RegisterWrite-Only0x00000F00 offset"]
SGI_LOGIC["SGI Generation Logic"]
TARGET_CPU0["Target CPU 0GicCpuInterface"]
TARGET_CPU1["Target CPU 1GicCpuInterface"]
TARGET_CPU2["Target CPU 2GicCpuInterface"]
TARGET_CPU3["Target CPU 3GicCpuInterface"]
INT0["Interrupt ID 0-15"]
INT1["Interrupt ID 0-15"]
INT2["Interrupt ID 0-15"]
INT3["Interrupt ID 0-15"]

CPU0 --> DISTRIBUTOR
CPU1 --> DISTRIBUTOR
CPU2 --> DISTRIBUTOR
CPU3 --> DISTRIBUTOR
DISTRIBUTOR --> GICD_SGIR
GICD_SGIR --> SGI_LOGIC
SGI_LOGIC --> TARGET_CPU0
SGI_LOGIC --> TARGET_CPU1
SGI_LOGIC --> TARGET_CPU2
SGI_LOGIC --> TARGET_CPU3
TARGET_CPU0 --> INT0
TARGET_CPU1 --> INT1
TARGET_CPU2 --> INT2
TARGET_CPU3 --> INT3
```

Sources: [src/regs/gicd_sgir.rs(L1 - L62)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L1-L62)

## Register Bit Field Layout

### GICD_SGIR Bit Field Structure

```mermaid
flowchart TD
subgraph subGraph0["32-bit GICD_SGIR Register Layout"]
    BITS31_26["Bits 31:26ReservedOFFSET(26) NUMBITS(6)"]
    BITS25_24["Bits 25:24TargetListFilterOFFSET(24) NUMBITS(2)"]
    BITS23_16["Bits 23:16CPUTargetListOFFSET(16) NUMBITS(8)"]
    BIT15["Bit 15NSATTOFFSET(15) NUMBITS(1)"]
    BITS14_4["Bits 14:4Reserved SBZOFFSET(4) NUMBITS(11)"]
    BITS3_0["Bits 3:0SGIINTIDOFFSET(0) NUMBITS(4)"]
end

BIT15 --> BITS14_4
```

Sources: [src/regs/gicd_sgir.rs(L21 - L58)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L21-L58)

## Register Field Definitions

### Core Field Components

|Field|Bits|Type|Purpose|
| --- | --- | --- | --- |
|TargetListFilter|25:24|Enum|Controls SGI routing behavior|
|CPUTargetList|23:16|Bitmap|Specifies target CPU interfaces|
|NSATT|15|Boolean|Security group selection|
|SGIINTID|3:0|Integer|SGI interrupt ID (0-15)|

Sources: [src/regs/gicd_sgir.rs(L25 - L56)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L25-L56)

### TargetListFilter Modes

The `TargetListFilter` field defines four distinct targeting modes implemented as register bitfield values:

```mermaid
flowchart TD
WRITE["Write to GICD_SGIR"]
FILTER_CHECK["Check TargetListFilter bits 25:24"]
MODE00["0b00: ForwardToCPUTargetListUse CPUTargetList bitmap"]
MODE01["0b01: ForwardToAllExceptRequesterAll CPUs except requesting CPU"]
MODE10["0b10: ForwardToRequesterOnly requesting CPU"]
MODE11["0b11: ReservedUndefined behavior"]
TARGET_BITMAP["Check CPUTargetList[7:0]Each bit = target CPU"]
ALL_EXCEPT["Generate SGI for allactive CPUs except requester"]
SELF_ONLY["Generate SGI only forrequesting CPU"]
SGI_GEN["Generate SGI(s)with SGIINTID value"]

ALL_EXCEPT --> SGI_GEN
FILTER_CHECK --> MODE00
FILTER_CHECK --> MODE01
FILTER_CHECK --> MODE10
FILTER_CHECK --> MODE11
MODE00 --> TARGET_BITMAP
MODE01 --> ALL_EXCEPT
MODE10 --> SELF_ONLY
SELF_ONLY --> SGI_GEN
TARGET_BITMAP --> SGI_GEN
WRITE --> FILTER_CHECK
```

Sources: [src/regs/gicd_sgir.rs(L31 - L35)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L31-L35)

## Register Implementation Details

### Rust Type Definition

The register is implemented as a write-only register using the tock-registers framework:

```
pub type GicdSgirReg = WriteOnly<u32, GICD_SGIR::Register>;
```

### Register Bitfield Structure

The implementation uses the `register_bitfields!` macro to define field layouts and valid values:

|Constant|Value|Description|
| --- | --- | --- |
|ForwardToCPUTargetList|0b00|Use explicit CPU target list|
|ForwardToAllExceptRequester|0b01|Broadcast except requester|
|ForwardToRequester|0b10|Self-targeting only|
|Reserved|0b11|Invalid/reserved value|

Sources: [src/regs/gicd_sgir.rs(L18 - L62)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L18-L62)

## CPU Targeting Mechanisms

### CPUTargetList Bitmap Interpretation

When `TargetListFilter` is set to `ForwardToCPUTargetList` (0b00), the `CPUTargetList` field operates as an 8-bit bitmap:

```mermaid
flowchart TD
subgraph subGraph0["CPUTargetList Bitmap (bits 23:16)"]
    BIT23["Bit 23CPU 7"]
    BIT22["Bit 22CPU 6"]
    BIT21["Bit 21CPU 5"]
    BIT20["Bit 20CPU 4"]
    BIT19["Bit 19CPU 3"]
    BIT18["Bit 18CPU 2"]
    BIT17["Bit 17CPU 1"]
    BIT16["Bit 16CPU 0"]
end
CPU7["GicCpuInterface 7"]
CPU6["GicCpuInterface 6"]
CPU5["GicCpuInterface 5"]
CPU4["GicCpuInterface 4"]
CPU3["GicCpuInterface 3"]
CPU2["GicCpuInterface 2"]
CPU1["GicCpuInterface 1"]
CPU0["GicCpuInterface 0"]

BIT16 --> CPU0
BIT17 --> CPU1
BIT18 --> CPU2
BIT19 --> CPU3
BIT20 --> CPU4
BIT21 --> CPU5
BIT22 --> CPU6
BIT23 --> CPU7
```

Sources: [src/regs/gicd_sgir.rs(L37 - L41)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L37-L41)

## Security Extensions Support

### NSATT Field Behavior

The NSATT (Non-Secure Attribute) field controls security group targeting when Security Extensions are implemented:

|NSATT Value|Group|Access Behavior|
| --- | --- | --- |
|0|Group 0|Forward SGI only if configured as Group 0|
|1|Group 1|Forward SGI only if configured as Group 1|

### Security Access Constraints

* **Secure Access**: Can write any value to NSATT field
* **Non-Secure Access**: Limited to Group 1 SGIs regardless of NSATT value written

Sources: [src/regs/gicd_sgir.rs(L42 - L49)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L42-L49)

## SGI Interrupt ID Specification

The `SGIINTID` field specifies which of the 16 available Software Generated Interrupts (0-15) to trigger. This field directly maps to interrupt IDs in the SGI range:

```mermaid
flowchart TD
SGIINTID_FIELD["SGIINTID bits 3:0"]
SGI_RANGE["SGI Interrupt IDs"]
ID0["ID 0: 0b0000"]
ID1["ID 1: 0b0001"]
ID2["ID 2: 0b0010"]
ID3["ID 3: 0b0011"]
DOTS["..."]
ID15["ID 15: 0b1111"]
CPU_INT0["CPU InterfaceSGI 0 pending"]
CPU_INT3["CPU InterfaceSGI 3 pending"]
CPU_INT15["CPU InterfaceSGI 15 pending"]

ID0 --> CPU_INT0
ID15 --> CPU_INT15
ID3 --> CPU_INT3
SGIINTID_FIELD --> SGI_RANGE
SGI_RANGE --> DOTS
SGI_RANGE --> ID0
SGI_RANGE --> ID1
SGI_RANGE --> ID15
SGI_RANGE --> ID2
SGI_RANGE --> ID3
```

Sources: [src/regs/gicd_sgir.rs(L52 - L56)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L52-L56)

## Implementation Considerations

### Write-Only Register Characteristics

The GICD_SGIR register is implemented as `WriteOnly<u32, GICD_SGIR::Register>`, meaning:

* No read operations are supported
* Each write triggers immediate SGI processing
* Register state is not persistent
* Effects are immediate upon write completion

### Reserved Field Handling

The implementation includes two reserved field ranges that must be handled appropriately:

* **Bits 31:26**: Reserved, implementation behavior undefined
* **Bits 14:4**: Reserved, Should Be Zero (SBZ)

Sources: [src/regs/gicd_sgir.rs(L24 - L61)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs#L24-L61)
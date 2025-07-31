# Register Address Translation

> **Relevant source files**
> * [src/consts.rs](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs)
> * [src/lib.rs](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs)

This document describes the address translation mechanism that converts guest memory accesses into internal register identifiers within the x86_vlapic crate. The translation layer serves as a bridge between two different APIC access methods (xAPIC MMIO and x2APIC MSR) and the unified internal register representation using the `ApicRegOffset` enum.

For information about the overall virtual register management system, see [Virtual Register Management](/arceos-hypervisor/x86_vlapic/2.2-virtual-register-management). For details about specific register layouts and functionality, see [Register System](/arceos-hypervisor/x86_vlapic/3-register-system).

## Translation Architecture Overview

The address translation system provides a unified interface for accessing APIC registers regardless of whether the guest uses xAPIC MMIO-based access or x2APIC MSR-based access. Both access methods are translated into the same internal `ApicRegOffset` enum representation.

```mermaid
flowchart TD
subgraph subGraph4["Register Access Handlers"]
    VLAPIC_REGS["VirtualApicRegshandle_read/write methods"]
end
subgraph subGraph3["Unified Register Space"]
    APIC_REG_OFFSET["ApicRegOffset enumsrc/consts.rs:61-114"]
    subgraph subGraph2["Register Categories"]
        CONTROL_REGS["Control RegistersID, Version, TPR, etc."]
        ARRAY_REGS["Array RegistersISR, TMR, IRR"]
        LVT_REGS["LVT RegistersTimer, Thermal, PMC, etc."]
        TIMER_REGS["Timer RegistersInitCount, CurCount, DivConf"]
    end
end
subgraph subGraph1["Translation Functions"]
    XAPIC_FUNC["xapic_mmio_access_reg_offset()src/consts.rs:200-202"]
    X2APIC_FUNC["x2apic_msr_access_reg()src/consts.rs:213-215"]
end
subgraph subGraph0["Guest Access Methods"]
    GUEST_MMIO["Guest xAPIC MMIO AccessGuestPhysAddr0xFEE0_0000-0xFEE0_0FFF"]
    GUEST_MSR["Guest x2APIC MSR AccessSysRegAddr0x800-0x8FF"]
end

APIC_REG_OFFSET --> ARRAY_REGS
APIC_REG_OFFSET --> CONTROL_REGS
APIC_REG_OFFSET --> LVT_REGS
APIC_REG_OFFSET --> TIMER_REGS
ARRAY_REGS --> VLAPIC_REGS
CONTROL_REGS --> VLAPIC_REGS
GUEST_MMIO --> XAPIC_FUNC
GUEST_MSR --> X2APIC_FUNC
LVT_REGS --> VLAPIC_REGS
TIMER_REGS --> VLAPIC_REGS
X2APIC_FUNC --> APIC_REG_OFFSET
XAPIC_FUNC --> APIC_REG_OFFSET
```

Sources: [src/consts.rs(L200 - L202)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L200-L202) [src/consts.rs(L213 - L215)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L213-L215) [src/consts.rs(L61 - L114)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L61-L114) [src/lib.rs(L90)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L90-L90) [src/lib.rs(L105)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L105-L105) [src/lib.rs(L137)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L137-L137) [src/lib.rs(L152)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L152-L152)

## xAPIC MMIO Address Translation

The xAPIC mode uses Memory-Mapped I/O (MMIO) for register access. Guest physical addresses in the range `0xFEE0_0000` to `0xFEE0_0FFF` are translated to register offsets using the `xapic_mmio_access_reg_offset()` function.

### Translation Process

```mermaid
flowchart TD
subgraph Constants["Constants"]
    APIC_BASE["DEFAULT_APIC_BASE0xFEE0_0000"]
    APIC_SIZE["APIC_MMIO_SIZE0x1000 (4KB)"]
end
START["Guest Physical AddressGuestPhysAddr"]
MASK["Apply MMIO Size Maskaddr & (APIC_MMIO_SIZE - 1)Extract lower 12 bits"]
SHIFT["Right Shift by 4result >> 4Convert to 16-byte aligned offset"]
CONVERT["ApicRegOffset::from()Map offset to enum variant"]
END["ApicRegOffset enum"]

CONVERT --> END
MASK --> SHIFT
SHIFT --> CONVERT
START --> MASK
```

The translation algorithm:

1. **Mask Address**: `addr.as_usize() & (APIC_MMIO_SIZE - 1)` extracts the offset within the 4KB APIC region
2. **Normalize to 16-byte boundaries**: `result >> 4` converts byte offset to register offset (APIC registers are 16-byte aligned)
3. **Convert to enum**: `ApicRegOffset::from(offset)` maps the numeric offset to the appropriate enum variant

### Address Mapping Examples

|Guest Physical Address|Masked Offset|Register Offset|ApicRegOffset Enum|
| --- | --- | --- | --- |
|0xFEE0_0020|0x020|0x2|ApicRegOffset::ID|
|0xFEE0_0030|0x030|0x3|ApicRegOffset::Version|
|0xFEE0_0100|0x100|0x10|ApicRegOffset::ISR(ISRIndex0)|
|0xFEE0_0320|0x320|0x32|ApicRegOffset::LvtTimer|

Sources: [src/consts.rs(L192 - L203)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L192-L203) [src/consts.rs(L197 - L198)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L197-L198) [src/lib.rs(L90)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L90-L90) [src/lib.rs(L105)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L105-L105)

## x2APIC MSR Address Translation

The x2APIC mode uses Model-Specific Registers (MSRs) for register access. MSR addresses in the range `0x800` to `0x8FF` are translated using the `x2apic_msr_access_reg()` function.

### Translation Process

```mermaid
flowchart TD
subgraph Constants["Constants"]
    MSR_BASE["X2APIC_MSE_REG_BASE0x800"]
    MSR_SIZE["X2APIC_MSE_REG_SIZE0x100 (256 registers)"]
end
START["System Register AddressSysRegAddr"]
SUBTRACT["Subtract Base Addressaddr.addr() - X2APIC_MSE_REG_BASEConvert to relative offset"]
CONVERT["ApicRegOffset::from()Map offset to enum variant"]
END["ApicRegOffset enum"]

CONVERT --> END
START --> SUBTRACT
SUBTRACT --> CONVERT
```

The translation is simpler than xAPIC because x2APIC MSRs directly correspond to register offsets:

1. **Calculate relative offset**: `addr.addr() - X2APIC_MSE_REG_BASE` gives the register offset
2. **Convert to enum**: `ApicRegOffset::from(offset)` maps the offset to the enum variant

### MSR Address Mapping Examples

|MSR Address|Relative Offset|ApicRegOffset Enum|
| --- | --- | --- |
|0x802|0x2|ApicRegOffset::ID|
|0x803|0x3|ApicRegOffset::Version|
|0x810|0x10|ApicRegOffset::ISR(ISRIndex0)|
|0x832|0x32|ApicRegOffset::LvtTimer|

Sources: [src/consts.rs(L205 - L216)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L205-L216) [src/consts.rs(L210 - L211)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L210-L211) [src/lib.rs(L137)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L137-L137) [src/lib.rs(L152)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/lib.rs#L152-L152)

## ApicRegOffset Enum Structure

The `ApicRegOffset` enum serves as the unified internal representation for all APIC registers. It categorizes registers into distinct types and handles special cases like register arrays.

```mermaid
flowchart TD
subgraph subGraph5["Index Enums"]
    ISR_INDEX["ISRIndexISRIndex0-ISRIndex7"]
    TMR_INDEX["TMRIndexTMRIndex0-TMRIndex7"]
    IRR_INDEX["IRRIndexIRRIndex0-IRRIndex7"]
end
subgraph subGraph4["ApicRegOffset Enum Variants"]
    LVT_LINT0["LvtLint0 (0x35)"]
    SIVR_REG["SIVR (0xF)"]
    subgraph subGraph1["Interrupt State Arrays"]
        ISR_ARRAY["ISR(ISRIndex)0x10-0x178 registers"]
        TMR_ARRAY["TMR(TMRIndex)0x18-0x1F8 registers"]
        IRR_ARRAY["IRR(IRRIndex)0x20-0x278 registers"]
    end
    subgraph subGraph3["Timer Control"]
        TIMER_INIT["TimerInitCount (0x38)"]
        TIMER_CUR["TimerCurCount (0x39)"]
        TIMER_DIV["TimerDivConf (0x3E)"]
        LVT_TIMER["LvtTimer (0x32)"]
        LVT_THERMAL["LvtThermal (0x33)"]
        LVT_PMC["LvtPmc (0x34)"]
        ID_REG["ID (0x2)"]
        VER_REG["Version (0x3)"]
        TPR_REG["TPR (0x8)"]
    end
    subgraph subGraph2["Local Vector Table"]
        LVT_LINT1["LvtLint1 (0x36)"]
        LVT_ERR["LvtErr (0x37)"]
        LVT_CMCI["LvtCMCI (0x2F)"]
        subgraph subGraph0["Basic Control Registers"]
            TIMER_INIT["TimerInitCount (0x38)"]
            TIMER_CUR["TimerCurCount (0x39)"]
            TIMER_DIV["TimerDivConf (0x3E)"]
            LVT_TIMER["LvtTimer (0x32)"]
            LVT_THERMAL["LvtThermal (0x33)"]
            LVT_PMC["LvtPmc (0x34)"]
            LVT_LINT0["LvtLint0 (0x35)"]
            ID_REG["ID (0x2)"]
            VER_REG["Version (0x3)"]
            TPR_REG["TPR (0x8)"]
            SIVR_REG["SIVR (0xF)"]
        end
    end
end

IRR_ARRAY --> IRR_INDEX
ISR_ARRAY --> ISR_INDEX
TMR_ARRAY --> TMR_INDEX
```

### Register Array Handling

The enum includes special handling for register arrays (ISR, TMR, IRR) that span multiple consecutive offsets. Each array uses a dedicated index enum generated by the `define_index_enum!` macro:

* **ISR (In-Service Register)**: 8 registers at offsets `0x10-0x17`
* **TMR (Trigger Mode Register)**: 8 registers at offsets `0x18-0x1F`
* **IRR (Interrupt Request Register)**: 8 registers at offsets `0x20-0x27`

The index calculation uses: `IndexType::from(offset - base_offset)` where `base_offset` is the starting offset for each array.

Sources: [src/consts.rs(L61 - L114)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L61-L114) [src/consts.rs(L129 - L131)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L129-L131) [src/consts.rs(L3 - L54)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L3-L54) [src/consts.rs(L56 - L58)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L56-L58)

## Translation Implementation Details

The core translation logic is implemented in the `ApicRegOffset::from()` method, which uses pattern matching to map numeric offsets to enum variants.

### Pattern Matching Strategy

```mermaid
flowchart TD
START["Numeric Offset (usize)"]
CAST["Cast to u32"]
MATCH["Pattern Match"]
SINGLE["Single Register0x2, 0x3, 0x8, etc."]
RANGE["Range Patterns0x10..=0x170x18..=0x1F0x20..=0x27"]
INVALID["Invalid Offsetpanic!"]
SINGLE_VARIANT["Direct Enum VariantApicRegOffset::ID"]
ARRAY_VARIANT["Array Enum VariantApicRegOffset::ISR(index)"]

CAST --> MATCH
MATCH --> INVALID
MATCH --> RANGE
MATCH --> SINGLE
RANGE --> ARRAY_VARIANT
SINGLE --> SINGLE_VARIANT
START --> CAST
```

The implementation handles three categories of offsets:

1. **Direct mappings**: Single offsets like `0x2 => ApicRegOffset::ID`
2. **Range mappings**: Consecutive ranges like `0x10..=0x17 => ApicRegOffset::ISR(index)`
3. **Invalid offsets**: Any unmapped offset triggers a panic

### Index Calculation for Arrays

For register arrays, the index is calculated by subtracting the base offset:

```
// Example for ISR array (offset 0x10-0x17)
0x10..=0x17 => ApicRegOffset::ISR(ISRIndex::from(value - 0x10))
```

This ensures that offset `0x10` maps to `ISRIndex0`, offset `0x11` maps to `ISRIndex1`, and so on.

Sources: [src/consts.rs(L116 - L148)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L116-L148) [src/consts.rs(L129 - L131)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L129-L131) [src/consts.rs(L19 - L31)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/src/consts.rs#L19-L31)
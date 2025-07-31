# CPU Interface (Vgicc)

> **Relevant source files**
> * [src/consts.rs](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/consts.rs)
> * [src/vgicc.rs](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs)

This document covers the `Vgicc` struct, which implements the per-CPU interface portion of the virtual Generic Interrupt Controller (GIC). The `Vgicc` manages CPU-specific interrupt state, list registers for interrupt virtualization, and processor state saving/restoration.

For information about the main VGIC controller that coordinates multiple CPU interfaces, see [Virtual GIC Controller (Vgic)](/arceos-hypervisor/arm_vgic/3.1-virtual-gic-controller-(vgic)). For system-wide constants and register layouts, see [Constants and Register Layout](/arceos-hypervisor/arm_vgic/3.4-constants-and-register-layout).

## Structure Overview

The `Vgicc` struct represents a single CPU's interface to the virtual interrupt controller. Each CPU core in the virtualized system has its own `Vgicc` instance that maintains independent interrupt state and configuration.

### Vgicc Struct Layout

```mermaid
flowchart TD
subgraph subGraph3["Vgicc Struct"]
    ID["id: u32CPU Identifier"]
    subgraph subGraph1["Processor State Registers"]
        SAVED_ELSR["saved_elsr0: u32Empty List Status Register"]
        SAVED_APR["saved_apr: u32Active Priority Register"]
        SAVED_HCR["saved_hcr: u32Hypervisor Control Register"]
    end
    subgraph subGraph0["List Register Management"]
        PENDING_LR["pending_lr: [u32; 512]SPI Pending List Registers"]
        SAVED_LR["saved_lr: [u32; 4]Hardware List Register State"]
    end
    subgraph subGraph2["Interrupt Configuration"]
        ISENABLER["isenabler: u32SGI/PPI Enable Bits 0-31"]
        PRIORITYR["priorityr: [u8; 32]PPI Priority Configuration"]
    end
end

ID --> PENDING_LR
ISENABLER --> PRIORITYR
SAVED_ELSR --> SAVED_LR
SAVED_LR --> PENDING_LR
```

Sources: [src/vgicc.rs(L3 - L14)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs#L3-L14)

### Field Categories and Responsibilities

|Category|Fields|Purpose|
| --- | --- | --- |
|CPU Identity|id|Uniquely identifies the CPU core this interface serves|
|List Register State|pending_lr,saved_lr|Manages virtual interrupt injection through hardware list registers|
|Processor Context|saved_elsr0,saved_apr,saved_hcr|Preserves CPU-specific GIC state during VM context switches|
|Interrupt Control|isenabler,priorityr|Configures interrupt enables and priorities for SGIs and PPIs|

Sources: [src/vgicc.rs(L4 - L13)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs#L4-L13) [src/consts.rs(L1 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/consts.rs#L1-L4)

## List Register Management

The `Vgicc` manages two types of list register arrays that are central to ARM GIC virtualization:

### Hardware List Registers vs Pending Arrays

```mermaid
flowchart TD
subgraph subGraph2["Interrupt Sources"]
    SPI["SPI Interrupts(32-543)"]
    PPI_SGI["PPI/SGI Interrupts(0-31)"]
end
subgraph subGraph1["Vgicc State"]
    SAVED_LR["saved_lr[4]Hardware LR Backup"]
    PENDING_LR["pending_lr[512]SPI Virtual Queue"]
    SAVED_ELSR["saved_elsr0Empty LR Status"]
end
subgraph subGraph0["Physical Hardware"]
    HW_LR["Physical List Registers(4 registers)"]
end

HW_LR --> SAVED_LR
PENDING_LR --> SAVED_LR
PPI_SGI --> HW_LR
SAVED_LR --> HW_LR
SAVED_LR --> SAVED_ELSR
SPI --> PENDING_LR
```

Sources: [src/vgicc.rs(L5 - L6)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs#L5-L6) [src/consts.rs(L3 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/consts.rs#L3-L4)

The `saved_lr` array holds the contents of the 4 physical hardware list registers, while `pending_lr` provides a much larger virtual queue for SPI interrupts that cannot fit in the limited hardware registers.

## Processor State Management

The `Vgicc` maintains three critical processor state registers that must be preserved across VM context switches:

### State Register Functions

```mermaid
flowchart TD
subgraph subGraph2["Hardware Registers"]
    HW_HCR["Physical GICH_HCR"]
    HW_APR["Physical GICH_APR"]
    HW_ELSR["Physical GICH_ELSR"]
end
subgraph subGraph1["Vgicc State Preservation"]
    HCR["saved_hcrHypervisor Control• Virtual IRQ/FIQ enables• List register usage"]
    APR["saved_aprActive Priority• Currently active interrupt priority• Priority grouping config"]
    ELSR["saved_elsr0Empty List Status• Which LRs are available• Hardware resource tracking"]
end
subgraph subGraph0["VM Context Switch"]
    VM_EXIT["VM Exit"]
    VM_ENTRY["VM Entry"]
end

APR --> HW_APR
APR --> VM_ENTRY
ELSR --> HW_ELSR
ELSR --> VM_ENTRY
HCR --> HW_HCR
HCR --> VM_ENTRY
VM_EXIT --> APR
VM_EXIT --> ELSR
VM_EXIT --> HCR
```

Sources: [src/vgicc.rs(L8 - L10)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs#L8-L10)

## Interrupt Configuration State

The `Vgicc` maintains per-CPU interrupt configuration for Software Generated Interrupts (SGIs) and Private Peripheral Interrupts (PPIs):

### SGI and PPI Management

```mermaid
flowchart TD
subgraph subGraph2["Per-bit Mapping"]
    BIT0["Bit 0: SGI 0"]
    BIT15["Bit 15: SGI 15"]
    BIT16["Bit 16: PPI 16"]
    BIT31["Bit 31: PPI 31"]
end
subgraph subGraph1["Vgicc Configuration"]
    ISENABLER["isenabler: u3232-bit enable maskOne bit per interrupt 0-31"]
    PRIORITYR["priorityr: [u8; 32]8-bit priority per PPIIndices 0-31 for PPI IDs 0-31"]
end
subgraph subGraph0["Interrupt ID Space"]
    SGI["SGI IDs 0-15Software Generated"]
    PPI["PPI IDs 16-31Private Peripheral"]
end

ISENABLER --> BIT0
ISENABLER --> BIT15
ISENABLER --> BIT16
ISENABLER --> BIT31
PPI --> ISENABLER
PPI --> PRIORITYR
SGI --> ISENABLER
```

Sources: [src/vgicc.rs(L12 - L13)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs#L12-L13) [src/consts.rs(L1 - L2)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/consts.rs#L1-L2)

The `isenabler` field uses individual bits to track enable/disable state for each of the 32 SGI and PPI interrupts, while `priorityr` stores 8-bit priority values specifically for the 32 PPI interrupts.

## Integration with VGIC System

The `Vgicc` operates as part of the larger VGIC virtualization system:

### System Integration Flow

```mermaid
flowchart TD
subgraph subGraph2["Physical Hardware"]
    PHYSICAL_GIC["arm_gicv2Physical GIC Driver"]
    HW_CPU_IF["Hardware CPUInterface Registers"]
end
subgraph subGraph1["VGIC System"]
    VGIC_MAIN["Vgic Controller(Main coordinator)"]
    VGICC_INSTANCES["Multiple Vgicc(Per-CPU instances)"]
end
subgraph subGraph0["Guest VM"]
    GUEST["Guest OSInterrupt Operations"]
end

GUEST --> VGIC_MAIN
PHYSICAL_GIC --> HW_CPU_IF
VGICC_INSTANCES --> HW_CPU_IF
VGICC_INSTANCES --> PHYSICAL_GIC
VGIC_MAIN --> VGICC_INSTANCES
```

Sources: [src/vgicc.rs(L1 - L14)&emsp;](https://github.com/arceos-hypervisor/arm_vgic/blob/2fa3fe56/src/vgicc.rs#L1-L14)

Each `Vgicc` instance manages the virtualization state for one CPU core, coordinating with the main `Vgic` controller to provide a complete virtual interrupt controller implementation to guest operating systems.
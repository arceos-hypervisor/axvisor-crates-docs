# Interrupt Types and Management

> **Relevant source files**
> * [src/gic_v2.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs)
> * [src/lib.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs)

This document covers the interrupt classification system and management operations provided by the `arm_gicv2` crate. It details the three interrupt types supported by ARM GICv2 (SGI, PPI, SPI), their characteristics, and how they are managed through the distributor and CPU interface components.

For information about the core GIC architecture and components, see [Core Architecture](/arceos-hypervisor/arm_gicv2/2-core-architecture). For detailed register-level operations, see [Register Interface](/arceos-hypervisor/arm_gicv2/4-register-interface).

## Interrupt Classification System

The ARM GICv2 specification defines three distinct interrupt types, each serving different purposes and having specific characteristics. The crate implements this classification through constants, enums, and a translation function.

### Interrupt Type Definitions

```mermaid
flowchart TD
subgraph subGraph2["Range Constants"]
    SGI_RANGE_CONST["SGI_RANGE: 0..16"]
    PPI_RANGE_CONST["PPI_RANGE: 16..32"]
    SPI_RANGE_CONST["SPI_RANGE: 32..1020"]
end
subgraph subGraph1["InterruptType Enum"]
    SGI_TYPE["InterruptType::SGI"]
    PPI_TYPE["InterruptType::PPI"]
    SPI_TYPE["InterruptType::SPI"]
end
subgraph subGraph0["Interrupt ID Space (0-1019)"]
    SGI_SPACE["SGI Range0-15Software Generated"]
    PPI_SPACE["PPI Range16-31Private Peripheral"]
    SPI_SPACE["SPI Range32-1019Shared Peripheral"]
end

PPI_RANGE_CONST --> PPI_SPACE
PPI_TYPE --> PPI_SPACE
SGI_RANGE_CONST --> SGI_SPACE
SGI_TYPE --> SGI_SPACE
SPI_RANGE_CONST --> SPI_SPACE
SPI_TYPE --> SPI_SPACE
```

**Sources**: [src/lib.rs(L14 - L29)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs#L14-L29) [src/lib.rs(L74 - L89)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs#L74-L89)

|Interrupt Type|Range|Purpose|CPU Scope|
| --- | --- | --- | --- |
|SGI (Software Generated)|0-15|Inter-processor communication|All CPUs|
|PPI (Private Peripheral)|16-31|CPU-specific peripherals|Single CPU|
|SPI (Shared Peripheral)|32-1019|System-wide peripherals|Multiple CPUs|

### Interrupt Translation System

The `translate_irq` function provides a safe way to convert logical interrupt IDs within each type to absolute GIC interrupt IDs.

```mermaid
flowchart TD
INPUT["Logical ID + InterruptType"]
TRANSLATE_IRQ["translate_irq()"]
SGI_CHECK["id < 16?"]
PPI_CHECK["id < 16?"]
SPI_CHECK["id < 988?"]
SGI_RESULT["Some(id)"]
SGI_NONE["None"]
PPI_RESULT["Some(id + 16)"]
PPI_NONE["None"]
SPI_RESULT["Some(id + 32)"]
SPI_NONE["None"]
GIC_INTID["GIC INTID"]

INPUT --> TRANSLATE_IRQ
PPI_CHECK --> PPI_NONE
PPI_CHECK --> PPI_RESULT
PPI_RESULT --> GIC_INTID
SGI_CHECK --> SGI_NONE
SGI_CHECK --> SGI_RESULT
SGI_RESULT --> GIC_INTID
SPI_CHECK --> SPI_NONE
SPI_CHECK --> SPI_RESULT
SPI_RESULT --> GIC_INTID
TRANSLATE_IRQ --> PPI_CHECK
TRANSLATE_IRQ --> SGI_CHECK
TRANSLATE_IRQ --> SPI_CHECK
```

**Sources**: [src/lib.rs(L91 - L116)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs#L91-L116)

## Interrupt Management Operations

The GIC provides different management capabilities depending on the interrupt type. These operations are primarily handled through the `GicDistributor` component.

### Core Management Functions

```mermaid
flowchart TD
subgraph subGraph1["Interrupt Type Applicability"]
    SGI_OPS["SGI Operations• send_sgi()• send_sgi_all_except_self()• send_sgi_to_self()• set_pend() (special)"]
    PPI_OPS["PPI Operations• set_enable()• set_priority()• set_state()"]
    SPI_OPS["SPI Operations• configure_interrupt()• set_enable()• set_priority()• set_target_cpu()• set_state()"]
end
subgraph subGraph0["GicDistributor Management Methods"]
    CONFIGURE["configure_interrupt()"]
    ENABLE["set_enable() / get_enable()"]
    PRIORITY["set_priority() / get_priority()"]
    TARGET["set_target_cpu() / get_target_cpu()"]
    STATE["set_state() / get_state()"]
    PEND["set_pend()"]
end

CONFIGURE --> SPI_OPS
ENABLE --> PPI_OPS
ENABLE --> SPI_OPS
PEND --> SGI_OPS
PRIORITY --> PPI_OPS
PRIORITY --> SPI_OPS
STATE --> PPI_OPS
STATE --> SGI_OPS
STATE --> SPI_OPS
TARGET --> SPI_OPS
```

**Sources**: [src/gic_v2.rs(L161 - L283)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L161-L283) [src/gic_v2.rs(L201 - L223)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L201-L223)

### Trigger Mode Configuration

Only SPI interrupts support configurable trigger modes. The system restricts trigger mode configuration to the SPI range.

```mermaid
flowchart TD
CONFIG_REQ["configure_interrupt(vector, trigger_mode)"]
RANGE_CHECK["vector >= 32?"]
EARLY_RETURN["return (invalid)"]
MAX_CHECK["vector < max_irqs?"]
CALCULATE["Calculate ICFGR register"]
REG_ACCESS["Access ICFGR[reg_idx]"]
MODE_CHECK["TriggerMode?"]
SET_EDGE["Set bit (Edge)"]
CLEAR_BIT["Clear bit (Level)"]
WRITE_REG["Write ICFGR register"]

CALCULATE --> REG_ACCESS
CLEAR_BIT --> WRITE_REG
CONFIG_REQ --> RANGE_CHECK
MAX_CHECK --> CALCULATE
MAX_CHECK --> EARLY_RETURN
MODE_CHECK --> CLEAR_BIT
MODE_CHECK --> SET_EDGE
RANGE_CHECK --> EARLY_RETURN
RANGE_CHECK --> MAX_CHECK
REG_ACCESS --> MODE_CHECK
SET_EDGE --> WRITE_REG
```

**Sources**: [src/gic_v2.rs(L161 - L178)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L161-L178)

## Software Generated Interrupt Management

SGIs have special handling due to their role in inter-processor communication. They use dedicated register interfaces and targeting mechanisms.

### SGI Generation Methods

|Method|Target|Use Case|
| --- | --- | --- |
|send_sgi(dest_cpu_id, sgi_num)|Specific CPU|Direct communication|
|send_sgi_all_except_self(sgi_num)|All other CPUs|Broadcast operations|
|send_sgi_to_self(sgi_num)|Current CPU|Self-signaling|

```mermaid
flowchart TD
subgraph subGraph1["GICD_SGIR Register Fields"]
    TARGET_FILTER["TargetListFilter"]
    CPU_LIST["CPUTargetList"]
    SGI_ID["SGIINTID"]
end
subgraph subGraph0["SGI Targeting Modes"]
    SPECIFIC["ForwardToCPUTargetList+ CPUTargetList"]
    BROADCAST["ForwardToAllExceptRequester"]
    SELF["ForwardToRequester"]
end
GICD_SGIR_REG["GICD_SGIR Register"]

BROADCAST --> TARGET_FILTER
CPU_LIST --> GICD_SGIR_REG
SELF --> TARGET_FILTER
SGI_ID --> GICD_SGIR_REG
SPECIFIC --> CPU_LIST
SPECIFIC --> TARGET_FILTER
TARGET_FILTER --> GICD_SGIR_REG
```

**Sources**: [src/gic_v2.rs(L201 - L223)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L201-L223) [src/regs/gicd_sgir.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/gicd_sgir.rs)

### SGI Pending State Management

SGIs require special handling for pending state management due to their per-CPU nature:

```mermaid
flowchart TD
SET_PEND["set_pend(int_id, is_pend, current_cpu_id)"]
SGI_CHECK["int_id in SGI_RANGE?"]
SGI_CALC["Calculate register offset"]
NORMAL_PEND["Use ISPENDR/ICPENDR"]
PEND_CHECK["is_pend?"]
SET_SPENDSGIR["Set SPENDSGIR bit"]
CLEAR_CPENDSGIR["Clear CPENDSGIR bits"]
CPU_SPECIFIC["Target specific CPU"]
ALL_CPUS["Clear for all CPUs"]

CLEAR_CPENDSGIR --> ALL_CPUS
PEND_CHECK --> CLEAR_CPENDSGIR
PEND_CHECK --> SET_SPENDSGIR
SET_PEND --> SGI_CHECK
SET_SPENDSGIR --> CPU_SPECIFIC
SGI_CALC --> PEND_CHECK
SGI_CHECK --> NORMAL_PEND
SGI_CHECK --> SGI_CALC
```

**Sources**: [src/gic_v2.rs(L264 - L283)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L264-L283)

## Integration with GIC Components

Interrupt management operations interface with both the distributor and CPU interface components, with different responsibilities for each type.

### Distributor vs CPU Interface Responsibilities

```mermaid
flowchart TD
subgraph subGraph2["Interrupt Flow"]
    PERIPHERAL["Peripheral Device"]
    HANDLER["Interrupt Handler"]
end
subgraph subGraph1["GicCpuInterface Responsibilities"]
    CPU_ACK["Acknowledgment• iar() register read• Interrupt priority"]
    CPU_EOI["Completion• eoi() priority drop• dir() deactivation"]
    CPU_MASK["Priority Masking• PMR register• Preemption control"]
end
subgraph subGraph0["GicDistributor Responsibilities"]
    DIST_CONFIG["Configuration• Trigger modes• Priority levels• CPU targeting"]
    DIST_ENABLE["Enable/Disable• Per-interrupt control• Group assignment"]
    DIST_SGI["SGI Generation• Inter-processor signals• Targeting modes"]
    DIST_STATE["State Management• Pending state• Active state"]
end

CPU_ACK --> HANDLER
DIST_CONFIG --> CPU_ACK
HANDLER --> CPU_EOI
PERIPHERAL --> DIST_CONFIG
```

**Sources**: [src/gic_v2.rs(L92 - L131)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L92-L131) [src/gic_v2.rs(L376 - L479)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L376-L479)

The interrupt management system provides a complete abstraction over ARM GICv2 hardware while maintaining type safety and proper encapsulation of interrupt-type-specific behaviors.
# Interrupt Processing Pipeline

> **Relevant source files**
> * [src/gic_v2.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs)
> * [src/lib.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs)

This document describes the complete interrupt processing pipeline in the ARM GICv2 implementation, from interrupt generation through final completion. It covers the coordination between the `GicDistributor` and `GicCpuInterface` components and the flow of control through the hardware abstraction layer.

For details about the individual components, see [GIC Distributor Component](/arceos-hypervisor/arm_gicv2/2.1-gic-distributor-component) and [CPU Interface Component](/arceos-hypervisor/arm_gicv2/2.2-cpu-interface-component). For information about interrupt classification, see [Interrupt Classification System](/arceos-hypervisor/arm_gicv2/3.1-interrupt-classification-system).

## Pipeline Overview

The interrupt processing pipeline consists of five main stages that coordinate between hardware interrupt sources, the GIC Distributor, CPU interfaces, and software interrupt handlers.

```mermaid
flowchart TD
HW_SOURCE["Hardware Peripheral"]
DIST_RECEIVE["GicDistributor receives interrupt"]
SW_SGI["Software SGI"]
DIST_CHECK["Distributor Processing:• Check enable/disable• Evaluate priority• Select target CPU• Check trigger mode"]
CPU_FORWARD["Forward to GicCpuInterface"]
CPU_SIGNAL["CPU Interface signals processor"]
PROC_ACK["Processor acknowledges via IAR read"]
HANDLER_EXEC["Interrupt handler execution"]
PROC_EOI["Processor signals completion via EOIR"]
OPTIONAL_DIR["Optional: Deactivation via DIR(EL2 mode only)"]
COMPLETE["Interrupt processing complete"]

CPU_FORWARD --> CPU_SIGNAL
CPU_SIGNAL --> PROC_ACK
DIST_CHECK --> CPU_FORWARD
DIST_RECEIVE --> DIST_CHECK
HANDLER_EXEC --> PROC_EOI
HW_SOURCE --> DIST_RECEIVE
OPTIONAL_DIR --> COMPLETE
PROC_ACK --> HANDLER_EXEC
PROC_EOI --> OPTIONAL_DIR
SW_SGI --> DIST_RECEIVE
```

Sources: [src/gic_v2.rs(L1 - L480)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L1-L480) [src/lib.rs(L1 - L117)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs#L1-L117)

## Interrupt Generation Stage

Interrupts enter the pipeline through three different mechanisms based on their type classification.

```mermaid
flowchart TD
subgraph SPI_GEN["SPI Generation (32-1019)"]
    UART_INT["UART interrupts"]
    GPIO_INT["GPIO interrupts"]
    OTHER_SPI["Other shared peripherals"]
end
subgraph PPI_GEN["PPI Generation (16-31)"]
    TIMER_INT["Timer interrupts"]
    PMU_INT["PMU interrupts"]
    OTHER_PPI["Other private peripherals"]
end
subgraph SGI_GEN["SGI Generation (0-15)"]
    SEND_SGI["send_sgi()"]
    SEND_ALL["send_sgi_all_except_self()"]
    SEND_SELF["send_sgi_to_self()"]
end
GICD_SGIR["GICD_SGIR register write"]
HW_SIGNAL["Hardware interrupt signal"]
DIST_INPUT["GicDistributor input"]

GICD_SGIR --> DIST_INPUT
GPIO_INT --> HW_SIGNAL
HW_SIGNAL --> DIST_INPUT
OTHER_PPI --> HW_SIGNAL
OTHER_SPI --> HW_SIGNAL
PMU_INT --> HW_SIGNAL
SEND_ALL --> GICD_SGIR
SEND_SELF --> GICD_SGIR
SEND_SGI --> GICD_SGIR
TIMER_INT --> HW_SIGNAL
UART_INT --> HW_SIGNAL
```

Sources: [src/gic_v2.rs(L201 - L223)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L201-L223) [src/lib.rs(L14 - L29)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs#L14-L29)

## Distributor Processing Stage

The `GicDistributor` performs comprehensive interrupt management including configuration validation, priority evaluation, and CPU targeting.

|Function|Register Access|Purpose|
| --- | --- | --- |
|set_enable()/get_enable()|ISENABLER/ICENABLER|Enable/disable interrupt delivery|
|set_priority()/get_priority()|IPRIORITYR|Configure interrupt priority levels|
|set_target_cpu()/get_target_cpu()|ITARGETSR|Select target processor for SPI interrupts|
|configure_interrupt()|ICFGR|Set edge/level trigger mode|
|set_state()/get_state()|ISPENDR/ICPENDR/ISACTIVER/ICACTIVER|Manage pending and active states|

```mermaid
flowchart TD
INT_INPUT["Interrupt arrives at Distributor"]
TYPE_CHECK["Check interrupt type"]
SGI_PROC["SGI Processing:• Check SPENDSGIR/CPENDSGIR• Inter-processor targeting"]
PPI_PROC["PPI Processing:• CPU-private handling• No target selection needed"]
SPI_PROC["SPI Processing:• Check ITARGETSR for target CPU• Multi-processor capable"]
ENABLE_CHECK["Check ISENABLER register"]
DROP["Drop interrupt"]
PRIORITY_CHECK["Check IPRIORITYR priority"]
TRIGGER_CHECK["Check ICFGR trigger mode"]
FORWARD["Forward to target GicCpuInterface"]

ENABLE_CHECK --> DROP
ENABLE_CHECK --> PRIORITY_CHECK
INT_INPUT --> TYPE_CHECK
PPI_PROC --> ENABLE_CHECK
PRIORITY_CHECK --> TRIGGER_CHECK
SGI_PROC --> ENABLE_CHECK
SPI_PROC --> ENABLE_CHECK
TRIGGER_CHECK --> FORWARD
TYPE_CHECK --> PPI_PROC
TYPE_CHECK --> SGI_PROC
TYPE_CHECK --> SPI_PROC
```

Sources: [src/gic_v2.rs(L180 - L199)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L180-L199) [src/gic_v2.rs(L225 - L261)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L225-L261) [src/gic_v2.rs(L263 - L283)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L263-L283)

## CPU Interface Processing Stage

The `GicCpuInterface` manages per-CPU interrupt delivery, acknowledgment, and completion signaling.

```mermaid
flowchart TD
CPU_RECEIVE["GicCpuInterface receives interrupt"]
PRIORITY_MASK["Check PMR priority mask"]
QUEUE["Queue interrupt"]
SIGNAL_CPU["Signal connected processor"]
PROC_READ_IAR["Processor reads GICC_IAR"]
IAR_VALUE["IAR returns interrupt ID"]
HANDLER_CALL["Call interrupt handler"]
IGNORE["Ignore spurious interrupt"]
HANDLER_COMPLETE["Handler completes"]
EOIR_WRITE["Write GICC_EOIR"]
EL2_CHECK["Check EL2 mode"]
COMPLETE_NORMAL["Interrupt complete"]
DIR_WRITE["Write GICC_DIR to deactivate"]
COMPLETE_EL2["Interrupt complete (EL2)"]

CPU_RECEIVE --> PRIORITY_MASK
DIR_WRITE --> COMPLETE_EL2
EL2_CHECK --> COMPLETE_NORMAL
EL2_CHECK --> DIR_WRITE
EOIR_WRITE --> EL2_CHECK
HANDLER_CALL --> HANDLER_COMPLETE
HANDLER_COMPLETE --> EOIR_WRITE
IAR_VALUE --> HANDLER_CALL
IAR_VALUE --> IGNORE
PRIORITY_MASK --> QUEUE
PRIORITY_MASK --> SIGNAL_CPU
PROC_READ_IAR --> IAR_VALUE
SIGNAL_CPU --> PROC_READ_IAR
```

Sources: [src/gic_v2.rs(L394 - L396)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L394-L396) [src/gic_v2.rs(L406 - L408)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L406-L408) [src/gic_v2.rs(L416 - L418)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L416-L418) [src/gic_v2.rs(L443 - L459)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L443-L459)

## Register-Level Pipeline Flow

This diagram shows the specific register interactions during interrupt processing, mapping high-level operations to actual hardware register access.

```mermaid
sequenceDiagram
    participant HardwareSoftware as "Hardware/Software"
    participant GICDistributorRegisters as "GIC Distributor Registers"
    participant CPUInterfaceRegisters as "CPU Interface Registers"
    participant ARMProcessor as "ARM Processor"

    Note over HardwareSoftware,ARMProcessor: Interrupt Generation
    HardwareSoftware ->> GICDistributorRegisters: Interrupt signal / GICD_SGIR write
    Note over GICDistributorRegisters: Distributor Processing
    GICDistributorRegisters ->> GICDistributorRegisters: Check ISENABLER[n]
    GICDistributorRegisters ->> GICDistributorRegisters: Check IPRIORITYR[n]
    GICDistributorRegisters ->> GICDistributorRegisters: Check ITARGETSR[n] (SPI only)
    GICDistributorRegisters ->> GICDistributorRegisters: Check ICFGR[n] trigger mode
    Note over GICDistributorRegisters,CPUInterfaceRegisters: Forward to CPU Interface
    GICDistributorRegisters ->> CPUInterfaceRegisters: Forward interrupt to target CPU
    Note over CPUInterfaceRegisters: CPU Interface Processing
    CPUInterfaceRegisters ->> CPUInterfaceRegisters: Check PMR priority mask
    CPUInterfaceRegisters ->> ARMProcessor: Signal interrupt to processor
    Note over ARMProcessor: Processor Handling
    ARMProcessor ->> CPUInterfaceRegisters: Read GICC_IAR
    CPUInterfaceRegisters ->> ARMProcessor: Return interrupt ID
    ARMProcessor ->> ARMProcessor: Execute interrupt handler
    ARMProcessor ->> CPUInterfaceRegisters: Write GICC_EOIR
    alt EL2 Mode Enabled
        ARMProcessor ->> CPUInterfaceRegisters: Write GICC_DIR
    end
    Note over HardwareSoftware,ARMProcessor: Interrupt Complete
```

Sources: [src/gic_v2.rs(L20 - L90)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L20-L90) [src/gic_v2.rs(L443 - L459)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L443-L459)

## Pipeline Integration Points

The pipeline integrates with external components through well-defined interfaces that abstract the hardware complexity.

```mermaid
flowchart TD
subgraph RUNTIME["Runtime Processing"]
    HANDLE_IRQ_FUNC["GicCpuInterface::handle_irq()"]
    TRANSLATE_FUNC["translate_irq()"]
end
subgraph INIT["Initialization Phase"]
    DIST_INIT["GicDistributor::init()"]
    CPU_INIT["GicCpuInterface::init()"]
end
subgraph CONFIG["Configuration Interface"]
    SET_ENABLE["set_enable()"]
    SET_PRIORITY["set_priority()"]
    SET_TARGET["set_target_cpu()"]
    CONFIG_INT["configure_interrupt()"]
end
IAR_READ["GICC_IAR read"]
HANDLER_EXEC["User handler execution"]
EOIR_WRITE["GICC_EOIR write"]
DIR_WRITE_OPT["Optional GICC_DIR write"]

EOIR_WRITE --> DIR_WRITE_OPT
HANDLER_EXEC --> EOIR_WRITE
HANDLE_IRQ_FUNC --> IAR_READ
IAR_READ --> HANDLER_EXEC
TRANSLATE_FUNC --> HANDLE_IRQ_FUNC
```

Sources: [src/gic_v2.rs(L342 - L373)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L342-L373) [src/gic_v2.rs(L461 - L478)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L461-L478) [src/gic_v2.rs(L443 - L459)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L443-L459) [src/lib.rs(L91 - L116)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/lib.rs#L91-L116)

## Error Handling and Edge Cases

The pipeline includes robust handling for edge cases and error conditions.

|Condition|Detection Method|Response|
| --- | --- | --- |
|Spurious Interrupt|IAR & 0x3ff >= 1020|No handler called, no EOIR write|
|Disabled Interrupt|ISENABLER[n] == 0|Interrupt dropped at distributor|
|Invalid IRQ Range|vector >= max_irqs|Early return from configuration functions|
|Priority Masking|Priority < PMR|Interrupt queued at CPU interface|

The `handle_irq()` function provides a complete wrapper that handles these edge cases automatically:

```javascript
// Simplified logic from handle_irq() method
let iar = self.iar();
let vector = iar & 0x3ff;
if vector < 1020 {
    handler(vector);
    self.eoi(iar);
    #[cfg(feature = "el2")]
    if self.regs().CTLR.get() & GICC_CTLR_EOIMODENS_BIT != 0 {
        self.dir(iar);
    }
} else {
    // spurious interrupt - no action taken
}
```

Sources: [src/gic_v2.rs(L443 - L459)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L443-L459) [src/gic_v2.rs(L180 - L192)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L180-L192) [src/gic_v2.rs(L162 - L178)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/gic_v2.rs#L162-L178)
# System Architecture

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs)
> * [src/pcpu.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs)
> * [src/vcpu.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs)

This document provides a detailed explanation of the overall system architecture for the arm_vcpu hypervisor implementation. It covers the core components, their relationships, and the data flow patterns that enable AArch64 virtualization. For specific implementation details of individual components, see [Virtual CPU Management](/arceos-hypervisor/arm_vcpu/2-virtual-cpu-management), [Exception Handling System](/arceos-hypervisor/arm_vcpu/4-exception-handling-system), and [Context Switching and State Management](/arceos-hypervisor/arm_vcpu/3-context-switching-and-state-management).

## Architecture Overview

The arm_vcpu system implements a Type-1 hypervisor architecture for AArch64 platforms, providing virtualization capabilities through the AArch64 virtualization extensions. The system operates at Exception Level 2 (EL2) and manages guest virtual machines running at EL1/EL0.

### Core Component Relationships

```

```

**Sources:** [src/vcpu.rs(L39 - L51)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L39-L51) [src/pcpu.rs(L10 - L16)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L10-L16) [src/lib.rs(L17 - L21)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L17-L21)

### System Initialization and Lifecycle

```mermaid
sequenceDiagram
    participant HostSystem as "Host System"
    participant Aarch64PerCpu as "Aarch64PerCpu"
    participant Aarch64VCpu as "Aarch64VCpu"
    participant GuestVM as "Guest VM"

    HostSystem ->> Aarch64PerCpu: new(cpu_id)
    Aarch64PerCpu ->> Aarch64PerCpu: Register IRQ_HANDLER
    HostSystem ->> Aarch64PerCpu: hardware_enable()
    Aarch64PerCpu ->> Aarch64PerCpu: Set VBAR_EL2 to exception_vector_base_vcpu
    Aarch64PerCpu ->> Aarch64PerCpu: Configure HCR_EL2 for virtualization
    HostSystem ->> Aarch64VCpu: new(Aarch64VCpuCreateConfig)
    Aarch64VCpu ->> Aarch64VCpu: Initialize TrapFrame and GuestSystemRegisters
    HostSystem ->> Aarch64VCpu: setup()
    Aarch64VCpu ->> Aarch64VCpu: init_hv() - Configure initial state
    HostSystem ->> Aarch64VCpu: set_entry(guest_entry_point)
    HostSystem ->> Aarch64VCpu: set_ept_root(page_table_root)
    loop VM Execution Cycle
        HostSystem ->> Aarch64VCpu: run()
        Aarch64VCpu ->> Aarch64VCpu: save_host_sp_el0()
        Aarch64VCpu ->> Aarch64VCpu: restore_vm_system_regs()
        Aarch64VCpu ->> GuestVM: Context switch to guest
        GuestVM -->> Aarch64VCpu: VM-Exit (trap/exception)
        Aarch64VCpu ->> Aarch64VCpu: vmexit_handler()
        Aarch64VCpu -->> HostSystem: Return AxVCpuExitReason
    end
```

**Sources:** [src/vcpu.rs(L69 - L85)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L69-L85) [src/vcpu.rs(L99 - L111)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L99-L111) [src/pcpu.rs(L49 - L67)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L49-L67)

## Core Components

### Virtual CPU (Aarch64VCpu)

The `Aarch64VCpu<H: AxVCpuHal>` structure serves as the primary abstraction for a virtual CPU. It maintains both guest context and runtime state required for virtualization.

|Field|Type|Purpose|
| --- | --- | --- |
|ctx|TrapFrame|Guest general-purpose registers and execution state|
|host_stack_top|u64|Host stack pointer for context switching|
|guest_system_regs|GuestSystemRegisters|Guest system control and configuration registers|
|mpidr|u64|Multiprocessor Affinity Register value for guest|

The VCPU implements the `AxArchVCpu` trait, providing standardized interfaces for:

* Creation and configuration via `Aarch64VCpuCreateConfig`
* Guest execution through the `run()` method
* Entry point and page table configuration
* Register manipulation interfaces

**Sources:** [src/vcpu.rs(L39 - L51)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L39-L51) [src/vcpu.rs(L64 - L124)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L64-L124)

### Per-CPU Management (Aarch64PerCpu)

The `Aarch64PerCpu<H: AxVCpuHal>` structure manages hardware virtualization features on a per-CPU basis:

```mermaid
flowchart TD
PCPU["Aarch64PerCpu"]
HCR["HCR_EL2 RegisterHypervisor Configuration"]
VBAR["VBAR_EL2 RegisterException Vector Base"]
IRQ["IRQ_HANDLERPer-CPU IRQ Dispatch"]
ORIG["ORI_EXCEPTION_VECTOR_BASEOriginal Vector Base"]
VM["VM Bit - Virtualization"]
RW["RW Bit - 64-bit Guest"]
IMO["IMO/FMO - Virtual Interrupts"]
TSC["TSC - SMC Instructions"]

HCR --> IMO
HCR --> RW
HCR --> TSC
HCR --> VM
PCPU --> HCR
PCPU --> IRQ
PCPU --> ORIG
PCPU --> VBAR
```

**Sources:** [src/pcpu.rs(L10 - L16)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L10-L16) [src/pcpu.rs(L49 - L67)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L49-L67) [src/pcpu.rs(L18 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L18-L26)

### Hardware Abstraction Layer

The system uses the `AxVCpuHal` trait to abstract platform-specific functionality:

```

```

**Sources:** [src/vcpu.rs(L278)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L278-L278) [src/pcpu.rs(L35 - L37)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L35-L37)

## Context Switching Architecture

The system implements a sophisticated context switching mechanism that preserves both general-purpose and system registers across VM entries and exits:

```mermaid
flowchart TD
subgraph subGraph2["Guest Context"]
    TRAP["TrapFrame(GPRs, SP_EL0, ELR, SPSR)"]
    GSYS["GuestSystemRegisters(System Control Registers)"]
end
subgraph subGraph1["Context Switch Operations"]
    SAVE_HOST["save_host_sp_el0()"]
    RESTORE_VM["restore_vm_system_regs()"]
    RUN_GUEST["run_guest()(Naked Function)"]
    VMEXIT["vmexit_handler()"]
end
subgraph subGraph0["Host Context"]
    HSP["Host SP_EL0(HOST_SP_EL0 per-CPU)"]
    HSTACK["Host Stack(save_regs_to_stack!)"]
    HSYS["Host System Registers(Original Hardware State)"]
end

GSYS --> VMEXIT
HSP --> SAVE_HOST
RESTORE_VM --> GSYS
RESTORE_VM --> RUN_GUEST
RUN_GUEST --> TRAP
SAVE_HOST --> RESTORE_VM
TRAP --> VMEXIT
VMEXIT --> HSP
VMEXIT --> HSTACK
```

**Sources:** [src/vcpu.rs(L182 - L214)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L182-L214) [src/vcpu.rs(L226 - L244)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L226-L244) [src/vcpu.rs(L255 - L282)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L255-L282)

## Exception and Interrupt Handling

The system provides a multi-layered exception handling architecture that processes VM-exits and routes them appropriately:

```mermaid
flowchart TD
subgraph subGraph3["Exit Reasons"]
    MMIO["AxVCpuExitReason::MmioRead/Write"]
    EXT_IRQ["AxVCpuExitReason::ExternalInterrupt"]
    SYSREG["AxVCpuExitReason::SystemRegister*"]
    PSCI["AxVCpuExitReason::CpuUp/Down"]
end
subgraph subGraph2["High-Level Dispatch"]
    SYNC_HANDLER["handle_exception_sync()"]
    IRQ_DISPATCH["IRQ_HANDLER per-CPU"]
    PANIC["invalid_exception_el2"]
end
subgraph subGraph1["Low-Level Processing"]
    VECTORS["exception_vector_base_vcpu(Assembly Vector Table)"]
    TRAMPOLINE["vmexit_trampoline(Context Save)"]
end
subgraph subGraph0["Exception Sources"]
    SYNC["Synchronous Exceptions(Data Aborts, HVC, SMC)"]
    IRQ["IRQ Interrupts"]
    INVALID["Invalid Exceptions"]
end

INVALID --> VECTORS
IRQ --> VECTORS
IRQ_DISPATCH --> EXT_IRQ
SYNC --> VECTORS
SYNC_HANDLER --> MMIO
SYNC_HANDLER --> PSCI
SYNC_HANDLER --> SYSREG
TRAMPOLINE --> IRQ_DISPATCH
TRAMPOLINE --> PANIC
TRAMPOLINE --> SYNC_HANDLER
VECTORS --> TRAMPOLINE
```

**Sources:** [src/vcpu.rs(L275 - L281)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L275-L281) [src/pcpu.rs(L18 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L18-L26) [src/pcpu.rs(L55 - L57)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L55-L57)

This architecture enables efficient virtualization by maintaining clear separation between host and guest contexts while providing comprehensive exception handling capabilities for all VM-exit scenarios.
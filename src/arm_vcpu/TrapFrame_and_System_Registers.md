# TrapFrame and System Registers

> **Relevant source files**
> * [src/context_frame.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs)

This document details the core data structures used for preserving CPU state during context switches in the arm_vcpu hypervisor. The system uses two primary structures: `Aarch64ContextFrame` for saving general-purpose registers and basic execution state, and `GuestSystemRegisters` for preserving system control registers and hypervisor-specific state.

For information about the broader context switching mechanism and how these structures are used during VM exits, see [Context Switching and State Management](/arceos-hypervisor/arm_vcpu/3-context-switching-and-state-management). For details on the assembly-level context switching code that manipulates these structures, see [Assembly Exception Vectors](/arceos-hypervisor/arm_vcpu/4.1-assembly-exception-vectors).

## TrapFrame Structure Overview

The `Aarch64ContextFrame` serves as the primary trap frame for saving guest execution context when exceptions occur. This structure captures the minimal set of registers needed to preserve the guest's execution state across VM exits.

```mermaid
flowchart TD
subgraph subGraph2["Special Handling"]
    ZERO["Register x31Always returns 0Cannot be modified(AArch64 Zero Register)"]
end
subgraph subGraph1["Access Methods"]
    GET["gpr(index) -> usizeset_gpr(index, val)exception_pc() -> usizeset_exception_pc(pc)set_argument(arg)"]
end
subgraph subGraph0["Aarch64ContextFrame Memory Layout"]
    GPR["gpr[31]: u64General Purpose Registersx0 through x30"]
    SP["sp_el0: u64EL0 Stack Pointer"]
    ELR["elr: u64Exception Link Register(Return Address)"]
    SPSR["spsr: u64Saved Program Status Register(CPU Flags & State)"]
end

ELR --> GET
GPR --> GET
GPR --> ZERO
SP --> GET
SPSR --> GET
```

The structure uses `#[repr(C)]` layout to ensure predictable memory organization for hardware and assembly code interaction. The default state configures SPSR to mask all exceptions and set EL1h mode for safe guest initialization.

**Sources:** [src/context_frame.rs(L17 - L63)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L17-L63)

## System Registers Structure

The `GuestSystemRegisters` structure manages the complete system-level state of a guest VM, including control registers, memory management settings, timer state, and hypervisor configuration.

```mermaid
flowchart TD
subgraph Operations["Operations"]
    STORE["store() - Save hardware to structUses MRS instructions"]
    RESTORE["restore() - Load struct to hardwareUses MSR instructions"]
    RESET["reset() - Zero all fields"]
end
subgraph subGraph0["GuestSystemRegisters Categories"]
    TIMER["Timer Registerscntvoff_el2, cntp_cval_el0cntv_cval_el0, cntkctl_el1cntvct_el0, cntp_ctl_el0cntv_ctl_el0, cntp_tval_el0cntv_tval_el0"]
    PROC["Processor IDvpidr_el2vmpidr_el2"]
    EL1["EL1/EL0 System Statesp_el0, sp_el1, elr_el1spsr_el1, sctlr_el1actlr_el1, cpacr_el1"]
    MMU["Memory Managementttbr0_el1, ttbr1_el1tcr_el1, mair_el1amair_el1, par_el1"]
    EXC["Exception Handlingesr_el1, far_el1vbar_el1, far_el2hpfar_el2"]
    THREAD["Thread Pointerstpidr_el0, tpidr_el1tpidrro_el0contextidr_el1"]
    HYP["Hypervisor Contexthcr_el2, vttbr_el2cptr_el2, hstr_el2pmcr_el0, vtcr_el2"]
end

EL1 --> STORE
EXC --> STORE
HYP --> STORE
MMU --> STORE
PROC --> STORE
RESTORE --> RESET
STORE --> RESTORE
THREAD --> STORE
TIMER --> STORE
```

The structure is aligned to 16 bytes (`#[repr(align(16))]`) for optimal memory access performance during context switches.

**Sources:** [src/context_frame.rs(L138 - L300)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L138-L300)

## Register State Preservation

The system employs a two-tier approach to state preservation, with different registers handled by different mechanisms based on their access patterns and importance.

### TrapFrame Register Operations

```mermaid
flowchart TD
subgraph subGraph2["Default State"]
    DEF["SPSR ConfigurationM: EL1h modeI,F,A,D: All maskedELR: 0, SP_EL0: 0"]
end
subgraph subGraph1["Special Operations"]
    PC["exception_pc()set_exception_pc(pc)Maps to ELR field"]
    ARG["set_argument(arg)Sets x0 registerFor return values"]
end
subgraph subGraph0["GPR Access Pattern"]
    INDEX["Register Index0-30: Normal GPRs31: Zero Register"]
    GET["gpr(index)Returns register valueIndex 31 -> 0"]
    SET["set_gpr(index, val)Sets register valueIndex 31 -> Warning"]
end

ARG --> SET
INDEX --> GET
INDEX --> SET
PC --> GET
SET --> DEF
```

**Sources:** [src/context_frame.rs(L65 - L136)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L65-L136)

### System Register Save/Restore Cycle

```mermaid
sequenceDiagram
    participant GuestVM as "Guest VM"
    participant HardwareRegisters as "Hardware Registers"
    participant GuestSystemRegisters as "GuestSystemRegisters"
    participant HostHypervisor as "Host/Hypervisor"

    Note over GuestVM,HostHypervisor: VM Exit Sequence
    GuestVM ->> HardwareRegisters: Exception/Trap occurs
    HardwareRegisters ->> GuestSystemRegisters: store() - MRS instructions
    Note over GuestSystemRegisters: All system state captured
    GuestSystemRegisters ->> HostHypervisor: Context available for inspection
    Note over GuestVM,HostHypervisor: VM Entry Sequence
    HostHypervisor ->> GuestSystemRegisters: Configure guest state
    GuestSystemRegisters ->> HardwareRegisters: restore() - MSR instructions
    Note over HardwareRegisters: Guest context loaded
    HardwareRegisters ->> GuestVM: ERET to guest execution
```

The `store()` and `restore()` methods use inline assembly with MRS (Move Register from System) and MSR (Move System Register) instructions to efficiently transfer register state between hardware and memory.

**Sources:** [src/context_frame.rs(L213 - L299)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L213-L299)

## Integration with Context Switching

These structures integrate closely with the assembly-level context switching mechanism to provide complete state preservation during VM transitions.

```mermaid
flowchart TD
subgraph subGraph2["State Structures"]
    TF["Aarch64ContextFrameGPRs, SP_EL0, ELR, SPSR"]
    GSR["GuestSystemRegistersSystem control state"]
end
subgraph subGraph1["VM Entry Flow"]
    SETUP["Host configuresguest state"]
    SYS_RESTORE["guest_system_regs.restore()Struct to hardware"]
    ASM_RESTORE["context_vm_entryAssembly restores from TrapFrame"]
    GUEST["Guest executionresumes"]
end
subgraph subGraph0["VM Exit Flow"]
    TRAP["Exception OccursHardware trap"]
    ASM_SAVE["SAVE_REGS_FROM_EL1Assembly saves to TrapFrame"]
    SYS_SAVE["guest_system_regs.store()System registers to struct"]
    HANDLER["Exception handlerProcesses exit reason"]
end

ASM_RESTORE --> GUEST
ASM_RESTORE --> TF
ASM_SAVE --> SYS_SAVE
ASM_SAVE --> TF
SETUP --> SYS_RESTORE
SYS_RESTORE --> ASM_RESTORE
SYS_RESTORE --> GSR
SYS_SAVE --> GSR
SYS_SAVE --> HANDLER
TRAP --> ASM_SAVE
```

The TrapFrame is manipulated directly by assembly code during the low-level context switch, while the GuestSystemRegisters structure is managed by Rust code using the `store()` and `restore()` methods.

**Sources:** [src/context_frame.rs(L1 - L301)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L1-L301)

## Register Categories and Usage

|Category|Registers|Purpose|Access Pattern|
| --- | --- | --- | --- |
|General Purpose|x0-x30, SP_EL0|Function arguments, local variables, stack management|Every VM exit/entry via assembly|
|Execution State|ELR, SPSR|Program counter and processor status|Every VM exit/entry via assembly|
|Memory Management|TTBR0_EL1, TTBR1_EL1, TCR_EL1, MAIR_EL1|Page table configuration, memory attributes|Restored on VM entry|
|System Control|SCTLR_EL1, CPACR_EL1, ACTLR_EL1|Processor features, access control|Restored on VM entry|
|Exception Handling|VBAR_EL1, ESR_EL1, FAR_EL1|Vector table, exception information|Preserved across exits|
|Timer Management|CNTVOFF_EL2, CNTKCTL_EL1, timer control/value registers|Virtual timer state|Managed by hypervisor|
|Hypervisor Control|HCR_EL2, VTTBR_EL2, VTCR_EL2|Virtualization configuration, stage-2 MMU|Set during VCPU setup|

The register categories reflect the layered nature of AArch64 virtualization, with some registers requiring preservation on every context switch while others are configured once during VCPU initialization.

**Sources:** [src/context_frame.rs(L148 - L197)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L148-L197)
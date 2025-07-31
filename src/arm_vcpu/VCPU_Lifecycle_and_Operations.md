# VCPU Lifecycle and Operations

> **Relevant source files**
> * [src/context_frame.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs)
> * [src/vcpu.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs)

This document details the virtual CPU (VCPU) creation, configuration, execution cycle, and VM exit handling mechanisms within the arm_vcpu hypervisor system. It covers the core `Aarch64VCpu` implementation, context management, and the complete lifecycle from VCPU instantiation to guest execution and exit handling.

For hardware abstraction and platform integration details, see [Hardware Abstraction and Platform Support](/arceos-hypervisor/arm_vcpu/5.2-hardware-abstraction-and-platform-support). For low-level exception vector implementation, see [Assembly Exception Vectors](/arceos-hypervisor/arm_vcpu/4.1-assembly-exception-vectors). For per-CPU state management across multiple VCPUs, see [Per-CPU State Management](/arceos-hypervisor/arm_vcpu/2.2-per-cpu-state-management).

## VCPU Structure and Components

The `Aarch64VCpu` struct serves as the primary virtual CPU implementation, containing all necessary state for guest execution and host-guest context switching.

### Core VCPU Structure

```mermaid
classDiagram
class Aarch64VCpu {
    +TrapFrame ctx
    +u64 host_stack_top
    +GuestSystemRegisters guest_system_regs
    +u64 mpidr
    +PhantomData~H~ _phantom
    +new(config) AxResult~Self~
    +setup(config) AxResult
    +run() AxResult~AxVCpuExitReason~
    +set_entry(entry) AxResult
    +set_ept_root(ept_root) AxResult
}

class TrapFrame {
    +u64[31] gpr
    +u64 sp_el0
    +u64 elr
    +u64 spsr
    +set_exception_pc(pc)
    +set_argument(arg)
    +set_gpr(idx, val)
    +gpr(idx) usize
}

class GuestSystemRegisters {
    +u64 cntvoff_el2
    +u32 cntkctl_el1
    +u64 sp_el0
    +u32 sctlr_el1
    +u64 hcr_el2
    +u64 vttbr_el2
    +u64 vmpidr_el2
    +store()
    +restore()
}

class Aarch64VCpuCreateConfig {
    +u64 mpidr_el1
    +usize dtb_addr
    
}

Aarch64VCpu  *--  TrapFrame : "contains"
Aarch64VCpu  *--  GuestSystemRegisters : "contains"
Aarch64VCpu  ..>  Aarch64VCpuCreateConfig : "uses for creation"
```

The VCPU structure maintains strict field ordering requirements for assembly code interaction. The `ctx` and `host_stack_top` fields must remain in their current positions and order to support the low-level context switching assembly routines.

**Sources:** [src/vcpu.rs(L40 - L51)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L40-L51) [src/context_frame.rs(L17 - L28)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L17-L28) [src/context_frame.rs(L145 - L197)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L145-L197)

## VCPU Creation and Configuration

### Creation Process

The VCPU creation follows a two-phase initialization pattern through the `AxArchVCpu` trait implementation:

```mermaid
sequenceDiagram
    participant Client as Client
    participant Aarch64VCpu as Aarch64VCpu
    participant TrapFrame as TrapFrame
    participant GuestSystemRegisters as GuestSystemRegisters

    Client ->> Aarch64VCpu: new(Aarch64VCpuCreateConfig)
    Aarch64VCpu ->> TrapFrame: TrapFrame::default()
    Aarch64VCpu ->> TrapFrame: set_argument(config.dtb_addr)
    Aarch64VCpu ->> GuestSystemRegisters: GuestSystemRegisters::default()
    Aarch64VCpu -->> Client: Aarch64VCpu instance
    Client ->> Aarch64VCpu: setup(())
    Aarch64VCpu ->> Aarch64VCpu: init_hv()
    Aarch64VCpu ->> Aarch64VCpu: init_vm_context()
    Aarch64VCpu -->> Client: AxResult
    Client ->> Aarch64VCpu: set_entry(guest_entry_point)
    Aarch64VCpu ->> TrapFrame: set_exception_pc(entry)
    Client ->> Aarch64VCpu: set_ept_root(page_table_root)
    Aarch64VCpu ->> GuestSystemRegisters: vttbr_el2 = ept_root
```

The `new()` method creates a VCPU instance with minimal initialization, setting up the trap context with the device tree address and initializing default system registers. The `setup()` method performs hypervisor-specific initialization including guest execution state configuration.

**Sources:** [src/vcpu.rs(L69 - L80)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L69-L80) [src/vcpu.rs(L82 - L85)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L82-L85) [src/vcpu.rs(L87 - L97)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L87-L97)

### Hypervisor Context Initialization

The `init_hv()` and `init_vm_context()` methods configure the guest execution environment:

|Register/Field|Value|Purpose|
| --- | --- | --- |
|SPSR_EL1|EL1h + All exceptions masked|Guest starts in EL1 with interrupts disabled|
|CNTHCTL_EL2|EL1PCEN + EL1PCTEN|Enable timer access from EL1|
|SCTLR_EL1|0x30C50830|System control register defaults|
|VTCR_EL2|40-bit PA, 4KB granule, stage-2 config|Stage-2 translation control|
|HCR_EL2|VM + RW + TSC|Enable virtualization, 64-bit EL1, trap SMC|
|VMPIDR_EL2|Configured MPIDR|Virtual multiprocessor ID|

**Sources:** [src/vcpu.rs(L128 - L136)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L128-L136) [src/vcpu.rs(L139 - L166)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L139-L166)

## VCPU Execution Cycle

### VM Entry and Exit Flow

The VCPU execution follows a carefully orchestrated context switching process:

```mermaid
flowchart TD
start["vcpu.run()"]
save_host["save_host_sp_el0()"]
restore_vm["restore_vm_system_regs()"]
run_guest["run_guest()"]
save_regs["save_regs_to_stack!()"]
save_stack["Save host stack to host_stack_top"]
vm_entry["context_vm_entry"]
guest_exec["Guest Execution"]
exception["Exception/Interrupt"]
vmexit["VM Exit"]
save_guest["SAVE_REGS_FROM_EL1"]
handler["vmexit_handler()"]
store_regs["guest_system_regs.store()"]
restore_host_sp["restore_host_sp_el0()"]
dispatch["Exception dispatch"]
sync["handle_exception_sync()"]
irq["ExternalInterrupt"]
exit_reason["AxVCpuExitReason"]

dispatch --> irq
dispatch --> sync
exception --> vmexit
exit_reason --> start
guest_exec --> exception
handler --> store_regs
irq --> exit_reason
restore_host_sp --> dispatch
restore_vm --> run_guest
run_guest --> save_regs
save_guest --> handler
save_host --> restore_vm
save_regs --> save_stack
save_stack --> vm_entry
start --> save_host
store_regs --> restore_host_sp
sync --> exit_reason
vm_entry --> guest_exec
vmexit --> save_guest
```

The execution cycle begins with the `run()` method, which saves host state, restores guest system registers, and transitions to guest execution through the `run_guest()` naked function.

**Sources:** [src/vcpu.rs(L99 - L111)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L99-L111) [src/vcpu.rs(L186 - L214)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L186-L214) [src/vcpu.rs(L255 - L282)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L255-L282)

### Context Switching Implementation

The `run_guest()` function implements a naked assembly function that performs the critical host-to-guest transition:

```mermaid
flowchart TD
subgraph subGraph2["Guest Context"]
    guest_regs["Guest GPRs"]
    guest_sys_regs["Guest System Registers"]
    guest_exec["Guest Execution"]
end
subgraph Transition["Transition"]
    save_macro["save_regs_to_stack!()"]
    save_sp["Save stack pointer to host_stack_top"]
    vm_entry["Branch to context_vm_entry"]
end
subgraph subGraph0["Host Context"]
    host_regs["Host GPRs (x19-x30)"]
    host_sp["Host SP_EL0"]
    host_stack["Host Stack"]
end

guest_regs --> guest_exec
guest_sys_regs --> guest_exec
host_regs --> save_macro
host_stack --> save_sp
save_macro --> vm_entry
save_sp --> vm_entry
vm_entry --> guest_regs
vm_entry --> guest_sys_regs
```

The naked function ensures no compiler-generated prologue/epilogue interferes with the precise register state management required for hypervisor operation.

**Sources:** [src/vcpu.rs(L186 - L214)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L186-L214) [src/vcpu.rs(L225 - L244)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L225-L244)

## VM Exit Handling

### Exit Reason Processing

The `vmexit_handler()` method processes VM exits and converts low-level trap information into structured exit reasons:

```mermaid
flowchart TD
vmexit["vmexit_handler(TrapKind)"]
store["guest_system_regs.store()"]
save_sp["Save guest SP_EL0 to ctx.sp_el0"]
restore_host["restore_host_sp_el0()"]
match_trap["Match TrapKind"]
sync["TrapKind::Synchronous"]
irq["TrapKind::Irq"]
other["Other TrapKind"]
handle_sync["handle_exception_sync(&mut ctx)"]
fetch_irq["H::irq_fetch()"]
panic["panic!(Unhandled exception)"]
exit_reason["AxVCpuExitReason"]
ext_int["AxVCpuExitReason::ExternalInterrupt"]
return_result["Return to hypervisor"]

exit_reason --> return_result
ext_int --> return_result
fetch_irq --> ext_int
handle_sync --> exit_reason
irq --> fetch_irq
match_trap --> irq
match_trap --> other
match_trap --> sync
other --> panic
restore_host --> match_trap
save_sp --> restore_host
store --> save_sp
sync --> handle_sync
vmexit --> store
```

The exit handler ensures proper state preservation by storing guest system registers and restoring host SP_EL0 before processing the specific exit cause.

**Sources:** [src/vcpu.rs(L255 - L282)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L255-L282)

### Exit Reason Types

The system generates specific exit reasons that inform the hypervisor about the cause of the VM exit:

|Exit Reason|Trigger|Handler Location|
| --- | --- | --- |
|MmioRead/Write|Data abort to device memory|handle_data_abort|
|SystemRegisterRead/Write|Trapped system register access|handle_system_register|
|Hypercall|HVC instruction|handle_psci_call|
|CpuUp/Down/SystemDown|PSCI power management|handle_psci_call|
|ExternalInterrupt|Physical interrupt|vmexit_handler|

**Sources:** [src/vcpu.rs(L276 - L281)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L276-L281)

## Context Management

### Register State Preservation

The VCPU maintains two primary context structures for complete state preservation:

```mermaid
flowchart TD
subgraph subGraph2["Hardware Registers"]
    hw_regs["Actual AArch64 System Registers"]
end
subgraph subGraph1["GuestSystemRegisters (System State)"]
    el1_state["EL1 State (sctlr_el1, vbar_el1, etc)"]
    subgraph subGraph0["TrapFrame (GPR State)"]
        timer["Timer Registers (cntvoff_el2, cntkctl_el1)"]
        vm_id["VM Identity (vmpidr_el2, vpidr_el2)"]
        memory["Memory Management (ttbr0/1_el1, tcr_el1)"]
        hyp_ctrl["Hypervisor Control (hcr_el2, vttbr_el2)"]
        gpr["gpr[31] - General Purpose Registers"]
        sp_el0["sp_el0 - EL0 Stack Pointer"]
        elr["elr - Exception Link Register"]
        spsr["spsr - Saved Program Status"]
    end
end
TrapFrame["TrapFrame"]
GuestSystemRegisters["GuestSystemRegisters"]

GuestSystemRegisters --> hw_regs
TrapFrame --> hw_regs
```

The `TrapFrame` captures the basic CPU execution state, while `GuestSystemRegisters` preserves the complete virtual machine system configuration.

**Sources:** [src/context_frame.rs(L17 - L136)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L17-L136) [src/context_frame.rs(L213 - L300)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/context_frame.rs#L213-L300)

### Host State Management

Host state preservation uses a combination of stack storage and per-CPU variables:

```mermaid
sequenceDiagram
    participant Host as Host
    participant PerCPU as PerCPU
    participant Guest as Guest
    participant stack as stack

    Note over Host,Guest: VM Entry Sequence
    Host ->> stack: save_regs_to_stack!() - GPRs x19-x30
    Host ->> PerCPU: save_host_sp_el0() - HOST_SP_EL0
    Host ->> Guest: Context switch to guest
    Note over Host,Guest: VM Exit Sequence
    Guest ->> PerCPU: Store guest SP_EL0 to ctx.sp_el0
    Guest ->> PerCPU: restore_host_sp_el0() - Restore HOST_SP_EL0
    Guest ->> stack: restore_regs_from_stack!() - Restore GPRs
    Guest ->> Host: Return to host execution
```

The per-CPU `HOST_SP_EL0` variable ensures each CPU core maintains independent host state, supporting multi-core hypervisor operation.

**Sources:** [src/vcpu.rs(L15 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L15-L26) [src/vcpu.rs(L262 - L273)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L262-L273)
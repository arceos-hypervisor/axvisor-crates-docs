# Virtual CPU Management

> **Relevant source files**
> * [src/vcpu.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs)

This document covers the core virtual CPU implementation in the arm_vcpu hypervisor system. It focuses on the `Aarch64VCpu` struct, its lifecycle management, configuration mechanisms, and the fundamental execution model for running guest virtual machines on AArch64 hardware.

For detailed information about VCPU lifecycle operations and VM exit handling, see [VCPU Lifecycle and Operations](/arceos-hypervisor/arm_vcpu/2.1-vcpu-lifecycle-and-operations). For per-CPU hardware state management and virtualization control, see [Per-CPU State Management](/arceos-hypervisor/arm_vcpu/2.2-per-cpu-state-management). For low-level context switching mechanics, see [Context Switching and State Management](/arceos-hypervisor/arm_vcpu/3-context-switching-and-state-management).

## Core VCPU Implementation

The `Aarch64VCpu<H: AxVCpuHal>` struct is the central component implementing virtual CPU functionality. It maintains both guest execution context and system register state, providing the foundation for running guest operating systems under hypervisor control.

### VCPU Structure and Components

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
    +set_entry(entry) AxResult
    +set_ept_root(ept_root) AxResult
    +run() AxResult~AxVCpuExitReason~
    +bind() AxResult
    +unbind() AxResult
    +set_gpr(idx, val)
}

class TrapFrame {
    +guest execution context
    +GPRs, SP_EL0, ELR, SPSR
    
}

class GuestSystemRegisters {
    +system control registers
    +timer registers
    +memory management registers
    
}

class VmCpuRegisters {
    +TrapFrame trap_context_regs
    +GuestSystemRegisters vm_system_regs
    
}

class Aarch64VCpuCreateConfig {
    +u64 mpidr_el1
    +usize dtb_addr
    
}

Aarch64VCpu  *--  TrapFrame : "contains"
Aarch64VCpu  *--  GuestSystemRegisters : "contains"
VmCpuRegisters  *--  TrapFrame : "aggregates"
VmCpuRegisters  *--  GuestSystemRegisters : "aggregates"
Aarch64VCpu  ..>  Aarch64VCpuCreateConfig : "uses for creation"
```

The VCPU structure maintains critical ordering constraints where `ctx` and `host_stack_top` must remain at the beginning of the struct to support assembly code expectations for context switching operations.

**Sources:** [src/vcpu.rs(L40 - L51)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L40-L51) [src/vcpu.rs(L30 - L37)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L30-L37) [src/vcpu.rs(L54 - L62)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L54-L62)

### Hardware Abstraction Integration

```

```

The generic `H: AxVCpuHal` parameter allows the VCPU implementation to work with different hardware abstraction layers while maintaining platform independence.

**Sources:** [src/vcpu.rs(L8)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L8-L8) [src/vcpu.rs(L64 - L65)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L64-L65) [src/vcpu.rs(L278)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L278-L278)

## VCPU Configuration and Initialization

### Creation and Setup Process

The VCPU follows a two-phase initialization pattern: creation with hardware-specific configuration, followed by hypervisor setup.

|Phase|Function|Purpose|Configuration|
| --- | --- | --- | --- |
|Creation|new(config)|Basic structure allocation|Aarch64VCpuCreateConfig|
|Setup|setup(config)|Hypervisor initialization|()(empty)|
|Entry Configuration|set_entry(entry)|Guest entry point|GuestPhysAddr|
|Memory Setup|set_ept_root(ept_root)|Extended page table root|HostPhysAddr|

**Sources:** [src/vcpu.rs(L69 - L84)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L69-L84) [src/vcpu.rs(L87 - L96)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L87-L96)

### System Register Initialization

```mermaid
flowchart TD
init_hv["init_hv()"]
init_spsr["Configure SPSR_EL1EL1h mode, masks set"]
init_vm_context["init_vm_context()"]
timer_config["Configure TimerCNTHCTL_EL2, CNTVOFF_EL2"]
sctlr_config["Configure SCTLR_EL10x30C50830"]
vtcr_config["Configure VTCR_EL240-bit PA, 4KB granules"]
hcr_config["Configure HCR_EL2VM enable, AArch64, SMC trap"]
vmpidr_config["Configure VMPIDR_EL2Based on mpidr parameter"]
ready["VCPU Ready for Execution"]

hcr_config --> ready
init_hv --> init_spsr
init_hv --> init_vm_context
init_vm_context --> hcr_config
init_vm_context --> sctlr_config
init_vm_context --> timer_config
init_vm_context --> vmpidr_config
init_vm_context --> vtcr_config
sctlr_config --> ready
timer_config --> ready
vmpidr_config --> ready
vtcr_config --> ready
```

The initialization process configures essential system registers to establish the proper hypervisor and guest execution environment, including memory management, exception handling, and CPU identification.

**Sources:** [src/vcpu.rs(L128 - L166)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L128-L166)

## Execution Model and Context Management

### Host-Guest Context Switching

```mermaid
sequenceDiagram
    participant HostContext as "Host Context"
    participant Aarch64VCpurun as "Aarch64VCpu::run()"
    participant GuestExecution as "Guest Execution"
    participant ExceptionHandler as "Exception Handler"

    HostContext ->> Aarch64VCpurun: run()
    Aarch64VCpurun ->> Aarch64VCpurun: save_host_sp_el0()
    Aarch64VCpurun ->> Aarch64VCpurun: restore_vm_system_regs()
    Aarch64VCpurun ->> Aarch64VCpurun: run_guest() [naked asm]
    Note over Aarch64VCpurun: save_regs_to_stack!()
    Note over Aarch64VCpurun: context_vm_entry
    Aarch64VCpurun ->> GuestExecution: Enter guest execution
    GuestExecution ->> ExceptionHandler: VM Exit (trap/interrupt)
    ExceptionHandler ->> Aarch64VCpurun: return exit_reason
    Aarch64VCpurun ->> Aarch64VCpurun: vmexit_handler(trap_kind)
    Aarch64VCpurun ->> Aarch64VCpurun: guest_system_regs.store()
    Aarch64VCpurun ->> Aarch64VCpurun: restore_host_sp_el0()
    Aarch64VCpurun ->> HostContext: return AxVCpuExitReason
```

The execution model implements a clean separation between host and guest contexts, with careful register state management and stack handling to ensure proper isolation.

**Sources:** [src/vcpu.rs(L99 - L111)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L99-L111) [src/vcpu.rs(L182 - L214)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L182-L214) [src/vcpu.rs(L255 - L282)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L255-L282)

### Per-CPU State Management

```mermaid
flowchart TD
subgraph subGraph2["VCPU Execution"]
    run_start["run() entry"]
    run_exit["vmexit_handler()"]
end
subgraph subGraph1["Context Operations"]
    save_host["save_host_sp_el0()"]
    restore_host["restore_host_sp_el0()"]
    SP_EL0_reg["SP_EL0 register"]
end
subgraph subGraph0["Per-CPU Storage"]
    HOST_SP_EL0["HOST_SP_EL0percpu static"]
end

HOST_SP_EL0 --> restore_host
restore_host --> SP_EL0_reg
run_exit --> restore_host
run_start --> save_host
save_host --> HOST_SP_EL0
save_host --> SP_EL0_reg
```

The per-CPU state management ensures that host context is properly preserved across guest execution cycles, using percpu storage to maintain thread-local state.

**Sources:** [src/vcpu.rs(L15 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L15-L26) [src/vcpu.rs(L102 - L104)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L102-L104) [src/vcpu.rs(L270 - L272)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L270-L272)

## VM Exit Processing

### Exit Reason Classification

The VCPU handles different types of VM exits through a structured dispatch mechanism:

```mermaid
flowchart TD
vmexit_handler["vmexit_handler(exit_reason)"]
store_guest["guest_system_regs.store()"]
restore_host_sp["restore_host_sp_el0()"]
dispatch["Match exit_reason"]
sync_handler["handle_exception_sync(&ctx)"]
irq_handler["AxVCpuExitReason::ExternalInterrupt"]
panic_handler["panic!(Unhandled exception)"]
return_reason["Return AxVCpuExitReason"]
irq_fetch["H::irq_fetch() for vector"]
system_halt["System Halt"]

dispatch --> irq_handler
dispatch --> panic_handler
dispatch --> sync_handler
irq_fetch --> return_reason
irq_handler --> irq_fetch
panic_handler --> system_halt
restore_host_sp --> dispatch
store_guest --> restore_host_sp
sync_handler --> return_reason
vmexit_handler --> store_guest
```

The exit processing maintains proper state transitions and delegates complex synchronous exceptions to specialized handlers while handling interrupts directly.

**Sources:** [src/vcpu.rs(L255 - L282)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L255-L282)

### Register State Preservation

During VM exits, the system carefully manages register state to maintain proper isolation:

|Register Set|Storage Location|Timing|
| --- | --- | --- |
|Guest GPRs|TrapFrame.ctx|During exception entry (assembly)|
|Guest System Regs|GuestSystemRegisters|Invmexit_handler()|
|Host SP_EL0|HOST_SP_EL0percpu|Before guest execution|
|Guest SP_EL0|ctx.sp_el0|Fromguest_system_regs.sp_el0|

**Sources:** [src/vcpu.rs(L262 - L272)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/vcpu.rs#L262-L272)
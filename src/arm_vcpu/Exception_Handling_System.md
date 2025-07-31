# Exception Handling System

> **Relevant source files**
> * [src/exception.S](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.S)
> * [src/exception.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs)
> * [src/exception_utils.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs)

This document covers the multi-layered exception handling architecture that manages VM exits, traps, and interrupts when a guest VM encounters exceptional conditions. The system provides a complete pipeline from low-level assembly exception vectors through register analysis utilities to high-level exception dispatch and handling.

For information about the VCPU lifecycle and context switching mechanisms, see [Virtual CPU Management](/arceos-hypervisor/arm_vcpu/2-virtual-cpu-management). For details about the assembly exception vectors specifically, see [Assembly Exception Vectors](/arceos-hypervisor/arm_vcpu/4.1-assembly-exception-vectors).

## Architecture Overview

The exception handling system operates across three distinct layers, each responsible for different aspects of exception processing:

**Exception Handling System Architecture**

```mermaid
flowchart TD
subgraph subGraph4["VCPU Integration"]
    TRAP_FRAME["TrapFrame"]
    EXIT_REASON["AxVCpuExitReason"]
    VCPU_RUN["Aarch64VCpu::run()"]
end
subgraph subGraph3["Handler Layer (exception.rs)"]
    HANDLE_EXCEPTION_SYNC["handle_exception_sync()"]
    HANDLE_DATA_ABORT["handle_data_abort()"]
    HANDLE_SYSTEM_REGISTER["handle_system_register()"]
    HANDLE_PSCI_CALL["handle_psci_call()"]
    HANDLE_SMC64["handle_smc64_exception()"]
end
subgraph subGraph2["Utility Layer (exception_utils.rs)"]
    EXCEPTION_ESR["exception_esr()"]
    EXCEPTION_CLASS["exception_class()"]
    EXCEPTION_FAULT_ADDR["exception_fault_addr()"]
    DATA_ABORT_UTILS["Data Abort Analysis Functions"]
    SYSREG_UTILS["System Register Analysis Functions"]
end
subgraph subGraph1["Assembly Layer (exception.S)"]
    VECTOR_TABLE["exception_vector_base_vcpu"]
    SAVE_REGS["SAVE_REGS_FROM_EL1"]
    RESTORE_REGS["RESTORE_REGS_INTO_EL1"]
    VMEXIT_TRAMP["vmexit_trampoline"]
    CONTEXT_VM_ENTRY["context_vm_entry"]
end
subgraph subGraph0["Guest VM (EL1/EL0)"]
    GUEST_CODE["Guest Code Execution"]
    GUEST_EXC["Exception/Trap Triggered"]
end

CONTEXT_VM_ENTRY --> RESTORE_REGS
EXIT_REASON --> VCPU_RUN
GUEST_CODE --> GUEST_EXC
GUEST_EXC --> VECTOR_TABLE
HANDLE_DATA_ABORT --> DATA_ABORT_UTILS
HANDLE_DATA_ABORT --> EXCEPTION_FAULT_ADDR
HANDLE_DATA_ABORT --> EXIT_REASON
HANDLE_EXCEPTION_SYNC --> EXCEPTION_CLASS
HANDLE_EXCEPTION_SYNC --> EXCEPTION_ESR
HANDLE_EXCEPTION_SYNC --> HANDLE_DATA_ABORT
HANDLE_EXCEPTION_SYNC --> HANDLE_PSCI_CALL
HANDLE_EXCEPTION_SYNC --> HANDLE_SMC64
HANDLE_EXCEPTION_SYNC --> HANDLE_SYSTEM_REGISTER
HANDLE_PSCI_CALL --> EXIT_REASON
HANDLE_SMC64 --> EXIT_REASON
HANDLE_SYSTEM_REGISTER --> EXIT_REASON
HANDLE_SYSTEM_REGISTER --> SYSREG_UTILS
RESTORE_REGS --> GUEST_CODE
RESTORE_REGS --> TRAP_FRAME
SAVE_REGS --> TRAP_FRAME
SAVE_REGS --> VMEXIT_TRAMP
VCPU_RUN --> CONTEXT_VM_ENTRY
VECTOR_TABLE --> SAVE_REGS
VMEXIT_TRAMP --> HANDLE_EXCEPTION_SYNC
```

Sources: [src/exception.S(L106 - L141)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.S#L106-L141) [src/exception.rs(L72 - L126)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L72-L126) [src/exception_utils.rs(L1 - L312)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L1-L312)

## Exception Processing Pipeline

The system processes exceptions through a structured pipeline that preserves guest state while analyzing and dispatching exceptions to appropriate handlers:

**Exception Processing Flow**

```mermaid
flowchart TD
subgraph subGraph4["Exit Reason Generation"]
    MMIO_READ["AxVCpuExitReason::MmioRead"]
    MMIO_WRITE["AxVCpuExitReason::MmioWrite"]
    HYPERCALL["AxVCpuExitReason::Hypercall"]
    CPU_UP["AxVCpuExitReason::CpuUp"]
    CPU_DOWN["AxVCpuExitReason::CpuDown"]
    SYSTEM_DOWN["AxVCpuExitReason::SystemDown"]
    SYSREG_READ["AxVCpuExitReason::SysRegRead"]
    SYSREG_WRITE["AxVCpuExitReason::SysRegWrite"]
    NOTHING["AxVCpuExitReason::Nothing"]
end
subgraph subGraph3["Specific Handlers"]
    DATA_ABORT["DataAbortLowerEL → handle_data_abort()"]
    HVC_CALL["HVC64 → handle_psci_call() or Hypercall"]
    SYS_REG["TrappedMsrMrs → handle_system_register()"]
    SMC_CALL["SMC64 → handle_smc64_exception()"]
    UNHANDLED["Unhandled → panic!"]
end
subgraph subGraph2["Exception Classification"]
    GET_ESR["exception_esr()"]
    GET_CLASS["exception_class()"]
    CLASS_MATCH["Match ESR_EL2::EC"]
end
subgraph subGraph1["Context Save & Dispatch"]
    SAVE_CONTEXT["SAVE_REGS_FROM_EL1 macro"]
    VMEXIT_FUNC["vmexit_trampoline()"]
    SYNC_HANDLER["handle_exception_sync()"]
end
subgraph subGraph0["Assembly Vector Dispatch"]
    VECTOR_BASE["exception_vector_base_vcpu"]
    CURRENT_EL_SYNC["current_el_sync_handler"]
    CURRENT_EL_IRQ["current_el_irq_handler"]
    LOWER_EL_SYNC["HANDLE_LOWER_SYNC_VCPU"]
    LOWER_EL_IRQ["HANDLE_LOWER_IRQ_VCPU"]
    INVALID_EXC["invalid_exception_el2"]
end
GUEST_TRAP["Guest VM Exception/Trap"]

CLASS_MATCH --> DATA_ABORT
CLASS_MATCH --> HVC_CALL
CLASS_MATCH --> SMC_CALL
CLASS_MATCH --> SYS_REG
CLASS_MATCH --> UNHANDLED
DATA_ABORT --> MMIO_READ
DATA_ABORT --> MMIO_WRITE
GET_CLASS --> CLASS_MATCH
GUEST_TRAP --> VECTOR_BASE
HVC_CALL --> CPU_DOWN
HVC_CALL --> CPU_UP
HVC_CALL --> HYPERCALL
HVC_CALL --> SYSTEM_DOWN
LOWER_EL_IRQ --> SAVE_CONTEXT
LOWER_EL_SYNC --> SAVE_CONTEXT
SAVE_CONTEXT --> VMEXIT_FUNC
SMC_CALL --> NOTHING
SYNC_HANDLER --> GET_CLASS
SYNC_HANDLER --> GET_ESR
SYS_REG --> SYSREG_READ
SYS_REG --> SYSREG_WRITE
VECTOR_BASE --> CURRENT_EL_IRQ
VECTOR_BASE --> CURRENT_EL_SYNC
VECTOR_BASE --> INVALID_EXC
VECTOR_BASE --> LOWER_EL_IRQ
VECTOR_BASE --> LOWER_EL_SYNC
VMEXIT_FUNC --> SYNC_HANDLER
```

Sources: [src/exception.S(L108 - L141)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.S#L108-L141) [src/exception.rs(L72 - L126)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L72-L126) [src/exception.rs(L276 - L290)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L276-L290)

## Exception Types and Dispatch Logic

The `handle_exception_sync` function serves as the primary dispatcher, using the Exception Class (EC) field from `ESR_EL2` to route exceptions to specialized handlers:

|Exception Class|ESR_EL2::EC Value|Handler Function|Exit Reason Generated|
| --- | --- | --- | --- |
|Data Abort Lower EL|DataAbortLowerEL|handle_data_abort()|MmioRead,MmioWrite|
|Hypervisor Call|HVC64|handle_psci_call()or Hypercall|CpuUp,CpuDown,SystemDown,Hypercall|
|System Register Access|TrappedMsrMrs|handle_system_register()|SysRegRead,SysRegWrite|
|Secure Monitor Call|SMC64|handle_smc64_exception()|Nothing(forwarded to ATF)|

Sources: [src/exception.rs(L72 - L126)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L72-L126)

### Data Abort Handling

Data aborts occur when the guest VM accesses memory that triggers a fault. The `handle_data_abort` function processes these by:

1. **Fault Analysis**: Using `exception_fault_addr()` to determine the guest physical address
2. **Access Classification**: Determining read/write direction and access width using utility functions
3. **Register Identification**: Finding which guest register was involved in the access
4. **MMIO Translation**: Converting the fault into an appropriate `AxVCpuExitReason`

The function performs several safety checks using `exception_data_abort_handleable()` and `exception_data_abort_is_translate_fault()` before proceeding with MMIO emulation.

Sources: [src/exception.rs(L128 - L182)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L128-L182) [src/exception_utils.rs(L132 - L142)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L132-L142) [src/exception_utils.rs(L202 - L254)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L202-L254)

### System Register Access Handling

System register accesses are trapped when the guest VM attempts to read or write control registers. The `handle_system_register` function:

1. **ISS Parsing**: Extracts the Instruction Specific Syndrome from `ESR_EL2`
2. **Register Identification**: Uses `exception_sysreg_addr()` to identify the target register
3. **Direction Detection**: Determines read vs write using `exception_sysreg_direction_write()`
4. **GPR Mapping**: Identifies the guest register involved using `exception_sysreg_gpr()`

Sources: [src/exception.rs(L184 - L211)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L184-L211) [src/exception_utils.rs(L174 - L195)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L174-L195)

### PSCI and Hypervisor Call Handling

The system supports both PSCI (Power State Coordination Interface) calls and general hypervisor calls through the HVC instruction. The `handle_psci_call` function recognizes PSCI function IDs in specific ranges:

* **32-bit PSCI**: Function IDs `0x8400_0000..=0x8400_001F`
* **64-bit PSCI**: Function IDs `0xC400_0000..=0xC400_001F`

Supported PSCI operations include:

* `PSCI_FN_CPU_OFF` → `AxVCpuExitReason::CpuDown`
* `PSCI_FN_CPU_ON` → `AxVCpuExitReason::CpuUp`
* `PSCI_FN_SYSTEM_OFF` → `AxVCpuExitReason::SystemDown`

Non-PSCI HVC calls are treated as general hypervisor calls with the call number in `x0` and arguments in `x1-x6`.

Sources: [src/exception.rs(L220 - L254)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L220-L254) [src/exception.rs(L80 - L102)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L80-L102)

### SMC Call Handling

Secure Monitor Calls (SMC) are either handled as PSCI calls or forwarded directly to the ARM Trusted Firmware (ATF). The `handle_smc64_exception` function first checks for PSCI compatibility, then uses the `smc_call` function to forward non-PSCI SMCs to the secure world.

Sources: [src/exception.rs(L256 - L271)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L256-L271) [src/exception.rs(L104 - L109)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L104-L109)

## Integration with Context Management

The exception handling system integrates tightly with the VCPU context management through several mechanisms:

**Context Integration Points**

```mermaid
flowchart TD
subgraph subGraph2["Exception Exit"]
    RESTORE_MACRO["RESTORE_REGS_INTO_EL1"]
    CONTEXT_ENTRY["context_vm_entry"]
    ERET_INST["eret instruction"]
end
subgraph subGraph1["Handler Processing"]
    CONTEXT_PARAM["ctx: &mut TrapFrame"]
    GPR_ACCESS["ctx.gpr[n] access"]
    PC_UPDATE["ctx.set_exception_pc()"]
end
subgraph subGraph0["Exception Entry"]
    SAVE_MACRO["SAVE_REGS_FROM_EL1"]
    TRAP_FRAME_SAVE["TrapFrame Population"]
end

CONTEXT_ENTRY --> ERET_INST
CONTEXT_PARAM --> GPR_ACCESS
CONTEXT_PARAM --> PC_UPDATE
GPR_ACCESS --> RESTORE_MACRO
PC_UPDATE --> RESTORE_MACRO
RESTORE_MACRO --> CONTEXT_ENTRY
SAVE_MACRO --> TRAP_FRAME_SAVE
TRAP_FRAME_SAVE --> CONTEXT_PARAM
```

The system maintains guest CPU state through the `TrapFrame` structure, allowing handlers to:

* Read guest register values for processing hypervisor calls and data aborts
* Update the program counter when advancing past trapped instructions
* Modify register contents for return values (e.g., MMIO read results)

Sources: [src/exception.S(L1 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.S#L1-L30) [src/exception.S(L32 - L58)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.S#L32-L58) [src/exception.rs(L75 - L77)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L75-L77) [src/exception.rs(L105 - L107)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception.rs#L105-L107)
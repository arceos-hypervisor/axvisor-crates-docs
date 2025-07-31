# Exception Analysis and Utilities

> **Relevant source files**
> * [src/exception_utils.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs)

This document covers the exception analysis and parsing utilities provided by the `exception_utils.rs` module. These utilities form a critical layer between the low-level assembly exception vectors and high-level exception handlers, providing functions to extract and interpret information from AArch64 exception syndrome registers, perform address translations, and manage register context during exception handling.

For information about the assembly exception vectors that capture exceptions, see [Assembly Exception Vectors](/arceos-hypervisor/arm_vcpu/4.1-assembly-exception-vectors). For details about high-level exception dispatch and handling logic, see [High-Level Exception Handling](/arceos-hypervisor/arm_vcpu/4.3-high-level-exception-handling).

## Exception Syndrome Register Analysis

The exception analysis system provides a comprehensive interface for reading and interpreting the Exception Syndrome Register (ESR_EL2), which contains detailed information about the cause and nature of exceptions that occur during guest execution.

### Exception Syndrome Register Reading Functions

```mermaid
flowchart TD
subgraph subGraph1["Exception Classification"]
    EC["Exception Class (EC)"]
    ISS["Instruction Specific Syndrome (ISS)"]
    IL["Instruction Length (IL)"]
end
subgraph subGraph0["Core Reading Functions"]
    exception_esr["exception_esr()"]
    exception_class["exception_class()"]
    exception_class_value["exception_class_value()"]
    exception_iss["exception_iss()"]
end
ESR_EL2["ESR_EL2 Hardware Register"]
DataAbort["Data Abort Analysis"]
SystemReg["System Register Analysis"]
HVC["HVC Call Analysis"]

EC --> DataAbort
EC --> HVC
EC --> SystemReg
ESR_EL2 --> exception_class
ESR_EL2 --> exception_class_value
ESR_EL2 --> exception_esr
ESR_EL2 --> exception_iss
ISS --> DataAbort
ISS --> SystemReg
exception_class --> EC
exception_esr --> IL
exception_iss --> ISS
```

The system provides several core functions for accessing different fields of the ESR_EL2 register:

* `exception_esr()` [src/exception_utils.rs(L12 - L14)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L12-L14) returns the complete ESR_EL2 register value
* `exception_class()` [src/exception_utils.rs(L21 - L23)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L21-L23) extracts the Exception Class field as an enum value
* `exception_class_value()` [src/exception_utils.rs(L30 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L30-L32) extracts the Exception Class as a raw numeric value
* `exception_iss()` [src/exception_utils.rs(L170 - L172)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L170-L172) extracts the Instruction Specific Syndrome field

Sources: [src/exception_utils.rs(L7 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L7-L32) [src/exception_utils.rs(L165 - L172)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L165-L172)

### Instruction Analysis

The system provides utilities for analyzing instruction-related information from exceptions:

|Function|Purpose|Return Value|
| --- | --- | --- |
|exception_instruction_length()|Determines if instruction is 16-bit or 32-bit|0 for 16-bit, 1 for 32-bit|
|exception_next_instruction_step()|Calculates step size for instruction pointer advancement|2 for 16-bit, 4 for 32-bit|

Sources: [src/exception_utils.rs(L144 - L163)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L144-L163)

## Fault Address Translation System

The fault address translation system handles the complex process of converting virtual fault addresses to physical addresses using the ARM Address Translation (AT) instructions and register coordination.

### Address Translation Flow

```mermaid
flowchart TD
subgraph subGraph2["Final Address Calculation"]
    Page_Offset["FAR & 0xfff"]
    Page_Number["HPFAR << 8"]
    Final_GPA["GuestPhysAddr"]
end
subgraph subGraph1["Translation Process"]
    arm_at["arm_at!(s1e1r, far)"]
    translate_far_to_hpfar["translate_far_to_hpfar()"]
    par_to_far["par_to_far() helper"]
    exception_hpfar["exception_hpfar()"]
end
subgraph subGraph0["Translation Decision Logic"]
    S1PTW_Check["Check ESR_ELx_S1PTW bit"]
    Permission_Check["exception_data_abort_is_permission_fault()"]
    Translation_Needed["Translation Needed?"]
end
FAR_EL2["FAR_EL2: Fault Address Register"]
HPFAR_EL2["HPFAR_EL2: Hypervisor Physical Fault Address Register"]
PAR_EL1["PAR_EL1: Physical Address Register"]

FAR_EL2 --> Page_Offset
FAR_EL2 --> S1PTW_Check
PAR_EL1 --> par_to_far
Page_Number --> Final_GPA
Page_Offset --> Final_GPA
Permission_Check --> Translation_Needed
S1PTW_Check --> Permission_Check
Translation_Needed --> exception_hpfar
Translation_Needed --> translate_far_to_hpfar
arm_at --> PAR_EL1
exception_hpfar --> Page_Number
par_to_far --> Page_Number
translate_far_to_hpfar --> arm_at
```

The `exception_fault_addr()` function [src/exception_utils.rs(L133 - L142)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L133-L142) implements a sophisticated address translation algorithm that:

1. Reads the FAR_EL2 register to get the virtual fault address
2. Determines whether address translation is needed based on the S1PTW bit and fault type
3. Either performs address translation using `translate_far_to_hpfar()` or directly reads HPFAR_EL2
4. Combines the page offset from FAR_EL2 with the page number from HPFAR_EL2

The `translate_far_to_hpfar()` function [src/exception_utils.rs(L92 - L113)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L92-L113) uses the ARM Address Translation instruction to convert a virtual address to physical, handling the PAR_EL1 register state and error conditions.

Sources: [src/exception_utils.rs(L34 - L142)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L34-L142)

## Data Abort Exception Analysis

The system provides comprehensive analysis capabilities for data abort exceptions, which are among the most common exceptions in virtualization scenarios involving MMIO operations.

### Data Abort Analysis Functions

```mermaid
flowchart TD
subgraph subGraph1["Access Pattern Analysis"]
    exception_data_abort_access_is_write["exception_data_abort_access_is_write()"]
    exception_data_abort_access_width["exception_data_abort_access_width()"]
    exception_data_abort_access_reg["exception_data_abort_access_reg()"]
    exception_data_abort_access_reg_width["exception_data_abort_access_reg_width()"]
    exception_data_abort_access_is_sign_ext["exception_data_abort_access_is_sign_ext()"]
end
subgraph subGraph0["Data Abort Classification"]
    exception_data_abort_is_permission_fault["exception_data_abort_is_permission_fault()"]
    exception_data_abort_is_translate_fault["exception_data_abort_is_translate_fault()"]
    exception_data_abort_handleable["exception_data_abort_handleable()"]
end
ESR_ISS["ESR_EL2.ISS Field"]

ESR_ISS --> exception_data_abort_access_is_sign_ext
ESR_ISS --> exception_data_abort_access_is_write
ESR_ISS --> exception_data_abort_access_reg
ESR_ISS --> exception_data_abort_access_reg_width
ESR_ISS --> exception_data_abort_access_width
ESR_ISS --> exception_data_abort_handleable
ESR_ISS --> exception_data_abort_is_permission_fault
ESR_ISS --> exception_data_abort_is_translate_fault
```

The data abort analysis functions extract specific information from the ISS field of ESR_EL2:

|Function|Bit Fields|Purpose|
| --- | --- | --- |
|exception_data_abort_is_permission_fault()|ISS[5:0] & 0xF0 == 12|Identifies permission faults vs translation faults|
|exception_data_abort_is_translate_fault()|ISS[5:0] & 0xF0 == 4|Identifies translation faults|
|exception_data_abort_access_is_write()|ISS[6]|Determines read vs write access|
|exception_data_abort_access_width()|ISS[23:22]|Access width (1, 2, 4, or 8 bytes)|
|exception_data_abort_access_reg()|ISS[20:16]|Register index (0-31)|
|exception_data_abort_handleable()|ISS[24] \| !ISS[10]|Determines if abort can be handled|

Sources: [src/exception_utils.rs(L197 - L255)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L197-L255)

## System Register Access Analysis

The system provides specialized parsing for system register access exceptions, which occur when guests attempt to access system registers that require hypervisor intervention.

### System Register Parsing Functions

```mermaid
flowchart TD
subgraph subGraph1["Parsing Functions"]
    exception_sysreg_direction_write["exception_sysreg_direction_write()"]
    exception_sysreg_gpr["exception_sysreg_gpr()"]
    exception_sysreg_addr["exception_sysreg_addr()"]
end
subgraph subGraph0["System Register ISS Format"]
    ISS_Field["ESR_EL2.ISS[24:0]"]
    Op0["Op0[21:20]"]
    Op2["Op2[19:17]"]
    Op1["Op1[16:14]"]
    CRn["CRn[13:10]"]
    CRm["CRm[4:1]"]
    Direction["Direction[0]"]
    RT["RT[9:5]"]
end

CRm --> exception_sysreg_addr
CRn --> exception_sysreg_addr
Direction --> exception_sysreg_direction_write
ISS_Field --> CRm
ISS_Field --> CRn
ISS_Field --> Direction
ISS_Field --> Op0
ISS_Field --> Op1
ISS_Field --> Op2
ISS_Field --> RT
Op0 --> exception_sysreg_addr
Op1 --> exception_sysreg_addr
Op2 --> exception_sysreg_addr
RT --> exception_sysreg_gpr
```

The system register analysis functions provide:

* `exception_sysreg_direction_write()` [src/exception_utils.rs(L175 - L178)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L175-L178) determines if the access is a write (true) or read (false)
* `exception_sysreg_gpr()` [src/exception_utils.rs(L181 - L186)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L181-L186) extracts the general-purpose register index (RT field)
* `exception_sysreg_addr()` [src/exception_utils.rs(L192 - L195)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L192-L195) constructs the system register address from encoding fields

The `exception_sysreg_addr()` function implements the ARMv8 system register encoding format, combining Op0, Op1, Op2, CRn, and CRm fields into a unique register identifier.

Sources: [src/exception_utils.rs(L174 - L195)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L174-L195)

## Register Context Management Macros

The system provides assembly macros for managing host register context during exception handling, ensuring proper preservation and restoration of callee-saved registers.

### Context Switching Macros

```mermaid
flowchart TD
subgraph subGraph1["Exception Flow Integration"]
    VM_Entry["VM Entry from Aarch64VCpu::run()"]
    VM_Exit["VM Exit to Exception Handler"]
    Return_to_Host["Return to Aarch64VCpu::run()"]
end
subgraph subGraph0["Host Context Management"]
    Host_Registers["Host Callee-Saved Registersx19-x30"]
    Stack_Frame["Stack Frame12 * 8 bytes"]
    save_regs_to_stack["save_regs_to_stack!"]
    restore_regs_from_stack["restore_regs_from_stack!"]
end
Exception_Processing["Exception Processing"]

Exception_Processing --> restore_regs_from_stack
Host_Registers --> Return_to_Host
Host_Registers --> Stack_Frame
Stack_Frame --> Host_Registers
VM_Entry --> save_regs_to_stack
VM_Exit --> Exception_Processing
restore_regs_from_stack --> Stack_Frame
save_regs_to_stack --> Host_Registers
```

The register context management system consists of two complementary macros:

**`save_regs_to_stack!`** [src/exception_utils.rs(L278 - L289)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L278-L289):

* Allocates 96 bytes (12 * 8) on the stack
* Saves registers x19-x30 using `stp` (store pair) instructions
* Used during VM entry to preserve host state

**`restore_regs_from_stack!`** [src/exception_utils.rs(L300 - L311)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L300-L311):

* Restores registers x19-x30 using `ldp` (load pair) instructions
* Deallocates the 96-byte stack frame
* Used during VM exit to restore host state

These macros ensure that the host's callee-saved registers are properly preserved across guest execution, maintaining the calling convention for the `Aarch64VCpu::run()` function.

Sources: [src/exception_utils.rs(L267 - L311)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L267-L311)

## Integration with Exception Handling Pipeline

The exception utilities integrate seamlessly with the broader exception handling system, providing the analytical foundation for exception dispatch and handling decisions.

### Utility Function Usage Pattern

```mermaid
flowchart TD
subgraph subGraph2["Exception Handlers"]
    handle_data_abort["handle_data_abort()"]
    handle_system_register["handle_system_register()"]
    handle_psci_call["handle_psci_call()"]
end
subgraph subGraph1["Exception Type Specific Analysis"]
    Data_Abort["Data Abort Functions"]
    System_Register["System Register Functions"]
    HVC_Analysis["HVC Call Analysis"]
end
subgraph subGraph0["Exception Analysis Phase"]
    exception_esr["exception_esr()"]
    exception_class["exception_class()"]
    exception_fault_addr["exception_fault_addr()"]
end
Exception_Vector["Assembly Exception Vector"]
Exception_Context["Exception Context Capture"]

Data_Abort --> exception_fault_addr
Data_Abort --> handle_data_abort
Exception_Context --> exception_esr
Exception_Vector --> Exception_Context
HVC_Analysis --> handle_psci_call
System_Register --> handle_system_register
exception_class --> Data_Abort
exception_class --> HVC_Analysis
exception_class --> System_Register
exception_esr --> exception_class
```

The utilities serve as the foundational layer that enables higher-level exception handlers to make informed decisions about how to process different types of exceptions, providing both classification and detailed analysis capabilities.

Sources: [src/exception_utils.rs(L1 - L311)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/exception_utils.rs#L1-L311)
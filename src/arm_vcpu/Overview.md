# Overview

> **Relevant source files**
> * [Cargo.toml](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/Cargo.toml)
> * [README.md](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/README.md)
> * [src/lib.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs)

This document provides a comprehensive overview of the `arm_vcpu` hypervisor system, an AArch64 virtualization implementation that provides virtual CPU management and hardware abstraction for hypervisor environments. The system implements low-level virtualization primitives including CPU context switching, exception handling, and guest state management for AArch64 platforms.

The `arm_vcpu` crate serves as a foundational component for hypervisor implementations, providing the core VCPU abstraction and associated hardware interfaces. For detailed information about specific subsystems, see [Virtual CPU Management](/arceos-hypervisor/arm_vcpu/2-virtual-cpu-management), [Exception Handling System](/arceos-hypervisor/arm_vcpu/4-exception-handling-system), and [System Integration](/arceos-hypervisor/arm_vcpu/5-system-integration).

## System Purpose and Scope

The `arm_vcpu` system implements virtualization support for AArch64 architecture, providing:

* **Virtual CPU Management**: Core VCPU lifecycle, state management, and execution control
* **Hardware Abstraction**: Platform-independent interfaces for virtualization hardware features
* **Exception Handling**: Comprehensive trap and interrupt handling for guest VMs
* **Context Switching**: Efficient state preservation and restoration between host and guest execution
* **Security Integration**: Secure Monitor Call (SMC) interface for trusted firmware interaction

Sources: [README.md(L1 - L5)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/README.md#L1-L5) [src/lib.rs(L1 - L33)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L1-L33)

## Primary System Components

The following diagram illustrates the main architectural components and their relationships within the `arm_vcpu` system:

**Core Component Architecture**

```mermaid
flowchart TD
subgraph subGraph2["External Dependencies"]
    AxVCpu["axvcpuCore VCPU Traits"]
    AxAddrSpace["axaddrspaceAddress Space Management"]
    PerCpuCrate["percpuPer-CPU Storage"]
    Aarch64Cpu["aarch64-cpuRegister Definitions"]
end
subgraph subGraph1["Internal Modules"]
    ContextFrame["context_frameState Structures"]
    Exception["exceptionHandler Logic"]
    ExceptionUtils["exception_utilsRegister Parsing"]
    PCpuMod["pcpuPer-CPU Implementation"]
    SMC["smcSecure Monitor Interface"]
    VCpuMod["vcpuVCPU Implementation"]
end
subgraph subGraph0["Primary Interfaces"]
    VCpu["Aarch64VCpuMain VCPU Implementation"]
    PerCpu["Aarch64PerCpuPer-CPU State Management"]
    TrapFrame["TrapFrame(Aarch64ContextFrame)"]
    Config["Aarch64VCpuCreateConfigVCPU Configuration"]
end

Exception --> Aarch64Cpu
Exception --> ExceptionUtils
PerCpu --> PCpuMod
PerCpu --> PerCpuCrate
TrapFrame --> ContextFrame
VCpu --> AxAddrSpace
VCpu --> AxVCpu
VCpu --> Config
VCpu --> PerCpu
VCpu --> TrapFrame
VCpu --> VCpuMod
VCpuMod --> Exception
VCpuMod --> SMC
```

Sources: [src/lib.rs(L9 - L21)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L9-L21) [Cargo.toml(L6 - L19)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/Cargo.toml#L6-L19)

## Code Entity Mapping

This diagram maps the natural language system concepts to specific code entities and file structures:

**Implementation Structure and Code Entities**

```mermaid
flowchart TD
subgraph subGraph2["Core Data Structures"]
    Aarch64VCpuStruct["struct Aarch64VCpu"]
    Aarch64PerCpuStruct["struct Aarch64PerCpu"]
    ContextFrameStruct["struct Aarch64ContextFrame"]
    ConfigStruct["struct Aarch64VCpuCreateConfig"]
end
subgraph subGraph1["Source File Organization"]
    VCpuRs["vcpu.rsVirtual CPU Logic"]
    PCpuRs["pcpu.rsPer-CPU Management"]
    ContextRs["context_frame.rsState Structures"]
    ExceptionRs["exception.rsException Handlers"]
    ExceptionS["exception.SAssembly Vectors"]
    UtilsRs["exception_utils.rsRegister Utilities"]
    SmcRs["smc.rsSecure Monitor Calls"]
end
subgraph subGraph0["Public API Surface"]
    LibRs["lib.rsMain Module"]
    PubVCpu["pub use Aarch64VCpu"]
    PubPerCpu["pub use Aarch64PerCpu"]
    PubConfig["pub use Aarch64VCpuCreateConfig"]
    PubTrapFrame["pub type TrapFrame"]
    HwSupport["has_hardware_support()"]
end

ContextRs --> ContextFrameStruct
ExceptionRs --> ExceptionS
ExceptionRs --> UtilsRs
LibRs --> HwSupport
LibRs --> PubConfig
LibRs --> PubPerCpu
LibRs --> PubTrapFrame
LibRs --> PubVCpu
PCpuRs --> Aarch64PerCpuStruct
PubPerCpu --> PCpuRs
PubTrapFrame --> ContextRs
PubVCpu --> VCpuRs
VCpuRs --> Aarch64VCpuStruct
VCpuRs --> ConfigStruct
VCpuRs --> ExceptionRs
VCpuRs --> SmcRs
```

Sources: [src/lib.rs(L17 - L21)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L17-L21) [src/lib.rs(L9 - L15)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L9-L15)

## Virtualization Hardware Integration

The system provides hardware abstraction for AArch64 virtualization extensions through a layered approach:

|Layer|Component|Responsibility|
| --- | --- | --- |
|Hardware|AArch64 CPU|Virtualization extensions (EL2, VHE)|
|Low-Level|exception.S|Assembly exception vectors and context switch|
|Abstraction|AxVCpuHaltrait|Hardware abstraction interface|
|Implementation|Aarch64VCpu|Concrete VCPU implementation|
|Integration|axvcpucrate|Generic VCPU traits and interfaces|

**Hardware Feature Detection and Support**

```mermaid
flowchart TD
subgraph subGraph2["Per-CPU Control"]
    HwSupport["has_hardware_support()Feature Detection"]
    IdRegs["ID_AA64MMFR1_EL1Virtualization Host Extensions"]
    DefaultTrue["Currently returns truePlaceholder Implementation"]
    PerCpuEnable["Aarch64PerCpu::hardware_enable()"]
    PerCpuDisable["Aarch64PerCpu::hardware_disable()"]
    VirtualizationState["Per-CPU Virtualization State"]
end
subgraph subGraph1["Virtualization Extensions"]
    EL2["Exception Level 2Hypervisor Mode"]
    VHE["Virtualization Host ExtensionsEnhanced Host Support"]
    HCR["HCR_EL2 RegisterHypervisor Configuration"]
    VBAR["VBAR_EL2 RegisterVector Base Address"]
end
subgraph subGraph0["Platform Detection"]
    HwSupport["has_hardware_support()Feature Detection"]
    IdRegs["ID_AA64MMFR1_EL1Virtualization Host Extensions"]
    DefaultTrue["Currently returns truePlaceholder Implementation"]
end

HwSupport --> DefaultTrue
HwSupport --> IdRegs
PerCpuEnable --> EL2
PerCpuEnable --> HCR
PerCpuEnable --> VBAR
VirtualizationState --> PerCpuDisable
VirtualizationState --> PerCpuEnable
```

Sources: [src/lib.rs(L23 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L23-L32) [Cargo.toml(L15)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/Cargo.toml#L15-L15)

## External Ecosystem Integration

The `arm_vcpu` crate integrates with the broader ArceOS hypervisor ecosystem through well-defined dependency relationships:

|Dependency|Purpose|Integration Point|
| --- | --- | --- |
|axvcpu|Core VCPU traits|AxVCpuHaltrait implementation|
|axaddrspace|Address space management|Memory virtualization support|
|percpu|Per-CPU data structures|Hardware state management|
|aarch64-cpu|Register definitions|Low-level hardware access|
|aarch64_sysreg|System register access|Control register manipulation|

The system operates at Exception Level 2 (EL2) and provides virtualization services for guest operating systems running at Exception Level 1 (EL1) and applications at Exception Level 0 (EL0).

Sources: [Cargo.toml(L14 - L19)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/Cargo.toml#L14-L19) [README.md(L5)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/README.md#L5-L5)
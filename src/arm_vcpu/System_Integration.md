# System Integration

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs)
> * [src/smc.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs)

This document covers the integration interfaces and hardware abstraction mechanisms that enable the arm_vcpu hypervisor to work with host systems, hardware platforms, and external components. It focuses on how the core virtualization functionality interfaces with the broader hypervisor ecosystem and underlying hardware.

For detailed information about the Secure Monitor Call interface, see [Secure Monitor Interface](/arceos-hypervisor/arm_vcpu/5.1-secure-monitor-interface). For platform-specific hardware support details, see [Hardware Abstraction and Platform Support](/arceos-hypervisor/arm_vcpu/5.2-hardware-abstraction-and-platform-support).

## Integration Architecture Overview

The arm_vcpu crate serves as a hardware-specific implementation that integrates with higher-level hypervisor frameworks through well-defined interfaces. The integration occurs at multiple layers, from hardware abstraction to external crate dependencies.

**Integration Architecture with Code Entities**

```mermaid
flowchart TD
subgraph subGraph3["Hardware/Firmware Interface"]
    SMC_IF["smc.rssmc_call()"]
    HW_DETECT["Hardware ExtensionDetection"]
    SECURE_FW["Secure Firmware(ATF/EL3)"]
end
subgraph subGraph2["External Crate Dependencies"]
    AXVCPU_CRATE["axvcpu crateCore VCPU traits"]
    AXADDR_CRATE["axaddrspace crateAddress space mgmt"]
    PERCPU_CRATE["percpu cratePer-CPU storage"]
    AARCH64_CRATE["aarch64-cpu crateRegister definitions"]
end
subgraph subGraph1["arm_vcpu Public Interface"]
    LIB["lib.rs"]
    VCPU_EXPORT["Aarch64VCpu"]
    PCPU_EXPORT["Aarch64PerCpu"]
    CONFIG_EXPORT["Aarch64VCpuCreateConfig"]
    TRAPFRAME_EXPORT["TrapFrame"]
    HW_SUPPORT["has_hardware_support()"]
end
subgraph subGraph0["Host Hypervisor System"]
    HOST["Host OS/Hypervisor"]
    HAL_TRAIT["AxVCpuHal trait(from axvcpu crate)"]
end

HAL_TRAIT --> VCPU_EXPORT
HOST --> HAL_TRAIT
HW_SUPPORT --> HW_DETECT
LIB --> AARCH64_CRATE
LIB --> CONFIG_EXPORT
LIB --> HW_SUPPORT
LIB --> PCPU_EXPORT
LIB --> TRAPFRAME_EXPORT
LIB --> VCPU_EXPORT
PCPU_EXPORT --> PERCPU_CRATE
SMC_IF --> SECURE_FW
VCPU_EXPORT --> AXADDR_CRATE
VCPU_EXPORT --> AXVCPU_CRATE
```

The integration architecture centers around the public interface defined in [src/lib.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs) which exports the core components that host hypervisors use to manage AArch64 virtual CPUs.

Sources: [src/lib.rs(L17 - L18)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L17-L18) [src/lib.rs(L21)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L21-L21) [src/lib.rs(L24 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L24-L32)

## Hardware Abstraction Layer Interface

The arm_vcpu crate implements hardware-specific functionality while depending on abstract interfaces for integration with higher-level hypervisor components. The primary abstraction mechanism is the `AxVCpuHal` trait from the axvcpu crate.

**Hardware Abstraction Layer Structure**

```mermaid
flowchart TD
subgraph subGraph2["Hardware Platform"]
    VIRT_EXT["AArch64 VirtualizationExtensions (VHE/EL2)"]
    CPU_REGS["aarch64-cpu crateRegister definitions"]
    PLATFORM_HW["Platform HardwareFeatures"]
end
subgraph subGraph1["arm_vcpu Implementation"]
    AARCH64_VCPU["Aarch64VCpuimplements AxVCpuHal"]
    AARCH64_PCPU["Aarch64PerCpuPer-CPU management"]
    HW_SUPPORT_FUNC["has_hardware_support()Platform detection"]
end
subgraph subGraph0["Abstract Interface Layer"]
    AXVCPU_HAL["AxVCpuHal trait(axvcpu crate)"]
    AXVCPU_CORE["AxVCpu core traits(axvcpu crate)"]
end

AARCH64_PCPU --> VIRT_EXT
AARCH64_VCPU --> AARCH64_PCPU
AARCH64_VCPU --> CPU_REGS
AXVCPU_CORE --> AARCH64_VCPU
AXVCPU_HAL --> AARCH64_VCPU
HW_SUPPORT_FUNC --> PLATFORM_HW
```

The hardware abstraction enables the arm_vcpu implementation to provide AArch64-specific virtualization functionality while remaining compatible with generic hypervisor frameworks that depend on the abstract `AxVCpuHal` interface.

Sources: [src/lib.rs(L17 - L18)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L17-L18) [src/lib.rs(L24 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L24-L32)

## External Dependency Integration

The arm_vcpu crate integrates with several external crates that provide fundamental hypervisor infrastructure. Each dependency serves a specific role in the overall system architecture.

|Crate|Purpose|Integration Point|
| --- | --- | --- |
|axvcpu|Core VCPU traits and interfaces|Aarch64VCpuimplementsAxVCpuHal|
|axaddrspace|Address space management|Used byAarch64VCpufor memory virtualization|
|percpu|Per-CPU data storage|Used byAarch64PerCpufor CPU-local state|
|aarch64-cpu|AArch64 register definitions|Used throughout for hardware register access|
|log|Logging infrastructure|Used for debug and error reporting|

**External Dependency Flow**

```mermaid
flowchart TD
subgraph subGraph1["External Crate APIs"]
    AXVCPU_API["axvcpu::AxVCpuHalHardware abstraction"]
    AXADDR_API["axaddrspaceMemory management"]
    PERCPU_API["percpuPer-CPU storage"]
    AARCH64_API["aarch64_cpuRegister access"]
    LOG_API["log macrosdebug!, warn!, error!"]
end
subgraph subGraph0["arm_vcpu Components"]
    VCPU_IMPL["src/vcpu.rsAarch64VCpu"]
    PCPU_IMPL["src/pcpu.rsAarch64PerCpu"]
    CONTEXT["src/context_frame.rsContext management"]
    EXCEPTION["src/exception.rsException handling"]
end

CONTEXT --> AARCH64_API
EXCEPTION --> AARCH64_API
EXCEPTION --> LOG_API
PCPU_IMPL --> PERCPU_API
VCPU_IMPL --> AXADDR_API
VCPU_IMPL --> AXVCPU_API
```

The external dependencies provide the foundational infrastructure that the arm_vcpu implementation builds upon, allowing it to focus on AArch64-specific virtualization concerns.

Sources: [src/lib.rs(L6 - L7)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L6-L7) [src/lib.rs(L17 - L18)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L17-L18)

## Platform Hardware Integration

The arm_vcpu crate provides platform hardware integration through hardware capability detection and secure firmware communication interfaces.

**Hardware Integration Points**

```mermaid
flowchart TD
subgraph subGraph2["Hardware Features"]
    VIRT_EXT["AArch64 VirtualizationExtensions"]
    EL2_MODE["Exception Level 2Hypervisor mode"]
    SECURE_STATE["Secure/Non-secureState separation"]
end
subgraph subGraph1["Secure Communication"]
    SMC_CALL["smc_call()smc.rs:10-26"]
    SMC_ASM["smc #0Assembly instruction"]
    ATF_EL3["ARM Trusted FirmwareEL3 Secure Monitor"]
end
subgraph subGraph0["Platform Detection"]
    HAS_HW_SUPPORT["has_hardware_support()lib.rs:24-32"]
    VHE_CHECK["ID_AA64MMFR1_EL1Virtualization Host Extensions"]
    CORTEX_A78["Cortex-A78Example platform"]
end

HAS_HW_SUPPORT --> VHE_CHECK
HAS_HW_SUPPORT --> VIRT_EXT
SMC_ASM --> ATF_EL3
SMC_CALL --> SECURE_STATE
SMC_CALL --> SMC_ASM
VHE_CHECK --> CORTEX_A78
VIRT_EXT --> EL2_MODE
```

The platform integration provides two key capabilities:

1. **Hardware Capability Detection**: The `has_hardware_support()` function determines if the current platform supports AArch64 virtualization extensions
2. **Secure Firmware Communication**: The `smc_call()` function enables communication with secure firmware components

Sources: [src/lib.rs(L24 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L24-L32) [src/smc.rs(L3 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L3-L26)

## Integration Safety and Constraints

The integration interfaces include several safety-critical components that require careful handling by host hypervisor systems.

The `smc_call()` function in [src/smc.rs(L10 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L10-L26) is marked as unsafe and requires the caller to ensure:

* `x0` contains a valid SMC function number per the SMC Calling Convention
* Arguments `x1`, `x2`, `x3` are valid for the specified SMC function
* The calling context is appropriate for secure monitor calls

The hardware support detection in [src/lib.rs(L24 - L32)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L24-L32) currently returns `true` by default but includes documentation for implementing proper detection using `ID_AA64MMFR1_EL1` register checks on platforms like Cortex-A78.

These safety constraints ensure that the arm_vcpu crate integrates properly with both hardware platforms and secure firmware components while maintaining the security boundaries required for hypervisor operation.

Sources: [src/smc.rs(L5 - L9)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L5-L9) [src/lib.rs(L25 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/lib.rs#L25-L30)
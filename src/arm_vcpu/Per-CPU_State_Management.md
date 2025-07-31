# Per-CPU State Management

> **Relevant source files**
> * [src/pcpu.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs)

This document covers the per-CPU state management system in the arm_vcpu hypervisor, which handles virtualization control and interrupt management on a per-processor basis. The system is responsible for enabling/disabling hardware virtualization features, managing exception vectors, and coordinating interrupt handling between the hypervisor and host OS.

For information about the overall VCPU lifecycle and operations, see [2.1](/arceos-hypervisor/arm_vcpu/2.1-vcpu-lifecycle-and-operations). For details about context switching and register state management, see [3](/arceos-hypervisor/arm_vcpu/3-context-switching-and-state-management).

## Per-CPU Data Structure

The core of per-CPU state management is the `Aarch64PerCpu` structure, which implements the `AxArchPerCpu` trait to provide processor-specific virtualization control.

### Aarch64PerCpu Structure

```mermaid
classDiagram
note for Aarch64PerCpu "4096-byte alignedRepresents per-CPU statefor virtualization control"
class Aarch64PerCpu {
    +cpu_id: usize
    -_phantom: PhantomData~H~
    +new(cpu_id: usize) AxResult~Self~
    +is_enabled() bool
    +hardware_enable() AxResult
    +hardware_disable() AxResult
}

class AxArchPerCpu {
    <<trait>>
    
    +new(cpu_id: usize) AxResult~Self~
    +is_enabled() bool
    +hardware_enable() AxResult
    +hardware_disable() AxResult
}

class AxVCpuHal {
    <<trait>>
    
    +irq_hanlder()
}

Aarch64PerCpu  ..|>  AxArchPerCpu : implements
Aarch64PerCpu  -->  AxVCpuHal : uses H generic
```

Sources: [src/pcpu.rs(L10 - L16)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L10-L16)

The structure is aligned to 4096 bytes and contains minimal data, serving primarily as a coordination point for per-CPU virtualization operations. The generic parameter `H` provides hardware abstraction through the `AxVCpuHal` trait.

## Per-CPU Variables

The system uses two key per-CPU variables to manage state across processor cores:

|Variable|Type|Purpose|
| --- | --- | --- |
|ORI_EXCEPTION_VECTOR_BASE|usize|Stores originalVBAR_EL2value before virtualization|
|IRQ_HANDLER|OnceCell<&(dyn Fn() + Send + Sync)>|Host OS IRQ handler for this CPU|

### Exception Vector Management

```mermaid
stateDiagram-v2
state HostVectors {
    [*] --> OriginalVBAR
    OriginalVBAR : "VBAR_EL2 = host exception vectors"
    OriginalVBAR : "Stored in ORI_EXCEPTION_VECTOR_BASE"
}
state VCpuVectors {
    [*] --> VCpuVBAR
    VCpuVBAR : "VBAR_EL2 = exception_vector_base_vcpu"
    VCpuVBAR : "Handles VM exits and traps"
}
[*] --> HostVectors : "Initial state"
HostVectors --> VCpuVectors : "hardware_enable()"
VCpuVectors --> HostVectors : "hardware_disable()"
note left of VCpuVectors : ['exception_vector_base_vcpu()<br>defined in assembly']
```

Sources: [src/pcpu.rs(L18 - L19)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L18-L19) [src/pcpu.rs(L28 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L28-L30) [src/pcpu.rs(L53 - L57)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L53-L57) [src/pcpu.rs(L74 - L77)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L74-L77)

## Hardware Virtualization Control

The per-CPU system manages the AArch64 virtualization extensions through the Hypervisor Configuration Register (HCR_EL2).

### Virtualization State Management

```mermaid
flowchart TD
subgraph subGraph0["HCR_EL2 Configuration"]
    G1["HCR_EL2::VM::Enable"]
    G2["HCR_EL2::RW::EL1IsAarch64"]
    G3["HCR_EL2::IMO::EnableVirtualIRQ"]
    G4["HCR_EL2::FMO::EnableVirtualFIQ"]
    G5["HCR_EL2::TSC::EnableTrapEl1SmcToEl2"]
end
A["new(cpu_id)"]
B["Register IRQ handler"]
C["Per-CPU instance ready"]
D["hardware_enable()"]
E["Save original VBAR_EL2"]
F["Set VCPU exception vectors"]
G["Configure HCR_EL2"]
H["Virtualization enabled"]
I["is_enabled() returns true"]
J["hardware_disable()"]
K["Restore original VBAR_EL2"]
L["Clear HCR_EL2.VM"]
M["Virtualization disabled"]

A --> B
B --> C
C --> D
D --> E
E --> F
F --> G
G --> G1
G --> G2
G --> G3
G --> G4
G --> G5
G --> H
H --> I
H --> J
J --> K
K --> L
L --> M
```

Sources: [src/pcpu.rs(L32 - L78)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L32-L78)

### HCR_EL2 Register Configuration

When enabling virtualization, the system configures several key control bits:

|Bit Field|Setting|Purpose|
| --- | --- | --- |
|VM|Enable|Enables virtualization extensions|
|RW|EL1IsAarch64|Sets EL1 execution state to AArch64|
|IMO|EnableVirtualIRQ|Routes IRQs to virtual interrupt controller|
|FMO|EnableVirtualFIQ|Routes FIQs to virtual interrupt controller|
|TSC|EnableTrapEl1SmcToEl2|Traps SMC instructions from EL1 to EL2|

Sources: [src/pcpu.rs(L59 - L65)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L59-L65)

## IRQ Handler Integration

The per-CPU system coordinates interrupt handling between the hypervisor and host OS through a registered handler mechanism.

### IRQ Handler Registration

```mermaid
sequenceDiagram
    participant HostOS as Host OS
    participant Aarch64PerCpunew as Aarch64PerCpu::new()
    participant IRQ_HANDLER as IRQ_HANDLER
    participant AxVCpuHalirq_hanlder as AxVCpuHal::irq_hanlder()

    HostOS ->> Aarch64PerCpunew: Create per-CPU instance
    Aarch64PerCpunew ->> IRQ_HANDLER: Get current CPU's IRQ_HANDLER
    Aarch64PerCpunew ->> AxVCpuHalirq_hanlder: Call H::irq_hanlder()
    AxVCpuHalirq_hanlder -->> Aarch64PerCpunew: Return handler function
    Aarch64PerCpunew ->> IRQ_HANDLER: Set handler in OnceCell
    IRQ_HANDLER -->> Aarch64PerCpunew: Registration complete
    Aarch64PerCpunew -->> HostOS: Per-CPU instance ready
    Note over IRQ_HANDLER: Handler stored as<br>per-CPU variable<br>for current processor
```

Sources: [src/pcpu.rs(L21 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L21-L26) [src/pcpu.rs(L34 - L37)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L34-L37)

The IRQ handler is set once per CPU during initialization and provides a mechanism for the hypervisor's exception handling system to dispatch interrupts back to the host OS when appropriate.

## Integration with VCPU System

The per-CPU state management integrates closely with the broader VCPU lifecycle:

### Per-CPU and VCPU Relationship

```mermaid
flowchart TD
subgraph subGraph2["Host OS Integration"]
    HostIRQ["Host IRQ Handler"]
    HostVectors["Host Exception Vectors"]
end
subgraph subGraph1["VCPU Operations"]
    VCPUCreate["Aarch64VCpu creation"]
    VCPURun["vcpu.run()"]
    ExceptionVectors["exception_vector_base_vcpu"]
end
subgraph subGraph0["CPU Core N"]
    PerCPU["Aarch64PerCpu<H>"]
    VBAR["VBAR_EL2 Register"]
    HCR["HCR_EL2 Register"]
    IRQVar["IRQ_HANDLER per-CPU var"]
end
note1["hardware_enable() sets upvirtualization for this CPU"]
note2["hardware_disable() restoresoriginal host state"]

ExceptionVectors --> VBAR
IRQVar --> HostIRQ
PerCPU --> HCR
PerCPU --> HostVectors
PerCPU --> IRQVar
PerCPU --> VBAR
PerCPU --> note1
PerCPU --> note2
VCPUCreate --> PerCPU
VCPURun --> PerCPU
```

Sources: [src/pcpu.rs(L45 - L78)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/pcpu.rs#L45-L78)

The per-CPU system provides the foundation for VCPU operations by ensuring that hardware virtualization extensions are properly configured and that interrupt handling is coordinated between the hypervisor and host OS on each processor core.
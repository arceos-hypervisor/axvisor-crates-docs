# Register Module Organization

> **Relevant source files**
> * [src/regs/mod.rs](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs)

## Purpose and Scope

This document covers the structural organization of the register abstraction layer within the arm_gicv2 crate. The register module provides hardware register definitions and abstractions that enable safe, type-safe access to ARM GICv2 hardware registers. This page focuses on how register definitions are organized, modularized, and exposed to the rest of the codebase.

For detailed information about specific register implementations, see [GICD_SGIR Register Details](/arceos-hypervisor/arm_gicv2/4.2-gicd_sgir-register-details). For broader context on how these registers integrate with the main GIC components, see [Core Architecture](/arceos-hypervisor/arm_gicv2/2-core-architecture).

## Module Structure Overview

The register module follows a hierarchical organization pattern where individual register definitions are contained in dedicated submodules and selectively re-exported through the main module interface.

### Register Module Hierarchy

```mermaid
flowchart TD
A["src/regs/mod.rs"]
B["gicd_sgir submodule"]
C["Public re-exports"]
D["gicd_sgir.rs file"]
E["GICD_SGIR register definition"]
F["pub use gicd_sgir::*"]
G["GicDistributor"]
H["GicCpuInterface"]
I["External consumers"]

A --> B
A --> C
A --> I
B --> D
C --> F
D --> E
G --> A
H --> A
```

**Sources**: [src/regs/mod.rs(L1 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs#L1-L4)

The current module organization demonstrates a focused approach where each hardware register receives its own dedicated submodule. The `mod.rs` file serves as the central coordination point that aggregates and exposes register definitions.

### Module Declaration and Re-export Pattern

The register module employs a straightforward declaration and re-export strategy:

|Module Component|Purpose|Implementation|
| --- | --- | --- |
|Submodule declaration|Declares register-specific modules|mod gicd_sgir;|
|Public re-export|Exposes register APIs to consumers|pub use gicd_sgir::*;|

**Sources**: [src/regs/mod.rs(L1 - L3)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs#L1-L3)

This pattern provides several architectural benefits:

* **Encapsulation**: Each register definition is isolated in its own module
* **Selective exposure**: Only intended APIs are re-exported through the main module
* **Maintainability**: New registers can be added by following the same pattern
* **Namespace management**: Wildcard re-exports flatten the API surface for consumers

## Register Organization Patterns

### Submodule Structure

```mermaid
flowchart TD
A["mod.rs"]
B["gicd_sgir"]
C["Register struct"]
D["Bit field definitions"]
E["Helper methods"]
F["Future register modules"]
G["Additional register types"]

A --> B
B --> C
B --> D
B --> E
F --> A
F --> G
```

**Sources**: [src/regs/mod.rs(L1)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs#L1-L1)

The modular approach allows for systematic expansion of register support. Each submodule can contain:

* Register structure definitions using `tock-registers` abstractions
* Bit field layouts and access patterns
* Register-specific helper functions and constants
* Documentation specific to that hardware register

### API Surface Management

The re-export strategy in the module creates a unified namespace for register access:

```mermaid
flowchart TD
A["src/regs/mod.rs"]
B["pub use gicd_sgir::*"]
C["GICD_SGIR register types"]
D["Related constants"]
E["Helper functions"]
F["GicDistributor"]
G["External crate users"]

A --> B
B --> C
B --> D
B --> E
F --> C
F --> D
F --> E
G --> A
```

**Sources**: [src/regs/mod.rs(L3)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs#L3-L3)

This design enables consumers to import register definitions through a single, well-defined module path while maintaining internal organization through submodules.

## Integration with Hardware Abstraction

### Register Module Role in GIC Architecture

```mermaid
flowchart TD
A["Hardware Layer"]
B["Register Abstractions"]
C["src/regs/mod.rs"]
D["gicd_sgir module"]
E["GicDistributor"]
F["GicCpuInterface"]
G["Memory-mapped hardware"]
H["Type-safe register access"]

A --> B
B --> C
C --> D
E --> C
F --> C
G --> A
H --> E
H --> F
```

**Sources**: [src/regs/mod.rs(L1 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs#L1-L4)

The register module serves as the foundational layer that translates raw hardware register layouts into type-safe Rust abstractions. This enables the higher-level `GicDistributor` and `GicCpuInterface` components to interact with hardware through well-defined, compile-time-verified interfaces.

## Extensibility and Future Growth

The current organization pattern supports systematic expansion for additional GIC registers:

|Register Category|Current Support|Extension Pattern|
| --- | --- | --- |
|Software Generated Interrupts|gicd_sgirmodule|Additional SGI-related registers|
|Distributor Control|Not yet implemented|gicd_ctlr,gicd_typermodules|
|CPU Interface|Not yet implemented|gicc_*register modules|
|Priority Management|Not yet implemented|Priority and masking register modules|

**Sources**: [src/regs/mod.rs(L1 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/src/regs/mod.rs#L1-L4)

Each new register would follow the established pattern of creating a dedicated submodule and adding appropriate re-exports to maintain the unified API surface while preserving internal modularity.
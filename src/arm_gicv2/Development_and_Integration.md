# Development and Integration

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml)
> * [Cargo.toml](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/Cargo.toml)

This document provides a comprehensive guide for developers on integrating and working with the `arm_gicv2` crate. It covers the development workflow, dependency management, build configuration, and quality assurance processes that support this hardware abstraction layer.

For specific technical details about crate dependencies and feature configuration, see [Crate Configuration and Dependencies](/arceos-hypervisor/arm_gicv2/5.1-crate-configuration-and-dependencies). For detailed information about the build system and CI/CD pipeline, see [Build System and Development Workflow](/arceos-hypervisor/arm_gicv2/5.2-build-system-and-development-workflow).

## Integration Overview

The `arm_gicv2` crate is designed as a foundational component for ARM-based systems requiring interrupt controller management. The crate follows embedded Rust best practices with `no_std` compatibility and minimal dependencies.

### Development Ecosystem Architecture

```mermaid
flowchart TD
subgraph subGraph3["Integration Targets"]
    ARCEOS["ArceOS Operating System"]
    EMBEDDED["Embedded Systems"]
    HYPERVISORS["Hypervisors/VMMs"]
    BARE_METAL["Bare Metal Applications"]
end
subgraph subGraph2["Target Platforms"]
    AARCH64_BARE["aarch64-unknown-none-softfloat"]
    X86_TEST["x86_64-unknown-linux-gnu (testing)"]
end
subgraph subGraph1["Core Dependencies"]
    TOCK_REGS["tock-registers 0.8"]
    RUST_NIGHTLY["Rust Nightly Toolchain"]
    NO_STD["no_std Environment"]
end
subgraph subGraph0["Development Infrastructure"]
    GITHUB["GitHub Repository"]
    CI_PIPELINE["CI Workflow (.github/workflows/ci.yml)"]
    DOCS_DEPLOY["GitHub Pages Documentation"]
    CARGO_TOML["Cargo.toml Configuration"]
end

AARCH64_BARE --> ARCEOS
AARCH64_BARE --> BARE_METAL
AARCH64_BARE --> EMBEDDED
AARCH64_BARE --> HYPERVISORS
CARGO_TOML --> CI_PIPELINE
CARGO_TOML --> TOCK_REGS
CI_PIPELINE --> AARCH64_BARE
CI_PIPELINE --> DOCS_DEPLOY
CI_PIPELINE --> X86_TEST
RUST_NIGHTLY --> NO_STD
TOCK_REGS --> RUST_NIGHTLY
```

**Sources:** [Cargo.toml(L1 - L19)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/Cargo.toml#L1-L19) [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L1-L56)

### Package Configuration and Metadata

The crate is configured for broad compatibility across the embedded systems ecosystem. Key configuration elements include:

|Configuration|Value|Purpose|
| --- | --- | --- |
|Edition|2021|Modern Rust language features|
|License|Triple-licensed (GPL-3.0-or-later, Apache-2.0, MulanPSL-2.0)|Maximum compatibility|
|Categories|embedded,no-std,hardware-support,os|Clear ecosystem positioning|
|Keywords|arceos,arm,aarch64,gic,interrupt-controller|Discoverability|

**Sources:** [Cargo.toml(L4 - L12)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/Cargo.toml#L4-L12)

## Development Workflow Integration

### Dependency Management Strategy

The crate maintains a minimal dependency footprint with a single external dependency:

```mermaid
flowchart TD
ARM_GICV2["arm_gicv2 crate"]
TOCK_REGS["tock-registers 0.8"]
REG_ABSTRACTION["Hardware Register Abstractions"]
SAFE_MMIO["Safe Memory-Mapped I/O"]
EL2_FEATURE["el2 feature flag"]
HYPERVISOR_SUPPORT["Hypervisor Extensions"]

ARM_GICV2 --> EL2_FEATURE
ARM_GICV2 --> TOCK_REGS
EL2_FEATURE --> HYPERVISOR_SUPPORT
REG_ABSTRACTION --> SAFE_MMIO
TOCK_REGS --> REG_ABSTRACTION
```

The `tock-registers` dependency provides type-safe hardware register access patterns essential for reliable interrupt controller management. The optional `el2` feature enables hypervisor-specific functionality without imposing overhead on standard use cases.

**Sources:** [Cargo.toml(L14 - L18)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/Cargo.toml#L14-L18)

### Build Target Configuration

The development process supports multiple build targets with different purposes:

```mermaid
flowchart TD
subgraph subGraph2["Build Process"]
    CARGO_BUILD["cargo build --target TARGET --all-features"]
    CARGO_TEST["cargo test --target x86_64-unknown-linux-gnu"]
    CARGO_CLIPPY["cargo clippy --target TARGET --all-features"]
end
subgraph subGraph1["Development Target"]
    X86_64["x86_64-unknown-linux-gnu"]
    UNIT_TESTING["Unit testing execution"]
end
subgraph subGraph0["Primary Target"]
    AARCH64["aarch64-unknown-none-softfloat"]
    BARE_METAL_USE["Bare metal ARM64 systems"]
end

AARCH64 --> BARE_METAL_USE
AARCH64 --> CARGO_BUILD
AARCH64 --> CARGO_CLIPPY
```

**Sources:** [.github/workflows/ci.yml(L12 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L12-L30)

## Quality Assurance Process

### Continuous Integration Pipeline

The CI system ensures code quality through multiple validation stages:

|Stage|Tool|Command|Purpose|
| --- | --- | --- | --- |
|Format Check|rustfmt|cargo fmt --all -- --check|Code style consistency|
|Linting|clippy|cargo clippy --target TARGET --all-features|Code quality analysis|
|Build Verification|cargo|cargo build --target TARGET --all-features|Compilation validation|
|Unit Testing|cargo|cargo test --target x86_64-unknown-linux-gnu|Functional verification|

**Sources:** [.github/workflows/ci.yml(L22 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L22-L30)

### Documentation Generation and Deployment

```mermaid
flowchart TD
subgraph subGraph1["Quality Gates"]
    BROKEN_LINKS["rustdoc::broken_intra_doc_links"]
    MISSING_DOCS["missing-docs warnings"]
end
subgraph subGraph0["Documentation Workflow"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
    DOC_FLAGS["RUSTDOCFLAGS validation"]
    INDEX_GEN["index.html generation"]
    GITHUB_PAGES["GitHub Pages deployment"]
end

DOC_BUILD --> DOC_FLAGS
DOC_BUILD --> INDEX_GEN
DOC_FLAGS --> BROKEN_LINKS
DOC_FLAGS --> MISSING_DOCS
INDEX_GEN --> GITHUB_PAGES
```

The documentation pipeline enforces strict quality standards, treating broken links and missing documentation as errors to ensure comprehensive API coverage.

**Sources:** [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L32-L55)

### Platform Compatibility Matrix

The crate supports multiple deployment scenarios with varying feature requirements:

|Use Case|Target Platform|Features Required|Integration Pattern|
| --- | --- | --- | --- |
|Embedded Systems|aarch64-unknown-none-softfloat|None|Direct hardware access|
|Hypervisors|aarch64-unknown-none-softfloat|el2|Extended privilege operations|
|Operating Systems|aarch64-unknown-none-softfloat|Optionalel2|Kernel-level integration|
|Development/Testing|x86_64-unknown-linux-gnu|All|Cross-compilation testing|

This matrix guides developers in selecting appropriate configuration options for their specific integration requirements.

**Sources:** [Cargo.toml(L17 - L18)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/Cargo.toml#L17-L18) [.github/workflows/ci.yml(L12 - L29)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L12-L29)
# Build System and Development Workflow

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.gitignore)

This document covers the automated build system, continuous integration pipeline, code quality assurance processes, and development workflow for the `arm_gicv2` crate. It details the GitHub Actions configuration, build targets, testing strategies, and documentation generation processes that ensure code quality and maintainability.

For information about crate dependencies and feature configuration, see [Crate Configuration and Dependencies](/arceos-hypervisor/arm_gicv2/5.1-crate-configuration-and-dependencies).

## CI/CD Pipeline Overview

The project uses GitHub Actions for continuous integration and deployment, with a comprehensive pipeline that ensures code quality, compatibility, and automated documentation deployment.

### CI/CD Pipeline Architecture

```mermaid
flowchart TD
subgraph DOC_JOB_STEPS["Documentation Job Steps"]
    DOC_CHECKOUT["actions/checkout@v4"]
    DOC_TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
    GH_PAGES["JamesIves/github-pages-deploy-action@v4"]
end
subgraph CI_JOB_STEPS["CI Job Steps"]
    CHECKOUT["actions/checkout@v4"]
    TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    VERSION_CHECK["Check rust version"]
    FORMAT["cargo fmt --all -- --check"]
    CLIPPY["cargo clippy --target aarch64-unknown-none-softfloat"]
    BUILD["cargo build --target aarch64-unknown-none-softfloat"]
    TEST["cargo test --target x86_64-unknown-linux-gnu"]
end
PR["Pull Request"]
GHA["GitHub Actions Trigger"]
PUSH["Push to Main"]
CI_JOB["ci job"]
DOC_JOB["doc job"]

BUILD --> TEST
CHECKOUT --> TOOLCHAIN
CI_JOB --> CHECKOUT
CLIPPY --> BUILD
DOC_BUILD --> GH_PAGES
DOC_CHECKOUT --> DOC_TOOLCHAIN
DOC_JOB --> DOC_CHECKOUT
DOC_TOOLCHAIN --> DOC_BUILD
FORMAT --> CLIPPY
GHA --> CI_JOB
GHA --> DOC_JOB
PR --> GHA
PUSH --> GHA
TOOLCHAIN --> VERSION_CHECK
VERSION_CHECK --> FORMAT
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L1-L56)

### Job Configuration Matrix

The CI pipeline uses a strategy matrix to test against multiple configurations:

|Parameter|Values|
| --- | --- |
|rust-toolchain|nightly|
|targets|aarch64-unknown-none-softfloat|

The pipeline is configured for fail-fast: false, allowing all matrix combinations to complete even if one fails.

Sources: [.github/workflows/ci.yml(L7 - L12)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L7-L12)

## Build Configuration

### Target Platforms

The build system supports multiple target platforms for different use cases:

```mermaid
flowchart TD
subgraph TOOLCHAIN_COMPONENTS["Toolchain Components"]
    RUST_SRC["rust-src"]
    CLIPPY_COMP["clippy"]
    RUSTFMT["rustfmt"]
end
subgraph USE_CASES["Use Cases"]
    EMBEDDED["Embedded Systems"]
    BARE_METAL["Bare Metal Applications"]
    TESTING["Unit Testing"]
end
subgraph BUILD_TARGETS["Build Targets"]
    AARCH64["aarch64-unknown-none-softfloat"]
    X86_TEST["x86_64-unknown-linux-gnu"]
end

AARCH64 --> BARE_METAL
AARCH64 --> EMBEDDED
CLIPPY_COMP --> AARCH64
RUSTFMT --> AARCH64
RUST_SRC --> AARCH64
X86_TEST --> TESTING
```

Sources: [.github/workflows/ci.yml(L12 - L19)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L12-L19)

### Primary Build Target

* **`aarch64-unknown-none-softfloat`**: The primary target for ARM64 bare-metal applications
* Used for clippy linting: `cargo clippy --target aarch64-unknown-none-softfloat`
* Used for building: `cargo build --target aarch64-unknown-none-softfloat`
* Supports all features: `--all-features` flag

### Testing Target

* **`x86_64-unknown-linux-gnu`**: Used exclusively for unit testing
* Conditional execution: Only runs when `matrix.targets == 'x86_64-unknown-linux-gnu'`
* Command: `cargo test --target x86_64-unknown-linux-gnu -- --nocapture`

Sources: [.github/workflows/ci.yml(L25 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L25-L30)

## Code Quality Assurance

The project implements multiple layers of code quality checks to maintain consistency and catch issues early.

### Code Quality Pipeline

```mermaid
sequenceDiagram
    participant Developer as "Developer"
    participant GitRepository as "Git Repository"
    participant CIPipeline as "CI Pipeline"
    participant cargofmt as "cargo fmt"
    participant cargoclippy as "cargo clippy"
    participant cargobuild as "cargo build"

    Developer ->> GitRepository: Push/PR
    GitRepository ->> CIPipeline: Trigger workflow
    CIPipeline ->> cargofmt: Check code format
    cargofmt -->> CIPipeline: Pass/Fail
    CIPipeline ->> cargoclippy: Run linter
    cargoclippy -->> CIPipeline: Pass/Fail (with exceptions)
    CIPipeline ->> cargobuild: Build target
    cargobuild -->> CIPipeline: Pass/Fail
```

Sources: [.github/workflows/ci.yml(L22 - L27)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L22-L27)

### Formatting Standards

* **Tool**: `cargo fmt` with `--all` flag
* **Enforcement**: `--check` flag ensures code is properly formatted
* **Command**: `cargo fmt --all -- --check`
* **Failure Behavior**: Pipeline fails if formatting is inconsistent

### Linting Configuration

* **Tool**: `cargo clippy` with comprehensive checks
* **Target-specific**: Runs against `aarch64-unknown-none-softfloat`
* **Features**: Uses `--all-features` to lint all code paths
* **Exceptions**: Allows `clippy::new_without_default` lint
* **Command**: `cargo clippy --target aarch64-unknown-none-softfloat --all-features -- -A clippy::new_without_default`

Sources: [.github/workflows/ci.yml(L23 - L25)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L23-L25)

## Testing Strategy

### Test Execution Model

```mermaid
flowchart TD
subgraph REASONS["Why x86_64 for Testing"]
    FAST_EXECUTION["Faster execution on CI runners"]
    STANDARD_LIB["Standard library availability"]
    DEBUG_OUTPUT["Better debugging capabilities"]
end
subgraph TEST_CONDITIONS["Test Execution Conditions"]
    TARGET_CHECK["matrix.targets == 'x86_64-unknown-linux-gnu'"]
    UNIT_TESTS["cargo test"]
    NOCAPTURE["--nocapture flag"]
end

DEBUG_OUTPUT --> NOCAPTURE
FAST_EXECUTION --> TARGET_CHECK
STANDARD_LIB --> TARGET_CHECK
TARGET_CHECK --> UNIT_TESTS
UNIT_TESTS --> NOCAPTURE
```

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L28-L30)

### Test Configuration

* **Conditional Execution**: Tests only run for x86_64 target
* **Output Visibility**: `--nocapture` flag ensures test output is visible
* **Target Specification**: Explicit target to avoid cross-compilation issues during testing

## Documentation System

### Documentation Generation Pipeline

The documentation system automatically builds and deploys API documentation to GitHub Pages.

```mermaid
flowchart TD
subgraph DEPLOYMENT["Deployment"]
    GH_PAGES_ACTION["JamesIves/github-pages-deploy-action@v4"]
    SINGLE_COMMIT["single-commit: true"]
    TARGET_FOLDER["folder: target/doc"]
end
subgraph DOC_BUILD["Documentation Build"]
    CARGO_DOC["cargo doc --no-deps --all-features"]
    INDEX_GEN["Generate index.html redirect"]
    RUSTDOC_FLAGS["RUSTDOCFLAGS validation"]
end
subgraph DOC_TRIGGERS["Documentation Triggers"]
    DEFAULT_BRANCH["Push to default branch"]
    PR_EVENT["Pull Request"]
end

CARGO_DOC --> INDEX_GEN
DEFAULT_BRANCH --> CARGO_DOC
GH_PAGES_ACTION --> SINGLE_COMMIT
INDEX_GEN --> RUSTDOC_FLAGS
PR_EVENT --> CARGO_DOC
RUSTDOC_FLAGS --> GH_PAGES_ACTION
SINGLE_COMMIT --> TARGET_FOLDER
```

Sources: [.github/workflows/ci.yml(L32 - L55)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L32-L55)

### Documentation Configuration

#### Build Settings

* **Command**: `cargo doc --no-deps --all-features`
* **Scope**: Excludes dependencies, includes all features
* **Output**: Generated to `target/doc` directory

#### Quality Enforcement

* **Environment Variable**: `RUSTDOCFLAGS="-D rustdoc::broken_intra_doc_links -D missing-docs"`
* **Broken Links**: Treats broken documentation links as errors
* **Missing Documentation**: Enforces documentation completeness

#### Index Generation

```
printf '<meta http-equiv="refresh" content="0;url=%s/index.html">' $(cargo tree | head -1 | cut -d' ' -f1) > target/doc/index.html
```

Sources: [.github/workflows/ci.yml(L40 - L48)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L40-L48)

### Deployment Strategy

* **Trigger**: Only deploys from the default branch (typically `main`)
* **Method**: GitHub Pages deployment action
* **Configuration**:
* `single-commit: true` - Keeps deployment history clean
* `branch: gh-pages` - Deploys to dedicated documentation branch
* `folder: target/doc` - Source folder for deployment

Sources: [.github/workflows/ci.yml(L49 - L55)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L49-L55)

## Development Environment Setup

### Required Toolchain Components

|Component|Purpose|
| --- | --- |
|rust-src|Source code for cross-compilation|
|clippy|Linting and code analysis|
|rustfmt|Code formatting|

### Toolchain Installation

The project uses the `dtolnay/rust-toolchain` action for consistent toolchain management:

```markdown
# Equivalent local setup
rustup toolchain install nightly
rustup component add rust-src clippy rustfmt
rustup target add aarch64-unknown-none-softfloat
```

Sources: [.github/workflows/ci.yml(L15 - L19)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L15-L19)

### Version Verification

The CI pipeline includes a version check step to ensure toolchain consistency:

* **Command**: `rustc --version --verbose`
* **Purpose**: Provides detailed version information for debugging and reproducibility

Sources: [.github/workflows/ci.yml(L20 - L21)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.github/workflows/ci.yml#L20-L21)

### Local Development Workflow

Developers should run these commands locally before pushing:

```markdown
# Format check
cargo fmt --all -- --check

# Linting
cargo clippy --target aarch64-unknown-none-softfloat --all-features -- -A clippy::new_without_default

# Build verification
cargo build --target aarch64-unknown-none-softfloat --all-features

# Documentation build (optional)
cargo doc --no-deps --all-features
```

### Git Configuration

The project uses standard Git ignore patterns for Rust projects:

|Pattern|Purpose|
| --- | --- |
|/target|Build artifacts|
|/.vscode|Editor configuration|
|.DS_Store|macOS system files|
|Cargo.lock|Dependency lock file (library crate)|

Sources: [.gitignore(L1 - L4)&emsp;](https://github.com/arceos-hypervisor/arm_gicv2/blob/eee14941/.gitignore#L1-L4)
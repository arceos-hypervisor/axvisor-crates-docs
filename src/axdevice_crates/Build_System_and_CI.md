# Build System and CI

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml)

This page documents the automated build system and continuous integration (CI) pipeline for the `axdevice_crates` repository. It covers the multi-architecture build strategy, code quality enforcement, testing procedures, and documentation generation workflow that ensures the codebase maintains compatibility across different target platforms in the no_std embedded environment.

For information about implementing new devices within this build system, see [Implementing New Devices](/arceos-hypervisor/axdevice_crates/4.2-implementing-new-devices). For details about the core architecture that this build system validates, see [Core Architecture](/arceos-hypervisor/axdevice_crates/2-core-architecture).

## CI Pipeline Overview

The repository uses GitHub Actions to implement a comprehensive CI pipeline that validates code across multiple architectures and maintains documentation quality. The pipeline consists of two primary jobs: `ci` for code validation and `doc` for documentation generation.

### CI Job Architecture

```mermaid
flowchart TD
subgraph subGraph2["Build Steps"]
    CHECKOUT["actions/checkout@v4"]
    TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    VERSION["rustc --version --verbose"]
    FMT["cargo fmt --all -- --check"]
    CLIPPY["cargo clippy --target TARGET --all-features"]
    BUILD["cargo build --target TARGET --all-features"]
    TEST["cargo test --target x86_64-unknown-linux-gnu"]
end
subgraph subGraph1["CI Job Matrix"]
    RUST["rust-toolchain: nightly"]
    T1["x86_64-unknown-linux-gnu"]
    T2["x86_64-unknown-none"]
    T3["riscv64gc-unknown-none-elf"]
    T4["aarch64-unknown-none-softfloat"]
end
subgraph subGraph0["GitHub Actions Triggers"]
    PUSH["push events"]
    PR["pull_request events"]
end

BUILD --> TEST
CHECKOUT --> TOOLCHAIN
CLIPPY --> BUILD
FMT --> CLIPPY
PR --> RUST
PUSH --> RUST
RUST --> T1
RUST --> T2
RUST --> T3
RUST --> T4
T1 --> CHECKOUT
T2 --> CHECKOUT
T3 --> CHECKOUT
T4 --> CHECKOUT
TOOLCHAIN --> VERSION
VERSION --> FMT
```

Sources: [.github/workflows/ci.yml(L1 - L31)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L1-L31)

### Documentation Job Pipeline

```mermaid
flowchart TD
subgraph subGraph2["Deployment Conditions"]
    MAIN_BRANCH["github.ref == default-branch"]
    GH_PAGES["branch: gh-pages"]
    DOC_FOLDER["folder: target/doc"]
end
subgraph subGraph1["Environment Configuration"]
    RUSTDOCFLAGS["-D rustdoc::broken_intra_doc_links -D missing-docs"]
    DEFAULT_BRANCH["refs/heads/default_branch"]
    PERMISSIONS["contents: write"]
end
subgraph subGraph0["Documentation Generation"]
    DOC_CHECKOUT["actions/checkout@v4"]
    DOC_TOOLCHAIN["dtolnay/rust-toolchain@nightly"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
    INDEX_GEN["Generate target/doc/index.html redirect"]
    PAGES_DEPLOY["JamesIves/github-pages-deploy-action@v4"]
end

DEFAULT_BRANCH --> MAIN_BRANCH
DOC_BUILD --> INDEX_GEN
DOC_CHECKOUT --> DOC_TOOLCHAIN
DOC_TOOLCHAIN --> DOC_BUILD
INDEX_GEN --> MAIN_BRANCH
MAIN_BRANCH --> PAGES_DEPLOY
PAGES_DEPLOY --> DOC_FOLDER
PAGES_DEPLOY --> GH_PAGES
PERMISSIONS --> PAGES_DEPLOY
RUSTDOCFLAGS --> DOC_BUILD
```

Sources: [.github/workflows/ci.yml(L32 - L56)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L32-L56)

## Multi-Architecture Build Strategy

The CI pipeline validates compatibility across four distinct target architectures, each serving different use cases in the hypervisor ecosystem:

|Target|Purpose|Test Coverage|
| --- | --- | --- |
|x86_64-unknown-linux-gnu|Standard Linux development and unit testing|Full (includingcargo test)|
|x86_64-unknown-none|Bare metal x86_64 hypervisor environments|Build and lint only|
|riscv64gc-unknown-none-elf|RISC-V hypervisor platforms|Build and lint only|
|aarch64-unknown-none-softfloat|ARM64 embedded hypervisor systems|Build and lint only|

### Build Matrix Configuration

The build matrix uses a `fail-fast: false` strategy to ensure all target architectures are tested independently, preventing early termination when one target fails.

```mermaid
flowchart TD
subgraph subGraph1["Build Validation Steps"]
    VERSION_CHECK["rustc --version --verbose"]
    FORMAT_CHECK["cargo fmt --all -- --check"]
    LINT_CHECK["cargo clippy --target TARGET --all-features"]
    BUILD_CHECK["cargo build --target TARGET --all-features"]
    UNIT_TEST["cargo test (x86_64-linux only)"]
end
subgraph subGraph0["Rust Toolchain Setup"]
    NIGHTLY["nightly toolchain"]
    COMPONENTS["rust-src, clippy, rustfmt"]
    TARGETS["matrix.targets"]
end

BUILD_CHECK --> UNIT_TEST
COMPONENTS --> FORMAT_CHECK
COMPONENTS --> LINT_CHECK
FORMAT_CHECK --> LINT_CHECK
LINT_CHECK --> BUILD_CHECK
NIGHTLY --> VERSION_CHECK
TARGETS --> BUILD_CHECK
TARGETS --> LINT_CHECK
VERSION_CHECK --> FORMAT_CHECK
```

Sources: [.github/workflows/ci.yml(L8 - L19)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L8-L19) [.github/workflows/ci.yml(L25 - L30)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L25-L30)

## Code Quality Enforcement

The CI pipeline enforces multiple layers of code quality validation before any changes can be merged.

### Formatting and Linting

|Check Type|Command|Purpose|
| --- | --- | --- |
|Code Formatting|cargo fmt --all -- --check|Ensures consistent code style across the codebase|
|Clippy Linting|cargo clippy --target $TARGET --all-features -- -A clippy::new_without_default|Catches common mistakes and enforces Rust best practices|
|Documentation|RUSTDOCFLAGS="-D rustdoc::broken_intra_doc_links -D missing-docs"|Enforces complete documentation with valid links|

The Clippy configuration specifically allows the `clippy::new_without_default` lint, indicating that the codebase may contain `new()` methods without corresponding `Default` implementations, which is common in no_std environments.

Sources: [.github/workflows/ci.yml(L22 - L25)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L22-L25) [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L40-L40)

## Testing Strategy

The testing approach is pragmatic, recognizing the constraints of no_std embedded development:

### Unit Testing Limitations

```mermaid
flowchart TD
subgraph subGraph2["Quality Assurance"]
    CLIPPY_ALL["Clippy runs on all targets"]
    FORMAT_ALL["Format check runs on all targets"]
    COMPILE_ALL["Compilation check on all targets"]
end
subgraph subGraph1["Test Execution"]
    LINUX_ONLY["Unit tests run only on x86_64-unknown-linux-gnu"]
    TEST_CMD["cargo test --target x86_64-unknown-linux-gnu -- --nocapture"]
    BUILD_VALIDATION["Build validation on all targets"]
end
subgraph subGraph0["Testing Constraints"]
    NOSTD["no_std environment"]
    EMBEDDED["Embedded targets"]
    LIMITED["Limited std library access"]
end

BUILD_VALIDATION --> COMPILE_ALL
CLIPPY_ALL --> FORMAT_ALL
COMPILE_ALL --> CLIPPY_ALL
EMBEDDED --> BUILD_VALIDATION
LIMITED --> LINUX_ONLY
LINUX_ONLY --> TEST_CMD
NOSTD --> LINUX_ONLY
```

Unit tests execute only on `x86_64-unknown-linux-gnu` because the other targets are bare metal environments that lack the standard library infrastructure required for Rust's test framework. However, the build validation ensures that the code compiles correctly for all target architectures.

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L28-L30)

## Documentation Generation and Deployment

The documentation system automatically generates and deploys API documentation using a dedicated job that runs in parallel with the main CI validation.

### Documentation Build Process

The documentation generation uses strict flags to ensure high-quality documentation:

```mermaid
flowchart TD
subgraph Deployment["Deployment"]
    BRANCH_CHECK["github.ref == default-branch"]
    DEPLOY_ACTION["JamesIves/github-pages-deploy-action@v4"]
    SINGLE_COMMIT["single-commit: true"]
    GH_PAGES_BRANCH["branch: gh-pages"]
    DOC_FOLDER["folder: target/doc"]
end
subgraph subGraph1["Build Process"]
    DOC_CMD["cargo doc --no-deps --all-features"]
    INDEX_CREATE["printf redirect to generated docs"]
    TREE_PARSE["cargo tree | head -1 | cut -d' ' -f1"]
end
subgraph subGraph0["Documentation Flags"]
    BROKEN_LINKS["-D rustdoc::broken_intra_doc_links"]
    MISSING_DOCS["-D missing-docs"]
    RUSTDOCFLAGS["RUSTDOCFLAGS environment"]
end

BRANCH_CHECK --> DEPLOY_ACTION
BROKEN_LINKS --> RUSTDOCFLAGS
DEPLOY_ACTION --> DOC_FOLDER
DEPLOY_ACTION --> GH_PAGES_BRANCH
DEPLOY_ACTION --> SINGLE_COMMIT
DOC_CMD --> TREE_PARSE
INDEX_CREATE --> BRANCH_CHECK
MISSING_DOCS --> RUSTDOCFLAGS
RUSTDOCFLAGS --> DOC_CMD
TREE_PARSE --> INDEX_CREATE
```

The `--no-deps` flag ensures that only the crate's own documentation is generated, not its dependencies. The automatic index.html generation creates a redirect to the main crate documentation.

Sources: [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L44 - L48)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L44-L48) [.github/workflows/ci.yml(L49 - L55)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L49-L55)

## Development Workflow Integration

The CI system is designed to support the typical development workflow for embedded hypervisor development:

### Branch Protection and Quality Gates

|Stage|Validation|Blocking|
| --- | --- | --- |
|Pull Request|All CI checks must pass|Yes|
|Format Check|cargo fmtvalidation|Yes|
|Lint Check|Clippy on all targets|Yes|
|Build Check|Compilation on all targets|Yes|
|Unit Tests|Tests on Linux target only|Yes|
|Documentation|Doc generation and link validation|Yes for main branch|

### Error Handling Strategy

The documentation job uses conditional error handling with `continue-on-error` set based on branch and event type, allowing documentation builds to fail on non-main branches and non-pull-request events without blocking the overall workflow.

```mermaid
flowchart TD
subgraph subGraph1["Build Outcomes"]
    MAIN_BRANCH_STRICT["Main branch: strict documentation"]
    PR_STRICT["Pull requests: strict documentation"]
    OTHER_LENIENT["Other contexts: lenient documentation"]
end
subgraph subGraph0["Error Handling Logic"]
    BRANCH_CHECK["github.ref != default-branch"]
    EVENT_CHECK["github.event_name != pull_request"]
    CONTINUE_ERROR["continue-on-error: true"]
    STRICT_MODE["continue-on-error: false"]
end

BRANCH_CHECK --> CONTINUE_ERROR
CONTINUE_ERROR --> OTHER_LENIENT
EVENT_CHECK --> CONTINUE_ERROR
STRICT_MODE --> MAIN_BRANCH_STRICT
STRICT_MODE --> PR_STRICT
```

This approach ensures that documentation quality is enforced where it matters most (main branch and pull requests) while allowing experimental branches to have temporary documentation issues.

Sources: [.github/workflows/ci.yml(L45)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L45-L45) [.github/workflows/ci.yml(L39)&emsp;](https://github.com/arceos-hypervisor/axdevice_crates/blob/28d49f14/.github/workflows/ci.yml#L39-L39)
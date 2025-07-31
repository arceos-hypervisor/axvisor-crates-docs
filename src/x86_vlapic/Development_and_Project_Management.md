# Development and Project Management

> **Relevant source files**
> * [.github/workflows/ci.yml](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml)
> * [.gitignore](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.gitignore)

This page documents the development workflow, continuous integration pipeline, build configuration, and testing procedures for the x86_vlapic crate. It covers the automated processes that ensure code quality, build consistency, and documentation deployment.

For information about the core system architecture and components, see [Core Architecture](/arceos-hypervisor/x86_vlapic/2-core-architecture). For details about the register system implementation, see [Register System](/arceos-hypervisor/x86_vlapic/3-register-system).

## CI/CD Pipeline Overview

The x86_vlapic project uses GitHub Actions for continuous integration and deployment. The pipeline is designed to ensure code quality, build consistency across targets, and automated documentation deployment.

### Pipeline Architecture

```mermaid
flowchart TD
subgraph subGraph3["Documentation Pipeline"]
    DOC_BUILD["cargo doc --no-deps --all-features"]
    DOC_INDEX["Generate index.html redirect"]
    DEPLOY["Deploy to gh-pages branch"]
end
subgraph subGraph2["Code Quality Gates"]
    FORMAT["cargo fmt --all --check"]
    CLIPPY["cargo clippy --all-features"]
    BUILD["cargo build --all-features"]
    TEST["cargo test (conditional)"]
end
subgraph subGraph1["CI Job Matrix"]
    MATRIX["Matrix Strategy"]
    RUST_NIGHTLY["rust-toolchain: nightly"]
    TARGET_X86["targets: x86_64-unknown-none"]
end
subgraph subGraph0["Trigger Events"]
    PUSH["Push to Repository"]
    PR["Pull Request"]
end

BUILD --> TEST
CLIPPY --> BUILD
DOC_BUILD --> DOC_INDEX
DOC_INDEX --> DEPLOY
FORMAT --> CLIPPY
MATRIX --> RUST_NIGHTLY
MATRIX --> TARGET_X86
PR --> MATRIX
PUSH --> MATRIX
RUST_NIGHTLY --> DOC_BUILD
RUST_NIGHTLY --> FORMAT
```

Sources: [.github/workflows/ci.yml(L1 - L56)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L1-L56)

### Toolchain Configuration

The project requires specific Rust toolchain configuration to support the bare-metal x86 target:

|Component|Configuration|Purpose|
| --- | --- | --- |
|Toolchain|nightly|Required for no_std features and advanced compiler options|
|Target|x86_64-unknown-none|Bare-metal x86_64 target for hypervisor environments|
|Components|rust-src|Source code for core library compilation|
|Components|clippy|Linting and code analysis|
|Components|rustfmt|Code formatting enforcement|

Sources: [.github/workflows/ci.yml(L11 - L19)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L11-L19)

## Build Process and Quality Checks

### Code Quality Workflow

```mermaid
sequenceDiagram
    participant Developer as "Developer"
    participant Repository as "Repository"
    participant CIPipeline as "CI Pipeline"
    participant GitHubPages as "GitHub Pages"

    Developer ->> Repository: "git push / create PR"
    Repository ->> CIPipeline: "Trigger workflow"
    Note over CIPipeline: Setup Phase
    CIPipeline ->> CIPipeline: "Setup Rust nightly toolchain"
    CIPipeline ->> CIPipeline: "Install components: rust-src, clippy, rustfmt"
    CIPipeline ->> CIPipeline: "Add target: x86_64-unknown-none"
    Note over CIPipeline: Quality Gates
    CIPipeline ->> CIPipeline: "cargo fmt --all --check"
    Note over GitHubPages,CIPipeline: Fails if code not formatted
    CIPipeline ->> CIPipeline: "cargo clippy --target x86_64-unknown-none --all-features"
    Note over GitHubPages,CIPipeline: Allows clippy::new_without_default
    CIPipeline ->> CIPipeline: "cargo build --target x86_64-unknown-none --all-features"
    Note over GitHubPages,CIPipeline: Must build successfully
    alt Target is x86_64-unknown-linux-gnu
        CIPipeline ->> CIPipeline: "cargo test --target x86_64-unknown-linux-gnu"
        Note over GitHubPages,CIPipeline: Unit tests with --nocapture
    end
    Note over CIPipeline: Documentation Pipeline (main branch only)
    CIPipeline ->> CIPipeline: "cargo doc --no-deps --all-features"
    CIPipeline ->> CIPipeline: "Generate index.html redirect"
    CIPipeline ->> GitHubPages: "Deploy to gh-pages branch"
```

Sources: [.github/workflows/ci.yml(L20 - L31)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L20-L31) [.github/workflows/ci.yml(L44 - L55)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L44-L55)

### Linting Configuration

The project uses specific Clippy configuration to balance code quality with practical considerations:

```
cargo clippy --target x86_64-unknown-none --all-features -- -A clippy::new_without_default
```

The `-A clippy::new_without_default` flag allows `new()` methods without requiring `Default` implementation, which is common in device driver patterns.

Sources: [.github/workflows/ci.yml(L25)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L25-L25)

## Documentation Management

### Documentation Build Process

The documentation pipeline includes special configuration for comprehensive API documentation:

|Configuration|Value|Purpose|
| --- | --- | --- |
|RUSTDOCFLAGS|-D rustdoc::broken_intra_doc_links|Fail on broken internal links|
|RUSTDOCFLAGS|-D missing-docs|Require documentation for all public items|
|Build flags|--no-deps --all-features|Generate docs only for this crate with all features|

Sources: [.github/workflows/ci.yml(L40)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L40-L40) [.github/workflows/ci.yml(L47)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L47-L47)

### Documentation Deployment Strategy

```mermaid
flowchart TD
subgraph subGraph2["Deployment Conditions"]
    MAIN_ONLY["Deploy only from main branch"]
    CONTINUE_ERROR["Continue on error for non-main"]
end
subgraph subGraph1["Documentation Actions"]
    BUILD["cargo doc build"]
    REDIRECT["Generate index.html redirect"]
    DEPLOY["Single-commit deployment"]
end
subgraph subGraph0["Branch Strategy"]
    MAIN["main branch"]
    GHPAGES["gh-pages branch"]
    PR["Pull Request"]
end

BUILD --> REDIRECT
DEPLOY --> GHPAGES
MAIN --> BUILD
MAIN_ONLY --> DEPLOY
PR --> BUILD
PR --> CONTINUE_ERROR
REDIRECT --> MAIN_ONLY
```

Sources: [.github/workflows/ci.yml(L45)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L45-L45) [.github/workflows/ci.yml(L50 - L55)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L50-L55)

## Testing Framework

### Test Execution Strategy

The testing approach is target-specific due to the bare-metal nature of the codebase:

```mermaid
flowchart TD
subgraph subGraph2["Test Limitations"]
    NO_STD["no_std environment"]
    BARE_METAL["Bare-metal target constraints"]
    HOST_ONLY["Host-only test execution"]
end
subgraph subGraph1["Test Execution"]
    UNIT_TESTS["cargo test --target x86_64-unknown-linux-gnu"]
    NOCAPTURE["--nocapture flag"]
    CONDITIONAL["Conditional execution"]
end
subgraph subGraph0["Test Targets"]
    X86_NONE["x86_64-unknown-none(Build Only)"]
    X86_LINUX["x86_64-unknown-linux-gnu(Build + Test)"]
end

BARE_METAL --> HOST_ONLY
HOST_ONLY --> CONDITIONAL
UNIT_TESTS --> CONDITIONAL
UNIT_TESTS --> NOCAPTURE
X86_LINUX --> UNIT_TESTS
X86_NONE --> BARE_METAL
```

Sources: [.github/workflows/ci.yml(L28 - L30)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L28-L30)

## Development Workflow

### Recommended Development Process

|Step|Command|Purpose|
| --- | --- | --- |
|1. Format|cargo fmt --all|Ensure consistent code formatting|
|2. Lint|cargo clippy --target x86_64-unknown-none --all-features|Check for common issues|
|3. Build|cargo build --target x86_64-unknown-none --all-features|Verify compilation|
|4. Test|cargo test(on host)|Run unit tests if available|
|5. Document|cargo doc --all-features|Generate and verify documentation|

### Repository Management

The project follows standard Git practices with specific CI integration:

```mermaid
flowchart TD
subgraph subGraph2["Quality Gates"]
    FORMAT_CHECK["Format Check"]
    CLIPPY_CHECK["Clippy Check"]
    BUILD_CHECK["Build Check"]
    MERGE["Merge to main"]
end
subgraph subGraph1["Remote Integration"]
    PUSH["git push"]
    PR["Create PR"]
    CI_CHECK["CI Pipeline"]
end
subgraph subGraph0["Local Development"]
    EDIT["Edit Code"]
    FORMAT["cargo fmt"]
    TEST["cargo test"]
end

BUILD_CHECK --> MERGE
CI_CHECK --> FORMAT_CHECK
CLIPPY_CHECK --> BUILD_CHECK
EDIT --> FORMAT
FORMAT --> TEST
FORMAT_CHECK --> CLIPPY_CHECK
PR --> CI_CHECK
PUSH --> CI_CHECK
TEST --> PUSH
```

Sources: [.github/workflows/ci.yml(L3)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.github/workflows/ci.yml#L3-L3) [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.gitignore#L1-L5)

### Build Artifacts and Exclusions

The `.gitignore` configuration excludes standard Rust build artifacts and development tools:

* `/target` - Cargo build output directory
* `/.vscode` - Visual Studio Code workspace files
* `.DS_Store` - macOS system files
* `Cargo.lock` - Lock file (typically excluded for libraries)

Sources: [.gitignore(L1 - L5)&emsp;](https://github.com/arceos-hypervisor/x86_vlapic/blob/9b85fb9d/.gitignore#L1-L5)

This configuration ensures that only source code and essential project files are tracked in version control, while build artifacts and editor-specific files are excluded.
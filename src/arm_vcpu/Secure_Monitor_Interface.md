# Secure Monitor Interface

> **Relevant source files**
> * [src/smc.rs](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs)

This document covers the Secure Monitor Call (SMC) interface implementation in the ARM vCPU hypervisor. The SMC interface provides a mechanism for the hypervisor to communicate with secure firmware, typically ARM Trusted Firmware (ATF), running at the secure monitor level (EL3). This interface is essential for implementing security features, power management operations, and other privileged system functions that require secure world access.

For information about how SMC exceptions from guest VMs are handled, see the exception handling documentation in [High-Level Exception Handling](/arceos-hypervisor/arm_vcpu/4.3-high-level-exception-handling).

## ARM Security Architecture and SMC Role

The SMC interface operates within ARM's TrustZone security architecture, which divides the system into secure and non-secure worlds across different exception levels.

#### ARM Security Model with SMC Interface

```mermaid
flowchart TD
subgraph subGraph1["Non-Secure World"]
    EL2_NS["EL2 Non-SecureHypervisorarm_vcpu"]
    EL1_NS["EL1 Non-SecureGuest OSVirtual Machines"]
    EL0_NS["EL0 Non-SecureGuest Applications"]
end
subgraph subGraph0["Secure World"]
    EL3_S["EL3 Secure MonitorARM Trusted FirmwareSecure Boot, PSCI"]
    EL1_S["EL1 SecureTrusted OSTEE Services"]
    EL0_S["EL0 SecureTrusted Applications"]
end

EL0_NS --> EL1_NS
EL0_S --> EL1_S
EL1_NS --> EL2_NS
EL1_S --> EL3_S
EL2_NS --> EL3_S
EL3_S --> EL2_NS
```

**Sources:** [src/smc.rs(L1 - L27)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L1-L27)

## SMC Call Implementation

The core SMC functionality is implemented through the `smc_call` function, which provides a direct interface to invoke secure monitor calls from the hypervisor.

#### SMC Call Function Interface

```mermaid
flowchart TD
subgraph subGraph2["Secure Monitor"]
    EL3_HANDLER["EL3 SMC HandlerFunction DispatchService Implementation"]
    RETURN["Return Valuesr0: Result/Statusr1-r3: Return Data"]
end
subgraph subGraph1["smc_call Function"]
    UNSAFE["unsafe fn smc_call(x0, x1, x2, x3)"]
    ASM["Inline Assemblysmc #0inout registers"]
end
subgraph subGraph0["Hypervisor Context"]
    CALLER["Caller FunctionPSCI HandlerSecure Service Request"]
    PARAMS["Parametersx0: Function IDx1-x3: Arguments"]
end

ASM --> EL3_HANDLER
CALLER --> PARAMS
EL3_HANDLER --> RETURN
PARAMS --> UNSAFE
RETURN --> CALLER
UNSAFE --> ASM
```

The `smc_call` function is marked as `unsafe` and `#[inline(never)]` to ensure proper handling of the security-sensitive operation:

|Parameter|Purpose|SMC Calling Convention|
| --- | --- | --- |
|x0|Function identifier|SMC function number per ARM SMC Calling Convention|
|x1-x3|Function arguments|Service-specific parameters|
|Returnr0|Status/result|Success/error code or primary return value|
|Returnr1-r3|Additional data|Service-specific return values|

**Sources:** [src/smc.rs(L3 - L26)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L3-L26)

## SMC Assembly Implementation

The actual secure monitor call is implemented using inline assembly with specific constraints to ensure safe execution.

#### SMC Assembly Execution Flow

```mermaid
flowchart TD
subgraph subGraph0["EL3 Processing"]
    EL3_ENTRY["EL3 SMC VectorContext Save"]
    EL3_DISPATCH["Function DispatchBased on x0"]
    EL3_SERVICE["Service ImplementationPSCI, Secure Services"]
    EL3_RETURN["Prepare Return ValuesLoad r0-r3"]
    EL3_EXIT["Context RestoreWorld Switch to EL2"]
end
ENTRY["Function Entrysmc_call(x0, x1, x2, x3)"]
SETUP["Register SetupLoad input parametersinto x0-x3"]
SMC_INST["Execute SMC #0Secure Monitor CallWorld Switch to EL3"]
CAPTURE["Register CaptureExtract r0-r3from output registers"]
RETURN_TUPLE["Return Tuple(r0, r1, r2, r3)"]

CAPTURE --> RETURN_TUPLE
EL3_DISPATCH --> EL3_SERVICE
EL3_ENTRY --> EL3_DISPATCH
EL3_EXIT --> CAPTURE
EL3_RETURN --> EL3_EXIT
EL3_SERVICE --> EL3_RETURN
ENTRY --> SETUP
SETUP --> SMC_INST
SMC_INST --> EL3_ENTRY
```

The assembly implementation uses specific constraints:

* `inout` constraints for registers `x0-x3` to handle both input and output
* `options(nomem, nostack)` to indicate the SMC instruction doesn't access memory or modify the stack
* `#[inline(never)]` attribute to prevent inlining and ensure the SMC happens at the correct execution context

**Sources:** [src/smc.rs(L15 - L25)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L15-L25)

## Integration with Exception Handling

When guest VMs attempt to execute SMC instructions, they are trapped by the hypervisor and can be handled through the exception system before potentially being forwarded to the secure monitor.

#### Guest SMC Exception Flow

```mermaid
flowchart TD
subgraph subGraph1["SMC Execution Paths"]
    FORWARD_SMC["Forward via smc_callHypervisor → EL3"]
    EMULATE_SMC["Emulate ResponseSynthetic Return Values"]
    DENY_SMC["Deny OperationInject Exception to Guest"]
end
subgraph subGraph0["SMC Decision Logic"]
    POLICY_CHECK["Security Policy CheckFunction ID Validation"]
    FORWARD_DECISION["Forward to EL3?"]
    EMULATE_DECISION["Emulate in EL2?"]
end
GUEST_SMC["Guest VMExecutes SMC"]
TRAP["HCR_EL2.TSC TrapException to EL2"]
EXC_HANDLER["handle_smc64_exceptionException Analysis"]
RETURN_GUEST["Return to GuestWith Results"]

DENY_SMC --> RETURN_GUEST
EMULATE_DECISION --> EMULATE_SMC
EMULATE_SMC --> RETURN_GUEST
EXC_HANDLER --> POLICY_CHECK
FORWARD_DECISION --> DENY_SMC
FORWARD_DECISION --> FORWARD_SMC
FORWARD_SMC --> RETURN_GUEST
GUEST_SMC --> TRAP
POLICY_CHECK --> EMULATE_DECISION
POLICY_CHECK --> FORWARD_DECISION
TRAP --> EXC_HANDLER
```

**Sources:** Based on system architecture patterns referenced in overview diagrams

## Security Considerations and Trust Boundaries

The SMC interface represents a critical trust boundary in the system, requiring careful consideration of security implications.

#### Trust Boundary and Attack Surface

```mermaid
flowchart TD
subgraph subGraph2["Secure Monitor Domain"]
    EL3_MONITOR["ARM Trusted FirmwareEL3 Secure MonitorRoot of Trust"]
    SECURE_SERVICES["Secure ServicesPSCI, Crypto, Boot"]
end
subgraph subGraph1["Hypervisor Trust Domain"]
    HYPERVISOR["arm_vcpu HypervisorEL2 Non-SecureTrusted Compute Base"]
    SMC_FILTER["SMC Policy EngineFunction ID FilteringParameter Validation"]
    SMC_CALL["smc_call FunctionDirect EL3 Interface"]
end
subgraph subGraph0["Untrusted Domain"]
    GUEST_VM["Guest VMsPotentially MaliciousEL1/EL0 Non-Secure"]
    GUEST_SMC["Guest SMC CallsTrapped and Filtered"]
end

EL3_MONITOR --> SECURE_SERVICES
GUEST_SMC --> SMC_FILTER
GUEST_VM --> GUEST_SMC
HYPERVISOR --> SMC_CALL
SMC_CALL --> EL3_MONITOR
SMC_FILTER --> SMC_CALL
```

### Security Properties

|Property|Implementation|Rationale|
| --- | --- | --- |
|Function ID validation|Policy checking before forwarding|Prevents unauthorized access to secure services|
|Parameter sanitization|Input validation in exception handlers|Mitigates parameter injection attacks|
|Direct hypervisor access|smc_callfunction for hypervisor use|Enables privileged operations like PSCI|
|Guest call filtering|Exception trapping and analysis|Maintains security boundaries between VMs|

**Sources:** [src/smc.rs(L6 - L9)&emsp;](https://github.com/arceos-hypervisor/arm_vcpu/blob/4dd7e5df/src/smc.rs#L6-L9) for safety documentation
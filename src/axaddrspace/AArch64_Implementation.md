# AArch64 Implementation

> **Relevant source files**
> * [src/npt/arch/aarch64.rs](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/aarch64.rs)

This document covers the AArch64-specific implementation of nested page tables within the axaddrspace crate. The AArch64 implementation provides VMSAv8-64 Stage 2 translation support for hypervisor environments, enabling guest physical address to host physical address translation on ARM 64-bit architectures.

This implementation specifically handles ARM's hypervisor-mode address translation. For information about other architecture implementations, see [x86_64 Implementation](/arceos-hypervisor/axaddrspace/3.3-x86_64-implementation) and [RISC-V Implementation](/arceos-hypervisor/axaddrspace/3.4-risc-v-implementation). For the generic nested page table interface, see [Architecture Selection](/arceos-hypervisor/axaddrspace/3.1-architecture-selection).

## Architecture Overview

The AArch64 implementation follows ARM's VMSAv8-64 specification for Stage 2 translation, which is used in hypervisor environments to translate Intermediate Physical Addresses (IPA) from guest virtual machines to Host Physical Addresses (HPA).

```mermaid
flowchart TD
subgraph subGraph3["Address Translation"]
    GPA["GuestPhysAddr"]
    HPA["HostPhysAddr"]
    IPARange["40-bit IPA Space"]
    PARange["48-bit PA Space"]
end
subgraph subGraph2["Memory Management"]
    MemType["MemType"]
    MappingFlags["MappingFlags"]
    TLBFlush["TLB Flush Operations"]
end
subgraph subGraph1["Core Components"]
    A64PTEHV["A64PTEHV"]
    A64Meta["A64HVPagingMetaData"]
    DescAttr["DescriptorAttr"]
end
subgraph subGraph0["AArch64 NPT Implementation"]
    NPT["NestedPageTable<H>"]
    A64PT["PageTable64<A64HVPagingMetaData, A64PTEHV, H>"]
end

A64Meta --> GPA
A64Meta --> IPARange
A64Meta --> PARange
A64Meta --> TLBFlush
A64PT --> A64Meta
A64PT --> A64PTEHV
A64PTEHV --> DescAttr
A64PTEHV --> HPA
A64PTEHV --> MappingFlags
DescAttr --> MemType
NPT --> A64PT
```

Sources: [src/npt/arch/aarch64.rs(L1 - L253)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/aarch64.rs#L1-L253)

The implementation provides a 3-level page table structure with:

* 40-bit Intermediate Physical Address (IPA) space for guest addresses
* 48-bit Physical Address (PA) space for host addresses
* VMSAv8-64 Stage 2 translation table format

## Page Table Entry Implementation

The `A64PTEHV` struct represents a single VMSAv8-64 translation table descriptor, implementing the `GenericPTE` trait to integrate with the generic page table framework.

```mermaid
flowchart TD
subgraph subGraph3["Attribute Management"]
    DescriptorAttr["DescriptorAttr"]
    ValidBit["VALID"]
    NonBlock["NON_BLOCK"]
    AccessFlag["AF"]
end
subgraph subGraph2["Physical Address Handling"]
    PhysMask["PHYS_ADDR_MASK0x0000_ffff_ffff_f000"]
    AddrBits["bits 12..48"]
end
subgraph subGraph1["GenericPTE Implementation"]
    NewPage["new_page()"]
    NewTable["new_table()"]
    GetPAddr["paddr()"]
    GetFlags["flags()"]
    SetPAddr["set_paddr()"]
    SetFlags["set_flags()"]
    IsPresent["is_present()"]
    IsHuge["is_huge()"]
end
subgraph subGraph0["A64PTEHV Structure"]
    PTE["A64PTEHV(u64)"]
    PTEBits["Raw 64-bit Value"]
end

DescriptorAttr --> AccessFlag
DescriptorAttr --> NonBlock
DescriptorAttr --> ValidBit
GetFlags --> DescriptorAttr
GetPAddr --> PhysMask
IsHuge --> NonBlock
IsPresent --> ValidBit
NewPage --> DescriptorAttr
NewTable --> DescriptorAttr
PTE --> GetFlags
PTE --> GetPAddr
PTE --> IsHuge
PTE --> IsPresent
PTE --> NewPage
PTE --> NewTable
PTE --> PTEBits
PTE --> SetFlags
PTE --> SetPAddr
PhysMask --> AddrBits
SetFlags --> DescriptorAttr
SetPAddr --> PhysMask
```

Sources: [src/npt/arch/aarch64.rs(L144 - L211)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/aarch64.rs#L144-L211)

### Key Features

|Feature|Implementation|Purpose|
| --- | --- | --- |
|Physical Address Mask|0x0000_ffff_ffff_f000|Extracts bits 12-48 for physical addressing|
|Page vs Block|NON_BLOCKbit|Distinguishes between 4KB pages and large blocks|
|Access Flag|AFbit|Required for all valid entries in ARM architecture|
|Valid Bit|VALIDbit|Indicates if the descriptor is valid for translation|

## Memory Attribute Management

The AArch64 implementation uses the `DescriptorAttr` bitflags structure to manage VMSAv8-64 memory attributes and access permissions.

```mermaid
flowchart TD
subgraph subGraph2["Conversion Logic"]
    ToMapping["to MappingFlags"]
    FromMapping["from MappingFlags"]
    MemTypeConv["Memory Type Conversion"]
end
subgraph subGraph1["Memory Types"]
    Device["Device Memory"]
    Normal["Normal Memory"]
    NormalNC["Normal Non-Cacheable"]
end
subgraph subGraph0["DescriptorAttr Bitflags"]
    DescAttr["DescriptorAttr"]
    ValidFlag["VALID (bit 0)"]
    NonBlockFlag["NON_BLOCK (bit 1)"]
    AttrIndex["ATTR (bits 2-5)"]
    S2AP_RO["S2AP_RO (bit 6)"]
    S2AP_WO["S2AP_WO (bit 7)"]
    Shareable["SHAREABLE (bit 9)"]
    AF["AF (bit 10)"]
    XN["XN (bit 54)"]
end

AttrIndex --> Device
AttrIndex --> Normal
AttrIndex --> NormalNC
DescAttr --> AF
DescAttr --> AttrIndex
DescAttr --> FromMapping
DescAttr --> MemTypeConv
DescAttr --> NonBlockFlag
DescAttr --> S2AP_RO
DescAttr --> S2AP_WO
DescAttr --> Shareable
DescAttr --> ToMapping
DescAttr --> ValidFlag
DescAttr --> XN
Device --> MemTypeConv
Normal --> MemTypeConv
NormalNC --> MemTypeConv
```

Sources: [src/npt/arch/aarch64.rs(L8 - L137)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/aarch64.rs#L8-L137)

### Memory Type Encoding

The implementation supports three memory types with specific attribute encodings:

|Memory Type|Encoding|Characteristics|
| --- | --- | --- |
|Device|0b0000|Device memory with shareability|
|Normal|0b1111|Write-back cacheable with shareability|
|NormalNonCache|0b0111|Inner cacheable, outer non-cacheable|

### Permission Mapping

The AArch64 Stage 2 access permissions are mapped to generic `MappingFlags`:

* **Read**: Controlled by `VALID` bit
* **Write**: Controlled by absence of `S2AP_WO` bit
* **Execute**: Controlled by absence of `XN` bit
* **Device**: Determined by memory attribute index

## Paging Metadata Configuration

The `A64HVPagingMetaData` struct provides architecture-specific configuration for the AArch64 hypervisor page tables.

```mermaid
flowchart TD
subgraph subGraph2["EL2 vs EL1 Operations"]
    EL2Ops["EL2 Operationsvae2is, alle2is"]
    EL1Ops["EL1 Operationsvaae1is, vmalle1"]
    ArmEL2Feature["arm-el2 feature"]
end
subgraph subGraph1["TLB Management"]
    FlushTLB["flush_tlb()"]
    SingleFlush["Single Address Flush"]
    FullFlush["Full TLB Flush"]
end
subgraph A64HVPagingMetaData["A64HVPagingMetaData"]
    Levels["LEVELS = 3"]
    PABits["PA_MAX_BITS = 48"]
    VABits["VA_MAX_BITS = 40"]
    VirtAddr["VirtAddr = GuestPhysAddr"]
end

ArmEL2Feature --> EL1Ops
ArmEL2Feature --> EL2Ops
FlushTLB --> FullFlush
FlushTLB --> SingleFlush
FullFlush --> EL1Ops
FullFlush --> EL2Ops
Levels --> FlushTLB
PABits --> FlushTLB
SingleFlush --> EL1Ops
SingleFlush --> EL2Ops
VABits --> FlushTLB
VirtAddr --> FlushTLB
```

Sources: [src/npt/arch/aarch64.rs(L213 - L253)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/aarch64.rs#L213-L253)

### TLB Flush Implementation

The TLB flush implementation supports both Exception Level 1 (EL1) and Exception Level 2 (EL2) operations:

**EL2 Mode** (with `arm-el2` feature):

* Single address: `tlbi vae2is` - Invalidate by VA in EL2
* Full flush: `tlbi alle2is` - Invalidate all EL2 TLB entries

**EL1 Mode** (default):

* Single address: `tlbi vaae1is` - Invalidate by VA, all ASID in EL1
* Full flush: `tlbi vmalle1` - Invalidate all EL1 TLB entries

All TLB operations include data synchronization barriers (`dsb sy`) and instruction synchronization barriers (`isb`) to ensure proper ordering.

## Integration with Generic Framework

The AArch64 implementation integrates with the generic page table framework through type aliases and trait implementations.

|Component|Type Definition|Purpose|
| --- | --- | --- |
|NestedPageTable<H>|PageTable64<A64HVPagingMetaData, A64PTEHV, H>|Main page table type for AArch64|
|Page Table Entry|A64PTEHV|VMSAv8-64 descriptor implementation|
|Metadata|A64HVPagingMetaData|Architecture-specific configuration|
|Virtual Address|GuestPhysAddr|Guest physical address type|

Sources: [src/npt/arch/aarch64.rs(L252 - L253)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/aarch64.rs#L252-L253)

The implementation ensures compatibility with the generic nested page table interface while providing ARM-specific optimizations for hypervisor workloads, including proper Stage 2 translation semantics and efficient TLB management for virtualized environments.
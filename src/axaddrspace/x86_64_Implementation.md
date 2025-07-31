# x86_64 Implementation

> **Relevant source files**
> * [src/npt/arch/x86_64.rs](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs)

This document covers the Intel x86_64 Extended Page Table (EPT) implementation for nested page tables in the axaddrspace crate. The EPT system provides hardware-assisted virtualization support for Intel VMX environments, enabling efficient guest-to-host physical address translation. For general nested page table concepts and architecture selection, see [Architecture Selection](/arceos-hypervisor/axaddrspace/3.1-architecture-selection). For implementations on other architectures, see [AArch64 Implementation](/arceos-hypervisor/axaddrspace/3.2-aarch64-implementation) and [RISC-V Implementation](/arceos-hypervisor/axaddrspace/3.4-risc-v-implementation).

## EPT Overview and Intel Virtualization Context

The x86_64 implementation leverages Intel's Extended Page Table technology, which is part of the VMX (Virtual Machine Extensions) feature set. EPT provides hardware-accelerated second-level address translation, mapping guest physical addresses to host physical addresses without requiring software intervention for most memory accesses.

**EPT System Architecture**

```mermaid
flowchart TD
subgraph subGraph3["Generic Framework Integration"]
    GenericPTE["GenericPTE trait"]
    PagingMetaData["PagingMetaData trait"]
    MappingFlags["MappingFlags"]
end
subgraph subGraph2["EPT Entry Components"]
    EPTEntry["EPTEntry"]
    EPTFlags["EPTFlags"]
    EPTMemType["EPTMemType"]
end
subgraph subGraph1["axaddrspace EPT Implementation"]
    ExtPageTable["ExtendedPageTable<H>"]
    EPTMeta["ExtendedPageTableMetadata"]
    PageTable64["PageTable64<ExtendedPageTableMetadata, EPTEntry, H>"]
end
subgraph subGraph0["Intel VMX Environment"]
    VMCS["VMCS (VM Control Structure)"]
    EPTP["EPTP (EPT Pointer)"]
    EPT_BASE["EPT Base Address"]
end

EPTEntry --> EPTFlags
EPTEntry --> EPTMemType
EPTEntry --> GenericPTE
EPTFlags --> MappingFlags
EPTMeta --> PagingMetaData
EPTP --> EPT_BASE
EPT_BASE --> ExtPageTable
ExtPageTable --> PageTable64
PageTable64 --> EPTEntry
PageTable64 --> EPTMeta
VMCS --> EPTP
```

Sources: [src/npt/arch/x86_64.rs(L182 - L184)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L182-L184)

## EPT Entry Structure and Flags

The `EPTEntry` structure represents a 64-bit EPT page table entry that follows Intel's EPT specification. Each entry contains physical address bits, access permissions, and memory type information.

**EPT Entry Bit Layout and Components**

```mermaid
flowchart TD
subgraph subGraph2["Memory Type Values"]
    Uncached["Uncached (0)"]
    WriteCombining["WriteCombining (1)"]
    WriteThrough["WriteThrough (4)"]
    WriteProtected["WriteProtected (5)"]
    WriteBack["WriteBack (6)"]
end
subgraph subGraph1["EPTFlags Components"]
    READ["READ (bit 0)Read access permission"]
    WRITE["WRITE (bit 1)Write access permission"]
    EXECUTE["EXECUTE (bit 2)Execute access permission"]
    MEM_TYPE_MASK["MEM_TYPE_MASK (bits 5:3)Memory type specification"]
    IGNORE_PAT["IGNORE_PAT (bit 6)Ignore PAT memory type"]
    HUGE_PAGE["HUGE_PAGE (bit 7)Large page indicator"]
    ACCESSED["ACCESSED (bit 8)Page accessed flag"]
    DIRTY["DIRTY (bit 9)Page dirty flag"]
    EXECUTE_FOR_USER["EXECUTE_FOR_USER (bit 10)User-mode execute access"]
end
subgraph subGraph0["EPTEntry 64-bit Structure"]
    RawBits["u64 raw value (self.0)"]
    PhysAddr["Physical Address [51:12]PHYS_ADDR_MASK: 0x000f_ffff_ffff_f000"]
    FlagsBits["Flags and Properties [11:0]"]
end

FlagsBits --> ACCESSED
FlagsBits --> DIRTY
FlagsBits --> EXECUTE
FlagsBits --> EXECUTE_FOR_USER
FlagsBits --> HUGE_PAGE
FlagsBits --> IGNORE_PAT
FlagsBits --> MEM_TYPE_MASK
FlagsBits --> READ
FlagsBits --> WRITE
MEM_TYPE_MASK --> Uncached
MEM_TYPE_MASK --> WriteBack
MEM_TYPE_MASK --> WriteCombining
MEM_TYPE_MASK --> WriteProtected
MEM_TYPE_MASK --> WriteThrough
RawBits --> FlagsBits
RawBits --> PhysAddr
```

Sources: [src/npt/arch/x86_64.rs(L9 - L32)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L9-L32) [src/npt/arch/x86_64.rs(L34 - L45)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L34-L45) [src/npt/arch/x86_64.rs(L99 - L107)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L99-L107)

### EPT Flags Implementation

The `EPTFlags` structure uses the `bitflags!` macro to provide type-safe manipulation of EPT entry flags. Key access permissions follow Intel's VMX specification:

|Flag|Bit Position|Purpose|
| --- | --- | --- |
|READ|0|Enables read access to the page|
|WRITE|1|Enables write access to the page|
|EXECUTE|2|Enables instruction execution from the page|
|MEM_TYPE_MASK|5:3|Specifies memory type for terminate pages|
|HUGE_PAGE|7|Indicates 2MB or 1GB page size|
|ACCESSED|8|Hardware-set when page is accessed|
|DIRTY|9|Hardware-set when page is written|

Sources: [src/npt/arch/x86_64.rs(L9 - L32)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L9-L32)

## Memory Type Management

EPT supports Intel's memory type system for controlling caching behavior and memory ordering. The `EPTMemType` enum defines the available memory types that can be encoded in EPT entries.

**Memory Type Configuration Flow**

```mermaid
flowchart TD
subgraph subGraph2["EPTFlags Methods"]
    SetMemType["EPTFlags::set_mem_type()"]
    GetMemType["EPTFlags::mem_type()"]
    BitsManip["BitField operationsbits.set_bits(3..6, type)"]
end
subgraph subGraph1["EPT Memory Type Selection"]
    MemTypeCheck["Device memory?"]
    WriteBackType["EPTMemType::WriteBack(normal memory)"]
    UncachedType["EPTMemType::Uncached(device memory)"]
end
subgraph subGraph0["MappingFlags Input"]
    DeviceFlag["MappingFlags::DEVICE"]
    StandardFlags["Standard R/W/X flags"]
end

BitsManip --> GetMemType
DeviceFlag --> MemTypeCheck
MemTypeCheck --> UncachedType
MemTypeCheck --> WriteBackType
SetMemType --> BitsManip
StandardFlags --> MemTypeCheck
UncachedType --> SetMemType
WriteBackType --> SetMemType
```

Sources: [src/npt/arch/x86_64.rs(L47 - L56)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L47-L56) [src/npt/arch/x86_64.rs(L58 - L78)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L58-L78) [src/npt/arch/x86_64.rs(L80 - L97)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L80-L97)

The memory type conversion logic ensures that:

* Normal memory regions use `WriteBack` caching for optimal performance
* Device memory regions use `Uncached` type to prevent caching side effects
* The conversion is bidirectional between `MappingFlags` and `EPTFlags`

## Generic Page Table Framework Integration

The `EPTEntry` implements the `GenericPTE` trait, enabling integration with the architecture-independent page table framework. This abstraction allows the same high-level page table operations to work across different architectures.

**GenericPTE Implementation for EPTEntry**

```mermaid
flowchart TD
subgraph subGraph2["Address Space Integration"]
    HostPhysAddr["HostPhysAddr type"]
    GuestPhysAddr["GuestPhysAddr type"]
    PageTable64Framework["PageTable64<Meta, PTE, H>"]
end
subgraph subGraph1["EPTEntry Implementation Details"]
    PhysAddrMask["PHYS_ADDR_MASK0x000f_ffff_ffff_f000"]
    FlagsConversion["EPTFlags ↔ MappingFlagsbidirectional conversion"]
    PresenceCheck["RWX bits != 0determines presence"]
    HugePageBit["HUGE_PAGE flagindicates large pages"]
end
subgraph subGraph0["GenericPTE Trait Methods"]
    NewPage["new_page(paddr, flags, is_huge)"]
    NewTable["new_table(paddr)"]
    GetPaddr["paddr() → HostPhysAddr"]
    GetFlags["flags() → MappingFlags"]
    SetPaddr["set_paddr(paddr)"]
    SetFlags["set_flags(flags, is_huge)"]
    IsPresent["is_present() → bool"]
    IsHuge["is_huge() → bool"]
    IsUnused["is_unused() → bool"]
    Clear["clear()"]
end

GetFlags --> FlagsConversion
GetPaddr --> HostPhysAddr
IsHuge --> HugePageBit
IsPresent --> PresenceCheck
NewPage --> FlagsConversion
NewPage --> PhysAddrMask
NewTable --> PhysAddrMask
PageTable64Framework --> GetPaddr
PageTable64Framework --> IsPresent
PageTable64Framework --> NewPage
SetFlags --> FlagsConversion
SetFlags --> HugePageBit
SetPaddr --> PhysAddrMask
```

Sources: [src/npt/arch/x86_64.rs(L109 - L154)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L109-L154)

### Key Implementation Details

The `EPTEntry` provides several critical implementation details:

|Method|Implementation|Purpose|
| --- | --- | --- |
|is_present()|self.0 & 0x7 != 0|Checks if any of R/W/X bits are set|
|paddr()|(self.0 & PHYS_ADDR_MASK)|Extracts 40-bit physical address|
|new_table()|Sets R+W+X for intermediate tables|Creates non-leaf EPT entries|
|set_flags()|ConvertsMappingFlagstoEPTFlags|Handles huge page flag setting|

Sources: [src/npt/arch/x86_64.rs(L117 - L149)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L117-L149)

## Extended Page Table Metadata

The `ExtendedPageTableMetadata` structure defines the architectural parameters for Intel EPT, implementing the `PagingMetaData` trait to provide page table configuration to the generic framework.

**EPT Configuration Parameters**

```mermaid
flowchart TD
subgraph subGraph1["Address Type Mapping"]
    VirtAddrType["type VirtAddr = GuestPhysAddr"]
    GuestContext["Guest Physical Address SpaceWhat guests see as 'physical'"]
end
subgraph subGraph0["ExtendedPageTableMetadata Constants"]
    LEVELS["LEVELS = 44-level page table"]
    PA_MAX_BITS["PA_MAX_BITS = 5252-bit physical addresses"]
    VA_MAX_BITS["VA_MAX_BITS = 4848-bit guest addresses"]
end
subgraph subGraph2["TLB Management"]
    FlushTlb["flush_tlb(vaddr)TODO: implement"]
    HardwareFlush["Hardware TLB invalidationINVEPT instruction"]
end

FlushTlb --> HardwareFlush
LEVELS --> VirtAddrType
PA_MAX_BITS --> VirtAddrType
VA_MAX_BITS --> VirtAddrType
VirtAddrType --> GuestContext
```

Sources: [src/npt/arch/x86_64.rs(L167 - L180)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L167-L180)

The metadata configuration aligns with Intel's EPT specification:

* **4-level hierarchy**: EPT4, EPT3, EPT2, EPT1 tables providing 48-bit guest address translation
* **52-bit physical addresses**: Supporting Intel's maximum physical address width
* **Guest physical address context**: EPT translates what guests consider "physical" addresses

## Type Alias and Framework Integration

The final component ties together all EPT-specific implementations with the generic page table framework through a type alias:

```
pub type ExtendedPageTable<H> = PageTable64<ExtendedPageTableMetadata, EPTEntry, H>;
```

This type alias creates the main EPT page table type that:

* Uses `ExtendedPageTableMetadata` for architectural parameters
* Uses `EPTEntry` for page table entries with Intel-specific features
* Accepts a generic hardware abstraction layer `H`
* Inherits all functionality from the `PageTable64` generic implementation

Sources: [src/npt/arch/x86_64.rs(L182 - L184)&emsp;](https://github.com/arceos-hypervisor/axaddrspace/blob/2ed4d076/src/npt/arch/x86_64.rs#L182-L184)

The resulting `ExtendedPageTable<H>` type provides a complete EPT implementation that integrates seamlessly with the broader address space management system while leveraging Intel's hardware virtualization features for optimal performance.
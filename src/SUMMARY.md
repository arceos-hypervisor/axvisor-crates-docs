# AxVisor Crates Book

# arm_vgic

- [Overview](./arm_vgic/Overview.md)

- [System Architecture](./arm_vgic/System_Architecture.md)

- [Core Components](./arm_vgic/Core_Components.md)
    - [Virtual GIC Controller (Vgic)](./arm_vgic/Virtual_GIC_Controller_(Vgic).md)
    - [CPU Interface (Vgicc)](./arm_vgic/CPU_Interface_(Vgicc).md)
    - [Device Operations Interface](./arm_vgic/Device_Operations_Interface.md)
    - [Constants and Register Layout](./arm_vgic/Constants_and_Register_Layout.md)

- [Dependencies and Integration](./arm_vgic/Dependencies_and_Integration.md)
    - [Dependency Analysis](./arm_vgic/Dependency_Analysis.md)
    - [Build Configuration](./arm_vgic/Build_Configuration.md)

# x86_vcpu

- [Overview](./x86_vcpu/Overview.md)

- [VMX Virtualization Engine](./x86_vcpu/VMX_Virtualization_Engine.md)
    - [Virtual CPU Management](./x86_vcpu/Virtual_CPU_Management.md)
    - [VMX Data Structures](./x86_vcpu/VMX_Data_Structures.md)
    - [VMCS Field Management](./x86_vcpu/VMCS_Field_Management.md)
    - [Per-CPU VMX State](./x86_vcpu/Per-CPU_VMX_State.md)

- [Memory Management](./x86_vcpu/Memory_Management.md)
    - [Physical Frame Management](./x86_vcpu/Physical_Frame_Management.md)
    - [Extended Page Tables and Guest Memory](./x86_vcpu/Extended_Page_Tables_and_Guest_Memory.md)

- [Supporting Systems](./x86_vcpu/Supporting_Systems.md)
    - [Register Management](./x86_vcpu/Register_Management.md)
    - [Model-Specific Register Access](./x86_vcpu/Model-Specific_Register_Access.md)
    - [VMX Definitions and Types](./x86_vcpu/VMX_Definitions_and_Types.md)
    - [VMX Instructions](./x86_vcpu/VMX_Instructions.md)

- [Development and Configuration](./x86_vcpu/Development_and_Configuration.md)
    - [Project Configuration](./x86_vcpu/Project_Configuration.md)
    - [Build System and CI](./x86_vcpu/Build_System_and_CI.md)

# axvisor

- [Overview](./axvisor/Overview.md)

- [Architecture](./axvisor/Architecture.md)
    - [System Components](./axvisor/System_Components.md)
    - [VM Management](./axvisor/VM_Management.md)
    - [Hardware Abstraction Layer](./axvisor/Hardware_Abstraction_Layer.md)
    - [Memory Management](./axvisor/Memory_Management.md)

- [Configuration](./axvisor/Configuration.md)
    - [VM Configuration](./axvisor/VM_Configuration.md)
    - [Platform-Specific Configuration](./axvisor/Platform-Specific_Configuration.md)

- [Building and Running](./axvisor/Building_and_Running.md)
    - [Guest VMs](./axvisor/Guest_VMs.md)
    - [Development Environment](./axvisor/Development_Environment.md)

- [Technical Reference](./axvisor/Technical_Reference.md)
    - [VMM Implementation](./axvisor/VMM_Implementation.md)
    - [VCPU Management](./axvisor/VCPU_Management.md)
    - [Timer Subsystem](./axvisor/Timer_Subsystem.md)

- [Contributing](./axvisor/Contributing.md)
    - [Testing Infrastructure](./axvisor/Testing_Infrastructure.md)
    - [CI/CD Pipeline](./axvisor/CI_CD_Pipeline.md)

# arm_vcpu

- [Overview](./arm_vcpu/Overview.md)
    - [System Architecture](./arm_vcpu/System_Architecture.md)
    - [Dependencies and Build System](./arm_vcpu/Dependencies_and_Build_System.md)

- [Virtual CPU Management](./arm_vcpu/Virtual_CPU_Management.md)
    - [VCPU Lifecycle and Operations](./arm_vcpu/VCPU_Lifecycle_and_Operations.md)
    - [Per-CPU State Management](./arm_vcpu/Per-CPU_State_Management.md)

- [Context Switching and State Management](./arm_vcpu/Context_Switching_and_State_Management.md)
    - [TrapFrame and System Registers](./arm_vcpu/TrapFrame_and_System_Registers.md)

- [Exception Handling System](./arm_vcpu/Exception_Handling_System.md)
    - [Assembly Exception Vectors](./arm_vcpu/Assembly_Exception_Vectors.md)
    - [Exception Analysis and Utilities](./arm_vcpu/Exception_Analysis_and_Utilities.md)
    - [High-Level Exception Handling](./arm_vcpu/High-Level_Exception_Handling.md)

- [System Integration](./arm_vcpu/System_Integration.md)
    - [Secure Monitor Interface](./arm_vcpu/Secure_Monitor_Interface.md)
    - [Hardware Abstraction and Platform Support](./arm_vcpu/Hardware_Abstraction_and_Platform_Support.md)

# axvcpu

- [Overview](./axvcpu/Overview.md)

- [Core VCPU Management](./axvcpu/Core_VCPU_Management.md)
    - [VCPU State Machine and Lifecycle](./axvcpu/VCPU_State_Machine_and_Lifecycle.md)
    - [Architecture Abstraction Layer](./axvcpu/Architecture_Abstraction_Layer.md)

- [Exit Handling System](./axvcpu/Exit_Handling_System.md)
    - [Exit Reasons and Categories](./axvcpu/Exit_Reasons_and_Categories.md)
    - [Memory Access and I/O Operations](./axvcpu/Memory_Access_and_I_O_Operations.md)

- [Implementation Details](./axvcpu/Implementation_Details.md)
    - [Per-CPU Virtualization State](./axvcpu/Per-CPU_Virtualization_State.md)
    - [Hardware Abstraction Layer](./axvcpu/Hardware_Abstraction_Layer.md)

- [Development and Build System](./axvcpu/Development_and_Build_System.md)
    - [Dependencies and Integration](./axvcpu/Dependencies_and_Integration.md)
    - [Testing and Continuous Integration](./axvcpu/Testing_and_Continuous_Integration.md)

- [Licensing and Legal](./axvcpu/Licensing_and_Legal.md)

# axdevice

- [Overview](./axdevice/Overview.md)

- [System Architecture](./axdevice/System_Architecture.md)

- [Core Components](./axdevice/Core_Components.md)
    - [Configuration Management](./axdevice/Configuration_Management.md)
    - [Device Emulation](./axdevice/Device_Emulation.md)

- [ArceOS Ecosystem Integration](./axdevice/ArceOS_Ecosystem_Integration.md)

- [Development Guide](./axdevice/Development_Guide.md)

# axdevice_crates

- [Overview](./axdevice_crates/Overview.md)
    - [Project Structure](./axdevice_crates/Project_Structure.md)

- [Core Architecture](./axdevice_crates/Core_Architecture.md)
    - [BaseDeviceOps Trait](./axdevice_crates/BaseDeviceOps_Trait.md)
    - [Device Type System](./axdevice_crates/Device_Type_System.md)
    - [Address Space Management](./axdevice_crates/Address_Space_Management.md)

- [Dependencies and Integration](./axdevice_crates/Dependencies_and_Integration.md)
    - [ArceOS Integration](./axdevice_crates/ArceOS_Integration.md)

- [Development Guide](./axdevice_crates/Development_Guide.md)
    - [Build System and CI](./axdevice_crates/Build_System_and_CI.md)
    - [Implementing New Devices](./axdevice_crates/Implementing_New_Devices.md)

# arm_gicv2

- [Overview](./arm_gicv2/Overview.md)

- [Core Architecture](./arm_gicv2/Core_Architecture.md)
    - [GIC Distributor Component](./arm_gicv2/GIC_Distributor_Component.md)
    - [CPU Interface Component](./arm_gicv2/CPU_Interface_Component.md)
    - [Interrupt Processing Pipeline](./arm_gicv2/Interrupt_Processing_Pipeline.md)

- [Interrupt Types and Management](./arm_gicv2/Interrupt_Types_and_Management.md)
    - [Interrupt Classification System](./arm_gicv2/Interrupt_Classification_System.md)
    - [Software Generated Interrupts](./arm_gicv2/Software_Generated_Interrupts.md)

- [Register Interface](./arm_gicv2/Register_Interface.md)
    - [Register Module Organization](./arm_gicv2/Register_Module_Organization.md)
    - [GICD_SGIR Register Details](./arm_gicv2/GICD_SGIR_Register_Details.md)

- [Development and Integration](./arm_gicv2/Development_and_Integration.md)
    - [Crate Configuration and Dependencies](./arm_gicv2/Crate_Configuration_and_Dependencies.md)
    - [Build System and Development Workflow](./arm_gicv2/Build_System_and_Development_Workflow.md)

# axaddrspace

- [Overview](./axaddrspace/Overview.md)

- [Core Architecture](./axaddrspace/Core_Architecture.md)
    - [Address Types and Spaces](./axaddrspace/Address_Types_and_Spaces.md)
    - [Address Space Management](./axaddrspace/Address_Space_Management.md)
    - [Hardware Abstraction Layer](./axaddrspace/Hardware_Abstraction_Layer.md)

- [Nested Page Tables](./axaddrspace/Nested_Page_Tables.md)
    - [Architecture Selection](./axaddrspace/Architecture_Selection.md)
    - [AArch64 Implementation](./axaddrspace/AArch64_Implementation.md)
    - [x86_64 Implementation](./axaddrspace/x86_64_Implementation.md)
    - [RISC-V Implementation](./axaddrspace/RISC-V_Implementation.md)

- [Memory Mapping Backends](./axaddrspace/Memory_Mapping_Backends.md)
    - [Linear Backend](./axaddrspace/Linear_Backend.md)
    - [Allocation Backend](./axaddrspace/Allocation_Backend.md)

- [Device Support](./axaddrspace/Device_Support.md)

- [Development Guide](./axaddrspace/Development_Guide.md)

# x86_vlapic

- [Overview](./x86_vlapic/Overview.md)

- [Core Architecture](./x86_vlapic/Core_Architecture.md)
    - [EmulatedLocalApic Device Interface](./x86_vlapic/EmulatedLocalApic_Device_Interface.md)
    - [Virtual Register Management](./x86_vlapic/Virtual_Register_Management.md)
    - [Register Address Translation](./x86_vlapic/Register_Address_Translation.md)

- [Register System](./x86_vlapic/Register_System.md)
    - [Register Constants and Offsets](./x86_vlapic/Register_Constants_and_Offsets.md)
    - [Local Vector Table (LVT)](./x86_vlapic/Local_Vector_Table_(LVT).md)
        - [Timer LVT Register](./x86_vlapic/Timer_LVT_Register.md)
        - [External Interrupt Pin Registers](./x86_vlapic/External_Interrupt_Pin_Registers.md)
        - [System Monitoring LVT Registers](./x86_vlapic/System_Monitoring_LVT_Registers.md)
        - [Error Handling LVT Registers](./x86_vlapic/Error_Handling_LVT_Registers.md)
    - [Control and Status Registers](./x86_vlapic/Control_and_Status_Registers.md)

- [Development and Project Management](./x86_vlapic/Development_and_Project_Management.md)

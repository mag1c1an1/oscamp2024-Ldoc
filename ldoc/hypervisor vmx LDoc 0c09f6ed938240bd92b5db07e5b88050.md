# hypervisor vmx LDoc

# VT-x

CPU可以通过 VMXON/VMXOFF 指令打开或关闭 VMX 操作模式。VMX 操作模式包含根模式（ root ）与非根（ non-root ）模式，其中 Hypervisor 运行在根模式，虚拟机则运行在非根模式.

CPU 从根模式切换为非根模式称为 VM-Entry，从非根模式切换为根模式则称为 VM-Exit。值得注意的是，根模式与非根模式都有各自的特权级 （ Ring0~Ring3 ），虚拟机操作系统和应用程序分别运行在非根模式的 Ring0 和 Ring3 特权级中，而 Hypervisor 通常运行在根模式的 Ring0 特权级

Intel VT-x 改变了非根模式下敏感非特权指令的语义，使它们能够触发 VM-Exit，而根模式指令的语义保持不变。处于非根模式的虚拟机执行敏感指令将会触发 VM-Exit，陷入 Hypervisor 进行处理。Hypervisor 读取 VM-Exit 相关信息，造成 VM-Exit 的原因（如I/O指令触发、外部中断触发等）并进行相应处理。处理完成后，Hypervisor 调用 VMLAUNCH/VMRESUME 指令从根模式切换到非根模式，恢复虚拟机运行。

![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled.png)

## VMCS

Intel VT-x 引入了 VMCS 。VMCS是内存中的一块区域，用于在 VM-Entry 和 VM-Exit 过程中保存和加载 Hypervisor 和虚拟机的寄存器状态。此外，VMCS 还包含一些控制域，用于控制 CPU 的行为

VMCS 与 vCPU 一一对应，每个 vCPU 都拥有一个 VMCS。当 vCPU 被调度到物理 CPU 上运行时，首先要将其 VMCS 与物理 CPU 绑定，物理 CPU 才能在 VM-Entry/VM-Exit 过程中将 vCPU 的寄存器状态保存到 VMCS 中。Intel VT-x提供了两条指令，分别用于绑定 VMCS 和解除 VMCS 绑定：

(1)VMPTRLD <VMCS地址>：将指定 VMCS 与当前CPU绑定。

(2)VMCLEAR <VMCS地址>：同样以 VMCS 地址为操作数，将 VMCS 与当前 CPU 解除绑定，该指令确保 CPU 缓存中的 VMCS 数据被写入内存中。

在发生 vCPU 迁移时，需要先在原物理 CPU 上执行 VMCLEAR 指令，而后在目的物理 CPU 上执行VMPTRLD 指令。此外，VMCS 虽然是内存区域，但是英特尔软件开发手册指出通过读写内存的方式读写 VMCS 数据是不可靠的，因为 VMCS 数据域格式与架构实现是相关的，而且部分 VMCS 数据可能位于CPU缓存中，尚未同步到内存中. Intel VT-x 提供 VMREAD 与 VMWRITE 指令用于读写VMCS数据域，格式如下：

(1)VMREAD <索引>：读取索引指定的VMCS数据域。

(2)VMWRITE <索引><数据>：将数据写入索引指定的VMCS数据域。

VMCS区域大小不固定，最多占用4KB，VMCS的具体大小可以通过查询MSR（Model Specific Register，特殊模块寄存器）IA32_VMX_BASIC[32∶44]得知

**vm-entry/vm-exit 过程中 VMCS 作用**

保存 guest 状态 → 加载 host 状态 → hypervisor 获取退出原因 → 保存 host 状态  → 加载 guest 状态

![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%201.png)

## 内存虚拟化-EPT

EPT表项的构成较为简单，其第0、1、2位分别表示了客户机物理页的可读、可写、可执行权限，并包含指向下一级页表页的指针 （ HPA ）。当EPTP（ 存在于VMCS中 ）的第6位为1时，会使能EPT的 A/D（ Accessed/Dirty，访问/脏 ）位。EPT中的 A/D 位和前文所述的进程页表的 A/D 位类似，D位仅存在于第四级页表项，即 PTE 中。A/D 位由处理器硬件置 1，由 Hypervisor 软件置 0。每当EPT 被使用时，对应的 EPT 表项的 A 位被处理器置 1；当客户机物理内存被写入时，对应的 EPT 表项的D位被置 1。需要注意的是，**对客户机页表 GPT 的任何访问均被视为写**，GPT 的页表页对应 EPT 表项的 D 位均被处理器硬件置1。该硬件特性将在本章多处提及，可用于实现一些软件功能，也可选择关闭该特性，例如可以用此硬件特性实现虚拟机热迁移。

![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%202.png)

和客户机操作系统中的GPT类似，EPT也是在缺页异常中由Hypervisor软件建立的。当刚启动的客户机中的某进程访问了一个虚拟地址，由于此时该进程的一级页表（GPT中的PML4）为空，故触发客户机操作系统中的缺页异常（步骤1）。客户机操作系统为了分配GPT对应的客户机物理页，需要查询EPT。此时，由于EPT尚未建立，客户机操作系统就退出到了Hypervisor（步骤2）。当客户机操作系统访问了一个缺失的EPT页表项，处理器产生EPT违例(EPT Violation)的VM-Exit，从而Hypervisor分配宿主机内存、建立EPT表项（步骤3）。触发EPT违例的详细原因会被硬件记录在VMCS的VM-Exit条件(VM-Exit Qualification)字段，供Hypervisor使用。宿主机操作系统完成宿主机物理页分配，建立对应的EPT表项，将返回到客户机操作系统（步骤3）。客户机操作系统继续访问GPT的下一级页表(PDPT)，重复步骤1、2，GPT和EPT的建立方可完成。这里又出现了软硬件的明确分工：软件维护页表，硬件查询页表。如果访问了EPT中的一个配置错误，不符合Intel规范的表项,处理器会触发EPT配置错误(EPT Misconfiguration)的VM-Exit。例如访问了一个不可读但可写的表项，此时硬件将不会记录发生EPT配置错误的原因，这类VM-Exit被Hypervisor用于模拟MMIO。

当处在非根模式下的CPU访问了一个GVA，MMU将首先查询GPT。gptr包含客户机页表的起始地址GPA，这会触发MMU交叉地查询EPT，将gptr中包含的GPA翻译成HPA，从而获取GPT的第一级表项。同理，为了获取GPT中每个层级的页表项，MMU都会查询EPT。在64位的系统中，Hypervisor使用GPA的低48位查询EPT页表，而EPT页表也使用了9+9+9+9的四级页表的形式。假设GPT也是四级页表，那么非根模式下的CPU为了读取一个GVA处的数据，如图3-6所示，需要额外读取24个页表项（图中加粗黑框的灰色长方形），因此需要额外的24次内存访问。即使MMU中有TLB缓存GVA到HPA的映射，TLB也无法覆盖越来越大的客户机虚拟地址空间，双层页表的查询将会造成巨大的开销。而在TLB未命中时，一个四级影子页表仅需访问4次内存即可得到HPA，相比于双层页表有巨大的优势。

## 中断虚拟化

通过虚拟机下陷将物理中断转换为虚拟中断，解决物理中断的路由问题。在此过程中，Hypervisor通过虚拟中断控制器模拟真实环境下中断的分发和传送，完成虚拟中断的注入.

当Hypervisor需要向正在运行的vCPU注入中断时，需要给vCPU发送一个信号，使其触发VM-Exit，从而在VM-Entry时注入中断。如果vCPU正处于屏蔽外部中断的状态，如vCPU的RFLAGS.IF=0，将不允许在VM-Entry时进行中断注入。此时可以将VM-Execution控制域中的中断窗口退出(Interrupt-Window Exiting)字段置为1，这样一旦vCPU进入能够接收中断的状态，便会产生一个VM-Exit，Hypervisor就可以注入刚才无法注入的中断，并将中断窗口退出字段置为0

## hypercraft exercises

1. 阅读[Intel SDM Vol. 3C](https://cdrdv2.intel.com/v1/dl/getContent/671447), Chapter 24: Introduction to Virtual Machine Extensions（共4页）
2. 阅读代码，描述在使能 VMX 的过程中 `vmx_region` 是如何分配和初始化的。（使能 VMX 的过程在 `crates/hypercraft/src/arch/x86_64/vmx/percpu.rs` 中的 `VmxPerCpuState::<H>::hardware_enable` 函数）
- Before executing VMXON, software should write the VMCS revision identifier (see Section 25.2) to the VMXON region. (Specifically, it should write the 31-bit VMCS revision identifier to bits 30:0 of the first 4 bytes of the VMXON region; bit 31 should be cleared to 0.) It need not initialize the VMXON region in any other way. Software should use a separate region for each logical processor and should not access or modify the VMXON region of a logical processor between execution of VMXON and VMXOFF on that logical processor. Doing otherwise may lead to unpredictable behavior (including behaviors identified in Section 25.11.1).
    
    ![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%203.png)
    
1. 阅读 Intel SDM Vol. 3C, Chapter 25: Virtual-Machine Control Structures 相关小节，回答以下问题：
    1. 如果让要 hypervisor 实现以下功能，应该如何配置 VMCS？
        1. 拦截 Guest `HLT` 指令
        2. 拦截 Guest `PAUSE` 指令
        3. 拦截外部设备产生的中断，而不是直通给 Guest
        4. 打开或关闭 Guest 的中断
        5. 拦截 Guest 缺页异常 (#PF)
        6. 拦截所有 Guest I/O 指令 (x86 `IN`/`OUT`/`INS`/`OUTS` 等)
        7. 只拦截 Guest 对串口的 I/O 读写 (I/O 端口为 `0x3f8`)
        8. 拦截所有 Guest MSR 读写
        9. 只拦截 Guest 对 `IA32_EFER` MSR 的写入
        10. 只拦截 Guest 对 `CR0` 控制寄存器 `PG` 位 (31 位) 的写入
    2. 如果要在单核 hypervisor 中交替运行两个 vCPU，应该如何操作 VMCS？
    - **Pin-Based VM-Execution Controls**
        
        external-interrupt-exiting: 0
        
        **Processor-Based VM-Execution Controls** 
        
        hlt: 7 
        
        pause: 30 
        
        uncondition i/o exiting: 24
        
        use i/o bimaps  25
        
        use msr bitmaps: 28
        
        **Secondary Processor-Based VM-Execution Controls**
        
        Virtual-interrupt delivery: 9
        
        直接修改 vmcs 中的 guest rflags
        
        **Execption Bitmap**
        
        The exception bitmap is a 32-bit field that contains one bit for each exception. When an exception occurs, its vector is used to select a bit in this field. If the bit is 1, the exception causes a VM exit. If the bit is 0, the exception is delivered normally through the IDT, using the descriptor corresponding to the exception’s vector.
        Whether a page fault (exception with vector 14) causes a VM exit is determined by bit 14 in the exception bitmap as well as the error code produced by the page fault and two 32-bit fields in the VMCS (the page-fault error-code mask and page-fault error-code match). See Section 26.2 for details.
        
        **i/o bitmap addresses**
        
        The VM-execution control fields include the 64-bit physical addresses of I/O bitmaps A and B (each of which are 4K Bytes in size). I/O bitmap A contains one bit for each I/O port in the range 0000H through 7FFFH; I/O bitmap B contains bits for ports in the range 8000H through FFFFH.
        A logical processor uses these bitmaps if and only if the “use I/O bitmaps” control is 1. If the bitmaps are used, execution of an I/O instruction causes a VM exit if any bit in the I/O bitmaps corresponding to a port it accesses is 1. See Section 26.1.3 for details. If the bitmaps are used, their addresses must be 4-KByte aligned.
        
        **msr-bitmap address**
        
        On processors that support the 1-setting of the “use MSR bitmaps” VM-execution control, the VM-execution control fields include the 64-bit physical address of four contiguous MSR bitmaps, which are each 1-KByte in size. This field does not exist on processors that do not support the 1-setting of that control.
        
        **Guest/Host Masks and Read Shadows for CR0 and CR4**
        
        VM-execution control fields include guest/host masks and read shadows for the CR0 and CR4 registers. These
        fields control executions of instructions that access those registers (including CLTS, LMSW, MOV CR, and SMSW). They are 64 bits on processors that support Intel 64 architecture and 32 bits on processors that do not.
        In general, bits set to 1 in a guest/host mask correspond to bits “owned” by the host:
        
        - Guest attempts to set them (using CLTS, LMSW, or MOV to CR) to values differing from the corresponding bits in the corresponding read shadow cause VM exits.
        - Guest reads (using MOV from CR or SMSW) return values for these bits from the corresponding read shadow.
        
        Bits cleared to 0 correspond to bits “owned” by the guest; guest attempts to modify them succeed and guest reads return values for these bits from the control register itself.
        
        ans:  首先要给每一个 vCPU 创建并配置一个 `VMCS`。在切换 vCPU 时，要把目标 vCPU 的 `VMCS` 设置为“当前 `VMCS`”，并切换其他不在 `VMCS` 中的 Guest 状态。
        
    1. 阅读代码，详细阐述：
        1. 在 VM-entry 和 VM-exit 的过程中，Host 的 `RSP` 寄存器的值是如何变化的？包括：哪些指令
        2. 在 VM-entry 和 VM-exit 的过程中，Guest 的通用寄存器的值是如何保存又如何恢复的？（提示：与`RSP`的修改有关）
        3. VM-exit 过程中是如何确保调用 `vmexit_handler` 函数时栈是可用的？
        
        vmcs 中是可以存一个 host rsp 这个就是用来vm-exit 退出的时候控制执行流。
        
        在 vm-launch 之前先通过 vmwrite 在 vmcs 设置 rsp 为 self.host_stack_top的地址
        
        进入 vmx_launch 将当前执行流的 rsp 存入 self.host_stack_top 之后将 rsp 设置为vcpu 的开始，这样就可以通过pop 回复寄存器 但是不回复 rsp 所有有 add rsp 8 之类的指令，
        
        vm-exit 时 rsp 被设置为了 host_stack_top 将现在 guest 的寄存器保存，将 host_stack_top 回复成 rsp 这是之前 vm-launch 执行流的 rsp 根据调用规范 call handler, 返回之后重新设置 rsp 回复 guest regs 然后 vmresume
        
    2. 假设一个 Hypervisor 中运行有 4 个 Guest VM，每个 Guest VM 中运行有 10 个应用 (需要 10 个 Guest 页表)，请问在用影子分页方式实现内存虚拟化时，Hypervisor 共需维护多少份影子页表？
    3. 假设在 Guest OS 中启用了 4 级页表，Guest 的一次访存 (使用 Guest 虚拟地址) 会导致多少次内存访问？（使用 4 级嵌套页表实现内存虚拟化；假设 TLB 全部失效；假设不出现缺页或 EPT Violation）  
    - guest 访问 5 次内存 每次访问真实的物理 5 次
        
        
    1. 简述：如果要改成按需分配内存，应该在代码上作出哪些修改？
    - os 处理 pf 一样， 但是换成了处理 ept-violation
    1. 结合之前学习到的知识，解释 Guest BIOS 核心代码中 `prot_gdt` 和 `prot_gdt_desc` 都是什么内容。
    2. 修改代码，使分配给 Guest OS 的内存容量从 16 MB 增加到 32 MB。
    3. 简述：如果要使 NimbOS OS 被加载的地址从`0x20_0000`更改到其他地址，需要做哪些修改？
    
    1. prot_gdt 是一个有两个有效项的全局描述符表，其中定义了一个代码段和一个数据段，均为4GB大小的对等映射。prot_gdt_desc 是指向这个全局描述符表的描述符，记录了其地址和大小，用作 lgdt 指令的参数；
    2. 修改 `GUEST_PHYS_MEMORY_SIZE` 为 `0x200_0000`；
    3. 修改 `GUEST_ENTRY` 以让 NimbOS 内核被复制到其他地址；同时修改 NimbOS BIOS 中的跳转地址。
    
    1. 编码实现一个新的虚拟PMIO设备，端口号自定：
        1. 实现其功能：Guest 写数据时打印一行日志，Guest 读数据时返回一个常数；
        2. 编码在 Guest BIOS 中使用 `IN`/`OUT` 指令调用这个设备。
    2. *编码实现一个新的虚拟PMIO设备，端口号自定
        1. *实现其功能：Guest 读数据时，返回 Guest 最后一次写入的数据；若 Guest 没有写过数据，返回0。
        2. *简述：为了让虚拟设备能够保存内部状态，代码要做哪些修改？
        
        ![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%204.png)
        
        ![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%205.png)
        
        ![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%206.png)
        
        ![Untitled](hypervisor%20vmx%20LDoc%200c09f6ed938240bd92b5db07e5b88050/Untitled%207.png)
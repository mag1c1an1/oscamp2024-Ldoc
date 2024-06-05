# hv x86 multicore

ArceOS 是支持 SMP 的

具体见 multiboot.S 中的 ap_entry32, ap_entry64

根据初始化协议 ap 收到 start-up INI 之后，ap 会开始启动，不考虑 multitask feature 最后会在 wait_irq 函数中

1. 【基础】基于ArceOS，完成其中一个架构（aarch64, x86, riscv）hypervisor支持多核心客户机的运行，要求代码可扩展性强（如果有余力，鼓励完成多个架构的实现）。此处的实现令一个物理CPU只与一个VCpu绑定，即一一对应，其上只需运行单个客户机。例如：4个物理CPU，每个CPU分别对应1个VCpu，共4个VCpu。单个客户机使用这4个VCpu。

思路： 到 hv 的 main 的时候其实 ap 已经初始化结束了，配置 vmcs 拦截 start ini ， host 发起 ini 给其他核并且解析 startup-ini 获得 ap entry address ，ap 创建 vcpu 进行运行。

目前 nimbos 不支持 smp 

尝试用 Arceos 作为 guest 或者 直接将 bios 换成 seabios 但后者太复杂

将 Arceos 作为 guest 运行之后 hypervisor 会 hang on

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled.png)

根据 qemu monitor 查看 $pc  和 寄存器

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%201.png)

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%202.png)

rax 中的值好像是一个地址

将 guest 反汇编之后 

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%203.png)

发现可能是页表不正确 

但是为什么这个不触发 vmexit 呢？或者是触发了我没注意到？

对比 nimbOS 和 ArceOS 的 multiboot.S 发现 确实 ArceOS 的 tmp PML4 多了些映射

 

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%204.png)

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%205.png)

2. 【进阶】基于【基础】的实现，令一个物理CPU与多个VCpu绑定，其上仍然只需运行单个客户机。一个物理CPU与多个VCpu绑定会涉及到调度算法以及上下文的切换，可以参考现有常用的调度算法（CFS、FIFO、RR）进行实现，或对其进行改进。例如：2个物理CPU，每个CPU分别对应2个VCpu，共4个VCpu。单个客户机使用这4个VCpu。

参考 sdm 卷三 31？
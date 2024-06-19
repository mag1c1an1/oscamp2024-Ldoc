# hv x86 multicore

guest 物理内存布局

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled.png)

ArceOS 是支持 SMP 的

具体见 multiboot.S 中的 ap_entry32, ap_entry64

根据初始化协议 ap 收到 start-up INI 之后，ap 会开始启动，不考虑 multitask feature 最后会在 wait_irq 函数中

1. 【基础】基于ArceOS，完成其中一个架构（aarch64, x86, riscv）hypervisor支持多核心客户机的运行，要求代码可扩展性强（如果有余力，鼓励完成多个架构的实现）。此处的实现令一个物理CPU只与一个VCpu绑定，即一一对应，其上只需运行单个客户机。例如：4个物理CPU，每个CPU分别对应1个VCpu，共4个VCpu。单个客户机使用这4个VCpu。

2024/06/06

思路： 到 hv 的 main 的时候其实 ap 已经初始化结束了，配置 vmcs 拦截 start ini ， host 发起 ini 给其他核并且解析 startup-ini 获得 ap entry address ，ap 创建 vcpu 进行运行。

目前 nimbos 不支持 smp 

尝试用 Arceos 作为 guest 或者 直接将 bios 换成 seabios 但后者太复杂

将 Arceos 作为 guest 运行之后 hypervisor 会 hang on

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%201.png)

根据 qemu monitor 查看 $pc  和 寄存器

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%202.png)

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%203.png)

rax 中的值好像是一个地址

将 guest 反汇编之后 

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%204.png)

发现可能是页表不正确 

但是为什么这个不触发 vmexit 呢？或者是触发了我没注意到？

对比 nimbOS 和 ArceOS 的 multiboot.S 发现 确实 ArceOS 的 tmp PML4 多了些映射

 

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%205.png)

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%206.png)

2024/06/13

配置了 pc-x86-hv-guest.toml 主要是限制了 内存大小

非常感谢 beta 老哥提供的 gdb 支持 [https://github.com/bet4it/arceos/wiki/使用方法](https://github.com/bet4it/arceos/wiki/%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95)

支持 hv 上直接运行  arceos hello-world 

通过 gdb 发现 之前 hang on 的原因是 nimbos bios 的 multiboot magic 是 0x1BADB002

但是 arceos os 中 定义的是 0x2BADB002（ 大概是 multiboot 1 和 2 的区别）

导致 进入 rust_entry 时 check 失败 直接 hlt 了     

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%207.png)

开启 smp 之后 mode = debug

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%208.png)

由 start-gpa -start-hpa 导致

vmx 收到 icr 的写入

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%209.png)

6.20 

双核情况下 运行 apps/helloworld 没有 问题

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%2010.png)

运行 apps/task/parallel 只要修改 NUM_DATA 和 NUM_TASKS  将其减小一点 双核就可以运行

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%2011.png)

在原来的设置下会分配超过 16M 内存会触发 ept violation ，之前试过将guest mem 设置为 128M 但是失败了会卡住， 感觉是qemu 直接加载 地址设置的问题。

使用的方法很简单 就是 注册一个软件中断  通过 virt_ipi_handler 来处理 消息，ap 收到 start 就 run

但是 超过双核 就会出现问题

![Untitled](hv%20x86%20multicore%2097ea87cf79804d9eaa384a2211c2a0d4/Untitled%2012.png)

这个 rip 地址在汇编里根本就不是可执行的 很奇怪， 感觉是 cpuid 出了问题 

现在才发现有 boot_arceos 分支 准备合并它

2. 【进阶】基于【基础】的实现，令一个物理CPU与多个VCpu绑定，其上仍然只需运行单个客户机。一个物理CPU与多个VCpu绑定会涉及到调度算法以及上下文的切换，可以参考现有常用的调度算法（CFS、FIFO、RR）进行实现，或对其进行改进。例如：2个物理CPU，每个CPU分别对应2个VCpu，共4个VCpu。单个客户机使用这4个VCpu。

参考 sdm 卷三 31？ 

我参考了 intel arcn 的实现思路 但是 还在设计 数据结构 C 和 Rust 还是有很多不同。。。。

太菜了 感觉来不及了 但是挺有收获的 😂
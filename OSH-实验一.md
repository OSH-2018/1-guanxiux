#实验一：调试操作系统的启动过程

##主要调试工具： gdb 和 qemu

####[gdb](http://www.gnu.org/software/gdb/documentation/) 

是由GUN开源组织发布的命令行式调试工具，实现的调试功能主要有以下四点：

1. 启动程序，判明任何可能影响其行为的因素
2. 使程序运行到指定条件时暂停
3. 在程序的运行暂停时检测当前发生的事件
4. 改变程序中的变量，这样可以修正一个bug造成的影响而让你继续去测试别的bug

要让gdb调试一个程序，必须要让这个程序在被编译时附带上debug info，具体是在用gcc编译时加上 -g 选项，在编译内核文件时使CONFIG_DEBUG_INFO=y。

#### gdb常用调试命令

```shell
gcc -g  main.c                      //在目标文件加入源代码的信息
gdb a.out       					//调试程序a.out

(gdb) start                         //开始调试
(gdb) next/n                        //一条一条执行，不进入调用的子函数
(gdb) step/s                        //执行下一条，可以进入子函数
(gdb) backtrace/bt                  //查看函数调用栈帧
(gdb) info/i locals                 //查看当前栈帧局部变量
(gdb) frame/f                       //选择栈帧，再查看局部变量
(gdb) print/p variable              //打印变量的值
(gdb) finish                        //运行到当前函数返回
(gdb) set var sum=0                 //修改变量值
(gdb) list/l $(行号或函数名)          //列出源码
(gdb) display/undisplay sum         //每次停下显示变量的值/取消跟踪
(gdb) break/b  $(行号或函数名)        //设置断点
(gdb) continue/c                    //连续运行
(gdb) info/i breakpoints            //查看已经设置的断点
(gdb) delete breakpoints 2          //删除某个断点
(gdb) disable/enable breakpoints 3  //禁用/启用某个断点
(gdb) break 9 if sum != 0           //满足条件才激活断点
(gdb) run/r                         //重新从程序开头连续执行
(gdb) watch input[4]                //设置观察点
(gdb) info/i watchpoints            //查看设置的观察点
(gdb) x/7b input                    //打印存储器内容，b--每个字节一组，7--7组
(gdb) disassemble                   //反汇编当前函数或指定函数
(gdb) si                            // 一条指令一条指令调试 而 s 是一行一行代码
(gdb) info registers                // 显示所有寄存器的当前值
(gdb) x/20 $esp                     //查看内存中开始的20个数
(gdb) file a.out					//调试程序a.out，相当于gdb a.out
(gdb) target remote:1234 			//连接1234端口以调试
```

#### [qemu](https://qemu.weilnetz.de/doc/qemu-doc.html)

是一个跨平台的纯软件实现的虚拟化模拟器，使用动态翻译来实现良好的仿真速度。它几乎能模拟任何硬件设备，例如 x86, arm 等，使用命令qemu-system-i386，qemu-system-x86_64 等来启动虚拟机。实验中使用qemu来模拟基于 x86 的64位处理器。同时qemu内嵌了一个 gdbserver，可以和gdb构成一个远程调试合作伙伴，让 gdb 通过 ip:port 网络方式或者是通过串口 /dev/ttyS*来控制观察虚拟机的运行，进行调试。

qemu 不仅是一个模拟器，它还集成了一系列搭建启动一个虚拟的系统的环境所需的工具，如 qemu-nbd 可以把一个 qemu disk image 通过nbd方式挂载到系统上，qemu-img 可以用来创建，修改和检查镜像文件，还能转换镜像文件的格式。

####用到的qemu命令

```shell 
qemu-system-x86_64 -kernel $(path)/bzImage -initrd rootfs.img -s -S
//-kernel bzImage 使用 bzImage 作为内核镜像，通常是 Linux 内核 
//-initrd file 使用 file 作为 initial ram disk，即临时根文件系统。initrd是为了支持linux启动的两个阶段，而设计的临时根文件系统。通常，initrd内包含多种可执行文件和驱动库，用于实现最后挂载真实的根文件系统，并在之后卸载临时的initrd根文件系统并释放相应的内存。在很多嵌入式的Linux系统中，没有使用其他真实的根文件系统，而采用initrd做为最后使用的根文件系统。initrd被绑定到内核，并伴随内核启动过程被加载。initrd内部包含了必要的可执行文件和系统文件以支持Linux第二阶段的启动过程
//-s 相当于 -gdb tcp::1234，在TCP上默认的1234端口打开gdbserver供gdb调试使用
//-S 不在startup阶段启动CPU，必须在调试器内输入 c (continue) 才能让内核继续启动
```

## 调试步骤

####工具的安装

安装 Linux17.10 Ubuntu 发行版，使用的软件版本分别为 GNU gdb 8.0.1 和 qemu 2.10。

####环境的搭建

```shell
mkdir Kernel
cd Kernel
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.18.103.tar.xz
tar xf linux-3.18.103.tar.xz
//获取内核源码

cd linux-3.18.103.tar.xz
make x86_64_defconfig
make -j8
//预先编译内核，-j8选项代表使用八核进行编译

cd ..
mkdir rootfs
git clone https://github.com/mengning/menu.git
cd menu
gcc -o init linktable.c menu.c test.c -m32 -static -pthread
cd ../rootfs
cp ../menu/init ./
find . | cpio -o -Hnewc |gzip -9 > ../rootfs.img
//制作initrd.img，这里命名作 rootfs.img，需要根据相关提示安装gzip等工具

cd ..
qemu-system-x86_64 -kernel linux-3.18.103/arch/x86_64/boot/bzImage -initrd rootfs.img 
//尝试启动内核

make menuconfig
//配置内核，设置64-bits kernel = y; Kernel hacking-->Compile-time checks and compiler options-->Compile the kernel with debug info = y。这样编译得到的是带有调试信息的64位内核。

make -j8
```

####开始调试

```shell
qemu-system-x86_64 -kernel linux-3.18.103/arch/x86_64/boot/bzImage -initrd rootfs.img -s -S
//启动内核并冻结，待gdb远程调试
```

另开一个terminal，

```shell
gdb
(gdb)file Kernel/linux-3.18.103/vmlinux
//装载调试的目标内核文件
(gdb)target remote:1234
//连接qemu的远程调试端口
(gdb)break n
//设置断点
(gdb)continue
//执行程序至断点，根据qemu官方文档，这是进一步调试必须的命令
...
//其他调试步骤
```

## 结果

内核启动初始化的入口在 linux-3.18.102/init/main.c 文件的 start_kernel() 函数中，里面调用的函数都可以说是内核启动的关键事件，这里着重追踪里面调用的 acpi_subsystem_init() 函数和 rest_init() 函数。

### acpi_subsystem_init()

in linux-3.18.102 /drivers/acpi/bus.c

​	这一函数对应的事件是完成内核启动阶段中 ACPI 的初始化的最终阶段。ACPI 是 Advanced Configuration and Power Interface (高级配置和电源管理接口)的缩写，其作用包括处理器，系统，外设的电源管理，温度管理等，能根据实际计算机的使用情况控制系统设备的功耗，提高系统性能和降低功耗。

​	这一事件执行完成后，系统平台的电源管理从简单的 bios 模式切换到了 ACPI 模式(当平台支持时)，同时初始化了 ACPI 事件的管理，并安装了中断和全局锁管理器。

#### 在该函数处设置断点, 初调用时:

```
(gdb) info stack
#0  acpi_subsystem_init () at drivers/acpi/bus.c:563
#1  0xffffffff81eefe64 in start_kernel () at init/main.c:670
#2  0xffffffff81eef2f2 in x86_64_start_reservations (
    real_mode_data=<optimized out>) at arch/x86/kernel/head64.c:195
#3  0xffffffff81eef413 in x86_64_start_kernel (
    real_mode_data=0x140c0 <error: Cannot access memory at address 0x140c0>) at arch/x86/kernel/head64.c:184
#4  0x0000000000000000 in ?? ()
```

```
(gdb) info registers
rax            0x0      0
rbx            0xffffffffffffffff       -1
rcx            0xffffffff81e03f60       -2116010144
rdx            0x0      0
rsi            0x80000000       2147483648
rdi            0x80000000       2147483648
rbp            0xffffffff81e03fb0       0xffffffff81e03fb0 <init_thread_union+16304>
rsp            0xffffffff81e03f80       0xffffffff81e03f80 <init_thread_union+16256>
r8             0x20d2000        34414592
r9             0xffff880000000000       -131941395333120
r10            0x8000000000000163       -9223372036854775453
r11            0x1      1
r12            0xffff880007f70b80       -131941261702272
r13            0xffffffff81f93920       -2114373344
r14            0xffffffff81f9b2e0       -2114342176
r15            0x0      0
rip            0xffffffff81f26dcf       0xffffffff81f26dcf <acpi_subsystem_init>
eflags         0x292    [ AF SF IF ]
cs             0x10     16
ss             0x0      0
ds             0x0      0
es             0x0      0
fs             0x0      0
gs             0x0      0
```

#### 随后单步执行该函数, 观察每一步的动作并记录:

```c
void __init acpi_subsystem_init(void)
{
	acpi_status status;
	//定义一个 status 结构体，储存 ACPI 的各种启动状况
	if (acpi_disabled) 
		return;
	//观察 acpi_disabled，检查是否在配置中禁用 ACPI 模式，若否，直接返回，否则，继续启动 ACPI 模式
    
	status = acpi_enable_subsystem(~ACPI_NO_ACPI_ENABLE);
    //尝试启动 acpi 子系统，根据启动的情况返回不同的 acpi_status 结构体赋值给 status
    //输入 ~ACPI_NO_ACPI_ENABLE 的 flag 是为了判断当前是否已经启动了 ACPI，若是，则子函数一些	  //步骤可据此跳过
	if (ACPI_FAILURE(status)) {
		printk(KERN_ERR PREFIX "Unable to enable ACPI\n");
		disable_acpi();
    //若 ACPI_FAILURE(status) 为真，表明 ACPI 启动失败，输出启动结构并禁用 ACPI
	} else {
		regulator_has_full_constraints();
	}
}
```

### rest_init()

in linux-3.18.102 /init/main.c

​	这一事件是内核初始化阶段到用户空间初始化阶段的分界，意指完成剩余部分的初始化工作。注意在调用函数 start_kernel() 前内核创建了一个pid = 0 的 0 号进程，它在内核初始化时唯一运行，为加载系统，初始化服务。后来这个进程演化为 idle 进程，负责进程调度，交换和系统空闲资源占用的工作。

#### 在该函数处设置断点, 初调用时:

```
(gdb) info stack
#0  rest_init () at init/main.c:394
#1  0xffffffff81eefe7e in start_kernel () at init/main.c:681
#2  0xffffffff81eef2f2 in x86_64_start_reservations (
    real_mode_data=<optimized out>) at arch/x86/kernel/head64.c:195
#3  0xffffffff81eef413 in x86_64_start_kernel (
    real_mode_data=0x140c0 <error: Cannot access memory at address 0x140c0>) at arch/x86/kernel/head64.c:184
#4  0x0000000000000000 in ?? ()
```

```
(gdb) info registers
rax            0x0      0
rbx            0xffffffffffffffff       -1
rcx            0x0      0
rdx            0xffffffff81be803a       -2118221766
rsi            0xa9     169
rdi            0x604    1540
rbp            0xffffffff81e03fb0       0xffffffff81e03fb0 <init_thread_union+16304>
rsp            0xffffffff81e03f80       0xffffffff81e03f80 <init_thread_union+16256>
r8             0xcf8    3320
r9             0x2      2
r10            0x8000000000000163       -9223372036854775453
r11            0x1      1
r12            0xffff880007f70b80       -131941261702272
r13            0xffffffff81f93920       -2114373344
r14            0xffffffff81f9b2e0       -2114342176
r15            0x0      0
rip            0xffffffff81832ae0       0xffffffff81832ae0 <rest_init>
eflags         0x246    [ PF ZF IF ]
cs             0x10     16
ss             0x0      0
ds             0x0      0
es             0x0      0
fs             0x0      0
gs             0x0      0
```

#### 随后单步执行该函数, 观察每一步的动作并记录:

```c
static noinline void __init_refok rest_init(void)
{
	int pid;
	//pid 是 process identification 的缩写，用来区分不同进程，相应的还有 tid，thread identification。pid 在进程运行过程中对于一个进程是唯一的。
	rcu_scheduler_starting();
	//启动 RCU scheduler。RCU，Read-copy-update 允许文件在某些情况下被多个读取者同时读取，这个时候读取到的实际上是原文件的副本，这个策略下原文件的修改就不会与读取冲突，但是会消耗更多空间。
	kernel_thread(kernel_init, NULL, CLONE_FS);
    //创建一号内核进程 kernel_init，进入子函数查看注意到返回值 nr = 1，它是系统中所有其它用户进程的祖先进程
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    //创建二号内核进程 kthread，注意到其返回值为 2，则 pid 被赋值为 2, 对应着kthreadd 进程，kthreadd 进程的功能是管理和帮助内核的其他部分创建别的内核进程。
	rcu_read_lock();
    //运行 RCU 的读阶段的 lock 功能，以保证下面读取函数调用的参数的稳定性。
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    // 获取kthreadd的进程描述符
	rcu_read_unlock();
    //解除 rcu_read_lock 功能以释放空间
	complete(&kthreadd_done);

	/*
	 * The boot idle thread must execute schedule()
	 * at least once to get things moving:
	 */
	init_idle_bootup_task(current);
    // 设置当前进程（0号进程）为idle进程类。 current 是全局项，在 /arch/x86/include/asm/current.h 里被定义，可以产生指向当前在运行的进程的 task_struct 结构体指针。
	schedule_preempt_disabled();

	cpu_startup_entry(CPUHP_ONLINE);
    // 0号进程完成kernel初始化的工作，进入idle循环，化身idle进程，并且禁止preempt，抢占
}
```










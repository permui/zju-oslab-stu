# Lab 4: RV64 虚拟内存管理

## 1 实验目的
* 学习虚拟内存的相关知识，实现物理地址到虚拟地址的切换。
* 了解 RISC-V 架构中 SV39 分页模式，实现虚拟地址到物理地址的映射，并对不同的段进行相应的权限设置。

## 2 实验环境

* Docker in Lab0

## 3 背景知识
 
在 [lab3](./lab3.md) 中我们赋予了 OS 对多个线程调度以及并发执行的能力，由于目前这些线程都是内核线程，因此他们可以共享运行空间，即运行不同线程对空间的修改是相互可见的。但是如果我们需要线程相互**隔离**，以及在多线程的情况下更加**高效**的使用内存，我们必须引入*虚拟内存*这个概念。

虚拟内存可以为进程提供独立的内存空间，制造一种每个进程的内存都是独立的假象。虚拟内存到物理内存的映射也包含了内存访问权限的信息，方便 kernel 完成权限检查。

在本次实验中，我们需要关注 OS 如何**开启虚拟内存**，如何通过设置页表来实现**地址映射**和**权限控制**。

### 3.1 Linux Kernel 的虚拟内存布局

```
start_address           end_address
    0x0                 0x3fffffffff
     │                       │
┌────┘                  ┌────┘
↓        256G           ↓                                
┌───────────────────────┬──────────┬────────────────┐
│      User Space       │    ...   │  Kernel Space  │
└───────────────────────┴──────────┴────────────────┘
                                   ↑     256G       ↑
                      ┌────────────┘                │ 
                      │                             │
              0xffffffc000000000           0xffffffffffffffff
                start_address                  end_address
```
上图是 Linux 5.15 中 RV64 架构的内存布局。它是基于 RISC-V 标准中定义的 Sv39 地址转换模式的（见下文）。我们看到 `0x0000004000000000` 以下的虚拟地址空间分配给 user space，`0xffffffc000000000` 及以上的虚拟地址空间分配给 `kernel space`。由于我们还未引入用户态程序，目前我们只需要关注 `kernel space`。原始说明文件在[此处](https://elixir.bootlin.com/linux/v5.15/source/Documentation/riscv/vm-layout.rst)。

Kernel 为了高效方便地访问 RAM，会把所有物理内存都映射到虚拟内存里 kernel space 中的一块区域。这块区域称为 *direct mapping area*。由于此处虚拟内存和物理内存的映射关系是 `VA == PA + OFFSET`， 所以这种映射也被称为 *linear mapping*（注意，这与线性代数中的 linear mapping 是不同的——那里的 linear mapping 一定把 0 映射到 0，而此处不是。在数学上此处的映射应称为 translation transformation，即平移变换）。在 RISC-V 架构的 Linux Kernel 中 direct mapping area 为 `0xffffffe000000000 ~ 0xffffffff00000000`, 共 124 GB 。

**本次实验的任务就是开启虚拟内存，实现 direct mapping，并让 lab3 中的多进程程序在 direct mapping area 中执行。**

为此，我们需要了解 RISC-V 架构中虚拟内存的工作方式。

### 3.2 RISC-V Virtual-Memory System

虚拟内存的本质是地址转换与保护，而这是通过页表来实现的。分四步来理解地址转换与保护：

1. S 态下如何开启转换，页表基地址存放在哪里：`satp` register
2. 我们使用的转换模式下，虚拟地址和物理地址如何划分
3. 页表项（Page Table Entry, PTE）的格式是什么
4. 当一切都设置好后，地址转换与保护是如何进行的

下面分别介绍这四点。

#### 3.2.1 The `satp` Register（Supervisor Address Translation and Protection Register）

`satp` 是 S 模式下控制地址转换与保护的 CSR 寄存器，它的格式如下图：

```c
 63      60 59                  44 43                                0
 ---------------------------------------------------------------------
|   MODE   |         ASID         |                PPN                |
 ---------------------------------------------------------------------
```

* MODE 字段控制地址转换模式，默认为 0，表示不做转换。它的所有取值如下图。当 `satp.MODE` 被设为 8 时，我们就开启了 Sv39 模式的地址转换。
```c
                             RV 64
     ----------------------------------------------------------
    |  Value  |  Name  |  Description                          |
    |----------------------------------------------------------|
    |    0    | Bare   | No translation or protection          |
    |  1 - 7  | ---    | Reserved for standard use             |
    |    8    | Sv39   | Page-based 39 bit virtual addressing  | <-- 我们使用的mode
    |    9    | Sv48   | Page-based 48 bit virtual addressing  |
    |    10   | Sv57   | Page-based 57 bit virtual addressing  |
    |    11   | Sv64   | Page-based 64 bit virtual addressing  |
    | 12 - 13 | ---    | Reserved for standard use             |
    | 14 - 15 | ---    | Reserved for standard use             |
     -----------------------------------------------------------
```
* ASID ( Address Space Identifier ) ： 此次实验中直接置 0 即可。
* PPN ( Physical Page Number ) ：顶级页表的**物理**页号。在 Sv39 中物理页的大小为 4KiB，所以 `PPN == PA >> 12`。

具体介绍请阅读 [RISC-V Privileged Spec 4.1.10](https://www.five-embeddev.com/riscv-isa-manual/latest/supervisor.html#sec:satp)。

#### 3.2.2 RISC-V Sv39 Virtual Address and Physical Address
```c
     38        30 29        21 20        12 11                           0
     ---------------------------------------------------------------------
    |   VPN[2]   |   VPN[1]   |   VPN[0]   |          page offset         |
     ---------------------------------------------------------------------
                            Sv39 virtual address

```

```c
 55                30 29        21 20        12 11                           0
 -----------------------------------------------------------------------------
|       PPN[2]       |   PPN[1]   |   PPN[0]   |          page offset         |
 -----------------------------------------------------------------------------
                            Sv39 physical address

```
Sv39 模式定义的物理地址有 56 位，虚拟地址有 64 位。但是，虚拟地址的 64 位只有低 39 位有效——在使用 64 位虚拟地址时，必须保证其 63 ~ 39 号位与 38 号位相等，否则就会触发 Page Fault Exception。当虚拟地址的 63 ~ 38 号位都为 0 时，该地址处于 user space 中；都为 1 时处于 kernel space 中。

Sv39 支持三级页表结构。

* VPN (Virtual Page Number) 是虚拟地址所在页的页号，在地址转换中是查询页表所用的下标。VPN[2] ~ VPN[0] 分别用于查询顶级页表、二级页表、三级页表。
* PPN (Physical Page Number) 是物理地址所在页的页号。
* Page Offset 是页内偏移。

必须指出，**并不是三级页表都一定要用的：我们可以只用第一级，或只用前两级**。

* 只用第一级页表。此时，假设我们拿 `VA` 中的 `VPN[2]` 查询顶级页表得到的 PTE 中物理页号部分为 `{PPN[2], PPN[1], PPN[0]}`，那么地址转换得到的物理地址就是 `{PPN[2], VPN[1], VPN[0], VA.offset}`。此时虚拟内存的一页大小为 1 GiB，因为 `{VPN[1], VPN[0], VA.offset}` 连起来被看作页内偏移，一共有 30 位。
* 只用前两级页表。此时虚拟内存的一页大小就是 2 MiB 。
* 当然，我们可以用三级页表，此时虚拟内存的一页大小就是 4 KiB。

具体介绍请阅读 [RISC-V Privileged Spec 4.4.1](https://www.five-embeddev.com/riscv-isa-manual/latest/supervisor.html#sec:sv39)。


#### 3.2.3 RISC-V Sv39 Page Table Entry
```c
 63      54 53        28 27        19 18        10 9   8 7 6 5 4 3 2 1 0
 -----------------------------------------------------------------------
| Reserved |   PPN[2]   |   PPN[1]   |   PPN[0]   | RSW |D|A|G|U|X|W|R|V|
 -----------------------------------------------------------------------
                                                     |   | | | | | | | |
                                                     |   | | | | | | | `---- V - Valid
                                                     |   | | | | | | `------ R - Readable
                                                     |   | | | | | `-------- W - Writable
                                                     |   | | | | `---------- X - Executable
                                                     |   | | | `------------ U - User
                                                     |   | | `-------------- G - Global
                                                     |   | `---------------- A - Accessed
                                                     |   `------------------ D - Dirty (0 in page directory)
                                                     `---------------------- Reserved for supervisor software
```

* 0 ～ 9 bit: protection bits
    * V : 有效位。访问到 V = 0 的 PTE 时会产生 Page Fault。
    * R : R = 1 该页可读。
    * W : W = 1 该页可写。
    * X : X = 1 该页可执行。
    * U , G , A , D , RSW 本次实验中设置为 0 即可。

具体介绍请阅读 [RISC-V Privileged Spec 4.4.1](https://www.five-embeddev.com/riscv-isa-manual/latest/supervisor.html#sec:sv39)。


#### 3.2.4 RISC-V Address Translation
虚拟地址转化为物理地址流程图如下，具体描述见 [RISC-V Privileged Spec 4.3.2](https://www.five-embeddev.com/riscv-isa-manual/latest/supervisor.html#sv32algorithm) :
```text
                                Virtual Address                                     Physical Address

                          9             9            9              12          55        12 11       0
   ┌────────────────┬────────────┬────────────┬─────────────┬────────────────┐ ┌────────────┬──────────┐
   │                │   VPN[2]   │   VPN[1]   │   VPN[0]    │     OFFSET     │ │     PPN    │  OFFSET  │
   └────────────────┴────┬───────┴─────┬──────┴──────┬──────┴───────┬────────┘ └────────────┴──────────┘
                         │             │             │              │                 ▲          ▲
                         │             │             │              │                 │          │
                         │             │             │              │                 │          │
┌────────────────────────┘             │             │              │                 │          │
│                                      │             │              │                 │          │
│                                      │             │              └─────────────────┼──────────┘
│    ┌─────────────────┐               │             │                                │
│511 │                 │  ┌────────────┘             │                                │
│    │                 │  │                          │                                │
│    │                 │  │     ┌─────────────────┐  │                                │
│    │                 │  │ 511 │                 │  │                                │
│    │                 │  │     │                 │  │                                │
│    │                 │  │     │                 │  │     ┌─────────────────┐        │
│    │   44       10   │  │     │                 │  │ 511 │                 │        │
│    ├────────┬────────┤  │     │                 │  │     │                 │        │
└───►│   PPN  │  flags │  │     │                 │  │     │                 │        │
     ├────┬───┴────────┤  │     │   44       10   │  │     │                 │        │
     │    │            │  │     ├────────┬────────┤  │     │                 │        │
     │    │            │  └────►│   PPN  │  flags │  │     │                 │        │
     │    │            │        ├────┬───┴────────┤  │     │   44       10   │        │
     │    │            │        │    │            │  │     ├────────┬────────┤        │
   1 │    │            │        │    │            │  └────►│   PPN  │  flags │        │
     │    │            │        │    │            │        ├────┬───┴────────┤        │
   0 │    │            │        │    │            │        │    │            │        │
     └────┼────────────┘      1 │    │            │        │    │            │        │
     ▲    │                     │    │            │        │    └────────────┼────────┘
     │    │                   0 │    │            │        │                 │
     │    └────────────────────►└────┼────────────┘      1 │                 │
     │                               │                     │                 │
 ┌───┴────┐                          │                   0 │                 │
 │  satp  │                          └────────────────────►└─────────────────┘
 └────────┘
```

## 4 实验指导

### 4.0 任务与途径

**本次实验的任务是开启虚拟内存，实现 direct mapping，并让 lab3 中的多进程程序在 direct mapping area 中执行。**

下面描述具体实现方式。为了描述清晰，我们先定义一些记号：

* $p_s$：物理地址的起始地址值，本实验中为 `0x80000000`。
* $p_e$：物理地址的终止地址值，本实验中为 `0x8020000`。
* $l$：物理地址段的长度，本实验中为 `0x8000000`，即 128MiB。
* $b$：为 OpenSBI 预留的地址长度，本实验中为 `0x200000`，即 2 MiB。 
* $v_s$：虚拟地址的起始地址值，本实验中为 `0xffffffe000000000`
* $v_e$：虚拟地址的终止地址值，本实验中为 `0xffffffe000200000`
* $t=v_s-p_s$，虚拟地址相对于物理地址的偏移量 (offset)
* $[a, b)\to [c, d)$：内存映射，把**虚拟地址空间**的 $[a, b)$ 映射到**物理地址空间**的 $[c, d)$。当然，要求两段长度相同。

完美开启虚拟内存分为两步：

**第一回映射，分两块**：

* $[p_s, p_s+d)\to [p_s, p_s+d)$，称为等值映射
* $[v_s, v_s+d)\to [p_s, p_s+d)$，称为偏移映射（即上文中的 linear mapping）

其中 $d$ = 1 GiB。

我们的物理地址段只有 $l$ = 128 MiB 这么大，为什么要做 1 GiB 的映射呢？这仅仅是为了方便。可以想象，如果使用两级或三级页表，就会涉及多个物理页的管理，比较麻烦；若只用第一级页表，我们就只需要一个物理页来存放它，非常方便。因此在第一回映射时我们就只用第一级页表，从而虚拟内存一页的大小就是 1 GiB（见上文）。只要我们实际访问的地址都在 128 MiB 的范围内，这样就是无害的。

**第二回映射**：

* $[v_s+b, v_e)\to [p_s+b, p_e)$ 

为什么左端点要加 $b$ 呢？因为 $[v_s, v_s+b)$ 这一段虚拟地址根本就用不着——若要访问这一段，访问到的是 OpenSBI，但我们在 S 态下不会（也不应该）访问它。需要访问它的是 M 态，而 M 态根本不受 `satp`（S 模式的地址转换）影响——在 M 态里程序直接用物理地址 $[p_s, p_s+b)$ 访问 OpenSBI。

第二回映射用三级页表来实现。

第二回映射会覆盖第一回映射。即，当我们把 `satp` 指向承载第二回映射的页表时，第一回映射就自动失效了。最后我们就得到了不多不少刚刚好的虚拟地址到物理地址的映射。

这里留下几个问题（也在思考题中），请同学思考：

* 为什么第一回映射里要做等值映射？
* 为什么要分这样两步呢？能否一步做完？

下面是具体编程的指引。

### 4.1 准备工作
此次实验基于 lab3 同学所实现的代码进行。

需要修改 `defs.h`，写入上面定义的常量。在 `defs.h` 中加入如下内容：
```c
#define OPENSBI_SIZE (0x200000)

#define VM_START (0xffffffe000000000)
#define VM_END   (0xffffffff00000000)
#define VM_SIZE  (VM_END - VM_START)

#define PA2VA_OFFSET (VM_START - PHY_START)
```

从 `repo` 同步以下代码: `vmlinux.lds.S`, `Makefile`。并按照以下步骤将这些文件正确放置。
```
.
└── arch
    └── riscv
        └── kernel
            ├── Makefile
            └── vmlinux.lds.S
```
这里我们通过 `vmlinux.lds.S` 模版生成 `vmlinux.lds`文件。链接脚本中的 `ramv` 代表 VMA (Virtual Memory Address)，即虚拟地址；`ram` 则代表 LMA (Load Memory Address)，即我们 OS image 被 load 的地址，可以理解为物理地址。使用以上的 `vmlinux.lds` 进行编译之后，得到的 `System.map` 以及 `vmlinux` 采用的都是虚拟地址，方便之后 debug。

### 4.2 第一回映射
下图描绘了第一回映射的状况：
```text
Physical Address
-------------------------------------------
                     | OpenSBI | Kernel |
-------------------------------------------
                     ^
                0x80000000
                     ├───────────────────────────────────────────────────┐
                     |                                                   |
Virtual Address      |                                                   |
-----------------------------------------------------------------------------------------------
                     | OpenSBI | Kernel |                                | OpenSBI | Kernel |
-----------------------------------------------------------------------------------------------
                     ^                                                   ^
                0x80000000                                       0xffffffe000000000
```

我们用两个函数实现第一回映射：

* `arch/riscv/kernel/vm.c : setup_vm`：配置好页表；
* `arch/riscv/kernel/head.S : relocate`：配置 `satp`，并返回到高地址。

```c
// arch/riscv/kernel/vm.c

/* early_pgtbl: 第一回映射的页表, 大小恰好为一页 4 KiB, 开头对齐页边界 */
unsigned long  early_pgtbl[512] __attribute__((__aligned__(0x1000)));

void setup_vm(void) {
    /* 
    1. 如前文所说，只使用第一级页表
    2. 因此 VA 的 64bit 作为如下划分： | high bit | 9 bit | 30 bit |
        high bit 可以忽略
        中间 9 bit 作为 early_pgtbl 的 index
        低 30 bit 作为页内偏移
    3. Page Table Entry 的权限 V | R | W | X 位设置为 1
    */
}
```
```asm
# arch/riscv/kernel/head.S

_start:

    call setup_vm
    call relocate

    ...

    j start_kernel

relocate:
    # set ra = ra + PA2VA_OFFSET
    # set sp = sp + PA2VA_OFFSET (If you have set the sp before)
   
    ###################### 
    #   YOUR CODE HERE   #
    ######################

    # set satp with early_pgtbl
    
    ###################### 
    #   YOUR CODE HERE   #
    ######################
    
    # flush tlb
    sfence.vma zero, zero

    ret

    .section .bss.stack
    .globl boot_stack
boot_stack:
    ...
```

> Hint 1: `sfence.vma` 指令用于刷新 TLB
>
> Hint 2: 在 set satp 前，我们只可以使用**物理地址**来打断点。设置 satp 之后，才可以使用虚拟地址打断点，同时之前设置的物理地址断点也会失效，需要删除。
>
> Hint 3: 如果要调试代码，可以使用 `.gdbinit` 做一些自动化。在启动 GDB 的那个目录下新建 `.gdbinit` 文件，在其中写入 GDB 指令，GDB 就会在启动时自动执行它们。比如在其中写入 `target remote :1234`、打断点指令、执行到断点处的指令，GDB 在启动后就会自动链接上 QEMU，打断点并跳到断点处。这能极大地提升调试体验。

### 4.3 第二回映射

下图描绘了第二回映射的状况：
```text
Physical Address
     PHY_START                           PHY_END
         ↓                                  ↓
--------------------------------------------------------
         | OpenSBI | Kernel |               |
--------------------------------------------------------
                   ^                        ^
               0x80200000                   └───────────────────────────────────────────────────┐
                   └───────────────────────────────────────────────────┐                        |
                                                                       |                        |
                                                                    VM_START                    |
Virtual Address                                                        |                        |
----------------------------------------------------------------------------------------------------
                                                             | OpenSBI | Kernel |               |
-----------------------------------------------------------------------------------------------------
                                                                       ^
                                                               0xffffffe000200000
```

第二回映射要使用三级页表，所以我们需要管理多个页面。一个好办法是先执行 `mm.c` 中的 `mm_init` 初始化内存管理模块，然后再配置页表。当我们需要新页时，调用 `kalloc()` 申请一页即可。**注意，本次实验与之前不同， `mm_init` 运行在虚拟地址上，所以 `mm_init` 函数需要一些修改**，请你自行完成。

第二回映射中，建立页表和设置 `satp` 用一个函数 `arch/riscv/kernel/vm.c : setup_vm_final` 来实现。

完成下面的代码，并在 `head.S` 的适当位置调用 `setup_vm_final`。

```c
// arch/riscv/kernel/vm.c 

/* swapper_pg_dir: kernel pagetable 根目录， 在 setup_vm_final 进行映射。 */
unsigned long  swapper_pg_dir[512] __attribute__((__aligned__(0x1000)));

void setup_vm_final(void) {
    memset(swapper_pg_dir, 0x0, PGSIZE);

    // No OpenSBI mapping required

    // mapping kernel text X|-|R|V
    create_mapping(...);

    // mapping kernel rodata -|-|R|V
    create_mapping(...);
    
    // mapping other memory -|W|R|V
    create_mapping(...);
    
    // set satp with swapper_pg_dir

    YOUR CODE HERE

    // flush TLB
    asm volatile("sfence.vma zero, zero");
    return;
}


/* 创建多级页表映射关系 */
create_mapping(uint64 *pgtbl, uint64 va, uint64 pa, uint64 sz, int perm) {
    /*
    pgtbl 为根页表的基地址
    va, pa 为需要映射的虚拟地址、物理地址
    sz 为映射的大小
    perm 为映射的读写权限

    创建多级页表的时候可以使用 kalloc() 来获取一页作为页表目录
    可以使用 V bit 来判断页表项是否存在
    */
}
```
### 4.4 编译及测试
由于加入了一些新的 C 文件，可能需要修改一些 Makefile 文件，请同学自己尝试修改，使项目可以编译并运行。

输出示例：
```bash
OpenSBI v0.9
  ____                    _____ ____ _____
 / __ \                  / ____|  _ \_   _|
| |  | |_ __   ___ _ __ | (___ | |_) || |
| |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
| |__| | |_) |  __/ | | |____) | |_) || |_
 \____/| .__/ \___|_| |_|_____/|____/_____|
       | |
       |_|

...

Boot HART MIDELEG         : 0x0000000000000222
Boot HART MEDELEG         : 0x000000000000b109

...mm_init done!
...proc_init done!
Hello RISC-V
idle process is running!

switch to [PID = 28 COUNTER = 1] 
[PID = 28] is running! thread space begin at 0xffffffe007fa2000

switch to [PID = 12 COUNTER = 1] 
[PID = 12] is running! thread space begin at 0xffffffe007fb2000

switch to [PID = 14 COUNTER = 2] 
[PID = 14] is running! thread space begin at 0xffffffe007fb0000
[PID = 14] is running! thread space begin at 0xffffffe007fb0000

switch to [PID = 9 COUNTER = 2] 
[PID = 9] is running! thread space begin at 0xffffffe007fb5000
[PID = 9] is running! thread space begin at 0xffffffe007fb5000

switch to [PID = 2 COUNTER = 2] 
[PID = 2] is running! thread space begin at 0xffffffe007fbc000
[PID = 2] is running! thread space begin at 0xffffffe007fbc000

switch to [PID = 1 COUNTER = 2] 
[PID = 1] is running! thread space begin at 0xffffffe007fbd000
[PID = 1] is running! thread space begin at 0xffffffe007fbd000

switch to [PID = 29 COUNTER = 3] 
[PID = 29] is running! thread space begin at 0xffffffe007fa1000
[PID = 29] is running! thread space begin at 0xffffffe007fa1000
[PID = 29] is running! thread space begin at 0xffffffe007fa1000

switch to [PID = 11 COUNTER = 3] 
[PID = 11] is running! thread space begin at 0xffffffe007fb3000
...

```

## 思考题
1. 验证 `.text`, `.rodata` 段的属性是否成功设置，给出截图。
2. 为什么我们在 `setup_vm` 中需要做等值映射?
3. 在 Linux 中，是不需要做等值映射的。请探索一下不在 `setup_vm` 中做等值映射的方法。

## 作业提交
同学需要提交实验报告以及整个工程代码。在提交前请使用 `make clean` 清除所有构建产物。

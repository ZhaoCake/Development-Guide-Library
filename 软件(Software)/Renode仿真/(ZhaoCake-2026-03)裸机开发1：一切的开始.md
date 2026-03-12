# 裸机开发1：一切的开始

> 虽然本文依然归于Renode，但是实际上对于真实设备上以及其他仿真工具上，原理也是适用的。

考虑到进行一个功能的开发对于大家都不是难事，因此对于一些背景介绍直接掠过，这篇文章主要介绍的是linker.ld以及startup.s。我们在各种平台的库上都看到这两个文件，但是不知道你对它是否有足够的了解。

## linker.ld：内存的“管家”

`linker.ld` 是 **链接脚本 (Linker Script)**。它的主要作用是告诉链接器（Linker）如何将编译生成的各个目标文件（.o）合并成一个最终的可执行文件（.elf）。

在裸机开发中，没有操作系统来帮你分配内存，你需要手动指定：

1. **内存布局 (Memory Layout)**：定义 ROM (Flash) 和 RAM 的起始地址及大小。
2. **段分配 (Section Placement)**：规定代码段 (`.text`)、只读数据段 (`.rodata`)、已初始化变量段 (`.data`) 和未初始化变量段 (`.bss`) 应该存放在内存的具体位置。
3. **符号定义**：定义一些特殊的符号（如栈顶地址 `_estack`），供启动代码或 C 语言使用。

## startup.s：处理器的“第一指令”

`startup.s` 是 **启动汇编代码**。它是处理器复位（Reset）后执行的第一段程序。

处理器的硬件逻辑非常简单：上电后，它跳转到一个固定的地址（复位向量），而那个地址上存放的就是启动代码。其核心任务包括：

1. **初始化栈指针 (SP)**：为 C 语言环境准备好“运行空间”。
2. **设置中断向量表**：告诉处理器发生中断（如定时器、外设中断）时该跳到哪去。
3. **数据段搬移**：将 `.data` 段从 ROM 拷贝到 RAM（因为变量在运行过程中需要被修改）。
4. **BSS 段清零**：将所有未初始化的全局变量清零。
5. **跳转到 main**：一切准备就绪后，调用我们熟悉的 `main()` 函数。

没有这两个文件，你的 C 代码就像一堆失去了地基和通电指令的砖块。接下来，我们将深入分析一些具体的代码实现。

## 一份极简的linker.ld与startup.s

```c
OUTPUT_ARCH( "riscv" )
ENTRY(_start)

/* FE310-G000 Memory Map */
/* Flash: 0x20000000 (usually mapped to 0x20400000 for code in some contexts, but standard is XIP at 0x20000000) */
/* DTIM (RAM): 0x80000000, Length 16KB */

MEMORY
{
  flash (rxai!w) : ORIGIN = 0x20000000, LENGTH = 512M
  ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = 16K
}

SECTIONS
{
  .init :
  {
    KEEP (*(.init))
  } > flash

  .text :
  {
    *(.text .text.*)
    *(.rodata .rodata.*)
    . = ALIGN(4);
    _etext = .;
  } > flash

  .data :
  {
    . = ALIGN(4);
    _sdata = .;
    *(.data .data.*)
    *(.sdata .sdata.*)
    . = ALIGN(4);
    _edata = .;
  } > ram AT > flash

  .bss :
  {
    . = ALIGN(4);
    _sbss = .;
    *(.bss .bss.*)
    *(.sbss .sbss.*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
  } > ram

  /* Stack Top */
  . = ALIGN(8);
  _stack_top = ORIGIN(ram) + LENGTH(ram);
}
```

```asm
.section .init, "ax"
.global _start

_start:
    /* Disable Interrupts (MIE bit in mstatus) */
    csrr t0, mstatus
    li t1, 0x8     /* MIE is bit 3 */
    not t1, t1
    and t0, t0, t1
    csrw mstatus, t0

    /* Initialize Stack Pointer */
    la sp, _stack_top

    /* Copy data segment */
    la a0, _sdata
    la a1, _edata
    la a2, _etext
    bge a0, a1, end_copy_data
loop_copy_data:
    lw t0, 0(a2)
    sw t0, 0(a0)
    addi a0, a0, 4
    addi a2, a2, 4
    blt a0, a1, loop_copy_data
end_copy_data:

    /* Zero fill bss segment */
    la a0, _sbss
    la a1, _ebss
    bge a0, a1, end_fill_bss
    li t0, 0
loop_fill_bss:
    sw t0, 0(a0)
    addi a0, a0, 4
    blt a0, a1, loop_fill_bss
end_fill_bss:

    /* Call main */
    call main

    /* Infinite Loop */
loop_forever:
    j loop_forever
```

## 关键代码解析

笔者在了解这一部分的时候，曾经犯了一个错误。由于很显然linker.ld中将rom ram flash等信息描述得很清楚了，笔者心下以为linker.ld的这些描述在编译器的作用下将会自动起作用，自动地完成各个数据搬运、初始化的工作；然而实际上不是这样的，linker.ld定义的只是符号，如果不通过 startup.s 手动搬运，定义在 .data 段的变量初值在 Flash 里，而在 RAM 中对应的地址处可能是一堆乱码。

指令 | 作用对象 | 实际效果
--- | --- | ---
`.section` | 内存位置 | 配合链接脚本，决定代码存放在 Flash 还是 RAM 的哪个偏移量。
`.global` | 符号可见性 | 配合链接脚本或 C 代码，允许交叉引用（如链接器找入口，或 C 调用汇编函数）。
`la sp, _stack_top` | 运行逻辑 | 在代码已经“存放到位”后，利用符号的具体数值地址完成硬件状态的初始化。

### Linker Script (`linker.ld`) 的奥秘

1. **`MEMORY` 块**：
   - 这里定义了硬件的物理特性。`flash` 起始于 `0x20000000`（执行代码的地方），`ram` 起始于 `0x80000000`（存储变量的地方）。
2. **`AT > flash`**：
   - 这是链接脚本中最精妙的地方。`.data` 段（已初始化的全局变量）的 **运行时地址 (VMA)** 在 RAM 中，但其 **加载地址 (LMA)** 在 Flash 中。这意味着变量的初始值被烧录在 Flash 里，程序启动时需要被搬运到 RAM。
3. **符号导出**：
   - `_sdata`、`_edata`、`_sbss` 等符号并不占用内存空间，它们只是一个**地址标记**。汇编代码通过这些标记知道从哪里开始搬运数据，搬运多少。

### Startup (`startup.s`) 的执行逻辑

1. **`csrw mstatus, t0`**：
   - 在进行初始化（特别是搬运数据）时，最稳妥的做法是关闭全局中断，防止程序跳入未准备好的中断处理函数中。
2. **`la sp, _stack_top`**：
   - 栈（Stack）是 C 语言函数调用（保存返回地址、局部变量）的基础。这行指令将链接脚本计算出的栈顶地址加载到 `sp` 寄存器。
3. **数据搬运循环 (`loop_copy_data`)**：
   - 它从 Flash (`_etext` 后面) 读取数据，然后写入 RAM (`_sdata`)。这保证了像 `int a = 10;` 这样的全局变量在 `main` 函数开始前就已经拥有了正确的值。
4. **BSS 清零 (`loop_fill_bss`)**：
   - C 语言规定未初始化的全局变量默认为 0。硬件上电时 RAM 的值是随机的，所以必须手动清零。
5. **`call main`**：
   - 所有的后台准备工作（地基）都已完成，现在正式将控制权交给用户编写的 C 代码。

## 最后一个问题

我们在平常编写C代码的时候并没有些什么linker.ld和startup.s，是只有嵌入式开发才需要吗？

并不是这样。在操作系统（如 Linux 或 Windows）上编写程序时，编译器会自动处理这些底层细节。

### 1. 默认链接脚本是从哪来的？
当你运行 `gcc` 链接程序时，它其实调用了链接器 `ld`。链接器内置了一套默认的链接脚本，以适应当前操作系统的内存模型（如 ELF 格式）。
*   **如何查看：** 你可以运行以下命令来导出编译器当前的默认链接脚本：

    ```bash
    ld --verbose
    # 或者对于交叉编译器
    riscv64-unknown-elf-ld --verbose
    ```
    你会看到一大段复杂的脚本，它规定了代码段起始地址（通常是 0x400000 左右）以及各种动态链接库的需求。

### 2. 启动代码是谁提供的？

在有操作系统的情况下，你的程序入口其实不是 `main`，而是由 C 标准库（如 `glibc`、`musl` 或 `newlib`）提供的启动文件。

*   **CRT (C Runtime)**：你可能在编译错误里见过 `crt1.o`, `crti.o`, `crtbegin.o` 这样的文件。它们就是操作系统的“startup.s”。
*   **职责转移**：在操作系统中，数据搬运和内存分配由**装载器 (Loader)** 在程序运行前完成，因此启动代码主要负责初始化 C 库的环境（如流 IO、线程本地存储）、调用全局构造函数，最后才调用你的 `main`。

### 3. 如果没有 C 库还会存在吗？

如果你使用 `-nostdlib` 选项告诉编译器不要链接标准库，那么你就必须**像在裸机开发中一样**，自己提供链接脚本和定义 `_start` 符号的汇编代码。否则，链接器会因为找不到程序入口（Entry Point）而报错。

**总结：** 裸机开发只是把这些隐藏在编译器背后的“黑盒”拆开了，让你亲手搭建程序运行的第一块砖。

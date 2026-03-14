# 裸机开发2：中断让你走向操作系统

在[上一篇文章](./(ZhaoCake-2026-03)裸机开发1：一切的开始.md)中，我们介绍了关于linker.ld和startup.s的相关内容，这部分内容看似与你熟悉的应用开发没有什么关联，但确实内存管理的基础。我们了解到startup一般而言有着“关中断、搬运初始化的全局变量、初始化bss、设置中断向量表”的任务，但是在上一篇文章中我们没有涉及到后者。在这篇文章中，我们将会进行这方面的探索。

> 为什么要将中断单独拿出来说呢？一方面因为中断的机制确实相对复杂，另一方面如果说上一篇文章对应了内存，那么这一篇文章就要对应多线程了。这将会作为我们从baremetal走向rtos的过渡。
> 中断这部分内容比较多，并且比较复杂，再加上现在LLM都有非常高的水平，只是还不足以直接给人以框架性的知识。

以RISCV为例，我们需要的前置知识是特权架构手册的相关CSR，主要指mstatus、mtval、mepc、mie、mip、mscrach等相关寄存器以及CLINT与PLIC的内容。

下面的内容将会建立在你已经知晓上面的相关知识的基础上。

## 设置中断向量表

首先我们来讲一下设置中断向量表。实际上设置中断向量表的过程是非常简单的，主要就是在startup.s中设置mtvec寄存器指向一个地址，这个地址就是中断向量表的起始地址。对于RISCV来说，中断向量表的格式是固定的，每个中断源都有一个对应的入口地址，这些入口地址按照顺序排列在中断向量表中。当一个中断发生时，处理器会根据中断源的编号从中断向量表中取出对应的入口地址，然后跳转到这个地址执行相应的中断处理函数。

```assembly
    la     t0, _vector_table
    ori    t0, t0, 1       /* mode = Vectored, 低2位是01 */
    csrw   mtvec, t0
```

在之前startup.s的基础上，在关闭中断之后加上上面的内容即可。_vector_table是一个标签，指向中断向量表的起始地址。关于中断向量表的内容，我们在下文进行介绍。现在我们继续看这三行代码，第一行是将_vector_table的地址加载到t0寄存器中，第二行是将t0寄存器的值与1进行按位或运算，这样就设置了中断向量表的模式为Vectored（即每个中断源都有一个独立的入口地址），最后一行是将t0寄存器的值写入mtvec寄存器中，这样就完成了中断向量表的设置。

## 中断向量表的内容

中断向量表的内容是由一系列的中断处理函数的入口地址组成的，这些入口地址按照中断源的编号顺序排列在中断向量表中。对于RISCV来说，中断源的编号是从0开始的，每个中断源都有一个对应的编号，例如外部中断0的编号是0，外部中断1的编号是1，软件中断0的编号是3，定时器中断0的编号是7等等。因此，在编写中断处理函数的时候，我们需要按照中断源的编号顺序将这些函数的入口地址放在中断向量表中。

```assembly
.section .vector_table, "a", @progbits
.global _vector_table
_vector_table:
    .word 0          /* 中断源0的入口地址 */
    .word 0          /* 中断源1的入口地址 */
    .word 0          /* 中断源2的入口地址 */
    j   msi_handler  /* 中断源3的入口地址 */
    .word 0          /* 中断源4的入口地址 */
    .word 0          /* 中断源5的入口地址 */
    .word 0          /* 中断源6的入口地址 */
    .word 0          /* 中断源7的入口地址 */
    /* 其他中断源的入口地址 */
```

在上面的代码中，我们定义了一个名为_vector_table的标签，指向中断向量表的起始地址。我们使用了.section指令将这个标签放在一个名为.vector_table的段中，这个段的属性是可读可执行的（"a"）并且是一个程序段（@progbits）。接下来，我们使用.word指令为每个中断源定义了一个入口地址，这些地址初始值为0，表示这些中断源暂时没有对应的处理函数。当我们编写了相应的中断处理函数之后，我们需要将这些函数的入口地址填入到中断向量表中，这样当对应的中断发生时，处理器就能够正确地跳转到相应的处理函数执行。

这里又再次涉及到了之前关于段的讨论中，需要补充的是这里有一个先有鸡还是先有蛋的问题。在linker.ld中我们定义了各种段和它们的部分，并且在汇编以及C代码中使用了这些段的符号。在编写代码时确实认为是先有符号再有内容。但是站在链接器的角度来看，链接器在处理代码时会先解析这些符号，然后根据链接脚本中的定义将它们放置在正确的位置上。因此，从链接器的角度来看，是编译器给出的目标已经带有了这些符号(通过汇编.section或者C的属性)，然后链接器根据链接脚本中的定义将它们放置在正确的位置上。

## 中断处理函数的编写

基本全部的中断处理函数的编写都是一样的，主要就是在函数中进行相应的处理，然后在函数结束时使用mret指令返回到中断发生前的状态。下面是一个简单的中断处理函数的示例：

```assembly
.global mti_handler
.align 2
mti_handler:
    addi    sp, sp, -(4 * 6)
    sw      ra,  20(sp)
    sw      a0,  16(sp)
    sw      a1,  12(sp)
    sw      a2,   8(sp)
    sw      t0,   4(sp)
    sw      t1,   0(sp)

    call    handle_timer_irq       /* implemented in main.c */

    lw      t1,   0(sp)
    lw      t0,   4(sp)
    lw      a2,   8(sp)
    lw      a1,  12(sp)
    lw      a0,  16(sp)
    lw      ra,  20(sp)
    addi    sp, sp, (4 * 6)
    mret
```

注意到这里我们并没有保存全部的寄存器，而是只保存了ra、a0-a2和t0-t1这几个寄存器。这是因为在这个中断处理函数中，我们只使用了这些寄存器，因此我们只需要保存这些寄存器的值即可。保存寄存器的过程是通过将它们的值压入栈中来实现的，在函数结束时我们再将它们从栈中弹出恢复到原来的状态。最后使用mret指令返回到中断发生前的状态。对于上下文切换和外部中断，我们对它们的预估是可能使用全部寄存器，因此在它们的处理中我们会保存全部的寄存器。

## 软件中断、上下文切换与多线程

上面我们给出的中断处理函数的示例是一个定时器中断的处理函数，软件中断和外部中断的处理函数的编写也是类似的，主要就是在函数中进行相应的处理，然后在函数结束时使用mret指令返回到中断发生前的状态。对于上下文切换和多线程，我们需要在中断处理函数中保存全部的寄存器，这样才能保证在切换到另一个线程时能够正确地恢复到原来的状态。

这里我们专门来说一说软件中断，这是实现上下文切换的重要手段之一。软件中断是一种特殊的中断，它是由软件触发的，而不是由硬件触发的。当我们需要进行上下文切换时，我们可以通过触发一个软件中断来实现。具体来说，我们可以在需要进行上下文切换的地方通过设置MSI（如果不知道这是什么，阅读CLINT相关文档）触发yield的方式设置一个软件中断，然后在软件中断的处理函数中进行上下文切换的相关操作，例如保存当前线程的状态、选择下一个要运行的线程、恢复下一个线程的状态等等。在更复杂的情况下，我们会使用定时器协作抢占辅以主动yield的方式实现线程的切换；对于更复杂的系统调用，使用ecall指令触发环境调用中断来实现系统调用的功能，这是一个异常类型。

```assembly
.section .text
.global  msi_handler
.extern  current_tcb
.extern  context_switch

.align 2
msi_handler:
/* ── Phase A: Save old thread's full context ────────────────────────── */

    /* A1. Swap t6 (x31) with mscratch.
     *     After this:
     *       t6       = &current_tcb  (the global pointer's address)
     *       mscratch = original x31 value  (stashed for later)         */
    csrrw   t6, mscratch, t6

    /* A2. Dereference to get the actual TCB pointer.
     *     t6  = current_tcb  (pointer to the running thread's TCB)     */
    lw      t6, 0(t6)

    /* A3. Save x1 .. x30 directly into tcb->regs[1..30].
     *     x0 is always 0; x31 is in mscratch (handled below).
     *     Offsets = register-index × 4.                                 */
    sw      x1,   ( 1*4)(t6)    /* ra  */
    sw      x2,   ( 2*4)(t6)    /* sp  – each thread's saved stack ptr  */
    ...
    sw      x30,  (30*4)(t6)    /* t5  */

    /* A4. Retrieve original x31 from mscratch and save it.
     *     Reuse x30 (already saved above) as scratch.                   */
    csrr    x30, mscratch
    sw      x30, (31*4)(t6)     /* regs[31] = original t6 */

    /* A5. Restore mscratch = &current_tcb so the NEXT MSI entry works.
     *     Use x30 (scratch) to hold the address.                        */
    la      x30, current_tcb
    csrw    mscratch, x30

    /* A6. Save mepc (resume PC) and mstatus (interrupt-enable bits).
     *     mret uses these to resume the thread with the correct mode.   */
    csrr    x30, mepc
    sw      x30, 128(t6)        /* tcb->pc      */

    csrr    x30, mstatus
    sw      x30, 132(t6)        /* tcb->mstatus */

/* ── Phase B: Select next thread ───────────────────────────────────── */

    /* B1. Clear MSIP so the interrupt cannot re-fire immediately after
     *     mret.  Do this before the call so context_switch() runs
     *     with a clean interrupt state.                                  */
    li      x30, 0x02000000     /* CLINT base */
    sw      zero, 0(x30)        /* MSIP0 = 0                             */

    /* B2. Call C scheduler: updates the global current_tcb pointer.
     *     We use the interrupted thread's stack (sp was saved to TCB).
     *     context_switch() may clobber any caller-saved register;
     *     that is fine because we restore everything from TCB below.    */
    call    context_switch

/* ── Phase C: Restore new thread's full context ─────────────────────── */

    /* C1. Reload the (now updated) TCB pointer into t6.
     *     x30 is available as scratch (will be restored shortly).       */
    la      x30, current_tcb
    lw      t6, 0(x30)          /* t6 = new current_tcb                  */

    /* C2. Restore mstatus first so mret behaviour is correct.           */
    lw      x30, 132(t6)
    csrw    mstatus, x30        /* restores MPIE, MPP → MIE after mret   */

    /* C3. Restore mepc so mret jumps to the right resume address.       */
    lw      x30, 128(t6)
    csrw    mepc, x30

    /* C4. Restore x1 .. x30.                                            */
    lw      x1,   ( 1*4)(t6)
    lw      x2,   ( 2*4)(t6)    /* sp  – switches to the new thread's stack */
    lw      x3,   ( 3*4)(t6)
    ...
    lw      x29,  (29*4)(t6)
    lw      x30,  (30*4)(t6)    /* restore t5 (x30) last among x1-x30   */

    /* C5. Restore x31 (t6) last – it holds the TCB pointer up to here. */
    lw      x31,  (31*4)(t6)

    /* C6. Return to the new thread.
     *     Hardware atomically:
     *       PC       ← mepc  (new thread's resume address)
     *       MIE      ← MPIE  (re-enables interrupts as the thread left them)
     *       priv     ← MPP   (stays Machine mode)                        */
    mret
```

上面这一段代码就是msi_handler的一个示例实现，这个函数的主要功能就是进行上下文切换。首先在Phase A中，我们保存了当前线程的全部寄存器的值，包括ra、sp、a0-a2、t0-t5以及t6（x31）。我们将t6寄存器的值与mscratch寄存器进行交换，这样就将current_tcb的地址保存在了t6寄存器中，同时将原来的t6寄存器的值保存在了mscratch寄存器中。接下来我们将当前线程的全部寄存器的值保存到当前线程的TCB中，然后将mscratch寄存器恢复为current_tcb的地址。在Phase B中，我们首先清除了MSIP寄存器中的对应位，这样就防止了中断的再次触发。然后我们调用了context_switch函数，这个函数是一个C函数，它的主要功能就是更新current_tcb指针，选择下一个要运行的线程。在Phase C中，我们首先重新加载了current_tcb的地址到t6寄存器中，然后恢复了新线程的mstatus和mepc寄存器的值，最后恢复了新线程的全部寄存器的值，最后使用mret指令返回到新线程的执行状态。

实际上到这一步，我们已经完成了从baremetal到rtos的过渡，接下来我们就可以在main函数中编写我们的应用程序了。在main函数中，我们可以创建线程、发送消息、进行同步等等，这些都是操作系统提供的功能。通过上面的中断处理函数的实现，我们已经为这些功能的实现打下了坚实的基础。

## 外部中断的处理

外部中断与软件中断和定时器中断不同，走的不是CLINT，而是PLIC。外部中断的处理函数的编写也是类似的，主要就是在函数中进行相应的处理，然后在函数结束时使用mret指令返回到中断发生前的状态。一旦理解了上面关于软件中断的处理函数的实现，外部中断的处理函数的实现就不难了。我们需要在外部中断的处理函数中首先保存当前线程的全部寄存器的值，然后调用相应的处理函数进行处理，最后使用mret指令返回到中断发生前的状态。

与软件中断相比，难点主要在于外设本身的使用逻辑上，对于有多个中断外设的情况来说，我们通过PLIC的寄存器来判断是哪个外设触发了中断，然后根据触发的外设调用相应的处理函数进行处理。对于不同的外设来说，它们的处理函数的实现可能会有所不同，因此我们需要根据具体的外设来编写相应的处理函数。

## 总结

在这一篇文章中，我们介绍了中断的相关内容，包括中断向量表的设置、中断处理函数的编写、软件中断和外部中断的处理等等。通过这些内容，我们已经完成了从baremetal到rtos的过渡，接下来我们就可以在main函数中编写我们的应用程序了。在main函数中，我们可以创建线程、发送消息、进行同步等等，这些都是操作系统提供的功能。通过上面的中断处理函数的实现，我们已经为这些功能的实现打下了坚实的基础。在下一篇文章中，我们将会介绍如何在RV32IMAC_zicsr架构上移植RT-Thread Nano操作系统，在移植的过程中，你会发现所使用到的知识在上面我们介绍的内容中都有涉及，因此你可以更好地理解到完成了上面这部分内容地实践实际上正是已经完成了一个最简化RTOS/多线程操作系统的实现了。
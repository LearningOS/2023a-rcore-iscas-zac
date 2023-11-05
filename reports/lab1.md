# 实验报告

## 实现的功能

在 task 包中加入递增系统调用计数的函数 `increase_current_syscall_count()` 和获取任务运行时间与相应系统调用次数的 `get_current_task_info`，在 `TaskControlBlock` 结构体中加入记录任务首次调度时间和各种系统调用次数的数组，在 `process` 模块中实现用结构体返回任务信息的 `sys_task_info()` 函数。

在实现之前，`BASE=0 make run` 产生的报错信息是 `Panicked at src/bin/ch3_taskinfo.rs:21, assertion failed: 3 <= info.syscall_times[SYSCALL_GETTIMEOFDAY]`。这意味着用户代码没有获得合适的任务信息。

## 问答题

1. 出现了
```
[rustsbi] RustSBI version 0.3.1, adapting to RISC-V SBI v1.0.0
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
...
[ERROR] [kernel] .bss [0x8026f000, 0x802a0000)
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003c4, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
```
，可见程序自动捕获了访问 S 态寄存器和 S 态指令产生的 `StoreFault`、 `IllegalInstruction` 错误并输出。

2.
    1. a0 放置着参数的地址，在本程序中是 `trap_handler(cx: &mut TrapContext) -> &mut TrapContext` 函数的返回值，也是 `__restore` 的输入，是一个 `&mut TrapContext` 类型的值。在因为系统调度而调用 `trap_handler` 之后，内核自动使用 `__restore` 恢复至 Trap 前的上下文，那么 a0 和 sp 一样都保存着用户态的栈顶信息。而在程序结束或异常终止之后，通过 `run_next_app` 提供下一个应用的上下文，再调用 `__switch` 和 `__restore` 来进入下一个应用的执行任务。

    2. 处理了 t0、t1、t2、sstatus、sepc、sscratch 这些寄存器，前三个是用来中转的通用寄存器，sstatus 是用来给出 Trap 发生时特权级信息的寄存器，sepc 是 Trap 发生时最后一条指令地址，sscratch 保存 Trap 发生时的栈顶，它们记录了用户程序的上下文，可以从 Trap 继续执行用户程序。

    3. 因为保存时跳过了 sp(x2) 和 tp(x4) 两个寄存器，tp 无需用到， sp 稍后通过 `csrrw sp, sscratch, sp` 恢复。

    4. 这条指令交换了 sp 和 sscratch 中的值，使 sp 指向用户栈顶，sscratch指向内核栈顶。

    5. `sret`，从[第二章](http://learningos.cn/rCore-Tutorial-Guide-2023A/chapter2/4trap-handling.html)可知，该指令结束了汇编代码的执行，回到保存的用户态代码上下文并继续执行。

    6. 这条指令交换了 sp 和 sscratch 中的值，使 sp 指向内核栈顶，sscratch指向用户栈顶。

    7. 以内核 Rust 程序为 S 态，是通过 `call trap_handler` 完成的。如果以汇编为 S 态，那么在 __alltraps 汇编函数就已经发生了。

## 反馈意见

在实现中，出现了 `sys_task_info()` 函数已经在输入结构体的地址上修改了系统调用次数而用户无法读取的情况，检查后发现是因为修改了 syscall_times 数组为 usize，而用户库中没有修改的缘故，改回 u32 通过了。也出现了时间计算出问题（一直小于 1 毫秒）的情况，检查发现是因为我每次调度下一个任务执行时都会覆写首次调度任务时间，通过判断是否为 0（为 0 时需要覆写）解决。

能否加入说明用户程序中 println “为什么 write 调用是两次”的解释？

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

无

2. 此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

[rCore 秋季课程文档](http://learningos.cn/rCore-Tutorial-Guide-2023A/honorcode.html)
[11 月 1 日作业答疑](https://sjodqtoogh.feishu.cn/docx/PQ4fd6LS9oTAeBxCnU8cQXisn7f)

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
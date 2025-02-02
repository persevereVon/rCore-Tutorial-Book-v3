引言
===========================================

本章导读
-------------------------------------------

在正式开始这一章的介绍之前，我们很高兴告诉读者：在前面的章节中基本涵盖了一个功能相对完善的内核抽象所需的所有硬件机制，而从本章开始我们所做的主要是一些软件上的工作，这会略微轻松一些。

在前面的章节中，随着应用的需求逐渐变得复杂，作为其执行环境的内核也需要在硬件提供的相关机制的支持之下努力为应用提供更多强大、易用且安全的抽象。让我们先来简单回顾一下：

- 第一章《RV64 裸机应用》中，由于我们从始至终只需运行一个应用，这时我们的内核看起来只是一个 **函数库** ，它会对应用的执行环境进行初始化，包括设置函数调用栈的位置使得应用能够正确使用内存。此外，它还将 SBI 接口函数进行了封装使得应用更容易使用这些功能。
- 第二章《批处理系统》中，我们需要自动加载并执行一个固定序列内的多个应用，当一个应用出错或者正常退出之后则切换到下一个。为了让这个流程能够稳定进行而不至于被某个应用的错误所破坏，内核需要借助硬件提供的 **特权级机制** 将应用代码放在 U 特权级执行，并对它的行为进行限制。一旦应用出现错误或者请求一些只有内核才能提供的服务时，控制权会移交给内核并对该 **Trap** 进行处理。
- 第三章《多道程序与分时多任务》中，出于一些对于总体性能或者交互性的要求，从 CPU 的角度看，它在执行一个应用一段时间之后，会暂停这个应用并切换出去，等到之后切换回来的时候再继续执行。其核心机制就是 **任务切换** 。对于每个应用来说，它会认为自己始终独占一个 CPU ，不过这只是内核对 CPU 资源的恰当抽象给它带来的一种幻象。
- 第四章《地址空间》中，我们利用一种经典的抽象—— **地址空间** 来代替先前对于物理内存的直接访问。这样做使得每个应用独占一个访存空间并与其他应用隔离起来，并由内核和硬件机制保证不同应用的数据被实际存放在物理内存上的位置也不相交。于是开发者在开发应用的时候无需顾及其他应用，整个系统的安全性也得到了一定保证。

事实上，由于我们还没有充分发掘这些抽象的能力，应用的开发和使用仍然比较受限，且用户在应用运行过程中的灵活性和交互性不够强，这尤其体现在交互能力上。目前为止，所有的应用都是在内核初始化阶段被一并加载到内存中的，之后也无法对应用进行动态增删，从用户的角度来看这和第二章的批处理系统似乎并没有什么不同。

.. _term-terminal:
.. _term-command-line:

于是，本章我们会开发一个用户 **终端** (Terminal) 或称 **命令行** 应用(Command Line Application, 俗称 **Shell** ) ，形成用户与操作系统进行交互的命令行界面(Command Line Interface)，它就和我们今天常用的 OS 中的命令行应用（如 Linux中的bash，Windows中的CMD等）没有什么不同：只需在其中输入命令即可启动或杀死应用，或者监控系统的运行状况。这自然是现代 OS 中不可缺少的一部分，并大大增加了系统的 **可交互性** ，使得用户可以更加灵活地控制系统。

为了方便开发，我们需要在已有抽象的基础上引入一个新的抽象：进程，还需要实现若干基于进程的功能强大的系统调用。

.. note::

   **任务和进程的关系与区别**

   第三章提到的 **任务** 和这里提到的 **进程** 有何关系和区别？ 这需要从二者对资源的占用和执行的过程这两个方面来进行分析。

   任务和进程都是一个程序的执行过程，或表示了一个运行的程序；都是能够被操作系统打断并通过切换来分时占用CPU资源；都需要 **地址空间** 来放置代码和数据；都有从开始运行到结束运行这样的生命周期。

   第三章提到的 **任务** 是这里提到的 **进程** 的初级阶段，还没进化到拥有更强大的动态变化的功能：进程可以在运行的过程中，创建 **子进程** 、 用新的 **程序** 内容覆盖已有的 **程序** 内容、可管理更多的 物理或虚拟的 **资源** 。
 


实践体验
-------------------------------------------

获取本章代码：

.. code-block:: console

   $ git clone https://github.com/rcore-os/rCore-Tutorial-v3.git
   $ cd rCore-Tutorial-v3
   $ git checkout ch5

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ cd os
   $ make run

将 Maix 系列开发板连接到 PC，并在上面运行本章代码：

.. code-block:: console

   $ cd os
   $ make run BOARD=k210

待内核初始化完毕之后，将在屏幕上打印可用的应用列表并进入用户终端（以 K210 平台为例）：

.. code-block::

   [rustsbi] RustSBI version 0.1.1
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

   [rustsbi] Platform: K210 (Version 0.1.0)
   [rustsbi] misa: RV64ACDFIMSU
   [rustsbi] mideleg: 0x22
   [rustsbi] medeleg: 0x1ab
   [rustsbi] Kernel entry: 0x80020000
   [kernel] Hello, world!
   last 808 Physical Frames.
   .text [0x80020000, 0x8002e000)
   .rodata [0x8002e000, 0x80032000)
   .data [0x80032000, 0x800c7000)
   .bss [0x800c7000, 0x802d8000)
   mapping .text section
   mapping .rodata section
   mapping .data section
   mapping .bss section
   mapping physical memory
   remap_test passed!
   after initproc!
   /**** APPS ****
   exit
   fantastic_text
   forktest
   forktest2
   forktest_simple
   forktree
   hello_world
   initproc
   matrix
   sleep
   sleep_simple
   stack_overflow
   user_shell
   usertests
   yield
   **************/
   Rust user shell
   >>  

其中 ``usertests`` 打包了很多应用，只要执行它就能够自动执行一系列应用。

只需输入应用的名称并回车即可在系统中执行该应用。如果输入错误的话可以使用退格键 (Backspace) 。以应用 ``exit`` 为例：

.. code-block::

    >> exit
    I am the parent. Forking the child...
    I am the child.
    I am parent, fork a child pid 3
    I am the parent, waiting now..
    waitpid 3 ok.
    exit pass.
    Shell: Process 2 exited with code 0
    >> 

当应用执行完毕后，将继续回到用户终端的命令输入模式。

本章代码树
--------------------------------------

.. code-block::
   :linenos:

   ├── bootloader
   │   ├── rustsbi-k210.bin
   │   └── rustsbi-qemu.bin
   ├── LICENSE
   ├── os
   │   ├── build.rs(修改：基于应用名的应用链接器)
   │   ├── Cargo.toml
   │   ├── Makefile
   │   └── src
   │       ├── config.rs
   │       ├── console.rs
   │       ├── entry.asm
   │       ├── lang_items.rs
   │       ├── link_app.S
   │       ├── linker-k210.ld
   │       ├── linker-qemu.ld
   │       ├── loader.rs(修改：基于应用名的应用加载器)
   │       ├── main.rs(修改)
   │       ├── mm(修改：为了支持本章的系统调用对此模块做若干增强)
   │       │   ├── address.rs
   │       │   ├── frame_allocator.rs
   │       │   ├── heap_allocator.rs
   │       │   ├── memory_set.rs
   │       │   ├── mod.rs
   │       │   └── page_table.rs
   │       ├── sbi.rs
   │       ├── syscall
   │       │   ├── fs.rs(修改：新增 sys_read)
   │       │   ├── mod.rs(修改：新的系统调用的分发处理)
   │       │   └── process.rs（修改：新增 sys_getpid/fork/exec/waitpid）
   │       ├── task
   │       │   ├── context.rs
   │       │   ├── manager.rs(新增：任务管理器，为上一章任务管理器功能的一部分)
   │       │   ├── mod.rs(修改：调整原来的接口实现以支持进程)
   │       │   ├── pid.rs(新增：进程标识符和内核栈的 Rust 抽象)
   │       │   ├── processor.rs(新增：处理器监视器，为上一章任务管理器功能的一部分)
   │       │   ├── switch.rs
   │       │   ├── switch.S
   │       │   └── task.rs(修改：支持进程机制的任务控制块)
   │       ├── timer.rs
   │       └── trap
   │           ├── context.rs
   │           ├── mod.rs(修改：对于系统调用的实现进行修改以支持进程系统调用)
   │           └── trap.S
   ├── README.md
   ├── rust-toolchain
   ├── tools
   │   ├── kflash.py
   │   ├── LICENSE
   │   ├── package.json
   │   ├── README.rst
   │   └── setup.py
   └── user(对于用户库 user_lib 进行修改，替换了一套新的测例)
      ├── Cargo.toml
      ├── Makefile
      └── src
         ├── bin
         │   ├── exit.rs
         │   ├── fantastic_text.rs
         │   ├── forktest2.rs
         │   ├── forktest.rs
         │   ├── forktest_simple.rs
         │   ├── forktree.rs
         │   ├── hello_world.rs
         │   ├── initproc.rs
         │   ├── matrix.rs
         │   ├── sleep.rs
         │   ├── sleep_simple.rs
         │   ├── stack_overflow.rs
         │   ├── user_shell.rs
         │   ├── usertests.rs
         │   └── yield.rs
         ├── console.rs
         ├── lang_items.rs
         ├── lib.rs
         ├── linker.ld
         └── syscall.rs
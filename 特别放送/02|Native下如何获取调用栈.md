# Native下如何获取调用栈

你好，我是 simsun，曾在微信从事 Android 开发，也是开源爱好者、Rust 语言“铁粉”。应绍文邀请，很高兴可以在“高手课”里和你分享一些编译方面的底层知识。
当我们在调试 Native 崩溃或者在做 profiling 的时候是十分依赖 backtrace 的，高质量的 backtrace 可以大大减少我们修复崩溃的时间。但你是否了解系统是如何生成 backtrace 的呢？今天我们就来探索一下 backtrace 背后的故事。
下面是一个常见的 Native 崩溃。通常崩溃本身并没有任何 backtrace 信息，可以直接获得的就是当前寄存器的值，但显然 backtrace 才是能够帮助我们修复 Bug 的关键。
pid: 4637, tid: 4637, name: crasher  >>> crasher <<<
signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
Abort message: 'some_file.c:123: some_function: assertion "false" failed'
    r0  00000000  r1  0000121d  r2  00000006  r3  00000008
    r4  0000121d  r5  0000121d  r6  ffb44a1c  r7  0000010c
    r8  00000000  r9  00000000  r10 00000000  r11 00000000
    ip  ffb44c20  sp  ffb44a08  lr  eace2b0b  pc  eace2b16
backtrace:
    #00 pc 0001cb16  /system/lib/libc.so (abort+57)
    #01 pc 0001cd8f  /system/lib/libc.so (__assert2+22)
    #02 pc 00001531  /system/bin/crasher (do_action+764)
    #03 pc 00002301  /system/bin/crasher (main+68)
    #04 pc 0008a809  /system/lib/libc.so (__libc_init+48)
    #05 pc 00001097  /system/bin/crasher (_start_main+38)
在阅读后面的内容之前，你可以先给自己 2 分钟时间，思考一下系统是如何生成 backtrace 的呢？我们通常把生成 backtrace 的过程叫作 unwind，unwind 看似和我们平时开发并没有什么关系，但其实很多功能都是依赖 unwind 的。举个例子，比如你要绘制火焰图或者是在崩溃发生时得到 backtrace，都需要依赖 unwind。
书本中的 unwind
1. 函数调用过程
如果你在大学时期修过汇编原理这门课程，相信你会对下面的内容还有印象。下图就是一个非常标准的函数调用的过程。
首先假设我们处于函数 main() 并准备调用函数 foo()，调用方会按倒序压入参数。此时第一个参数会在调用栈栈顶。
调用 invoke foo() 伪指令，压入当前寄存器 EIP 的值到栈，然后载入函数 foo() 的地址到 EIP。
此时，由于我们已经更改了 EIP 的值（为 foo() 的地址），相当于我们已经进入了函数 foo()。在执行一个函数之前，编译器都会给每个函数写一段序言（prologue），这里会压入旧的 EBP 值，并赋予当前 EBP 和 ESP 新的值，从而形成新的一个函数栈。
下一步就行执行函数 foo() 本身的代码了。
结束执行函数 foo() 并准备返回，这里编译器也会给每个函数插入一段尾声（epilogue）用于恢复调用方的 ESP 和 EBP 来重建之前函数的栈和恢复寄存器。
执行返回指令（ret），被调用函数的尾声（epilogue）已经恢复了 EBP 和 ESP，然后我们可以从被恢复的栈中依次 pop 出 EIP、所有的参数以及被暂存的寄存器的值。
读到这里，相信如果没有学过汇编原理的同学肯定会有一些懵，我来解释一下上面提到的寄存器缩写的具体含义，上述命名均使用了 x86 的命名方式。讲这些是希望你对函数调用有一个初步的理解，其中有很多细节在不同体系结构、不同编译器上的行为都有所区别，所以请你放松心情，跟我一起继续向后看。
EBP：基址指针寄存器，指向栈帧的底部。
在 ARM 体系结构中，R11（ARM code）或者 R7（Thumb code）起到了类似的作用。在 ARM64 中，此寄存器为 X29。
ESP：栈指针寄存器，指向栈帧的栈顶 ， 在 ARM 下寄存器为 R13。
EIP：指令寄存器，存储的是 CPU 下次要执行的指令的地址，ARM 下为 PC，寄存器为 R15。
2. 恢复调用帧
如果我们把上述过程缩小，站在更高一层视角去看，所有的函数调用栈都会形成调用帧（stack frame），每一个帧中都保存了足够的信息可以恢复调用函数的栈帧。
我们这里忽略掉其他不相关的细节，重点关注一下 EBP、ESP 和 EIP。你可以看到 EBP 和 ESP 分别指向执行函数栈的栈底和栈顶。每次函数调用都会保存 EBP 和 EIP 用于在返回时恢复函数栈帧。这里所有被保存的 EBP 就像一个链表指针，不断地指向调用函数的 EBP。 这样我们就可以此为线索，十分优雅地恢复整个调用栈。
这里我们可以用下面的伪代码来恢复调用栈：
void debugger::print_backtrace() {
    auto curr_func = get_func_from_pc(get_pc());
    output_frame(curr_func);
    auto frame_pointer = get_register_value(m_pid, reg::rbp);
    auto return_address = read_mem(frame_pointer+8);
    while (dwarf::at_name(curr_func) != "main") {
        curr_func = get_func_from_pc(ret_addr);
        output_frame(curr_func);
        frame_pointer = read_mem(frame_pointer);
        return_address = read_mem(frame_pointer+8);
    }
但是在 ARM 体系结构中，出于性能的考虑，天才的开发者为了节约 R7/R11 寄存器，使其可以作为通用寄存器来使用，因此无法保证保存足够的信息来形成上述调用栈的（即使你向编译器传入了“-fno-omit-frame-pointer”）。比如下面两种情况，ARM 就会不遵循一般意义上的序言（prologue），感兴趣的同学可以具体查看APCS Doc。
函数为叶函数，即在函数体内再没有任何函数调用。
函数体非常小。
Android 中的 unwind
我们知道大部分 Android 手机使用的都是 ARM 体系结构，那在 Android 中需要如何进行 unwind 呢？我们需要分两种情况分别讨论。
1. Debug 版本 unwind
如果是 Debug 版本，我们可以通过“.debug_frame”（有兴趣的同学可以了解一下DWARF）来帮助我们进行 unwind。这种方法十分高效也十分准确，但缺点是调试信息本身很大，甚至会比程序段（.TEXT 段）更大，所以我们是无法在 Release 版本中包含这个信息的。
DWARF 是一种标准调试信息格式。DWARF 最早与 ELF 文件格式一起设计, 但 DWARF 本身是独立的一种对象文件格式。本身 DAWRF 和 ELF 的名字并没有任何意义（侏儒、精灵，是不是像魔兽世界的角色 :)），后续为了方便宣传，才命名为 Debugging With Attributed Record Formats。引自wiki
2. Release 版本 unwind
对于 Release 版本，系统使用了一种类似“.debug_frame”的段落，是更加紧凑的方法，我们可以称之为 unwind table，具体来说在 x86 和 ARM64 平台上是“.eh_frame”和“.eh_frame_hdr”，在 ARM32 平台上为“.ARM.extab”和“.ARM.exidx”。
由于 ARM32 的标准早于 DWARF 的方法，所有 ARM 使用了自己的实现，不过它们的原理十分接近，后续我们只讨论“.eh_frame”，如果你对 ARM32 的实现特别感兴趣，可以参考ARM-Unwinding-Tutorial。
“.eh_frame section”也是遵循 DWARF 的格式的，但 DWARF 本身十分琐碎也十分复杂，这里我们就不深入进去了，只涉及一些比较浅显的内容，你只需要了解 DAWRF 使用了一个被称为 DI（Debugging Information Entry）的数据结构，去表示每个变量、变量类型和函数等在 debug 程序时需要用到的内容。
“.eh_frame”使用了一种很聪明的方法构成了一个非常大的表，表中包含了每个程序段的地址对应的相应寄存器的值以及返回地址等相关信息，下面就是这张表的示例（你可以使用readelf --debug-dump=frames-interp去查看相应的信息，Release 版中会精简一些信息，但所有帮助我们 unwind 的寄存器信息都会被保留）。
“.eh_frame section”至少包含了一个 CFI（Call Frame Information）。每个 CFI 都包含了两个条目：独立的 CIE（Common Information Entry）和至少一个 FDE（Frame Description Entry）。通常来讲 CFI 都对应一个对象文件，FDE 则描述一个函数。
“.eh_frame_hdr section”包含了一系列属性，除了一些基础的 meta 信息，还包含了一列有序信息（初始地址，指向“.eh_frame”中 FDE 的指针），这些信息按照 function 排序，从而可以使用二分查找加速搜索。
总结
总的来说，unwind 第一个栈帧是最难的，由于 ARM 无法保证会压基址指针寄存器（EBP）进栈，所以我们需要借助一些额外的信息（.eh_frame）来帮助我们得到相应的基址指针寄存器的值。即使如此，生产环境还是会有各种栈破坏，所以还是有许多工作需要做，比如不同的调试器（GDB、LLDB）或者 breakpad 都实现了一些搜索算法去寻找潜在的栈帧，这里我们就不展开讨论了，感兴趣的同学可以查阅相关代码。
扩展阅读
下面给你一些外部链接，你可以阅读 GCC 中实现 unwind 的关键函数，有兴趣的同学可以在调试器中实现自己的 unwinder。
_Unwind_Backtrace
uw_frame_state_for
uw_update_context
uw_update_context_1
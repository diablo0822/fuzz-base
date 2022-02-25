# fuzz-base
Fuzz是什么？
如何做好Fuzz


关于 fuzz 的一些思考
fuzz ，一般翻译成 模糊测试。 这里面包含了两个方面，一是模糊，二是测试。
很多人的以为在于模糊就是fuzz的全部，以为随机化的数据，就是fuzz。其实，random != fuzz.
“测试” 才是核心， 随机化的数据，只是一种测试的手段。

既然是测试，测试什么呢？测试的其实是一种预期。 凡是与预期不一致的，都应该视为“异常”。
这里的“异常”，不一定会导致通常所见的crash，其实表现在逻辑层面，就是一些条件分支，
怎么捕获这些非传统类型的“异常” ， 也是一个好的fuzzer需要考虑的。

通常，这种与预期的不一致，在软件层面，有些表现成为bug ，有一些是设计文档之外的副作用。
bug != vul ,一些漏洞虽然是一些简单的bug , 一些漏洞其实也是功能特性。

这个测试的预期包括，设计上的预期和实现上的预期。
对于小规模的软件，也许能从设计上规避很多问题，但是对于大规模的软件，比如windows，
代码复杂度已经达到了设计者无法预期所有状态的地步。
大规模软件的分工协作开发，由于每个实现者的安全意识和水平不一致，也会导致设计者没有预料的问题。

软件或者说单元之的执行流程，其实就是一个个状态机，数据的输入与输出解析，产生状态的转移。
当转移到某个状态没有被处理，或者处理不正确时，它就产生了前面所说的“异常”。

软件系统的脆弱点，或者说漏洞的触发的源头，其实就在于对于外部数据的解析。
一切的外部输入都是有害的，当对软件黑盒测试的时候，关注点就可以放在数据的整个生命周期上。
数据从哪里产生，怎么传输，在哪里解析，在哪里释放。
在fuzz的时候，我们一般只需要关心在哪里解析就可以了，
只有后期利用的时候，才需要关心在哪里产生等问题。

数据格式越复杂，整个熵值越高，解析就越容易出错。制造足够的复杂度，也是fuzzer的武器，
制造复杂度，最容易想到的就是随机化 。
纯粹的随机化是没什么意义的，往往会被第一层的校验就挡住，所以就需要拆分成最小单元来测试。

所谓的最小测试单元法则，就是不要把整个系统带着一块测试，而是需要测试哪个单元，就把那个单元单独提取出来测试。
这个最小单元粒度上可以是模块，区段，函数，甚至是基本块。
分层的具体体现方法很多，比如对于进程的测试，我们可以注入代码，进行内存测试。
对于内核的测试，我们可以加载驱动，采用直接调用的方式测试。

分层是一个很重要的思路，第三方的单元，无论是代码注入，内核直接调用，其实都是给了黑盒代码一个适配中间层。
让我们的异常数据得以绕过那些不必要的检查，直接传递到解析模块。

比如 syscall 通常有usermode 的stub ,在这些stub里本身进行了很多检查和校验，但其实这些校验是可以绕过的，
直接syscall的方式，就相当于在你的fuzzer代码和目标内核本身，加了一个适配层，辅助数据的传递。

对于内核涉及到格式解析的，我们甚至可以用分层和单元化的思想，单独提取解析部分的代码，
直接在用户态以函数的形式来测试这部分代码，加快速度。

面对一个大型的黑盒系统，fuzzer的时间成本很高 , 所以我们尽可能的节约自己的时间，但是信息收集这部分
的时间必不可少，俗称“踩点” ，了解系统在哪里解析外部的数据，怎样用分层的思想，直接的测试这个子系统。

整个流程就是，

掘黑盒系统里解析外部数据的子系统，
分层化测试这个子系统
获系统由于解析产生的“异常”
分析这些异常。
但由于时间成本的关系， 通常只会关心一些具体的表现，比如 调试器捕获的中断，verifier捕获的BSOD。

好的fuzzer的评判标准。

良好的覆盖率。
覆盖率是后面一切测试的保证，怎么保证覆盖率，又可以说几天了，这里不展开。
匪夷所思与众不同丧心病狂的思路与测试点。
土豪如google和MS，可以拼机器。穷人只能想奇技淫巧。
日志记录与重放系统。 将运行时产生的日志，直接重放，就是一个没有精简过的POC。
非传统异常类型的捕获。
也就是说，将本来不能捕获的异常，想办法知道。也就是 PIN 和 DigTool 的部分。 如果还能即时的反馈给fuzzer系统，那覆盖率就更高了。
自身的流程可控。
fuzzer本身也要对自身的执行状态，高度可控制。以便处理callback类型的fuzz。
js-kernel-fuzzer 目前只做到了第三点，和部分的第四点，还需要更多的改进。
用的还是比较笨的IDA逆向的办法，提高覆盖率，如果你有好办法，恳请告知。

其他的非必须，但是也是很有用的部分。

可拆分成单元测试，以支持分布式的fuzz.
配合重放系统和虚拟机的所做的自动化精简系统。
js-kernel-fuzzer 目前还是笨办法，人工在精简 , 每次都费时费力的精简之后才发给MSRC , 期望有时间能做下这个。
总结，fuzz是一个很有用的方法，也是一种很复杂的思路。
各个厂商自己也越来越重视fuzz , 以后挖掘的难度也会越来越高。
以上，只是自己挖掘的一点浅显的总结。

当你觉得已经理解了fuzz时，再好好想想 , 也许有更多的收获。

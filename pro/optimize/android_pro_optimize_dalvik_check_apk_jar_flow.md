# Android Dalvik检查和优化.apk.jar的流程introduce

dalvik的目标平台是Android这样的小RAM，低速度flash memory，运行标准Linux系统的设备。针对这样的平台特性，要想做到更好，我们需要考虑以下几点：

- 1、为了减少系统的内存使用，字节码可以多进程共享。但出于安全性考虑，这样的字节码不可以编辑。
- 2、为了保证响应速度，加载一个新的APP所需时间尽量少。
- 3、标准Java中把多个类文件分别存放导致了大量的冗余，为了节省APP的占用空间，这个问题要解决。
- 4、加载类的时候解析类的字段成员会导致额外的消耗，如果改成像C一样直接访问会比较好。
- 5、字节码verification很有必要，但很慢，我们需要把验证与APP执行分开。
- 6、字节码optimization(比如指令优化、方法pruning)可以在很大程度上影响执行速度和电池消耗。

标准VM都是程序启动时把每单个的类文件解压放入heap，每个进程都有一份copy。这样的做法在内存占用和时间上面都有损失，但方便了对指令的优化。

现在看看dalvik是怎么做的：

- 1、多个类被集成进单一的DEX文件。
- 2、DEX文件在进程间以只读方式共享。
- 3、byte ordering和word alignment根据local system来做调整。
- 4、字节码verification尽可能提前。
- 5、需要修改字节码的optimization必须提前进行。

这样做的好处在下面一一介绍。

###VM Operation

系统中的应用程序代码以.jar或.apk文件存在。其实它们都是.zip的文档，只不过多了一些文件头信息。DEX文件也就是解压.apk后的classes.dex文件。classes.dex中的字节码是经过压缩处理的，而且文件头部不一定是word aligned，所以不能直接mmap到内存直接执行，而是先解压，然后做一些realignment,optimization,verification操作。下面详细介绍一下这个过程。

###Preparation

做到DEX文件的执行前优化（优化后的DEX叫做ODEX，Optimization DEX），至少有三种方式:

- 1、VM的JIT技术。优化后的文件放在/data/dalvik-cache目录下。这种方式在模拟器和eng模式下编译的系统中有效，只有这两种情况下操作dalvik-cache目录才不会有权限问题。
- 2、安装应用程序时，system installer做优化。这需要dalvik-cache目录的写权限。
- 3、编译系统源码时进行优化。这样优化不会修改jar/apk文件，但会对classes.dex进行优化，优化后的DEX与原文件放在同一个目录下一起写入system image。

系统中的/data/dalvik-cache目录属于system/system，权限是0771。存储在这个目录下的ODEX文件被system和应用程序所属的group拥有，权限是0644。DRM-locked的应用程序使用640权限。底线是你可以读你自己的DEX文件和其它的大多数应用程序，但不能创建、修改或删除它们。

使用JIT和system installer做DEX文件的Preparation要分成三步：

- 1、由system installer创建dalvik-cache文件夹，这个程序运行在有root权限的installd进程中。
- 2、classes.dex被解压出来，并在文件头部预留一些空间存放ODEX头信息。
- 3、为了方便使用和做一些针对特定系统的微调，把它mmap。比如byte-swapping，structure realigning等。我们还会做一些像文件偏移量和数据索引是否越界等方面的基本检查。

编译系统使用一个很复杂的流程来做这些事：启动模拟器，强制对所有相关DEX文件执行JIT优化，最后把优化后的结果从dalvik-cache中提取出来。之所以这样做而不是在PC上面使用一个工具来完成，在后面解释Optimization时可以看到原因。

当代码的byte-swapping和align完成时，我们的preparation就完成了。再做完verification和optimization，最后，我们就会把一些相关计算出来的信息添加到ODEX文件的头部然后开始执行。

###dexopt

其实，如果我们想优化DEX中的类文件的话，最简单最安全的办法就是把所有类加载到VM中然后运行一遍，运行失败的就是没有verification和optimization的。但是，这样会分配一些很难释放的资源。比如，加载本地库时。所以，不能使用运行程序的那个VM来做。

我们的解决方案就是使用dexopt这个程序，它会初始化一个VM，加载DEX文件并执行verification和optimization过程。完成后，进程退出，释放所有资源。这个过程中，也可以多个VM使用同一个DEX。file lock会让dexopt只运行一次。

###verification

字节码的verification过程涉及到每个DEX文件中的所有类和类中的所有方法中的指令。目标就是检查非法指令序列，这样做完以后，运行的时候就不必管了。这个过程中涉及到的许多计算也存在于GC过程中。

出于效率上的考虑，下一节提到的optimization会假设verification已经成功运行通过。默认情况下，dalvik会对所有类进行verification，而只对verification成功的类执行optimization。在进行verification过程中出现失败时，我们不一定会报告（比如在不同的包中调用一个作用范围为包内的类），我们会在执行时抛出一个异常。因为检查每个方法的访问权限很慢。

执行verification成功的类在ODEX文件中有一个flag set，当它们被加载时，就不会再进行verification。linux系统的安全机制会防止这个文件被破坏，但如果你能绕过去，还是能去破坏它的。ODEX文件有一个32-bit的checksum，但只能做一个快速检查。

###Optimization

VM解释器在第一次运行一段代码时会做一些optimization。比如，把常量池引用替换成指向内部数据结构的指针，一些永远成功的操作或固定的代码被替换成更简单的形式。做这些optimization需要的信息有的只能在运行时得到，有的可以推断出来。

dalvik做的optimization包含下面这些：

- 1、对于虚方法的调用，把方法索引修改成vtable索引。
- 2、把field的get/put修改成字节偏移量。把boolean/byte/char/short等类型的变量合并到一个32-bit的形式，更少的代码可以更有效地利用CPU的I-cache。
- 3、把一些大量使用的简单方法进行inline，比如String.length()。这样能减少方法调用的开销。
- 4、删除空方法。
- 5、加入一些计算好的数据。比如，VM需要一个hash table来查找类名字，我们就可以在Optimization阶段进行计算，不用放到DEX加载的时候了。

所有的指令修改都是使用一个Dalvik标准没有定义的指令去替换原有指令。这样，我们就可以让优化和没有优化的指令自由搭配。具体的操作与VM版本有关。

Optimization过程有两个地方需要我们注意：

- 1、VM如果更新的话，vtable索引和字节偏移量可能会更新。
- 2、如果两个DEX互相信赖，而其中一个DEX更新的话，确保优化后的索引和偏移量有效。

###Dependencies and Limitations

优化后的DEX会包含一个它信赖的DEX文件列表，并添加了CRC-32和修改时间。文件列表中包含了dalvik-cache目录下的文件的路径和相应的SHA-1签名。而文件在设备上的timestamp不可信也不能用。另外还有VM版本号。

如果当前DEX所依赖的DEX有更新，我们也需要更新当前DEX。如果我们可以做一个JIT的dexopt调用，更新过程很easy。但如果只能依赖installer daemon，或者这个DEX被装到ODEX中的话，VM只能拒绝它了。

dexopt的输出与平台版本，VM版本有关，想编写一个运行在PC上，而优化后的输出在其它设备使用的dexopt很难。因此，dexopt是在目标设备上或者目标设备的模拟器上运行。
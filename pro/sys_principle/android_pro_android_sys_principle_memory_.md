# Android手机内存的运行机制

使用android手机的用户可能都安装了任务管理的软件，使用android手机真的有必要安装结束任务的软件吗？大家在使用中也都发现了，很多软件在被结束后，马上就会又出现在任务列表里，或是稍等一会自己也会出现，任务管理不停的结束后台程序，也没见给手机的运行速度带来多少提升，这是为什么呢？

其实大家不用那么在意android手机剩余内存的大小。很多人都是把使用其他系统的习惯带到了android手机上，不是所有的智能手机系统都一样的。android大多数应用没有退出的设计其实是有道理的，这和系统对进程的调度机制有关系。如果你知道java，就能更清楚这机制了。其实和java的垃圾回收机制类似，系统有一个规则来回收内存。进行内存调度有个阀值，只有低于这个值系统才会按一个列表来关闭用户不需要的东西。当然这个值默认设置得很小，所以你会看到内存老在很少的数值徘徊。但事实上他并不影响速度。相反加快了下次启动应用的速度。这本来也是android的优势之一，如果人为去关闭进程，没有太大必要。特别是自动关进程的软件。

可能有人会说了，那为什么内存少的时候运行大型程序会慢呢？其实很简单，在内存剩余不多时打开大型程序，会触发系统自身的调进程调度策略，这是十分消耗系统资源的操作，特别是在一个程序频繁向系统申请内存的时候。这种情况下系统并不会关闭所有打开的进程，而是选择性关闭，频繁的调度自然会拖慢系统。

那么，进程管理软件到底还有存在的价值吗？其实还是有的，在运行大型程序之前，你可以手动关闭一些进程释放内存，可以显著的提高运行速度。但一些小程序，完全可交由系统自己管理。很多朋友还有个疑问，如果不关程序是不是会更耗电？这里也解释一下，android的应用在被切换到后台时，它其实已经被暂停了，并不会消耗cpu资源，只保留了运行状态。所以为什么有的程序切出去重新进入，还会到主界面。但是，一个程序如果想要在后台处理些东西，如音乐播放，它就会开启一个服务，服务可在后台持续运行，所以在后台耗电的也只有带服务的应用了。这个在进程管理软件里能看到，名字是service。所以没有带服务的应用在后台是完全不耗电的，没有必要关闭。这种设计本来就是一个非常好的设计，下次启动程序时，会更快，因为不需要读取界面资源，何必要关掉他们抹杀这个android的优点呢？

还有一点，为什么android应用看起来那么耗内存？大家知道，android上的应用是java，当然需要虚拟机，而android上的应用是带有独立虚拟机的，也就是每开一个应用就会打开一个独立的虚拟机。这样设计的原因是可以避免虚拟机崩溃导致整个系统崩溃，但代价就是需要更多内存。

至于为什么开了大程序或者开了好几个程序之后切换会变慢，具体分析如下:

已经开启了一个大程序，占用70%内存，如果再想运行一个程序，此时还需要50%的内存，则就需要一个从大程序占用的内存中释放或者压缩的过程，所以表现出来的就是慢一会儿。

已经开启了几个程序共占用内存80%，运行新程序时又需要20%的内存，系统内存因为没见过剩余0的时候，也就是应该剩一部分空闲内存，那么就需要从之前开启的这几个程序中选择一个或者几个来关闭，这一过程也需要耗费系统资源，所以会慢一会儿。也就是说你手动去结束程序的时候，就是替系统在释放内存，就算你不去结束，在需要内存的时候系统也会自动结束程序释放内存。

不在后台运行的程序（没服务的），即使不结束也不会耗电。在后台运行的（有服务的）程序，如一些播放器或实时监控的软件，自然会耗电。这就说明结束进程并不是没用，我们只需要看哪个带服务耗电哪个程序后台一直在运行，看服务就能看出来，这样的软件如果用不到的时候就结束了吧。

以QQ举例，正常的退出，会在进程管理里留下qq的运行过的状态，但不耗电不占cpu，如果你只是切换出去（按房子键而不是退出）那么自然会耗电，因为程序还在运行，QQ还在线呢。

这里就有个要注意的地方了，虽然房子键和那个返回键都可以将程序切换出去，但是两者的效果差异是很大的，返回键可以视作程序已经退出了，而按房子键，则是将程序切换到了后台来运行，软件并没有退出哦！

以上这些设计都是为了确保了android的稳定性，正常情况下最多单个程序崩溃，但整个系统不会崩溃，也永远没有内存不足的提示出现。大家可能是被windows毒害得太深了，总想保留更多的内存，但实际上这并不一定会提升速度，相反却丧失了程序启动快的这一系统特色，得不偿失。大家不妨换种观念习惯来使用android系统。
# reuseport_example
SO_REUSEPORT example, with a udp server and a udp client.

FROM http://blog.csdn.net/c359719435/article/details/51721902
Linux端口重用SO_REUSEPORT使用详解

最近有个我写的模块，性能有问题，晚高峰cpu总是很高。分析了一下，觉得问题可能出现在线程模型上。之前的线程模型是：1个listener线程+N个worker线程，listener线程收到客户端请求，必须跟某个worker线程有一次交互，这次交互通过线程间的队列实现，会有锁的开销。当并发量大的时候，锁的开销会影响整个程序的性能。几年前用过SO_REUSEPORT，所以用Linux的这个特性改了一下代码，果然效果很明显，整个程序的CPU使用率降低了50%左右。本文用于记录SO_REUSEPORT的使用，以备不时之需。
端口重用SO_REUSEPORT介绍
使用详解
SO_REUSEPORT和SO_REUSEADDR
参考文献
端口重用SO_REUSEPORT介绍

Linux kernel 3.9之前的版本，一个ip+port组合，只能被监听bind一次。这样在多核环境下，往往只能有一个线程（或者进程）是listener，在高并发情况下，往往这就是性能瓶颈。于是Linux kernel 3.9之后，Linux推出了端口重用SO_REUSEPORT选项。 
SO_REUSEPORT允许多个线程/进程，绑定在同一个端口上。这样的话，多个socket可以同时bind同一个tcp/udp端口（ip+port组合）。同时内核保证多个这样的socket的负载均衡。其核心实现有一下3点：

扩展 socket option，增加 SO_REUSEPORT 选项，用来设置 reuseport。
修改 bind 系统调用实现，以便支持可以绑定到相同的 IP 和端口。
修改处理新建连接的实现，查找 listener 的时候，能够支持在监听相同 IP 和端口的多个 sock 之间均衡选择。且同一个源ip和端口的所有包，必须要发送到同一个listener。
使用详解

使用起来很简单，无论tcp还是udp，bind之前，通过setsockopt设定一下SO_REUSEPORT即可。代码可能像下面一样：

int opt_val = 1;
if(setsockopt(mSockFD, SOL_SOCKET, SO_REUSEPORT, &opt_val, sizeof(opt_val))){
    cout << "set reuseport error:  " << errno << endl;
        return -errno;
        }
        对于服务器的线程模型，用上SO_REUSEPORT后，就变得简单了。直接起N个worker线程，每个线程创建socket，通过类似以上的代码设定SO_REUSEPORT后，bind到同一个ip+port，然后每个worker线程收到请求处理请求即可。省掉了单独的listener线程以及线程之间的队列、锁等逻辑，简单、干净、利落。

        一个简单的代码例子reuseport_example
        SO_REUSEPORT和SO_REUSEADDR

        从字面意思理解，SO_REUSEPORT是端口重用，SO_REUSEADDR是地址重用。两者的区别：

        （1）SO_REUSEPORT是允许多个socket绑定到同一个ip+port上。SO_REUSEADDR用于对TCP套接字处于TIME_WAIT状态下的socket，才可以重复绑定使用。

        （2）两者使用场景完全不同。SO_REUSEADDR这个套接字选项通知内核，如果端口忙，但TCP状态位于TIME_WAIT，可以重用端口。这个一般用于当你的程序停止后想立即重启的时候，如果没有设定这个选项，会报错EADDRINUSE，需要等到TIME_WAIT结束才能重新绑定到同一个ip+port上。而SO_REUSEPORT用于多核环境下，允许多个线程或者进程绑定和监听同一个ip+port，无论UDP、TCP（以及TCP是什么状态）。

        （3）对于多播，两者意义相同。
        参考文献

        [1] 多个进程绑定相同端口的实现分析[Google Patch] 
        [2] SO_REUSEPORT学习笔记 
        [3] Tengine & Nginx性能测试

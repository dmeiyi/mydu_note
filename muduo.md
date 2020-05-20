1. codec的基本功能之一是做TCP分包，不需要继承，没有基类，只要把它当成普通data member来使用P230
2. LengthHeaderCodec:如果只收到了半条消息，那么不会触发消息事件回调，数据会停留在Buffer里（数据已经读到Buffer）等待收到完整的消息在通知处理函数 p230
3. Channel
	+ 每个Channel自始至终只属于一个EventLoop，因此每个Channel对象都属于一个IO线程，每个对象自始至终只负责一个文件描述符（fd）的IO事件分发，但并不拥有fd，在析构函数中也不会close(fd)
	
	+ Channel是IO事件回调的分发器（dispatcher），它在handleEvent()中根据事件的具体类型分别回调ReadCallback、WriteCallback等，
	
	+ Channel也使用boost::function来表示函数回调，muduo用户一般不直接使用Channel，而会使用更上层的封装，如TcpConnection，Channel的生命期由其owner class负责管理，一般是其他class的直接或间接成员
，每一个Channel对象服务于一个文件描述符，，
4. EventLoop：其主要功能是运行循环事件EventLoop::loop() 

5. Poller是EventLoop的间接成员，只供其 owner EventLoop在IO线程调用，因此无需加锁，其生命期与EventLoop相等，Poller并不拥有Channel，Channel在析构之前必须字节unregister(Eventloop::removeChannel)
#### EventLoop类
他是事件循环（反应器Reactor），每个线程只能由一个EventLoop实体，它负责IO和定时器事件的分派。他用TimerQueue作为计时器管理，用Poller作为IO多路复用。TimerQueue底层使用timerfd_系列函数将定时器转换为fd添加到事件循环中，当时间到达后就会自动出发事件，其内部使用set管理一些注册好的Timer，由于set有自动排序功能，所以注册到事件循环的总是第一个需要处理的Timer。Poller是IO多路复用的实现，他是一个抽象类，具体实现由其子类PollPoller/EpollPoller实现，通过虚函数提供回调功能。poll中的updateChannel方法用于注册和更新关注的事件，所有的fd都需要调用它添加到事件循环中。除了用TimeQueue和Poller管理时间事件和IO事件外EventLoop还包含一个任务队列，它用来做一些计算任务，你可以将自己的任务添加到任务队列中，

#### 阻塞非阻塞
+ 阻塞和非阻塞是指进程访问的数据如果尚未就绪，进程是否需要等待，这相当于函数内部的实现区别，也就是未就绪时时直接返回还是等待就绪
#### 同步和异步
+ 同步和异步是指访问数据的机制，同步一般指主动请求并等待IO操作完毕的方式，当数据就绪后在读写的时候必须阻塞。异步则指主动请求数据后便可继续处理其他任务，随后等待IO操作完毕的通知，这可以使进程在数据读写时也不阻塞
#### IO多路复用
+ IO多路复用是指使用一个线程来检查多个文件描述符（socket）的就绪状态，比如调用select和poll函数，传入多个文件描述符，如果有一个文件描述符就绪则返回，否则就阻塞直到超时，得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行（比如用线程池）
+ 一般情况下，IO复用机制需要事件分发器，事件分发器的作用，将那些读写事件源分发给各读写事件的处理者
+ Reactor 同步IO
## epoll
epoll只有epoll_create,epoll_ctl,epoll_wait 3个系统调用。
####  int epoll_create(int size)
+ 创建一个epoll的句柄
####  int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
+ epoll的事件注册函数，它不同于select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。
+ 第一个参数是epoll_create()的返回值。
+ 第二个参数表示动作，用三个宏来表示：
	```
	EPOLL_CTL_ADD：注册新的fd到epfd中；
	EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
	EPOLL_CTL_DEL：从epfd中删除一个fd；
	```
+ 第三个参数是需要监听的fd。
+ 第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
	```
	//保存触发事件的某个文件描述符相关的数据（与具体使用方式有关）
 
	typedef union epoll_data {
		void *ptr;
		int fd;
		__uint32_t u32;
		__uint64_t u64;
	} epoll_data_t;
	//感兴趣的事件和被触发的事件
	struct epoll_event {
		__uint32_t events; /* Epoll events */
		epoll_data_t data; /* User data variable */
	};
	
	events可以是以下几个宏的集合：

		EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
		EPOLLOUT：表示对应的文件描述符可以写；
		EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
		EPOLLERR：表示对应的文件描述符发生错误；
		EPOLLHUP：表示对应的文件描述符被挂断；
		EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
		EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
	```
#### int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
+ 收集在epoll监控的事件中已经发送的事件。参数events是分配好的epoll_event结构体数组，epoll将会把发生的事件赋值到events数组中（events不可以是空指针，内核只负责把数据复制到这个events数组中，不会去帮助我们在用户态中分配内存）。maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。如果函数调用成功，返回对应I/O上已准备好的文件描述符数目，如返回0表示已超时。

### epoll工作原理
+ epoll同样只告知那些就绪的文件描述符，而且当调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可，这里也使用了内存映射（mmap）技术，这样便彻底省掉了这些文件描述符在系统调用时复制的开销。
+ 另一个本质的改进在于epoll采用基于事件的就绪通知方式。在select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。


#### __thread

thread变量每一个线程有一份独立实体，各个线程的值互不干扰。可以用来修饰那些带有全局性且值可能变，但是又不值得用全局变量保护的变量。


### Reactor简述
+ 什么是Reactor
	+ Reactor是一种基于事件驱动的设计模式，即通过回调机制，将事件的接口注册到Reactor，当事件发生之后，就会回调注册的接口
+ Reactor必要组件：
	+ Event Multiplexer事件分发器：即一些I/O复用机制select、poll、epoll等，程序将事件源注册到分发器上，等待事件的出发，做相应的处理
	+ Handle事件源：用于表示一个事件，Linux上是文件描述符
	+ Reactor反应器：用于管理事件的调度及注册删除，当有激活的事件时，则调用回调函数处理，没有则继续事件循环
	+ Event Handler事件处理器：管理已注册事件和的调度，分成不同类型的事件（读/写/定时）当事件发生，调用对应的回调函数处理

### assert
assert这个关键字我们称之为“断言”，当这个关键字后边的条件为假的时候，程序自动崩溃并抛出AssertionError的异常。
什么情况下我们会需要这样的代码呢？
当我们在测试程序的时候就很好用，因为与其让错误的条件导致程序今后莫名其妙地崩溃，不如在错误条件出现的那一瞬间我们实现“自爆”。
一般来说我们可以用Ta再程序中置入检查点，当需要确保程序中的某个条件一定为真才能让程序正常工作的话，assert关键字就非常有用了

#### level trigger
条件触发：是只要满足条件就发生一个io事件； 


#### shutdown   close
（1）两种半关闭状态

①关闭读一半

套接字不再接受数据，套接字接受缓冲区中的现有数据都会被丢弃。

②关闭写一半

称为半关闭

套接字先将当前发送缓冲区中的数据发送完，然后发送TCP正常连接终止序列。

不能对套接字调用任何写函数（协议栈仍然能自动发送FIN的确认信息）。



注：①、②说的缓冲区都是说的内核协议栈缓冲区，不是我们之前研究的Buffer，这是应用层的缓冲区。

（2）close的坏处

#### move
C++ 标准库使用比如vector::push_back 等这类函数时,会对参数的对象进行复制,连数据也会复制.这就会造成对象内存的额外创建, 本来原意是想把参数push_back进去就行了,通过std::move，可以避免不必要的拷贝操作。
std::move是将对象的状态或者所有权从一个对象转移到另一个对象，只是转移，没有内存的搬迁或者内存拷贝所以可以提高利用效率,改善性能.
对指针类型的标准库对象并不需要这么做.

#### boost::function boost::bind
+ boost::function是一个函数包装器，也即一个函数模板，可以用来代替用有相同返回类型，相同参数类型，以及相同参数个数的各个不同的函数
	```
#include<boost/function.hpp>
#include<iostream>
typedef boost::function<int(int ,char)> Func;

int test(int num,char sign)
{
   std::cout<<num<<sign<<std::endl
}

int main()
{
  Func f;
  f=&test;  //or f=test
  f(1,'A');
}
这样在不同的地方用不同的函数来替代 f 可以得到类似于C++中多态的效果。
	```
+ 但是这样有一定的局限性，例如我想实现的线程池需要执行不同的任务，这些任务的返回类型，函数参数个数，参数类型肯定是不同的，所以不能用上面的方法实现，那么怎么样定义一个函数模板来包含各种返回类型，函数参数个数，参数类型不同的各种函数呢？
	```
	typedef boost::function<void()> Func;
	//or
	typedef boost::function<void(void)> Func;
	```
+ void类型的返回类型表示我们可以返回各种不同的类型了，那我们怎么样传入参数呢？可以用boost::bind。
	```
	#include<boost/function.hpp>
	#include<boost/bind.hpp>
	#include<iostream>
	typdef boost::function<void(void)> Func;

	int test(int num)
	{
		std::cout<<"In test"<<std::endl;    
	}

	int main()
	{
		Func f(boost::bind<test,6>);
		f();
	}
	```

##### 问题
1. TCP 计算机网络有几层，HTTP报文  GET /POST区别 
2. 项目工作流程 Eventloop   为什么线程池是固定数量，数量如何确定有计算公式，并发并行有什么区别，多线程一般用来处理什么类型的任务，
什么时候用多线程，死锁情况，线程同步，项目如何优化






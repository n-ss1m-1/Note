# main函数启动


## 启动日志


## 启动底层网络模块


### 创建+构造mainloop


1.初始化一些属性()
2.绑定主线程(One Loop Per Thread)
3.创建Poller_(监听连接请求)
4.创建wakeupfd和wakeupChannel(用于线程间通信)
5.wakeupChannel设置可读回调+设置关心可读事件

#### main thread


#### Poller(监听连接请求)


#### wakeupfd


#### wakeupChannel


wakeupChannel设置可读回调+设置关心可读事件

### 创建+构造TcpServer


1.已封装为EchoServer类
2.EchoServer构造：给TcpServer设置回调(连接状态变化回调、可读写事件回调+通过TcpServer设置线程池大小
▲3.TcpServer的构造：
创建和构造acceptor_+设置新连接回调
创建和构造threadpool_


#### 构造accptor_+设置新连接回调


初始化其成员：
1.mainloop
2.acceptSocket(createNonblocking())
3.acceptChannel
4.设置socket属性+绑定服务器地址结构(main函数初始化并传入)
5.设置Channel可读回调函数(新连接请求)

##### mainloop


##### acceptSocket(非阻塞)


##### acceptChannel


#### 构造threadpool_


threadpool_构造函数：
1.绑定base loop
2.设置启动标志
3.初始化线程数量

注：线程池中不包含base loop的线程

问：为什么eventloopthread使用unique_ptr保存，而subloop使用普通的指针

### 启动TcpServer:start()


threadpool_启动->EventThreadPool::start()
acceptor_启动监听->Acceptor::listen()

注意：
1.先启动线程池，再启动监听器：防止先启动监听+新连接快速到来->无法找到可用的subLoop
2.listen()使用runInLoop()保证在主线程中执行

#### threadpool_启动 EventLoopThreadPool::start(cb)


▲1.创建指定数量的EventLoopThread
▲2.subLoop通过此处调用的EventLoopThread->startLoop()
      创建并作为返回值

##### EventLoopThread的构造(1)


构造函数：
1.创建Thread(非底层thread)，bind绑定设置其回调函数EventLoopThread::threadFunc()
->用于创建subloop



###### Thread的创建(构造函数)


初始化一些值+move设置回调

##### EventLoopThread::startLoop()(2)


父线程执行
▲1.调用Thread::start()创建真正的底层子线程，并让子线程调用其回调threadFunc()
2.使用条件变量(loop不为空) 阻塞等待子线程创建并传递subloop
3.将创建好的subloop通过返回值传递给Pool

###### Thread::start()创建真正的子线程+调用其回调函数(3)


start：(父线程执行)
1.真正地创建底层thread(shared_ptr)->回调仿函数
{
2.子线程获取并将tid设置到封装它的Thread中
(通过CurrentThread类)
(需要信号量实现父子线程同步)
3.子线程调用回调函数threadFunc()
}
4.父线程等待信号量(避免读取到无效tid) 然后销毁信号量

问：为什么使用shared_ptr<thread>

   * EventLoopThread::threadFunc()(4)


子线程执行
1.创建subloop
(子线程通过bind绑定，回调执行相应EventLoopThread对象的方法)
(加锁+条件变量唤醒阻塞在startLoop()的父线程)
通过bind绑定，将创建的loop传递给提前绑定的EventLoopThread对象
2.启动subloop：loop.loop()
3.清理工作

#### acceptor_启动监听 Acceptor::listen()


listen()使用runInLoop()保证在mainloop中执行:
因为acceptor属于mainloop->为保证线程安全
(listen()的acceptChannel调用enableReading() 这需要在主线程对应的poller_上设置监听事件 不能设置到其他eventLoop上)


##### acceptSocket调用listen()


设置同时三次握手的最大连接数

##### acceptChannel调用enableReading() 


设置关心可读事件(有新连接到来) 并挂上监听树

### 启动mainLoop:loop()


while(!quit)循环


#### poll()阻塞 等待事件发生


由EpollPoller具体实现
流程：
1.调用epoll_wait()阻塞监听事件
2.调用fillActiveChannels() 返回有事件发生的Channel

▲注：
此处的channel是通过events_.data.ptr -> activeChannel得到的
该ptr指针是在EpollPoller::update()处初始化的

#### 遍历activeChannel 调用Channel->handleEvent()


##### handleEvent()


尝试将TcpConnection弱引用提升为强引用
->保证在回调期间 该TcpConnection对象始终存活(why保证?)
->调用handleEventWithGuard()

###### handleEventWithGuard()


使用 & 判断发生的事件
并调用相应的回调函数

   * 连接挂起事件(且已没有数据可读)


EPOLLHUP
closeCallback_

      * TcpConnection::handleClose()


         * setState(KDIsconne)


         * channel->disableAll()


设置不监听任何事件

         * shared_from_this()


保证断开连接(回调执行)期间，该TcpConnection对象不会被销毁，让回调能安全执行

         * connectionCallback_


连接状态变化 回调通知应用层

         * closeCallback_


执行关闭连接的回调 
执行的是TcpServer::removeConnection回调方法
(runInLoop 任务队列中执行)

            * TcpServer::removeConnectionInLoop()


1.从连接表中移除
2.在相应的loop中执行TcpConnection::connectDestroyed()
注：执行方式-queueInLoop+bind
-使用queueInLoop是为了让前面的任务都先完成
-使用bind是绑定该连接对象 实现具体的连接销毁

               * connectDestroyed()


1.再次检查状态+相应操作
(同handleClose()的1234)
2.channel->remove()，
将channel从监听树上摘下

注：此处Channel::remove()
->封装了EventLoop::removeChannel(channel)
->封装了EpollPoller::removeChannel()

                  * EpollPoller::removeChannel()


1.erase：从Epoller维护的channelMap中移除
2.检查channel状态index=aAdded，再真正移除：update(EPOLL_CTL_DEL)
3.channel->set_index() 设置为初始状态

   * 错误事件


EPOLLERR
errorCallback_

      * TcpConnection::handleError()


getsockopt()获取更精确的错误原因

   * 新连接事件(也是可读)


EPOLLIN || EPOLLPRI
readCallback_

      * Acceptor::handleRead()


         * acceptorSocket.accept()


1.封装accept()接受连接
2.获取并设置peer地址

         * TcpServer::newConnection()


1.threadPool::getNextLoop() 轮询subLoop 分配新连接的Channel(负载均衡)
2.获取其绑定的本机的ip和port：getsockname()
3.创建TcpConnection对象，管理新连接
4.将TcpConnection加入连接映射表
5.设置TcpConnection回调函数
6.在子线程中执行(runInLoop)连接建立操作:TcpConnection::connectEstablished()

            * TcpConnection的构造


1.绑定subLoop
2.创建+构造Socket
3.创建+构造Channel
4.绑定localAddr和peerAddr
5.初始化一些属性

            * TcpConnection::connectEstablished()


               * TcpConnection::setState()


               * Channel->tie()


绑定Channel和TcpConnection
Channel持有该TcpConnection的weak_ptr
在执行回调时提升为shared_ptr
避免执行回调时被销毁
(什么时候可能会销毁？？？)

               * Channel->enableReading()


设置监听Channel的可读事件

               * connectionCallback_


通知应用层 连接建立完毕

   * 可读事件


EPOLLIN || EPOLLPRI
readCallback_

      * TcpConnection::handleRead()


         * inputBuffer_.readFd()


读取客户发送到服务器的数据

            * TcpConnection::messageCallback_()


业务逻辑：处理客户发送的数据
(此处处理方式：Echo)

注：shared_from_this()保证回调期间，该TcpConnection对象不会被销毁

               * TcpConnection::send()


在当前TcpConnection对应的Loop对应的Thread中---执行SendInLoop()
若不在，则使用runInLoop()+bind()，添加到任务队列

                  * TcpConnection::sendInLoop()


                     * 第一次开始写数据+outputBuffer无数据：::write()直接写


                        * 无剩余数据：回调writeCompleteCallback_，通知上层应用


                     * 还有剩余数据：高水位检查+拷贝到outputBuffer+设置监听EPOLLOUT


                        * 可写：使用handleWrite()发送数据


                           * 无剩余数据：注销EPOLLOUT+回调writeCompleteCallback_


                     * 


   * 可写事件


EPOLLOUT
writeCallback_

注：注册监听EPOLLOUT的前提---sendInLoop需要多次发送数据

      * TcpConnection::handleWrite()


         * outputBuffer_.writeFd()


         * 移动读指针，回收空间


         * 检查outputBuffer，发送完毕：注销监听EPOLLOUT+向对应loop的任务队列添加回调writeCompleteCallback_


#### 处理任务队列


##### EventLoop::doPendingFunctors()


1.lock
▲2.swap：
将任务拷贝出来再执行 而非在临界区执行任务
减少占据临界区的时间
3.遍历任务队列+调用

## 启动LFU缓存


## 启动内存池


# TcpConnection+Buffer补充


# Poller的构造


Poller::newDefaultPoller(this)传出具体对象(Epoller或Poller或Selector)
1.隐藏底层的IO多路复用技术
2.此处使用epoll,在EpollPoller类(继承Poller)的构造函数中调用epoll_create()初始化

# eventfd+任务队列：线程间通信机制(无锁)


eventfd的作用：唤醒loop所属线程 执行任务队列中的任务
注：One Loop Per Thread，Loop的任务队列必须由其相应的Thread执行->避免数据混乱、负载不均衡


## runInLoop(cb)


当前运行的线程欲执行新增到某Loop的任务cb

### isInLoopThread()


检查当前Thread和Loop是否对应绑定

#### 已对应：执行cb


#### 不对应：queueInLoop(cb)


1.(加锁)将cb加入到该Loop的任务队列中
2.wakeup()唤醒对应的线程执行任务

##### wakeup()唤醒线程的机制：


向当前loop的wakeupfd写一字节数据
->poll()监听到有可读事件
->解除阻塞、返回activeChannel
->处理activeChannel的事件
▲->处理任务队列
(具体见启动mainLoop后的流程)

# 切记！现在写的是流程图，关系图另写，细节另外补充


# channel的回调怎么设置的？


TcpConnection的成员函数，在TcpConnection的构造函数中设置。

# Channel的感兴趣事件是什么时候设置的？


TcpConnection::ConnectionEstablished()中设置监听可读事件(在TcpServer::NewConnection末尾 让mainloop执行)。

TcpConnection::sendInLoop()中若要多次写 则将剩余数据写到发送缓冲区中+设置监听可写事件

# acceptChannel与其他Channel的不同点


在TcpServer构造函数中设置新连接回调 。TcpServer.start()中调用Acceptor::listen() 设置关心可读事件

# channel和TcpConnection什么时候绑定的？


TcpConnection：ConnectionEstablised

# 什么时候使用queueInLoop() 什么时候使用runInLoop()


loop_->runInLoop(cb)
立即或异步执行。如果当前线程就是该 EventLoop 的 IO 线程，则立即同步执行回调 cb；否则，将其加入队列并异步执行。
	

loop_->queueInLoop(cb)
总是异步排队。无论当前是否在 IO 线程，都只将回调 cb放入任务队列，绝不立即执行。

runInLoop并非无用武之地。它适用于那些需要立即生效或者执行非常快速的操作。例如，在 TcpConnection::connectEstablished方法中，设置 Channel 的 tie关联（用于生命周期管理）和启用读事件监听（channel_->enableReading()）这类轻量级且关键的操作，就适合潜在的立即执行。

# 为什么像send这样的方法，一定要使用sendInLoop，在对应的线程中执行任务呢：


1.避免数据竞争：多线程写入数据，会导致数据混乱
2.如果使用加锁，则会导致效率低下
此处使用One Loop Per Thread的原则，利用eventfd和任务队列，实现无锁的线程操作
->高效、无误、职责分明

# 补充：什么时候设置的监听这俩？


# Channel::enablexxx()


1.设置Channel的events
2.调用EpollPoller::update()

## update()≈epoll_ctl()


1.此处Channel::update()
->封装了EventLoop::updateChannel(channel)
->封装了EpollPoller::update(channel)

注：EpollPoller::update()
即epoll_ctl的一系列操作
并初始化ptr指针指向相应的Channel对象

# 补充：TcpConnection的状态转换/状态有什么作用？


# 一个Loop  <->  一个Thread <-> 一个Polller <-> 多个Channel


# 为什么此处采用多层封装  而不能让channel组合相应的poller对象呢


# TcpConnection


# Buffer


## CheapPrepend+InialSize


1.缓冲区最前端 初始预留的空间大小 可用于添加协议头
2.默认缓冲区大小

## 构造+析构


## readableBytes()+wtitableBytes()+prependableBytes()


1.获取待读数据长度
2.获取可写空间长度
3.获取前置空间长度(已读 可重新利用)

返回值类型size_t    const函数

## readerIndex_+writerIndex_


1.读指针：指向下一个可读数据位置
2.写指针：指向下一个可写位置

## peek()


返回待读数据的起始地址

## begin()


vector数组首元素的地址 也就是数组的起始地址

## beginWrite()


获取可写空间的起始地址

## ssize_t writeFd(int fd, int *saveErrno)


将buffer上的数据 写到fd

## ssize_t readFd(int fd, int *saveErrno)


从fd上读取数据到buffer

## vector<char> buffer_


## void retrieve(size_t len)


回收已读数据空间：移动读指针

## void retrieveAll()


将所有指针重置到初始位置

## std::string retrieveAllAsString()


调用retrieveAsString()

## std::string retrieveAsString(size_t len)


把onMessage函数上报的Buffer数据 转成string类型的数据返回

## void ensureWritableBytes(size_t len)


确保有足够的可写空间 不足时扩容

## void makeSpace(size_t len)


扩容/腾出空间

## void append(const char *data, size_t len)


把[data, data+len]内存上的数据添加到writable缓冲区当中

# 需要改变某个loop的状态/某个loop的组件的状态/某个loop有任务需要执行->使用下面两个函数 保证线程安全


具体例子：
1.acceptor启动：listen()
2.

# ！！！总结不同的回调的设置、使用的时机


# ！！！不同关键组件的生命周期 例如TcpConnection


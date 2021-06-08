---
title: "ceph网络通信"
date: 2020-07-10T21:53:01+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---


## 网络通信框架

源代码路径src/msg下有三个子目录，分别是Simple、Async、XIO。msg目录下定义了网络通信的抽象框架，它完成了通信接口和具体实现的分离，三个子目录就是其对应的具体实现。

Simple是最早的网络层的实现，当时的网络模型还是线程模型。Simple就是采用的这种线程模型。它最大的特点就是：一个链接的每一端，都会创建两个线程，一个专门用于接收，一个专门用于发送。这种模型实现简单，但对于大规模的集群，创建大量的线程，会非常消耗资源，影响性能。

Async基于事件的I/O多路复用模式，目前网络通信广泛采用的模式。

XIO使用了开源的网络通信库accelio来实现，需要依赖第三方库，前两种方式只支持TCP/IP协议，XIO可以支持Infiniband网络。

| type     		  | position            |   
| --------------- |:------------------- |
| Message         | msg/Message.h 	    |
| ceph_msg_header | include/msgr.h      |
| ceph_msg_footer | include/msgr.h      |
| Connection      | msg/Connection.h    |
| Dispatcher      | msg/Dispatcher.h    |
| Messenger       | msg/Messenger.h     |
| Policy          | msg/Policy.h        |


### Message

类Message是所有消息的基类，格式：header、user_data、footer，header是消息头，user_data是具体发送的数据，footer标志消息结尾。任何要发送的消息，都继承于该类，不同类型的消息定义在messages目录下。

```
class Message : public RefCountedObject {
  ceph_msg_header  header;      // headerelope 消息头
  ceph_msg_footer  footer; //消息尾
  //用户数据
  bufferlist       payload;  // "front" unaligned blob //一般操作相关的元数据
  bufferlist       middle;   // "middle" unaligned blob //预留字段
  bufferlist       data;     // data payload (page-alignment will be preserved where possible) //一般为读写的数据

  /* recv_stamp is set when the Messenger starts reading the
   * Message off the wire */
  utime_t recv_stamp; //开始接收数据的时间戳
  /* dispatch_stamp is set when the Messenger starts calling dispatch() on
   * its endpoints */
  utime_t dispatch_stamp; //dispatch的时间戳
  /* throttle_stamp is the point at which we got throttle */
  utime_t throttle_stamp; //获取throttle的时间戳
  /* time at which message was fully read */
  utime_t recv_complete_stamp; //接收完成的时间戳

  ConnectionRef connection; //网络连接类

  uint32_t magic = 0; //消息的魔术字

  bi::list_member_hook<> dispatch_q; //boost::instrusive需要的字段
}

struct ceph_msg_header {
    __le64 seq;       /* message seq# for this session */ //当前session内消息的唯一序号
    __le64 tid;       /* transaction id */ //消息的全局唯一的id
    __le16 type;      /* message type */ //消息类型
    __le16 priority;  /* priority.  higher value == higher priority */ //优先级
    __le16 version;   /* version of message encoding */ //消息编码的版本
  
    __le32 front_len; /* bytes in main payload */ //payload的长度
    __le32 middle_len;/* bytes in middle payload */ //middle的长度
    __le32 data_len;  /* bytes of data payload */ //data的长度
    __le16 data_off;  /* sender: include full offset; //对象的数据偏移量
                 receiver: mask against ~PAGE_MASK */
  
    struct ceph_entity_name src; //消息源
  
    /* oldest code we think can decode this.  unknown if zero. */
    __le16 compat_version;
    __le16 reserved;
    __le32 crc;       /* header crc32c */
} __attribute__ ((packed));
```
因为payload／middle／data大小一般是变长，因此，为了能正确地解析三者，header中纪录了三者的长度：front_len、middle_len、data_len。

```
struct ceph_msg_footer {     
    __le32 front_crc, middle_crc, data_crc;
    // sig holds the 64 bits of the digital signature for the message PLR
    __le64  sig;
    __u8 flags;
} __attribute__ ((packed));
```

在footer中会计算payload／middle／data的crc，填入front_crc middle_crc和data_crc。

在src/messages下定义了系统具体的消息实现，其都是Message的子类。

### Connection

类Connection对应端到端的socket链接的封装，负责维护socket链接以及ceph的协议栈操作接口，向上提供发送消息接口及转发底层消息给DispacthQueue，向下发送和接收消息。

```
struct Connection : public RefCountedObject {
    mutable Mutex lock;
    Messenger *msgr;
    RefCountedPtr priv; //链接的私有数据
    int peer_type; //链接的peer类型
    int64_t peer_id = -1;  // [msgr2 only] the 0 of osd.0, 4567 or client.4567
    safe_item_history<entity_addrvec_t> peer_addrs; //peer地址
    utime_t last_keepalive, last_keepalive_ack; //最后一次发送keepalive的时间和最后一次接收keepalive的ACK的时间
private:
    uint64_t features; //一些features标志位
public:
    bool failed; // true if we are a lossy connection that has failed. //当值为true时，该链接为lossy(有损)链接已经失效了 

    int rx_buffers_version; //接收缓冲区的版本
    map<ceph_tid_t,pair<bufferlist,int> > rx_buffers; //接收缓冲区。消息的标识ceph_tid_t -> (bufferlist, rx_buffers_version)的映射

    virtual int send_message(Message *m) = 0; //发送消息的接口
};
```

在Simple模式下，PipeConnection继承于Connection，其PipeConnection::send_message(Message *m)调用static_cast<SimpleMessenger*>(msgr)->send_message(m, this);

### Dispatcher

类Dispatcher是消息分发的接口，其分发的接口为：

```
virtual void ms_dispatch(Message *m)；
virtual void ms_fast_dispatch(Message *m);
```

Server端注册该Dispatcher类用于把接收到的Message请求分发到具体处理的应用层。

Client端需要实现一个Dispatcher函数，用于接收收到的ACK应答消息。

### Messenger

Messenger是整个网络抽象模块，定义了网络模块的基本API接口，网络模块对外提供的基本功能，就是能在节点之间发送和接收消息。
向一个节点发送消息的命令：

```
virtual int send_message(Message *m, const entity_inst_t &dst = 0);
void add_dispatcher_head(Dispatcher *d)；
```

```
class Messenger {
private:
    std::deque<Dispatcher*> dispatchers;
    std::deque<Dispatcher*> fast_dispatchers;
    /// the "name" of the local daemon. eg client.99
    entity_name_t my_name;

    /// my addr
    safe_item_history<entity_addrvec_t> my_addrs;
};
```

### Policy

Policy定义了Messenger处理Connection的一些策略：

```
struct Policy {
  /// If true, the Connection is tossed out on errors.
  bool lossy; //如果为true，当该连接出现错误时就删除
  /// If true, the underlying connection can't be re-established from this end.
  bool server; //如果为true，为服务端，都是被动连接
  /// If true, we will standby when idle
  bool standby; //如果为true，该连接处于等待状态
  /// If true, we will try to detect session resets
  bool resetcheck; //如果为true，该连接出错后重连
  /** 
   *  The throttler is used to limit how much data is held by Messages from
   *  the associated Connection(s). When reading in a new Message, the Messenger
   *  will call throttler->throttle() for the size of the new Message.
   */
  //该连接相关的流控操作
  ThrottleType* throttler_bytes;
  ThrottleType* throttler_messages;
  
  /// Specify features supported locally by the endpoint.
  uint64_t features_supported; //本地端的一些feature标志
  /// Specify features any remotes must have to talk to this endpoint.
  uint64_t features_required;//远程端需要的一些feature标志
}
```

### DispatchQueue

DispatchQueue类用于把接收到的请求保存在内部，通过其内部的线程，调用比如SimpleMessenger类注册的Dispatch类的处理函数来处理相应的消息。

```
class DispatchQueue {
    CephContext *cct;
    Messenger *msgr;
    mutable Mutex lock;
    Cond cond;

    PrioritizedQueue<QueueItem, uint64_t> mqueue; //接收消息的优先队列

    std::set<pair<double, Message::ref>> marrival; //接收到的消息集合pair为(recv_time,message)
    map<Message::ref, decltype(marrival)::iterator> marrival_map; //消息->所在集合位置的映射
};
```

其内部的mqueue为优先级队列，用来保存消息，marrival保存了接收到的消息。marrival_map保存消息在集合中的位置。

函数DispatchQueue::enqueue用来把接收到的消息添加到消息队列中，函数DispatchQueue::entry为线程的处理函数，用于处理消息。

### Linux下的socket通信模型

```
int socket(int domain, int type, int protocol); 
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int listen(int sockfd, int backlog);
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

## SimpleMessenger

### SimpleMessenger

类SimpleMessenger实现了Messenger接口。

```
class SimpleMessenger : public SimplePolicyMessenger {
    Accepter accepter; //接受客户端的连接请求
    DispatchQueue dispatch_queue; //接收到的请求的消息分发队列

    friend class Accepter;

    bool did_bind; //是否绑定。如果绑定了具体端口，为ture；如果调用了Accepter::stop()，重置为false
    /// counter for the global seq our connection protocol uses
    __u32 global_seq; //生成全局的消息seq
    /// lock to protect the global_seq
    ceph::spinlock global_seq_lock;

    entity_addr_t my_addr; //地址信息

    /**
     * hash map of addresses to Pipes
     *
     * NOTE: a Pipe* with state CLOSED may still be in the map but is considered
     * invalid and can be replaced by anyone holding the msgr lock
     */
    ceph::unordered_map<entity_addr_t, Pipe*> rank_pipe; //地址->Pipe映射
    /**
     * list of pipes are in the process of accepting
     *
     * These are not yet in the rank_pipe map.
     */
    set<Pipe*> accepting_pipes; //正在处理的Pipes集合
    /// a set of all the Pipes we have which are somehow active
    set<Pipe*>      pipes; //所有的Pipe集合
    /// a list of Pipes we want to tear down
    list<Pipe*>     pipe_reap_queue; //准备释放的Pipe集合

    /// internal cluster protocol version, if any, for talking to entities of the same type.
    int cluster_protocol; //内部集群的协议版本
};
```

### Accepter

类Accepter用来在Server端监听端口，接收链接，它继承了Thread类，本身是一个线程，来不断的监听Server的端口。

对于socket通信，主要做了socket()、bind()、listen()、accept()这些事情。

```
class Accepter : public Thread {
    SimpleMessenger *msgr;
    bool done;
    int listen_sd; //监听的端口
    uint64_t nonce;
    int shutdown_rd_fd;
    int shutdown_wr_fd;
    int create_selfpipe(int *pipe_rd, int *pipe_wr);

    void *entry() override; //继承于Thread，线程的主函数，实现了accept()。
    void stop();
    int bind(const entity_addr_t &bind_addr, const set<int>& avoid_ports); //包括socket()、bind()、listen()。
    int rebind(const set<int>& avoid_port);
    int start();
};
```
SimpleMessenger会调用bind，SimpleMessenger的bind，将这些事情委托给了成员accepter，即调用accepter->bind()，而accepter->bind其实就是做了socket()、bind()、listen()这些事情。具体可参考SimpleMessenger::bind()、Accepter::bind()的具体实现。

socket,bind,listen,都已经有着落了，但是目前还没有accept的踪影。Accepter类继承自Thread，它本质是个线程, class Accepter : public Thread。

Accepter这个线程的主要任务，就是accept，接受到来的连接。但这个线程不会处理客户的通信请求，而是接到请求，创建Pipe来负责处理该请求，然后自己继续accept，等待新的请求。

```
int Accepter::start()
{ 
    ldout(msgr->cct,1) << __func__ << dendl;

    // start thread
    create("ms_accepter");

    return 0;
}

void *Accepter::entry()
{
    ldout(msgr->cct,1) << __func__ << " start" << dendl;

    int errors = 0;

    struct pollfd pfd[2];
    memset(pfd, 0, sizeof(pfd));

    pfd[0].fd = listen_sd;
    pfd[0].events = POLLIN | POLLERR | POLLNVAL | POLLHUP;
    pfd[1].fd = shutdown_rd_fd;
    pfd[1].events = POLLIN | POLLERR | POLLNVAL | POLLHUP;
    while (!done) {
        ldout(msgr->cct,20) << __func__ << " calling poll for sd:" << listen_sd << dendl;
        int r = poll(pfd, 2, -1);
        if (r < 0) {
            if (errno == EINTR) {
                continue;
            }
            ldout(msgr->cct,1) << __func__ << " poll got error"
                << " errno " << errno << " " << cpp_strerror(errno) << dendl;
            ceph_abort();
        }
        ldout(msgr->cct,10) << __func__ << " poll returned oke: " << r << dendl;
        ldout(msgr->cct,20) << __func__ <<  " pfd.revents[0]=" << pfd[0].revents << dendl;
        ldout(msgr->cct,20) << __func__ <<  " pfd.revents[1]=" << pfd[1].revents << dendl;

        if (pfd[0].revents & (POLLERR | POLLNVAL | POLLHUP)) {
            ldout(msgr->cct,1) << __func__ << " poll got errors in revents "
                <<  pfd[0].revents << dendl;
            ceph_abort();
        }
        if (pfd[1].revents & (POLLIN | POLLERR | POLLNVAL | POLLHUP)) {
            // We got "signaled" to exit the poll
            // clean the selfpipe
            char ch;
            if (::read(shutdown_rd_fd, &ch, sizeof(ch)) == -1) {
                if (errno != EAGAIN)
                    ldout(msgr->cct,1) << __func__ << " Cannot read selfpipe: "
                        << " errno " << errno << " " << cpp_strerror(errno) << dendl;
            }
            break;
        }
        if (done) break;

        // accept
        sockaddr_storage ss;
        socklen_t slen = sizeof(ss);
        int sd = accept_cloexec(listen_sd, (sockaddr*)&ss, &slen);
        if (sd >= 0) {
            errors = 0;
            ldout(msgr->cct,10) << __func__ << " incoming on sd " << sd << dendl;

            msgr->add_accept_pipe(sd); //创建pipe,由专门的线程负责与客户端通信，此线程一直accept新的连接
        } else {
            int e = errno;
            ldout(msgr->cct,0) << __func__ << " no incoming connection?  sd = " << sd
                << " errno " << e << " " << cpp_strerror(e) << dendl;
            if (++errors > msgr->cct->_conf->ms_max_accept_failures) {
                lderr(msgr->cct) << "accetper has encoutered enough errors, just do ceph_abort()." << dendl;
                ceph_abort();
            }
        }
    }

    ldout(msgr->cct,20) << __func__ << " closing" << dendl;
    // socket is closed right after the thread has joined.
    // closing it here might race
    if (shutdown_rd_fd >= 0) {
        ::close(shutdown_rd_fd);
        shutdown_rd_fd = -1;
    }

    ldout(msgr->cct,10) << __func__ << " stopping" << dendl;
    return 0;
}

```

这个线程是个死循环，不停的等待客户端的连接请求，收到连接请求后，调用msgr->add_accept_pipe，可以理解为创建pipe，由专门的线程负责与客户端通信，而本线程继续accept。

然后SimpleMessenger的add_accept_pipe函数：

```
Pipe *SimpleMessenger::add_accept_pipe(int sd) 
{
    lock.Lock();
    Pipe *p = new Pipe(this, Pipe::STATE_ACCEPTING, NULL); //创建pipe,接管与Client端的通信,状态为STATE_ACCEPTING
    p->sd = sd; 
    p->pipe_lock.Lock();
    p->start_reader(); //服务端，启动Pipe读线程
    p->pipe_lock.Unlock();
    pipes.insert(p); //放入到SimpleMessenger的pipes和accepting_pipes两个集合中去
    accepting_pipes.insert(p);
    lock.Unlock();
    return p;
}
```

就是说，服务器端，accept之后，创建了一个新的Pipe数据结构，然后讲新的Pipe放入到SimpleMessenger的pipes和accepting_pipes两个集合中去。

此处是服务器端，我们看一些client端如何连接服务器的监听端口，client端的程序一般这样写，参考ceph/src/test/messenger/simple_client.cc：

```
//创建messenger实例
messenger = Messenger::create(g_ceph_context, g_conf().get_val<std::string>("ms_type"),
                entity_name_t::MON(-1),
                "client",
                getpid(), 0);
// enable timing prints
messenger->set_magic(MSG_MAGIC_TRACE_CTR);
messenger->set_default_policy(Messenger::Policy::lossy_client(0));

//创建Dispatcher 类并添加到messenger，用于分发处理消息
dispatcher = new SimpleDispatcher(messenger);
messenger->add_dispatcher_head(dispatcher);
dispatcher->set_active(); // this side is the pinger

//启动messenger
r = messenger->start();

//获得服务器端的连接
conn = messenger->connect_to_mon(dest_addrs);

//发送消息
conn->send_message(m);
```

messenger的connect_to_mon操作，调用了messenger->connect_to，该函数就如同linux socket通信中的connect系统调用，该函数的实现：

```
ConnectionRef SimpleMessenger::connect_to(int type,
        const entity_addrvec_t& addrs)
{
    Mutex::Locker l(lock);
    if (my_addr == addrs.front()) {
        // local
        return local_connection;
    }

    // remote
    while (true) {
        Pipe *pipe = _lookup_pipe(addrs.legacy_addr());
        if (pipe) {
            ldout(cct, 10) << "get_connection " << addrs << " existing " << pipe << dendl;
        } else {
            pipe = connect_rank(addrs.legacy_addr(), type, NULL, NULL);
            ldout(cct, 10) << "get_connection " << addrs << " new " << pipe << dendl;
        }   
        Mutex::Locker l(pipe->pipe_lock);
        if (pipe->connection_state)
            return pipe->connection_state;
        // we failed too quickly!  retry.  FIXME.
    }
}
```

首先是尝试查找已经存在的Pipe(通过_lookup_pipe)，如果可以复用，就不再创建，否则就调用connect_rank来创建Pipe，如下所示：

```
Pipe *SimpleMessenger::connect_rank(const entity_addr_t& addr,
        int type,
        PipeConnection *con,
        Message *first)
{
    ceph_assert(lock.is_locked());
    ceph_assert(addr != my_addr);

    ldout(cct,10) << "connect_rank to " << addr << ", creating pipe and registering" << dendl;

    // create pipe //创建pipe，并且把Pipe的状态设置为STATE_CONNECTING
    Pipe *pipe = new Pipe(this, Pipe::STATE_CONNECTING,
            static_cast<PipeConnection*>(con));
    pipe->pipe_lock.Lock();
    pipe->set_peer_type(type);
    pipe->set_peer_addr(addr);
    pipe->policy = get_policy(type);
    pipe->start_writer(); //客户端，启动Pipe写线程
    if (first)
        pipe->_send(first);
    pipe->pipe_lock.Unlock();
    pipe->register_pipe();
    pipes.insert(pipe);

    return pipe;
}
```

connect_to是client的行为，类似于socket通信中的connect系统调用，它真正通信之前，创建了Pipe数据结构。而再回到服务器端，服务器端，accept收到连接请求之后，立刻创建了Pipe，它就返回了，继续accept，等待新的连接。所以，Pipe才是通信的双方。


### Pipe

类Pipe实现了PipeConnection的接口，它实现了两个端口之间的类似管道的功能。

对于每一个Pipe，内部都有一个Reader和一个Writer线程，分别用来处理这个Pipe有关的消息接收和请求的发送。

```
class Pipe : public RefCountedObject {
    /** 
     * The Reader thread handles all reads off the socket -- not just
     * Messages, but also acks and other protocol bits (excepting startup,
     * when the Writer does a couple of reads).
     * All the work is implemented in Pipe itself, of course.
    */
    class Reader : public Thread {
      Pipe *pipe;
    public:
      explicit Reader(Pipe *p) : pipe(p) {}
      void *entry() override { pipe->reader(); return 0; }
    } reader_thread; //接收线程，用于接收数据

    /** 
     * The Writer thread handles all writes to the socket (after startup).
     * All the work is implemented in Pipe itself, of course.
     */
    class Writer : public Thread {
      Pipe *pipe;
    public:
      explicit Writer(Pipe *p) : pipe(p) {}
      void *entry() override { pipe->writer(); return 0; }
    } writer_thread; //发送线程，用于发送数据

    class DelayedDelivery;
    DelayedDelivery *delay_thread;

    SimpleMessenger *msgr; //msgr的指针
    uint64_t conn_id; //分配给pipe自己唯一的id

    char *recv_buf; //接收缓冲区
    size_t recv_max_prefetch; //接收缓冲区一次预取的最大值
    size_t recv_ofs; //接收的偏移量
    size_t recv_len; //接收的长度

private:
    int sd; //pipe对应的socket fd
    struct iovec msgvec[SM_IOV_MAX]; //发送消息的iovec结构

public:
    int port; //链接端口
    int peer_type; //链接对方的类型
    entity_addr_t peer_addr; //对方地址
    Messenger::Policy policy; //策略

    Mutex pipe_lock;
    int state; //当前链接的状态
    std::atomic<bool> state_closed = { false }; // true iff state = STATE_CLOSED //true，如果为STATE_CLOSED

    // session_security handles any signatures or encryptions required for this pipe's msgs. PLR

    std::shared_ptr<AuthSessionHandler> session_security;

protected:
    friend class SimpleMessenger;
    PipeConnectionRef connection_state; //PipeConnection的引用

    utime_t backoff;         // backoff time //backoff时间

    bool reader_running, reader_needs_join;
    bool reader_dispatching; /// reader thread is dispatching without pipe_lock
    bool notify_on_dispatch_done; /// something wants a signal when dispatch done
    bool writer_running;

    map<int, list<Message*> > out_q;  // priority queue for outbound msgs //准备发送的消息优先级队列
    DispatchQueue *in_q; //接收消息的DispatchQueue
    list<Message*> sent; //要发送的消息
    Cond cond;
    bool send_keepalive;
    bool send_keepalive_ack;
    utime_t keepalive_ack_stamp;
    bool halt_delivery; //if a pipe's queue is destroyed, stop adding to it //如果Pipe队列销毁，停止增加
    __u32 connect_seq, peer_global_seq;
    uint64_t out_seq; //发送消息的序号
    uint64_t in_seq, in_seq_acked; //接收到的消息序号和ack序号
};
```

在client端SimpleMessenger::connect_to->SimpleMessenger::connect_rank下会执行pipe->start_writer()来启动Pipe的写线程。

在server端Accepter::entry()->SimpleMessenger::add_accept_pipe下执行pipe->start_reader()来启动Pipe的读线程。

Pipe的reader_thread的主函数是Pipe::reader()，writer_thread的主函数是Pipe::writer()。

### 流程

发送消息流程:

![Local Picture](/images/ceph/messenger/simple_sendmsg.jpg "send_msg")

接收消息流程:

![Local Picture](/images/ceph/messenger/simple_recvmsg.jpg "recv_msg")





## AsyncMessenger

AsyncMessenger消息模块是基于epoll的事件驱动方式。

### AsyncMessenger

AsyncMessenger是一个抽象的消息管理器，定义了主要的成员和方法。继承于SimplePolicyMessenger类，SimplePolicyMessenger是对消息管理器的一些连接的策略进行定义的设置。

```
class AsyncMessenger: public SimplePolicyMessenger {
    NetworkStack *stack;
    std::vector<Processor*> processors;  //主要用来监听连接
    friend class Processor;
    DispatchQueue dispatch_queue; //用来分发消息

    // the worker run messenger's cron jobs
    Worker *local_worker; //用来运行定时器任务

    std::string ms_type;

    /**
     *  false; set to true if the AsyncMessenger bound to a specific address;
     *  and set false again by Accepter::stop().
     */
    bool did_bind;
    /// counter for the global seq our connection protocol uses
    __u32 global_seq;

    ceph::unordered_map<entity_addrvec_t, AsyncConnectionRef> conns; //地址和连接的map表，创建一个新的连接时将连接和和地址信息加入到这个map表中，在发送消息时先根据地址对这个map进行一次查找，如果找到了返回连接，如果没有找到创建一个新的连接。
    set<AsyncConnectionRef> accepting_conns; //接收连接的集合，这个集合主要存放那些已经接收的连接
    set<AsyncConnectionRef> deleted_conns; //已经关闭并且需要清理的连接的集合
    int cluster_protocol;

    AsyncConnectionRef local_connection;  //初始化传的是local_work,协议为LoopbackProtocolV1
};
```

### Processor

Processor类主要完成启动、绑定、监听到来的连接。

```
class Processor {
  AsyncMessenger *msgr; //AsyncMessenger的指针实例
  NetHandler net;
  Worker *worker; //工作线程
  vector<ServerSocket> listen_sockets;
  EventCallbackRef listen_handler; //accept的处理函数，对应C_processor_accept，其对应Processor::accept()函数

  class C_processor_accept;

 public:
  Processor(AsyncMessenger *r, Worker *w, CephContext *c);
  ~Processor() { delete listen_handler; };

  void stop();
  int bind(const entity_addrvec_t &bind_addrs, //执行绑定套接字的具体过程
       const set<int>& avoid_ports,
       entity_addrvec_t* bound_addrs);
  void start(); //执行消息模块的start，具体就是启动线程，让其处于工作状态
  void accept(); //建立连接的过程，如果连接建立成功，则通过add_accept()函数将连接加入到accepting_conns集合中
};
```
### Worker

worker类是工作线程的抽象接口，具体实现为PosixWorker。

```
class Worker {
    bool done = false; //如果线程的工作完成置为true，否则false

    CephContext *cct;
    PerfCounters *perf_logger;
    unsigned id; 

    std::atomic_uint references;
    EventCenter center; //EventCenter的实例，在Worker的构造函数中执行EventCenter的初始化工作, 用来保存要处理的相关事件

    //server端
    virtual int listen(entity_addr_t &addr, unsigned addr_slot,
            const SocketOptions &opts, ServerSocket *) = 0;
    //client主动连接
    virtual int connect(const entity_addr_t &addr,
            const SocketOptions &opts, ConnectedSocket *socket) = 0;
};
```

### EventCenter

EventCenter维护一系列文件描述符和处理已经注册的事件。其保存所有事件，提供处理事件的相关函数。

```
class EventCenter {
 public:
  // should be enough;
  static const int MAX_EVENTCENTER = 24;
  struct FileEvent { //文件事件
    int mask;
    EventCallbackRef read_cb;
    EventCallbackRef write_cb;
    FileEvent(): mask(0), read_cb(NULL), write_cb(NULL) {}
  };  

  struct TimeEvent { //时间事件
    uint64_t id; 
    EventCallbackRef time_cb;

    TimeEvent(): id(0), time_cb(NULL) {}
  }

  CephContext *cct;
  std::string type;
  int nevent;
  // Used only to external event
  pthread_t owner = 0;

  ///外部事件
  std::mutex external_lock;
  std::atomic_ulong external_num_events;
  deque<EventCallbackRef> external_events; //用于存放外部事件的队列

  ///socket事件，其下标是socket对应的fd
  vector<FileEvent> file_events; //FileEvent的实例
  EventDriver *driver;  //EventDriver的实例

  ///时间事件
  std::multimap<clock_type::time_point, TimeEvent> time_events; //时间事件的容器
  // Keeps track of all of the pollers currently defined.  We don't
  // use an intrusive list here because it isn't reentrant: we need
  // to add/remove elements while the center is traversing the list.
  std::vector<Poller*> pollers; //A Poller object is invoked once each time through the dispatcher's inner polling loop //Poller用于轮询事件，用于dpdk模式
  std::map<uint64_t, std::multimap<clock_type::time_point, TimeEvent>::iterator> event_map; //事件map映射

  uint64_t time_event_next_id;
  int notify_receive_fd;
  int notify_send_fd;
  NetHandler net;
  EventCallbackRef notify_handler;
  unsigned idx;
  AssociatedCenters *global_centers = nullptr;
}
```

### EventDriver

EventDriver是一个抽象的接口，定义了添加事件监听，删除事件监听，获取触发的事件的接口。针对不同的IO多路复用机制，实现了不同的类。SelectDriver实现了select的方式。EpollDriver实现了epoll的网络事件处理方式。KqueueDriver是FreeBSD实现kqueue事件处理模型。

```
/*
 * EventDriver is a wrap of event mechanisms depends on different OS.
 * For example, Linux will use epoll(2), BSD will use kqueue(2) and select will
 * be used for worst condition.
 */
class EventDriver {
 public:
  virtual ~EventDriver() {}       // we want a virtual destructor!!!
  virtual int init(EventCenter *center, int nevent) = 0;
  virtual int add_event(int fd, int cur_mask, int mask) = 0;
  virtual int del_event(int fd, int cur_mask, int del_mask) = 0;
  virtual int event_wait(vector<FiredFileEvent> &fired_events, struct timeval *tp) = 0;
  virtual int resize_events(int newsize) = 0;
  virtual bool need_wakeup() { return true; }
};
```

### NetHandler

NetHandler是AsyncMessenger模块中用于网络处理的类, 封装了Socket的基本的功能。

```
class NetHandler {
    int generic_connect(const entity_addr_t& addr, const entity_addr_t& bind_addr, bool nonblock);

    CephContext *cct;
   public:
    int create_socket(int domain, bool reuse_addr=false);
    explicit NetHandler(CephContext *c): cct(c) {}
    int set_nonblock(int sd);
    int set_socket_options(int sd, bool nodelay, int size);
    int connect(const entity_addr_t &addr, const entity_addr_t& bind_addr);
    
    /** 
     * Try to reconnect the socket.
     *
     * @return    0         success
     *            > 0       just break, and wait for event
     *            < 0       need to goto fail
     */
    int reconnect(const entity_addr_t &addr, int sd);
    int nonblock_connect(const entity_addr_t &addr, const entity_addr_t& bind_addr);
    void set_priority(int sd, int priority, int domain);
};
```
### AsyncConnection

AsyncConnection是整个Async消息模块的核心，连接的创建和删除、数据的读写指令、连接的重建、消息的处理等都是在这个类中进行的。

```
class AsyncConnection {
    AsyncMessenger *async_msgr;
    uint64_t conn_id;
    PerfCounters *logger;
    int state;
    ConnectedSocket cs;  //对应的socket
    int port;
    Messenger::Policy policy;

    DispatchQueue *dispatch_queue;

    // lockfree, only used in own thread
    bufferlist outcoming_bl; //临时存放消息的bl
    bool open_write = false;

    EventCallbackRef read_handler;
    EventCallbackRef write_handler;
    EventCallbackRef write_callback_handler;
    EventCallbackRef wakeup_handler;
    EventCallbackRef tick_handler;
    char *recv_buf; //用于从套接字中接收消息的buf
    uint32_t recv_max_prefetch;
    uint32_t recv_start;
    uint32_t recv_end;
    set<uint64_t> register_time_events; // need to delete it if stop
    ceph::coarse_mono_clock::time_point last_connect_started;
    ceph::coarse_mono_clock::time_point last_active;
    ceph::mono_clock::time_point recv_start_time;
    uint64_t last_tick_id = 0;
    const uint64_t connect_timeout_us;
    const uint64_t inactive_timeout_us;

    // Tis section are temp variables used by state transition

    // Accepting state
    bool msgr2 = false;
    entity_addr_t socket_addr;  ///< local socket addr
    entity_addr_t target_addr;  ///< which of the peer_addrs we're connecting to (as clienet) or should reconnect to (as peer)

    entity_addr_t _infer_target_addr(const entity_addrvec_t& av);

    // used only by "read_until"
    uint64_t state_offset;
    Worker *worker; //对应的工作线程
    EventCenter *center; //对应的事件中心，也就是本Connection的所有的事件都由center处理

    std::unique_ptr<Protocol> protocol;
};
```

### StackSingleton

```
struct StackSingleton {
  CephContext *cct;
  std::shared_ptr<NetworkStack> stack;

  explicit StackSingleton(CephContext *c): cct(c) {}
  void ready(std::string &type) {
    if (!stack)
      stack = NetworkStack::create(cct, type);
  }
  ~StackSingleton() {
    stack->stop();
  }
};
```

### NetworkStack

NetworkStack是底层协议栈的抽象，子类有PosixNetworkStack、RDMAStack、DPDKStack。

```
class NetworkStack {
    std::string type;
    unsigned num_workers = 0;
    ceph::spinlock pool_spin;
    bool started = false;

    std::function<void ()> add_thread(unsigned i);

  protected:
    CephContext *cct;
    vector<Worker*> workers;
};
```
NetworkStack中的workers用来保存多个Worker，每个Worker都会创建一个Epoll（Kqueue或select）。

### 流程

参考ceph_osd.cc中OSD的部分，来看下Async的流程。

![Local Picture](/images/ceph/messenger/AsyncMessenger_flow.jpg "AsyncMessenger flow")


AsyncMessenger的初始化

```
AsyncMessenger::AsyncMessenger(CephContext *cct, entity_name_t name,
        const std::string &type, string mname, uint64_t _nonce)
    : SimplePolicyMessenger(cct, name,mname, _nonce),
    dispatch_queue(cct, this, mname),
    lock("AsyncMessenger::lock"),
    nonce(_nonce), need_addr(true), did_bind(false),
    global_seq(0), deleted_lock("AsyncMessenger::deleted_lock"),
    cluster_protocol(0), stopped(true)
{
    std::string transport_type = "posix";
    if (type.find("rdma") != std::string::npos)
        transport_type = "rdma";
    else if (type.find("dpdk") != std::string::npos)
        transport_type = "dpdk";

    auto single = &cct->lookup_or_create_singleton_object<StackSingleton>(
            "AsyncMessenger::NetworkStack::" + transport_type, true, cct);
    single->ready(transport_type);
    stack = single->stack.get();
    stack->start();
    local_worker = stack->get_worker();
    local_connection = new AsyncConnection(cct, this, &dispatch_queue,
            local_worker, true, true);
    init_local_connection();
    reap_handler = new C_handle_reap(this);
    unsigned processor_num = 1;
    if (stack->support_local_listen_table())
        processor_num = stack->get_num_worker();
    for (unsigned i = 0; i < processor_num; ++i)
        processors.push_back(new Processor(this, stack->get_worker(i), cct));
}

1. 初始化一个StackSingleton，single->ready(transport_type)会调用NetworkStack::create()，NetworkStack::create根据类型，比如posix来初始化PosixNetworkStack，调用PosixNetworkStack构造函数，调用基类NetworkStack::NetworkStack()构造函数，其根据num_works(cct->_conf->ms_async_op_threads)创建Woker实例，并初始化worker->EventCenter，然后将worker将入NetworkStack类内的vector<Worker*> workers里，最后返回PosixNetworkStack实例；
2. NetworkStack::start(), 初始化并启动Worker的线程函数，其通过epoll监听事件，用注册的回调函数进行处理。
3. 初始化一个local_worker和一个local_connection，用来处理定时cron事件。
4. 根据processor_num来初始化Processor，并放入到processors里。

```

AsyncMessenge的bind

```
int AsyncMessenger::bind(const entity_addr_t &bind_addr)
{
    ldout(cct,10) << __func__ << " " << bind_addr << dendl;
    // old bind() can take entity_addr_t(). new bindv() can take a
    // 0.0.0.0-like address but needs type and family to be set.
    auto a = bind_addr;
    if (a == entity_addr_t()) {
        a.set_type(entity_addr_t::TYPE_LEGACY);
        if (cct->_conf->ms_bind_ipv6) {
            a.set_family(AF_INET6);
        } else {
            a.set_family(AF_INET);
        }   
    }
    return bindv(entity_addrvec_t(a));
}

int AsyncMessenger::bindv(const entity_addrvec_t &bind_addrs)
{
    lock.Lock();

    if (!pending_bind && started) {
        ldout(cct,10) << __func__ << " already started" << dendl;
        lock.Unlock();
        return -1;
    }

    ldout(cct,10) << __func__ << " " << bind_addrs << dendl;

    if (!stack->is_ready()) {
        ldout(cct, 10) << __func__ << " Network Stack is not ready for bind yet - postponed" << dendl;
        pending_bind_addrs = bind_addrs;
        pending_bind = true;
        lock.Unlock();
        return 0;
    }

    lock.Unlock();

    // bind to a socket
    set<int> avoid_ports;
    entity_addrvec_t bound_addrs;
    unsigned i = 0;
    for (auto &&p : processors) {
        int r = p->bind(bind_addrs, avoid_ports, &bound_addrs);
        if (r) {
            // Note: this is related to local tcp listen table problem.
            // Posix(default kernel implementation) backend shares listen table
            // in the kernel, so all threads can use the same listen table naturally
            // and only one thread need to bind. But other backends(like dpdk) uses local
            // listen table, we need to bind/listen tcp port for each worker. So if the
            // first worker failed to bind, it could be think the normal error then handle
            // it, like port is used case. But if the first worker successfully to bind
            // but the second worker failed, it's not expected and we need to assert
            // here
            ceph_assert(i == 0);
            return r;
        }
        ++i;
    }
    _finish_bind(bind_addrs, bound_addrs);
    return 0;
}

//主要根据processors数，调用processor->bind来进行初始化，processor->bind根据bind_addrs.v.size()来依次完成绑定操作，需调用worker->center.submit_to()。

worker->center.submit_to(
        worker->center.get_id(),
        [this, k, &listen_addr, &opts, &r]() {
        r = worker->listen(listen_addr, k, opts, &listen_sockets[k]);
        }, false);

template <typename func>
void submit_to(int i, func &&f, bool nowait = false) {
    ceph_assert(i < MAX_EVENTCENTER && global_centers);
    EventCenter *c = global_centers->centers[i];
    ceph_assert(c);
    if (!nowait && c->in_thread()) {
        f();
        return ;
    }   
    if (nowait) {
        C_submit_event<func> *event = new C_submit_event<func>(std::move(f), true);
        c->dispatch_event_external(event);
    } else {
        C_submit_event<func> event(std::move(f), false);
        c->dispatch_event_external(&event);
        event.wait();
    }   
};

//submit_to就是将worker->listen(listen_addr, k, opts, &listen_sockets[k])这个事件通过EventCenter->
//dispatch_event_external添加到deque<EventCallbackRef> external_events队列中去。
//这里的listen即PosixWorker::listen()。

```

AsyncMessenger的add_dispatcher_head，其内部会调用ready()

```
void Messenger::add_dispatcher_head(Dispatcher *d) {
    bool first = dispatchers.empty();
    dispatchers.push_front(d);
    if (d->ms_can_fast_dispatch_any())
        fast_dispatchers.push_front(d);
    if (first)
        ready();
}


void AsyncMessenger::ready()
{
    ldout(cct,10) << __func__ << " " << get_myaddrs() << dendl;

    stack->ready();
    if (pending_bind) {
        int err = bindv(pending_bind_addrs);
        if (err) {
            lderr(cct) << __func__ << " postponed bind failed" << dendl;
            ceph_abort();
        }   
    }

    Mutex::Locker l(lock);
    for (auto &&p : processors)
        p->start();
    dispatch_queue.start();
}

void Processor::start()
{
    ldout(msgr->cct, 1) << __func__ << dendl;

    // start thread
    worker->center.submit_to(worker->center.get_id(), [this]() {
                for (auto& l : listen_sockets) {
                    if (l) {
                        worker->center.create_file_event(l.fd(), EVENT_READABLE,
                            listen_handler);
                    }
                }
            }, false);
}

//在AsyncMessenger::bindv()中processor->bind()，完成了socket绑定监听，将监听端口给了listen_sockets。
//在Processor::start函数里，向EventCenter投递了外部事件，该外部事件的回调函数里实现了向EventCenter注册listen socket的读事件监听，该事件的处理函数为listen_handler。
//listen_handler对应的 处理函数为 processor::accept函数，其处理接收连接的事件。

class Processor::C_processor_accept : public EventCallback {
    Processor *pro;

    public:
    explicit C_processor_accept(Processor *p): pro(p) {}
    void do_request(uint64_t id) override {
        pro->accept();
    }
};

void Processor::accept()
{
    ...

    for (auto& listen_socket : listen_sockets) {

        while (true) {
            entity_addr_t addr;
            ConnectedSocket cli_socket;
            Worker *w = worker;
            if (!msgr->get_stack()->support_local_listen_table())
                w = msgr->get_stack()->get_worker();
            else
                ++w->references;
            int r = listen_socket.accept(&cli_socket, opts, &addr, w);
            if (r == 0) {
                ldout(msgr->cct, 10) << __func__ << " accepted incoming on sd "
                    << cli_socket.fd() << dendl;

                msgr->add_accept(
                        w, std::move(cli_socket),
                        msgr->get_myaddrs().v[listen_socket.get_addr_slot()],
                        addr);
                accept_error_num = 0;
                continue;

            } else {

            }
        }
   }
}

//在函数Processor::accept里，首先获取了一个worker，通过调用accept函数接收该连接。并调用msgr->add_accept函数。

void AsyncMessenger::add_accept(Worker *w, ConnectedSocket cli_socket,
        const entity_addr_t &listen_addr,
        const entity_addr_t &peer_addr)
{
    lock.Lock();
    //创建连接，该Connection已经指定了worker处理该Connection上所有的事件。
    AsyncConnectionRef conn = new AsyncConnection(cct, this, &dispatch_queue, w,
            listen_addr.is_msgr2(), false);
    conn->accept(std::move(cli_socket), listen_addr, peer_addr);
    accepting_conns.insert(conn);
    lock.Unlock();
}


//在AsyncConnection::accept()进行处理，将state设置成STATE_ACCEPTING，调用center->dispatch_event_external(read_handler)，监听read事件
void AsyncConnection::accept(ConnectedSocket socket,
                 const entity_addr_t &listen_addr,
                 const entity_addr_t &peer_addr)
{
  ldout(async_msgr->cct, 10) << __func__ << " sd=" << socket.fd()
                 << " listen_addr " << listen_addr
                 << " peer_addr " << peer_addr << dendl;
  ceph_assert(socket.fd() >= 0); 

  std::lock_guard<std::mutex> l(lock);
  cs = std::move(socket);
  socket_addr = listen_addr;
  target_addr = peer_addr; // until we know better
  state = STATE_ACCEPTING;
  protocol->accept();
  // rescheduler connection in order to avoid lock dep
  center->dispatch_event_external(read_handler);
}
```

AsyncMessenger的start，主要用来启动本地连接。

```
int AsyncMessenger::start()
{
    lock.Lock();
    ldout(cct,1) << __func__ << " start" << dendl;

    // register at least one entity, first!
    ceph_assert(my_name.type() >= 0); 

    ceph_assert(!started);
    started = true;
    stopped = false;

    if (!did_bind) {
        entity_addrvec_t newaddrs = *my_addrs;
        for (auto& a : newaddrs.v) {
            a.nonce = nonce;
        }   
        set_myaddrs(newaddrs);
        _init_local_connection();
    }

    lock.Unlock();
    return 0;
}
```

#### Client端连接过程

```
AsyncConnectionRef AsyncMessenger::create_connect(
        const entity_addrvec_t& addrs, int type)
{
    ceph_assert(lock.is_locked());

    ldout(cct, 10) << __func__ << " " << addrs
        << ", creating connection and registering" << dendl;

    // here is where we decide which of the addrs to connect to.  always prefer
    // the first one, if we support it.
    entity_addr_t target;
    for (auto& a : addrs.v) {
        if (!a.is_msgr2() && !a.is_legacy()) {
            continue;
        }
        // FIXME: for ipv4 vs ipv6, check whether local host can handle ipv6 before
        // trying it?  for now, just pick whichever is listed first.
        target = a;
        break;
    }

    // create connection
    Worker *w = stack->get_worker();
    AsyncConnectionRef conn = new AsyncConnection(cct, this, &dispatch_queue, w,
            target.is_msgr2(), false);
    conn->connect(addrs, type, target);
    ceph_assert(!conns.count(addrs));
    ldout(cct, 10) << __func__ << " " << conn << " " << addrs << " "
        << *conn->peer_addrs << dendl;
    conns[addrs] = conn;
    w->get_perf_counter()->inc(l_msgr_active_connections);

    return conn;
}


//函数AsyncConnection::_connect设置了状态为STATE_CONNECTING，向对应的EventCenter投递外部外部事件，其read_handler为void AsyncConnection::process()函数。
void AsyncConnection::_connect()
{
    ldout(async_msgr->cct, 10) << __func__ << dendl;

    state = STATE_CONNECTING;
    protocol->connect();
    // rescheduler connection in order to avoid lock dep
    // may called by external thread(send_message)
    center->dispatch_event_external(read_handler);
}

```

#### 消息的接收-Server

由AsyncMessenger::add_accept()已经知道其调用AsyncConnection::accept()进行处理，先将state的值置为STATE_ACCEPTING，继而调用center->dispatch_event_external(read_handler)，监听read事件，其回调函数为AsyncConnection::process()。

```
//AsyncConnection::process()可以看做是一个状态驱动器，根据state来处理不同的事件。
void AsyncConnection::process() {
    std::lock_guard<std::mutex> l(lock);
    last_active = ceph::coarse_mono_clock::now();
    recv_start_time = ceph::mono_clock::now();

    ldout(async_msgr->cct, 20) << __func__ << dendl;

    switch (state) {
    case STATE_NONE: {
	return;
    }
    case STATE_CLOSED: {
	return;
    }
    case STATE_CONNECTING: {
       ceph_assert(!policy.server);

       // clear timer (if any) since we are connecting/re-connecting
       if (last_tick_id) {
           center->delete_time_event(last_tick_id);
           last_tick_id = 0;
       }

       if (cs) {
           center->delete_file_event(cs.fd(), EVENT_READABLE | EVENT_WRITABLE);
           cs.close();
       }

       SocketOptions opts;
       opts.priority = async_msgr->get_socket_priority();
       opts.connect_bind_addr = msgr->get_myaddrs().front();
       ssize_t r = worker->connect(target_addr, opts, &cs);
       if (r < 0) {
           protocol->fault();
           return;
       }

       center->create_file_event(cs.fd(), EVENT_READABLE, read_handler);
       state = STATE_CONNECTING_RE;
   }
   case STATE_CONNECTING_RE: {
       state = STATE_CONNECTION_ESTABLISHED;
       ...
       break;
   }
   case STATE_ACCEPTING: {
      center->create_file_event(cs.fd(), EVENT_READABLE, read_handler);
      state = STATE_CONNECTION_ESTABLISHED;
      break;
    }   
    case STATE_CONNECTION_ESTABLISHED: {
      if (pendingReadLen) {
        ssize_t r = read(*pendingReadLen, read_buffer, readCallback);
        if (r <= 0) { // read all bytes, or an error occured
          pendingReadLen.reset();
          char *buf_tmp = read_buffer;
          read_buffer = nullptr;
          readCallback(buf_tmp, r); 
        }   
        return;
      }   
      break;
    }
   }
}
```


消息处理的流程图:

![Local Picture](/images/ceph/messenger/AsyncConnection_server.jpg "AsyncConnection server")

#### 消息的发送-Client

连接时会将state的值置为STATE_CONNECTING。在AsyncConnection::process()会先case到STATE_CONNECTING中，调用protocol->read_event()时会先case到START_CONNECT中。不作详述。




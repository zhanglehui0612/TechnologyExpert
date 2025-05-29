## 一 同步和异步、阻塞和非阻塞、同步和阻塞之间区别
### 1.1 同步和异步比较
✅ 同步: 指的是程序在执行过程中，需要等待上一步骤完成后才继续执行，发起操作后必须等待完成，期间不能干别的  
✅ 异步: 指的是程序在执行过程中，不需要等待上一步骤完成后才继续执行，发起操作后不等待结果，操作完成后会通过回调、事件通知调用者  

### 1.2 阻塞和非阻塞比较
✅ 阻塞: 在I/O操作中，需要等待数据备好才返回，如果没有准备好则挂起线程进行等待  
✅ 非阻塞: 在I/O操作中，数据准备好或者没准备好都返回，不会一直等待; 如果没准备好，立即返回一个状态或错误，不让调用者卡住  

### 1.3 同步和阻塞比较
✅ 同步: 关注的是调用者是否等待结果  
✅ 阻塞: 关注的线程是否被挂起  

## 二 同步阻塞I/O模型
✅ 当程序发起一个 I/O 操作（比如读取 socket 数据）时，如果数据没有准备好，程序会被挂起，一直等待，直到数据准备好  
🎯 **特点:**  
🔷 程序会一直等待，不能做其他事，效率较低    
```go
package main

import (
	"fmt"
	"net"
)

func main() {
	ln, _ := net.Listen("tcp", ":8080")
	conn, _ := ln.Accept()          // 阻塞，直到有连接
	buf := make([]byte, 1024)
	n, _ := conn.Read(buf)          // 阻塞，直到有数据
	fmt.Println("Received:", string(buf[:n]))
}
```

```rust
use std::net::TcpListener;
use std::io::Read;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    let (mut stream, _) = listener.accept().unwrap();  // 阻塞直到有连接

    let mut buffer = [0; 1024];
    let n = stream.read(&mut buffer).unwrap();         // 阻塞直到有数据
    println!("Received: {}", String::from_utf8_lossy(&buffer[..n]));
}
```

## 三 同步非阻塞I/O模型
✅ 当程序发起一个 I/O 操作时，如果数据没有准备好，不会挂起程序，而是立即返回一个错误或空数据  
🎯 **特点:**  
🔷 程序可以继续执行其他任务  
🔷 通常与轮询或事件机制(如 select、poll)配合使用, 需要自己去轮询数据是否准备好，而不是系统通知你  

```go
package main

import (
	"fmt"
	"syscall"
	"time"
)

func main() {
	fd, _ := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
	addr := syscall.SockaddrInet4{Port: 8080}
	copy(addr.Addr[:], []byte{127, 0, 0, 1})
	syscall.Bind(fd, &addr)
	syscall.Listen(fd, 10)
	syscall.SetNonblock(fd, true)  // 设置为非阻塞

	for {
		nfd, err := syscall.Accept(fd)
		if err != nil {
			fmt.Println("No connection yet...")
			time.Sleep(1 * time.Second)
			continue
		}
		fmt.Println("Accepted connection:", nfd)
		break
	}
}

```

```rust
use mio::{Events, Interest, Poll, Token};
use mio::net::TcpListener;
use std::time::Duration;

fn main() {
    let mut poll = Poll::new().unwrap();
    let mut events = Events::with_capacity(128);

    let addr = "127.0.0.1:8080".parse().unwrap();
    let mut listener = TcpListener::bind(addr).unwrap();

    poll.registry().register(&mut listener, Token(0), Interest::READABLE).unwrap();

    println!("Waiting for connection...");

    loop {
        poll.poll(&mut events, Some(Duration::from_secs(1))).unwrap();

        for event in events.iter() {
            if event.token() == Token(0) {
                let (stream, _) = listener.accept().unwrap();
                println!("Accepted connection from client.");
                return;
            }
        }

        println!("No connection yet...");
    }
}
```


## 四 异步非阻塞I/O模型
✅ 当你发起一个 I/O 请求(如读取数据)时，不会等待其完成(异步)，也不会阻塞当前线程(非阻塞)。系统会在操作完成后主动通知你（通常通过回调、事件或 future/promise  
🎯 **特点:**  
🔷 用户线程在发起 I/O 后可以立即干别的事  
🔷 当数据准备好时，操作系统会通过事件/回调机制通知你  

![](.IO模型专题_images/210d809b.png)  

```rust
use tokio::net::TcpListener;
use tokio::io::AsyncReadExt;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    let (mut socket, _) = listener.accept().await.unwrap();

    let mut buf = [0u8; 1024];
    let n = socket.read(&mut buf).await.unwrap();  // 异步非阻塞读取
    println!("Received: {}", String::from_utf8_lossy(&buf[..n]));
}
```

## 五 多路复用I/O模型
### 5.1 什么I/O多路复用
I/O多路复用指的是允许一个线程或者进程同时监听多个I/O描述符，当一个或者多个描述符有事件就绪的时候通知应用程序
应用程序就可以对这些就绪的描述符进行实际的读/写操作
多个I/O描述符就是你需要传递的描述符，表示你对哪些I/O描述符需要监听


### 5.2 select
用户态: 
用户进程调用 select() 并传入监听的 fd 集合及超时时间
会将用户传入的参数拷贝到内核
```
int select(int nfds,                // 传入当前最大描述符+1，用于内核遍历
           fd_set *readfds,         // 传入的读描述符集合，是一个位图的数据结构，最多只允许传入1024个
           fd_set *writefds,        // 传入的写描述符集合，是一个位图的数据结构，最多只允许传入1024个
           fd_set *exceptfds,       // 传入的异常描述符集合，是一个位图的数据结构
           struct timeval *timeout);// 超时等待时间
```
注意:
文件描述符是用数字表示
传入的描述符集合是一个位图，fd_set索引代表文件描述符，如果该位置标记位1，表示需要监听
返回之后，位图会被清掉，只会对就绪的描述符标记成1，应用程序遍历位图，获取就绪描述符，然后进行读写
位图只允许传入1024个描述符，因此有限制
用户程序在监听的话需要对描述符判断，大于1024的描述符，select是无法监视的

内核态: 
内核开始扫描是否有描述符fd就绪
遍历范围: 遍历从 0 到 nfds - 1 的每个fd
如果发现至少一个描述符fd满足条件（如有数据可读），则立即返回，告诉用户哪个 fd 就绪，即重置fd_set, 将就绪的fd_set标记位1
如果没有就绪 fd，则根据 timeout 参数决定等待行为：
1. [ ] timeout = 0：立即返回，非阻塞查询
2. [ ] timeout > 0：阻塞等待，直到有 fd 就绪或超时
3. [ ] timeout = NULL：无限阻塞，直到有 fd 就绪

如果没有就绪 fd，且需要等待, 则挂起线程等待
当某个 fd 发生感兴趣的事件时，内核会唤醒进程，结束阻塞

用户态:
用户态收到select调用返回的数据，如果就绪个数是0，则进行下一轮处理
如果不是0则遍历client socket 数组，然后根据他们的描述符在fd_set检查是不是是就绪的socket fd，如果是则开始进行read调用
此时的内核缓冲区肯定是有数据的，所以不会阻塞等待，直接读取数据就行, 处理完进行下一轮select


### 5.3 poll
我们知道select使用的是位图fd_set数据结构，有1024长度限制，poll通过pollfd数组解决了长度限制问题
但是本质和select一样，还是用户程序需要遍历全部描述符，才知道哪些描述符有就绪事件
```
struct pollfd {
    int   fd;       // 需要监听的文件描述符
    short events;   // 感兴趣的事件（输入）
    short revents;  // 实际发生的事件（输出，由内核填充）
};

int poll(struct pollfd fds[],     // 指向 pollfd 结构体数组
         nfds_t nfds,             // 数组中 pollfd 元素个数
         int timeout);            // 超时时间
```

用户态:
调用poll, 传入需要监视的描述符结构体pollfd数组, 并且指定数组中描述符集合个数和超时时间
将参数拷贝内核中

内核态:
内核遍历pollfd描述符数组，检查描述符是否有事件就绪
如果有事件就绪则更新revents,指定就绪事件; 如果没有则遍历下一个
遍历完后，有就绪事件发生就返回; 没有的话则挂起线程阻塞等待，直到有就绪事件发生，内核会唤醒进程，结束阻塞
如果没有就绪 fd，则根据 timeout 参数决定等待行为：
1. [ ] timeout = 0：立即返回，非阻塞查询
2. [ ] timeout > 0：阻塞等待，直到有 fd 就绪或超时
3. [ ] timeout = NULL：无限阻塞，直到有 fd 就绪

用户态:
用户态收到poll调用返回的数据，如果就绪个数是0，则进行下一轮处理
如果不是0则遍历pollfd数组，找到revents不为空，即有就绪事件的描述符，然后开始进行读写
此时的内核缓冲区肯定是有数据的，所以不会阻塞等待，直接读取数据就行, 处理完进行下一轮poll

### 5.4 epoll
#### 5.4.1 概念和操作
##### 5.4.1.1 epfd
epfd指的就是epoll文件描述符, 它是通过epoll_create创建出来的，代表一个epoll实例
epfd就是一个“事件管理器”或“事件注册中心”，保存你监听的所有 fd
每次你调用 epoll_ctl()、epoll_wait() 时，都要传入 epfd 表示你操作哪个 epoll 实例
epfd数据结构:
红黑树(rbtree):	保存你注册的所有 fd 和它们感兴趣的事件
就绪队列(ready list): 当事件发生，fd 被加入这个队列，供 epoll_wait 返回

##### 5.4.1.2 epoll_event
表示一个epoll事件
events: epoll事件集合，EPOLLIN(读事件)、EPOLLOUT(写)、EPOLLERR(异常)等，可以是一个或者多个
data: 用户数据

##### 5.4.1.3 epoll_create()
int epoll_create(int size): 其实就是在内核申请一块空间存放一些数据，用于保存监听socket和就绪的socket
epfd中主要是一个红黑树结构的socket监听集合；一个是socket就绪队列
size: 设置监听的文件描述符数量
返回值: epoll的文件描述符

##### 5.4.1.3 epoll_ctl()
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *events):向epoll的socket监听集合添加、修改、删除用户感兴趣的事件
epfd: epoll的文件描述符，epoll_create创建时返回
op: 操作类型，是增加、修改还是删除
fd: socket文件描述符
events: 感兴趣的事件： EPOLLIN(读事件)、EPOLLOUT(写)、EPOLLERR(异常)等
返回值表示是否操作成功,成功返回0，失败返回-1

##### 5.4.1.3 epoll_wait()
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout): 等待就绪事件，timeout决定是否阻塞等待
epfd: epoll_create创建的区域的文件描述符
events: 返回就绪的事件列表，就绪的事件就会往这里放
maxevents: 每次能处理的最大事件数量，这个和工作模式ET/IT有点关系
timeout: -1表示一直阻塞；0表示不阻塞； >0表示阻塞指定时长
返回值表示已就绪文件描述符数量

#### 5.4.2 工作流程

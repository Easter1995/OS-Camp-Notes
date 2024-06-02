# Rust异步编程

type: Post
status: Invisible
date: 2024/05/23
tags: 编程, 课程
category: 操作系统

***异步：***不用等待一个任务执行完成就可以开始执行下一个任务。与多线程不同的是，它并不需要创建额外的线程，此外，所有函数调用都是静态分发的，也没有堆分配。

异步操作意味着当你执行某个操作时，程序不会等待该操作完成，而是继续执行其他操作

![Untitled](https://cdn.jsdelivr.net/gh/Easter1995/blog-image/202406020945786.png)

***为什么不直接就用线程来实现”同时“执行多个任务呢？***

- 线程开销大
- 创建一个线程需要使用很多系统调用，开销很大
- 操作系统反应可能不及时

# 以web Server为例

从**TcpStream**里面读取请求内容：一旦连接建立成功，就可以使用 `TcpStream` 对象来发送和接收数据

- handle_connection：读取发送过来的请求，并响应“hello world”
- concat!：是一个宏，可以把里面的字符串字面量连接为一个单独的字符串

```rust
use std::net::{TcpListener, TcpStream};
use std::io;
use std::io::Read;
use std::io::Write;

fn main() {
    // 绑定主机端口
    let listener = TcpListener::bind("localhost:3000").unwrap();

    loop {
        let (connection, _) = listener.accept().unwrap();

        if let Err(e) = handle_connection(connection) {
            println!("failed to handle connection: {e}")
        }
    }
}

fn handle_connection(mut connection: TcpStream) -> io::Result<()> {
    let mut read = 0;
    let mut request = [0u8; 1024];

    loop {
        // try reading from the stream
        let num_bytes = connection.read(&mut request[read..])?;

        // the client disconnected
        if num_bytes == 0 { // 👈
            println!("client disconnected unexpectedly");
            return Ok(());
        }

        // keep track of how many bytes we've read
        read += num_bytes;

        // have we reached the end of the request?
        if request.get(read - 4..read) == Some(b"\r\n\r\n") {
            break;
        }
    }

    let request = String::from_utf8_lossy(&request[..read]);
    println!("{request}");

    // "Hello World!" in HTTP
    let response = concat!(
        "HTTP/1.1 200 OK\r\n",
        "Content-Length: 12\n",
        "Connection: close\r\n\r\n",
        "Hello world!"
    );

    let mut written = 0;

    loop {
        // write the remaining response bytes
        let num_bytes = connection.write(response[written..].as_bytes())?;

        // the client disconnected
        if num_bytes == 0 {
            println!("client disconnected unexpectedly");
            return Ok(());
        }

        written += num_bytes;

        // have we written the whole response yet?
        if written == response.len() {
            break;
        }
    }

    // flush the response
    connection.flush()
}
```

问题：一次只能响应一个请求

引入线程：一个线程在等待请求时，其余线程可以处理这个请求并发送响应

***阻塞IO的问题：***

```rust
loop {
    // 调用accept之前检查ctrl+c信号
    if got_ctrl_c() {
        break;
    }

    // **如果ctrl+c在这里发生，会出现什么情况?**
    // 如果在这里发生ctrl+c，程序并不会立马停止，因为此时在内核控制下程序会一直等待直到下一个连接到来
    let (connection, _) = listener.accept().unwrap();

    // 在新的连接被接受之前，这不会被调用
    if got_ctrl_c() {
        break;
    }

    std::thread::spawn(|| /* ... */);
}
```

因为I/O阻塞导致我们丧失了程序的控制权，除了等它执行完毕，没有更好的方式取消一个线程的执行——**因为应用程序的控制器完全交给了内核，导致实现事件驱动逻辑变得非常困难**

## 非阻塞IO

### WouldBlock

```rust
use std::io;

// ...
listener.set_nonblocking(true).unwrap();

loop {
    let connection = match listener.accept() {
        Ok((connection, _)) => connection,
        Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
            // the operation was not performed
            // WouldBlock将控制权从控制器交回到我们手里，我们可以决定下一步干嘛
        }
        Err(e) => panic!("{e}"),
    };
		// 在这里将监听器设置为阻塞模式
    connection.set_nonblocking(true).unwrap();
    // ...
}
```

将监听器设置为非阻塞模式，如果I/O请求不能立即完成，内核将返回一个 WouldBlock 错误代码。尽管被表示为错误代码，但WouldBlock并不是真正的错误。只是意味着当前操作无法立即执行完毕，我们可以决定接下来要做什么。就不会像之前一样，让监听器必须等待到一个新的连接才能进行下一步了。

自旋：无休止地接受连接但是什么也不干

```rust
let mut connections = Vec::new(); // 👈

loop {
    match listener.accept() {
        Ok((connection, _)) => {
            connection.set_nonblocking(true).unwrap();
            // 在主线程中持续监听新的连接，并将接受到的连接存储在 connections 向量中
            connections.push(connection); // 👈
        },
        // 接收连接失败
        Err(e) if e.kind() == io::ErrorKind::WouldBlock => {}
        Err(e) => panic!("{e}"),
    }
}
```

所以现在必须为每个**状态（Read、Write、Flush）**都设置响应的操作，模仿内核调度进程

```rust
// ...
loop {
    match listener.accept() {
        // ...
    }

    for (connection, state) in connections.iter_mut() {
        if let ConnectionState::Read { request, read } = state {
            // ... 继续读操作
        }

        if let ConnectionState::Write { response, written } = state {
            // ... 继续写操作
        }

        if let ConnectionState::Flush = state {
            // ...
        }
    }
}
```

比如说读操作：

```rust
let mut completed = Vec::new(); // 👈

'next: for (i, (connection, state)) in connections.iter_mut().enumerate() {
    if let ConnectionState::Read { request, read } = state {
        loop {
            // 尝试从流中读取数据
            match connection.read(&mut request[*read..]) {
                Ok(0) => {
                    println!("client disconnected unexpectedly");
                    completed.push(i); // 👈
                    continue 'next;
                }
                Ok(n) => *read += n,
                // 当前连接未准备好，先处理下一个活动连接
                Err(e) if e.kind() == io::ErrorKind::WouldBlock => continue 'next,
                Err(e) => panic!("{e}"),
            }

            // ...
        }

        // ...
    }
    if let ConnectionState::Flush = state {
    match connection.flush() {
	        Ok(_) => {
	            completed.push(i); // 👈
	        },
	        Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
	            // not ready yet, move on to the next connection
	            continue 'next;
	        }
	        Err(e) => panic!("{e}"),
	    }
		}
}

// 按相反顺序迭代以保留索引
for i in completed.into_iter().rev() {
    connections.remove(i); // 👈
}
```

- 继续读，读的时候查看当前这个读操作的状态，如果是WouldBlock，说明当前connection现在没法继续读了，就要从剩下的连接中找到状态为Read的连接，看看它能不能继续读。总的来说就是一直遍历所有连接，找到那个可以继续往下读的，让它先把读操作进行完毕
- 同时进行差错处理：读到0字节，手动将这个连接放入一个表示已完成连接的向量，并按照先进先出的顺序将其从连接向量中移除
- 写操作同理
- 刷新完毕后连接完成

缺点：

- 只用了一个线程来处理所有的连接请求，对CPU的利用率低
- 检查是否可读写使用了系统调用，开销大

## 多路复用

***epoll的概念：***

监视多个文件描述符，看看是否有任何一个可以进行I/O操作。可以往epoll里添加资源的文件描述符和事件，表示对这个文件的这个事件感兴趣，要把它加入interest list，那么epoll就可以帮忙监视这个文件事件是否发生了。

文件：做了rCore实验，其实Linux上一切存在电脑上的东西都可以被称作文件，包括TCP socket和外部设备

调用epoll_wait()将阻塞，直到以下任一情况发生：

- 文件描述符传递了一个事件；
- 调用被信号处理器中断；
- 超时到期。

所以epoll也是一种阻塞，之前的阻塞是通过自旋来完成的，自旋就是不断查询连接是否可以继续进行相关操作，连接错误类型为WouldBlock时继续查询下一个连接状态；连接状态为Ok时执行操作。

epoll这种同时阻塞多个操作的方法被称为I/O多路复用。

```rust
use epoll::{Event, Events, ControlOptions::*};
use std::os::fd::AsRawFd;

fn main() {
    let listener = TcpListener::bind("localhost:3000").unwrap();
    listener.set_nonblocking(true).unwrap();

    // 添加 listener 到 epoll
    // event的interest flag为EPOLLIN，表示有新连接加入
    // event的data field为listener的文件描述符
    let event = Event::new(Events::EPOLLIN | Events::EPOLLOUT, fd as _);
    // event作为epoll监视的事件
    epoll::ctl(epoll, EPOLL_CTL_ADD, listener.as_raw_fd(), event).unwrap(); // 👈
}
```

```rust
let mut connections = HashMap::new(); // 哈希表存储连接，连接的文件描述符作为键

loop {
    let mut events = [Event::new(Events::empty(), 0); 1024];
    let timeout = -1; // 阻塞，直到某些事情发生
    let num_events = epoll::wait(epoll, timeout, &mut events).unwrap(); // 👈
		
    for event in &events[..num_events] {
		    // event的data field为文件描述符
        let fd = event.data as i32;

		    // listener准备就绪了吗?
		    if fd == listener.as_raw_fd() {
		        // 尝试建立一个连接
		        match listener.accept() {
		            Ok((connection, _)) => {
		                connection.set_nonblocking(true).unwrap();
		                 let fd = connection.as_raw_fd();
		
		                 // 注册新连接到epoll，连接in或out都算作一个事件
		                 let event = Event::new(Events::EPOLLIN | Events::EPOLLOUT, fd as _);
		                 epoll::ctl(epoll, EPOLL_CTL_ADD, fd, event).unwrap(); // 👈
		                 // 新连接状态为正在读
		                 let state = ConnectionState::Read {
                        request: [0u8; 1024],
                        read: 0,
	                   };
										 // 将新连接和它的状态以及文件描述符加入connections向量
                     connections.insert(fd, (connection, state)); // 👈
		            }
		            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {}
		            Err(e) => panic!("{e}"),
		        }
		    }
		    // otherwise, a connection must be ready ？
		    // events里面都有事件了，说明肯定有一个连接是准备好了的，不然现在还在阻塞状态
        let (connection, state) = connections.get_mut(&fd).unwrap(); // 👈
        
        // 尝试基于状态推动当前 connection 的执行
        if let ConnectionState::Read { request, read } = state {
            // connection.read...
            *state = ConnectionState::Write {
                response: response.as_bytes(),
                written: 0,
            };
        }

        if let ConnectionState::Write { response, written } = state {
            // connection.write...
            *state = ConnectionState::Flush;
        }

        if let ConnectionState::Flush = state {
            // connection.flush...
        }
        
        for fd in completed {
            let (connection, _state) = connections.remove(&fd).unwrap();
            // connection 会自动从 epoll 注销
            drop(connection);
        }
    }
}
```

- epoll::wait
  
    接受events作为参数，初始化时，events是空的事件数据，而epoll::wait将监视注册在epoll里的文件对应的事件，如果有事件变得就绪，就会将这个事件装进events数组，返回就绪事件个数
    
- 首先将listener接收到一个新连接作为一个事件注册进epoll，listener一收到连接，epoll就会有知道并在loop里调用`epoll::wait`时将其加入events
- 然后listener接到连接，将这个连接可读或可写也作为一个事件注册进epoll，这样一旦有一个连接可以继续被进行读写操作，就可以将它拿出来执行读写
    - `Events::EPOLLIN | Events::EPOLLOUT`：
        - `EPOLLIN` 表示对应文件描述符上有数据可读事件，`EPOLLOUT` 表示对应文件描述符上有数据可写事件。
        - 这样做的目的是告诉 epoll 监听器，我们希望监听的是文件描述符上的可读和可写事件。
- 之前的操作是一直遍历connections数组，直到找到一个可读写的连接；但现在epoll自己就会帮忙找出所有可执行的连接
- 现在实现了在同一个线程中处理多个连接请求

# Futures

## 概念

Rust的Future与js的Promise有很多相似之处，但是Promise一旦被创建，就会马上开始run那个task，而Future却是惰性的，也就是说使用它之前必须调用它一次

```jsx
function timer(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
}

timer(200)
.then(() => timer(100))
.then(() => timer(50))
.then(() => console.log("I'm the last one"));
```

- 可以看出使用Promise，避免了setTimeout的嵌套使用，而且语义也更清晰
- 当第一个 Promise resolve 后，执行 `then(() => timer(100))`，返回一个新的 Promise，在 100 毫秒后 resolve。
- 当第二个 Promise resolve 后，执行 `then(() => timer(50))`，返回一个新的 Promise，在 50 毫秒后 resolve。
- 当第三个 Promise resolve 后，执行 `then(() => console.log("I'm the last one"))`，这时会输出 "I'm the last one"。

***什么是future？***

用来代表某个在未来将会被完成的任务

***Rust中的异步实现基于轮询，每个异步任务分成三个阶段：***

1. **轮询阶段（The Poll phase）**。一个`Future`被轮询到（polled）后，会开始执行，直到被阻塞。我们经常把轮询一个Future这部分称之为执行器（executor）
2. **等待阶段**。事件源(通常称为reactor)注册Future等待一个事件发生，并确保当该事件准备好时唤醒相应的`Future`
3. **唤醒阶段**。事件发生了且Future被唤醒了。现在轮询Future的那个执行器（executor）应该调度Future再次被轮询，直到它结束或者到达下一个断点

## 异步编程

将任务写成独立的单元，能集中在一个地方处理所有任务的调度和事件处理，从而重新获得流程控制权

```rust
trait Future {
    type Output;

    fn poll(&mut self) -> Option<Self::Output>;
}
```

- 任务：处理请求、读写数据本质上都是一个任务，是一段要被执行的代码，代表着它**将在未来某个时候需要得到解析**——Future的意思就是这个
- fn poll：询问这个任务是否就绪，如果就绪就改变它的状态并执行它

```rust
impl Scheduler {
    fn spawn<T>(&self, mut future: T) {
        let id = rand();
        // 对 future 调用一次 poll 让它运转起来，传入的参数是它的 ID
        future.poll(event.id);
        // 保存 future
        self.tasks.insert(id, future);
    }

    fn run(self) {
        // ...

        for event in epoll_events {
            // 根据事件 ID poll 相应的 future
            let future = self.tasks.get(&event.id).unwrap();
            future.poll(event.id);
        }
    }
}
```

### 唤醒器

使future运转起来：

- 首先给每个future分配一个id，将(id，future)这个键值对加入任务表
- 然后调度器要让哪个future运转的话，只用根据id就可以查找到任务表里对应的future，poll它就可以使其运转起来

<aside>
💡 如果 future 是 I/O 事件，在 接到 epoll 通知时我们就知道它可以执行了。问题是我们不知道 epoll 事件对应的是哪个future, 因为 future 的执行过程都在内部的 poll 中。

我们需要 future 传递一个 ID 给调度器，它可以用这个 ID 而不是文件描述符向 epoll 注册任何 I/O 资源。通过这种方式，调度器就能把 epoll 事件和 future 对应起来了。

意思就是future是一个有待执行的任务，现在用epoll来监听事件，它当然能知道哪个事件发生了，但是它不知道此时应该使哪个future运转，因此引入ID

</aside>

```rust
impl Scheduler {
    fn spawn<T>(&self, mut future: T) {
        let id = rand();
        // 对 future 调用一次 poll 让它运转起来，传入的参数是它的 ID
        future.poll(event.id);
        // 保存 future
        self.tasks.insert(id, future);
    }

    fn run(self) {
        // ...

        for event in epoll_events {
            // 根据事件 ID poll 相应的 future
            let future = self.tasks.get(&event.id).unwrap();
            future.poll(event.id);
        }
    }
}
```

现在的情况是：调度器可以根据event的id，使用`future.poll(event.id)` 唤醒一个future，但我们想要不靠这个id，靠一个通用的方法就可以唤醒future

```rust
#[derive(Clone)]
struct Waker(Arc<dyn Fn() + Send + Sync>); // 里面唯一的元素是那个闭包

impl Waker {
    fn wake(&self) {
        (self.0)()
    }
}

trait Future {
    type Output;

    fn poll(&mut self, waker: Waker) -> Option<Self::Output>;
}
```

Waker：每个future都具有一个Waker，可以唤醒自己并通知调度器它可以被执行

- 调度器可以为每个 future 提供一个回调函数，它被调用时更新该 future 在调度器中的状态，标记 future 为就绪。这样调度器就完全和 epoll 或 其他任何独立通知系统解耦了。
  
    这样一来，future 可以自行决定何时通知调度器它已经准备好执行，而调度器则可以专注于轮询事件并执行相应的 future
    
- 现在要唤醒一个任务，只用为任务实现future特征，然后传入它的waker即可`poll(waker)`

wake函数：

- Waker实例的那个闭包，执行它就可以做到唤醒一个Future
- 这样一来每个Future都可以通过在poll时传入waker，自定义自己独有的唤醒方式

### 反应器

future被唤醒了就需要开始执行，但是现在唯一能执行的途径就是通过epoll返回的那个事件集合找到对应的任务

***反应器：***

负责注册事件、添加任务、驱动epoll、调用waker的wake函数唤醒任务

```rust
thread_local! {
    static REACTOR: Reactor = Reactor::new();
}

struct Reactor {
    epoll: RawFd, // epoll的文件描述符
    tasks: RefCell<HashMap<RawFd, Waker>>, // 任务文件描述符为键，waker为值
}

impl Reactor {
    pub fn new() -> Reactor {
        Reactor {
            epoll: epoll::create(false).unwrap(),
            tasks: RefCell::new(HashMap::new()),
        }
    }
}
```

添加任务和删除任务：

- 添加任务：在epoll里注册任务和它的事件
- 删除任务：将任务从任务队列移除

```rust
impl Reactor {
    // 添加一个关注读和写的事件描述符
    //
    // 当事件被触发时`waker` 将会被调用
    pub fn add(&self, fd: RawFd, waker: Waker) {
		    // 创建一个事件
        let event = epoll::Event::new(Events::EPOLLIN | Events::EPOLLOUT, fd as u64);
        // 在epoll里面注册文件对应事件
        epoll::ctl(self.epoll, EPOLL_CTL_ADD, fd, event).unwrap();
        // tasks添加键值对，以便通过任务找到它对应的唤醒器
        self.tasks.borrow_mut().insert(fd, waker);
    }
    
    // 从 epoll 移除指定的描述符
    //
    // 移除后任务将不再得到该通知
    pub fn remove(&self, fd: RawFd) {
        self.tasks.borrow_mut().remove(&fd);
    }
}
```

- wait任务：找出可执行的任务，调用其waker的wake函数将它唤醒

```rust
impl Reactor {
    // 驱动任务前进，然后一直阻塞，直到有事件到达。
    pub fn wait(&self) {
       let mut events = [Event::new(Events::empty(), 0); 1024];
       let timeout = -1; // 永不超时
       let num_events = epoll::wait(self.epoll, timeout, &mut events).unwrap();

       for event in &events[..num_events] {
           let fd = event.data as i32;

           // 唤醒任务
           // 任务对应的文件描述符在反应器的任务列表里才调用wake唤醒任务
           if let Some(waker) = self.tasks.borrow().get(&fd) {
               waker.wake();
           }
       }
    }
}
```

### 调度器

- spawn：往可执行任务里面添加一个任务
- run：执行任务（执行那个闭包）
    - 创建一个唤醒器，也就是一个闭包：作用是将一个任务推到runnable队列
    - 将任务poll一次
- 调度器一直轮询每一个目前在runnable里面的任务，并且在目前没有要执行的任务时一直等待新任务

```rust
// 任务都实现了Future特征
type SharedTask = Arc<Mutex<dyn Future<Output = ()> + Send>>;

struct Scheduler {
		// 只记录可以被执行的任务
    runnable: Mutex<VecDeque<SharedTask>>,
}

impl Scheduler {
    impl Scheduler {
		    // 创建任务时将其加入队尾
		    // future 只在它可以执行的时候才会被 poll。它们在创建时总是会执行一次，然后直到wake方法被调用才会被唤醒
		    pub fn spawn(&self, task: impl Future<Output = ()> + Send + 'static) {
		        self.runnable.lock().unwrap().push_back(Arc::new(Mutex::new(task)));
		    }
		}

		pub fn run(&self) {
		    loop {
		        loop {
		            // 从队列中弹出一个可执行的任务
		            // 没有可执行任务就break
		            let Some(task) = self.runnable.lock().unwrap().pop_front() else { break };
		            let t2 = task.clone();
		
		            // 创建一个唤醒器，它的作用是把任务推回队列，这样任务就会被poll一次，达到唤醒的目的
		            // 唤醒器可能会被复制很多份
		            let wake = Arc::new(move || {
		                SCHEDULER.runnable.lock().unwrap().push_back(t2.clone());
		            });
		
		            // 调用该任务的 poll 方法
		            task.lock().unwrap().poll(Waker(wake));
		        }
		
		        // 如果没有可执行的任务，阻塞 epoll 直到某些任务变得就绪
		        // reactor直接调用epoll::wait
		        REACTOR.with(|reactor| reactor.wait()); // 👈
		    }
		}
}
```

调度器和反应器共同构成了一个 future 的运行时。调度器（runnable tasks）会跟踪哪些任务是可运行的，并轮询它们，当 epoll 告诉我们它们感兴趣的内容准备就绪时，反应器会将任务标记为可运行。

总的代码：也是异步web服务的运行时

```rust
trait Future {
    type Output;
    fn poll(&mut self, waker: Waker) -> Option<Self::Output>;
}

static SCHEDULER: Scheduler = Scheduler { /* ... */ };

// The scheduler.
#[derive(Default)]
struct Scheduler {
    runnable: Mutex<VecDeque<SharedTask>>,
}

type SharedTask = Arc<Mutex<dyn Future<Output = ()> + Send>>;

impl Scheduler {
		// 创建一个任务
    pub fn spawn(&self, task: impl Future<Output = ()> + Send + 'static);
    // 尝试唤醒一个任务
    pub fn run(&self);
}

// 意味着每个线程都会有一个独立的 REACTOR 实例，互相之间不会产生影响
thread_local! {
    static REACTOR: Reactor = Reactor::new();
}

// The reactor.
struct Reactor {
    epoll: RawFd,
    tasks: RefCell<HashMap<RawFd, Waker>>,
}

impl Reactor {
    pub fn new() -> Reactor;
    pub fn add(&self, fd: RawFd, waker: Waker);
    pub fn remove(&self, fd: RawFd);
    // epoll::wait直到有可执行任务并将其唤醒
    pub fn wait(&self);
}
```

- thread_local!
    - 用于定义线程局部变量
    - 线程局部变量在每个线程中都有独立的实例，彼此之间不会互相干扰，也就是说这个东西不需要在线程之间共享，不会被多个线程同时访问

## **Leaf futures**

Runtime crate里面的leaf future可以表示像嵌套字一样的资源

```rust
// stream is a leaf-future
let mut stream = tokio::net::TcpStream::connect("127.0.0.1:3000");
```

stream是一个叶子future，在这种情况下，通过返回一个 future，**程序可以在等待套接字读取完成时执行其他操作**，而不必阻塞等待数据的到达。

## **Non-leaf-futures**

使用 async 关键字编写的 future，它们代表了一个可以在执行器上运行的任务。在异步程序中，大部分的工作都会由这些非叶子 future 完成。通常，这样的任务**会等待叶子 future**（如上文所述的套接字读取操作）作为其中的一个操作，以完成任务。

非叶子 future 指的是那些代表了**一组操作的 future**，而不是单一操作的 future

```rust
// Non-leaf-future
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap();// <- yield
    println!("connected!");
    let result = stream.write(b"hello world\n").await; // <- yield
    println!("message sent!");
    ...
};
```

比如说这个代码，non_leaf 是一个非叶子 future，因为它包含了多个异步操作，而不是单一的异步操作。

- 第一个异步操作是stream那一步，它是一个异步连接操作，它尝试连接到指定的 IP 地址和端口。连接完成后它会返回一个socket
- 第二个异步操作是result那一步，它尝试将数据发送到套接字中。与连接操作类似，程序将在此处暂停执行当前任务，并等待写操作完成。写操作完成后，它会返回一个结果（可能是写入的字节数）
- 第二个异步操作必须等待第一个执行完成

## **Runtimes 运行时**

运行时（runtime）是指一种在程序执行过程中提供支持的**软件环境**。

在异步编程中，运行时负责管理**异步任务的执行和调度**。它提供了异步任务的执行环境，包括任务队列、线程池、异步任务的调度器等。

## **A useful mental model of an async runtime**

***rust的异步系统可以被分为三部分：***

1. ***Reactor 反应器***
   
    它负责**监听事件并将其分发给相应的future**。在异步编程中，事件可以是例如套接字准备好读取或写入数据，定时器触发等。反应器使用事件驱动的方式来管理和调度异步操作
    
2. ***Executor 执行器***
   
    执行器是负责实际**执行异步任务（Future）**的组件。它从反应器接收分发的任务，并在合适的时机执行这些任务
    
3. ***Future*** 
   
    代表了一个可能会在未来某个时间点完成的计算**任务**。未来允许我们以非阻塞的方式处理异步操作，通过异步操作的结果来触发后续的操作。
    
    在异步编程中，Future 代表了一个异步操作的结果。每当异步操作需要进行时，它会返回一个 Future 对象，该对象表示异步操作的状态和结果。Future 可以被轮询（poll）以检查操作是否已完成，如果操作尚未完成，Future 将返回一个 **Poll::Pending** 来指示调用者暂时挂起。
    

***这三个部分work together：***

通过一个 the `Waker` ，反应器可以告诉执行器，现在有一个Future准备好开始运行了。

life cycle：

- Waker是被执行器创建的
- 当future第一次被executor轮询的时候，它会被分配一个Waker对象（由这个executor创建的）的**clone**。Waker是一个共享的object，因此实际上所有的克隆都是指向同一个底层的原始的对象。因此，任何一个对Waker和它的clone的调用都可以唤醒一个与之绑定的Future
    - Waker 是一种异步编程的机制，用于**唤醒处于挂起状态的 Future**，以便它们可以继续执行。Waker 包含一个指向 Future 的引用，当操作完成并且数据可用时，**Future 将调用与其关联的 Waker 来通知调度器（executor）它可以继续执行**。
- 上面讲到的future会将分配到的Waker对象交给反应器，后面会用到

## **I/O密集型 VS CPU密集型任务**

```rust
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap(); // <-- yield

    // request a large dataset
    let result = stream.write(get_dataset_request).await.unwrap(); // <-- yield

    // wait for the dataset
    let mut response = vec![];
    stream.read(&mut response).await.unwrap(); // <-- yield

    // do some CPU-intensive analysis on the dataset
    let report = analyzer::analyze_data(response).unwrap();

    // send the results back
    stream.write(report).await.unwrap(); // <-- yield
};
```

两个yield之间的代码和executor是在同一个线程上运行的，也就是说当analyzer在分析数据的时候，executor也在计算数据，而不能处理新的请求

解决方法：

1. 我们可以创建一个新的`leaf future`，它将我们的任务发送到另一个线程，并在任务完成时解析。 我们可以像等待其他Future一样等待这个`leaf-future`。
2. 运行时可以有某种类型的管理程序来监视不同的任务占用多少时间，并将 executor 移动到不同的线程，这样即使我们的分析程序任务阻塞了原始的执行程序线程，它也可以继续运行。
3. 您可以自己创建一个与运行时兼容的`reactor`，以您认为合适的任何方式进行分析，并返回一个可以等待的future。

# **异步的web服务**

为主任务实现Future特征：

```rust
// 主任务的状态枚举
enum Main {
    Start, // 起始状态，需要初始化监听器、往反应器添加监听器的事件、改变主任务状态
    Accept { listener: TcpListener }, // 👈
}

// impl Future for Main {
fn poll(&mut self, waker: Waker) -> Option<()> {
		// 起始状态
    if let Main::Start = self {
		    // 设置一个非阻塞的监听器
        let listener = TcpListener::bind("localhost:3000").unwrap();
        listener.set_nonblocking(true).unwrap();
		    
		    // 在REACTOR添加这个事件（监听器听到新连接）
		    REACTOR.with(|reactor| {
		        reactor.add(listener.as_raw_fd(), waker);
		    });
		    *self = Main::Accept { listener }; // 更新主任务状态
    }
    
    // 已经开始监听连接的状态，尝试提取出listener
    if let Main::Accept { listener } = self {
        match listener.accept() {
            Ok((connection, _)) => {
                connection.set_nonblocking(true).unwrap();
                // 使用调度器来创建一个任务
                SCHEDULER.spawn(Handler { // 👈
                    connection,
                    state: HandlerState::Start,
                });
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                return None;
            }
            Err(e) => panic!("{e}"),
        }
    }
}
```

- 编写主任务：主任务掌握整个可执行任务队列的开关（waker）
- 主任务处于起始状态时：调用REACTOR的add函数，在epoll中注册“监听器监听到有新连接”事件
- 主任务处于Accept状态时：尝试获取listener，要是listener就绪，就尝试捕获一个连接
    - 使用调度器来创建一个任务，这个任务就是处理这个新连接
      
        反应器：监听事件，调度器：任务调度
        

***任务：处理连接***

逻辑跟之前一样，根据连接状态来判断任务可执行的情况下接下来应该干嘛；并且使用epoll来监听任务何时可执行——现在epoll已经被反应器封装好了，在epoll注册事件变为对反应器调用add

```rust
struct Handler {
    connection: TcpStream,
    state: HandlerState,
}

enum HandlerState {
    Start,
    Read {
        request: [u8; 1024],
        read: usize,
    },
    Write {
        response: &'static [u8],
        written: usize,
    },
    Flush,
}

impl Future for Handler {
    type Output = ();
		
    fn poll(&mut self, waker: Waker) -> Option<Self::Output> {
		    // 初始状态：在反应器注册事件并调用一次任务的poll
        if let HandlerState::Start = self.state {
            // 注册当前连接以便得到通知
            REACTOR.with(|reactor| {
                reactor.add(self.connection.as_raw_fd(), waker);
            });

            self.state = HandlerState::Read {
                request: [0u8; 1024],
                read: 0,
            };
        }
        // 其他状态：能执行的任务就让它继续执行
        // read the request
		    if let HandlerState::Read { request, read } = &mut self.state {
		        loop {
		            match self.connection.read(&mut request[*read..]) {
		                Ok(0) => {
		                    println!("client disconnected unexpectedly");
		                    return Some(());
		                }
		                Ok(n) => *read += n,
		                // 当前任务暂时没法执行
		                Err(e) if e.kind() == io::ErrorKind::WouldBlock => return None, // 👈
		                Err(e) => panic!("{e}"),
		            }
		
		            // did we reach the end of the request?
		            let read = *read;
		            if read >= 4 && &request[read - 4..read] == b"\r\n\r\n" {
		                break;
		            }
		        }
		
		        // we're done, print the request
		        let request = String::from_utf8_lossy(&request[..*read]);
		        println!("{}", request);
		
		        // and move into the write state
		        let response = /* ... */;
		
		        self.state = HandlerState::Write {
		            response: response.as_bytes(),
		            written: 0,
		        };
		    }
		
		    // write the response
		    if let HandlerState::Write { response, written } = &mut self.state {
		        // self.connection.write...
		
		        // successfully wrote the response, try flushing next
		        self.state = HandlerState::Flush;
		    }
		
		    // flush the response
		    if let HandlerState::Flush = self.state {
		        match self.connection.flush() {
		            Ok(_) => {}
		            Err(e) if e.kind() == io::ErrorKind::WouldBlock => return None, // 👈
		            Err(e) => panic!("{e}"),
		        }
		    }
    }
    // 任务执行完了，在反应器移除任务
    REACTOR.with(|reactor| {
        reactor.remove(self.connection.as_raw_fd());
    });

    Some(())
}
```

# 全功能的Web服务

## poll_fn

`poll_fn`的实现：

```rust
// create a future with the given `poll` function
// 函数返回值是一个实现了Future特征的类型
pub fn poll_fn<F, T>(f: F) -> impl Future<Output = T>
where
    F: FnMut(Waker) -> Option<T>,
{
    struct PollFn<F>(F);

    impl<F, T> Future for PollFn<F>
    where
        F: FnMut(Waker) -> Option<T>,
    {
        type Output = T;

        fn poll(&mut self, waker: Waker) -> Option<Self::Output> {
            (self.0)(waker)
        }
    }

    PollFn(f)
}
```

1. 定义`poll_fn`函数：接收一个闭包`f`，该闭包接受一个`Waker`并返回`Option<T>`。
2. 在内部定义了结构体PollFn，并为它实现了Future特征。这个结构体里面有一个闭包，当调用结构体的poll函数时，会将waker作为结构体里面闭包的参数

<aside>
💡 **Closure Call**

In Rust, if you have **a variable that holds a closure**, you can call it just like a function by using parentheses **`()`** and passing the required arguments.

</aside>

用法：

```rust
fn main() {
    SCHEDULER.spawn(listen());
    SCHEDULER.run();
}

// 使用poll_fn为listen创建了两个Future
fn listen() -> impl Future<Output = ()> {
		// 现在start是一个具有Future特征的类型PollFn
		// 当它被poll时，就会注册一个监听器，并且把传入的waker注册到reactor
    let start = poll_fn(|waker| {
        let listener = TcpListener::bind("localhost:3000").unwrap();

        listener.set_nonblocking(true).unwrap();

        REACTOR.with(|reactor| {
            reactor.add(listener.as_raw_fd(), waker);
        });

        Some(listener) // 返回listener
    });

    let accept = poll_fn(|_| match listener.accept() {
        // 尝试捕获一个连接
        Ok((connection, _)) => {
            connection.set_nonblocking(true).unwrap();
						// 捕获成功就创建一个任务交给调度器
            SCHEDULER.spawn(Handler {
                connection,
                state: HandlerState::Start,
            });

            None
        }
        Err(e) if e.kind() == io::ErrorKind::WouldBlock => None,
        Err(e) => panic!("{e}"),
    });
}
```

## chain

Chain future允许将两个future链接在一起，使用提供的闭包从第一个future过渡到第二个future。

```rust
enum Chain<T1, F, T2> {
		// transition是一个闭包，用于将第一个future的输出转换为第二个future
    First { future1: T1, transition: F },
    Second { future2: T2 },
}

impl<T1, F, T2> Future for Chain<T1, F, T2>
where
    T1: Future,
    F: FnOnce(T1::Output) -> T2, // 接收第一个Future的输出作为参数，输出第二个Future
    T2: Future,
{
    type Output = T2::Output;

    fn poll(&mut self, waker: Waker) -> Option<Self::Output> {
        if let Chain::First { future1, transition } = self {
            // poll the first future
            match future1.poll(waker.clone()) {
                Some(value) => {
                    // first future is done, transition into the second
                    // 闭包执行完一次后就会将值设为None
                    let future2 = (**transition**.take().unwrap())(value); // 👈使用闭包，用第一个Future的结果来创建第二个Future
                    *self = Chain::Second { future2 };
                }
                // first future is not ready, return
                None => return None,
            }
        }

        if let Chain::Second { future2 } = self {
            // first future is already done, poll the second
            return future2.poll(waker); // 👈
        }

        None
    }
}
```

为Future特征定义chain函数：

```rust
fn chain<F, T>(self, transition: F) -> Chain<Self, F, T>
where
    F: FnOnce(Self::Output) -> T,
    T: Future,
    Self: Sized,
{
    Chain::First {
        future1: self, // 第一个future是调用chain函数的这个future
        transition: Some(transition),
    }
}
```

- chain函数接受一个闭包transition，该闭包的操作是将一个Future的输出转换为另一个Future，chain函数返回值是一个chain枚举
- 函数创建一个Chain::First变量，将调用函数时传入的闭包transition作为这个枚举变量的transition，并将Chain::First作为函数返回值
- 当使用chain函数获得一个Chain类型的变量，调用它的poll函数时，就会产生一个迷你自动机：future1被poll之后，状态转移，Chain枚举变量由Chain::First变为Chain::Second，然后future2被poll

之前需要调用两次poll_fn来处理两个future，而这两个future之间又有联系，即listener必须先注册了才能开始接收连接。之前的处理方法是：定义一个枚举Main，表示主任务的状态，当主任务的状态为Start的时候，添加一个TCP监听器并将状态改为Accept…这样相当于是手动把自动机的状态进行转移。现在有了Chain，就可以通过poll一个Chain枚举类型，将状态进行自动转移了

```rust
// 手动状态转移

// 主任务的状态枚举
enum Main {
    Start, // 起始状态，需要初始化监听器、往反应器添加监听器的事件、改变主任务状态
    Accept { listener: TcpListener }, // 👈
}

// impl Future for Main {
fn poll(&mut self, waker: Waker) -> Option<()> {
		// 起始状态
    if let Main::Start = self {
		    // 设置一个非阻塞的监听器
        let listener = TcpListener::bind("localhost:3000").unwrap();
        listener.set_nonblocking(true).unwrap();
		    
		    // 在REACTOR添加这个事件（监听器听到新连接）
		    REACTOR.with(|reactor| {
		        reactor.add(listener.as_raw_fd(), waker);
		    });
		    *self = Main::Accept { listener }; // 更新主任务状态
    }
    
    // 已经开始监听连接的状态，尝试提取出listener
    if let Main::Accept { listener } = self {
        match listener.accept() {
            Ok((connection, _)) => {
                connection.set_nonblocking(true).unwrap();
                // 使用调度器来创建一个任务
                SCHEDULER.spawn(Handler { // 👈
                    connection,
                    state: HandlerState::Start,
                });
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                return None;
            }
            Err(e) => panic!("{e}"),
        }
    }
}
```

现在有了Chain Future就可以把两个future连在一起了：

```rust
fn main() {
    SCHEDULER.spawn(listen());
    SCHEDULER.run();
}

fn listen() -> impl Future<Output = ()> {
    poll_fn(|waker| {
        let listener = TcpListener::bind("localhost:3000").unwrap();
        // ...

        REACTOR.with(|reactor| {
            reactor.add(listener.as_raw_fd(), waker);
        });

        Some(listener) // poll_fn在这里返回了一个Future表示第一个Future完成
    }) // 👈 下一个状态
    .chain(|listener| {
			  // chain函数的参数：translation闭包
			  // 即使用future1的输出listener，来作为future2的输入，来监听连接，最终future2返回None
        poll_fn(move |_| match listener.accept() {
            Ok((connection, _)) => {
                // ...
                SCHEDULER.spawn(Handler {
                    connection,
                    state: HandlerState::Start,
                });

                None
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => None,
            Err(e) => panic!("{e}"),
        })
    })
}
```

将handle任务也这样用chain来组合：遇到报错，因为chain获取了connection的所有权

其实只需要将connection放在堆上就可以了：这样一来，就可以确保每个chain函数都可以安全引用connection

```rust
let connection = Arc::new(connection); // 👈
```

因为在创建future的时候要使用data的引用，现在data的生命周期已经明了，但是它的位置会变化，因此使用堆上的智能指针，让它的位置不变

Scheduler和Reactor是怎么连在一起的？

# **优雅的Web服务**

优雅退出：当按下 `ctrl+c` 时，不是粗暴地立刻关闭程序，而是应该立刻停止接受新的连接请求，同时等待已建立连接的请求执行完毕，超过30秒没执行完毕的请求将会被终止，然后服务端程序才退出

```rust
use signal_hook::consts::signal::SIGINT;
use signal_hook::iterator::Signals;

fn ctrl_c() {
    let mut signal = Signals::new(&[SIGINT]).unwrap();
    let _ctrl_c = signal.forever().next().unwrap(); // 阻塞线程
}
```

使用signal来接受ctrl C信号， forever().next() 会阻塞线程直到收到信号。但现在我们的程序只有一个线程，阻塞线程意味着所有的其他任务都会被阻塞，所以这个接收信号的动作也应该是个future

在epoll中注册这个信号，并在接收到信号时通知主线程

## spawn_blocking

让阻塞任务单独在一个线程上执行：

```rust
fn spawn_blocking(blocking_work: impl FnOnce() + Send + 'static) -> impl Future<Output = ()> {
    let state: Arc<Mutex<(bool, Option<Waker>)>> = Arc::default();
    let state_handle = state.clone();

    // run the blocking work on a separate thread
    std::thread::spawn(move || {
        // run the work
        blocking_work();

        // mark the task as done
        let (done, waker) = &mut *state_handle.lock().unwrap();
        *done = true;

        // wake the waker
        if let Some(waker) = waker.take() {
            waker.wake();
        }
    });

    poll_fn(move |waker| match &mut *state.lock().unwrap() {
        // work is not completed, store our waker and come back later
        (false, state) => {
            *state = Some(waker);
            None
        }
        // the work is completed
        (true, _) => Some(()),
    })
}
```

- 要执行阻塞任务，首先必须让这个会使整个线程阻塞的任务在一个单独的线程上执行、
- 并且要为这个阻塞任务设置状态，以便主线程知道阻塞任务完成了
- 如果状态中存储了一个唤醒器，它将被获取并调用以唤醒Future
- 用poll_fn创建一个Future，如果阻塞任务还没有完成的话，就在状态中存入一个waker
- 返回一个future，如果任务完成了，就返回一个Some(())，表示future准备好了；如果任务没完成就返回一个None，表示future没准备好

```rust
fn ctrl_c() -> impl Future<Output = ()> {
		// 调用spawn_blocking，传入一个阻塞操作，阻塞操作完成后自然会返回一个future
    spawn_blocking(|| {
        let mut signal = Signals::new(&[SIGINT]).unwrap();
        let _ctrl_c = signal.forever().next().unwrap();
    })
}
```

spawn_blocking 服务将该 future 作为异步版本的 JoinHandle 返回。阻塞任务在单独的线程中运行时，主线程可以异步地等待它完成

- `spawn_blocking` 返回一个 `Future`，这个 `Future` 可以在异步上下文中被 `await`。
- 这个 `Future` 的 `poll` 方法会检查阻塞任务是否已经完成，如果完成则返回 `Some(())`，否则返回 `None` 并存储 `Waker`
- `spawn_blocking` 是一种方便的抽象，允许我们在单独的线程中运行阻塞任务，同时返回一个 `Future`，这个 `Future` 可以在主线程中被异步等待。

## select

> spawn_blocking 是一个非常方便的抽象，常用于处理异步程序中的阻塞 API。
> 
> 
> 好的，我们有能等待 ctrl+c 信号的 future 了。
> 
> 如果您还记得我们的服务端使用阻塞I/O时的情景，当时我们想知道如何监视信号，才能在信号到达后立即中止连接侦听器循环。当时我们就意识到需要以某种方式来**同时**监听传入连接和ctrl+c信号。
> 
> 因为 accept 是阻塞的，这并不容易，但是有了 future 就让这变得可能了！
> 
> 我们将这包装成另一个 future。给定两个 future，我们能创建一个包装器实现选择功能：哪个 future 先完成，就将它的输出作为包装器的输出。
> 

这个意思就是：

- 通过将 accept 和信号监听封装成 Future，我们可以在异步上下文中处理这两种操作，即同时监听两个信号
- 我们可以创建一个包装器 `Future`，这个包装器实现了选择功能：监听多个 `Future`，并且哪个 `Future` 先完成，就将它的输出作为包装器的输出。

```rust
// select函数，允许同时等待两个Future

fn select<L, R>(left: L, right: R) -> Select<L, R> {
    Select { left, right } // 两个Future
}

struct Select<L, R> {
    left: L,
    right: R
}

enum Either<L, R> {
    Left(L),
    Right(R)
}

impl<L, R> Future for Select<L, R> {
    type Output = Either<L, R>;
		
		// 将同一个waker交给两个future
    fn poll(&mut self, waker: Waker) -> Option<Self::Output> {
		    // 首先轮询left
        if let Some(output) = self.left.poll(waker.clone()) {
		        // 如果完成，就返回这个future的输出
            return Some(Either::Left(output));
        }
				// 然后轮询right
        if let Some(output) = self.right.poll(waker) {
            return Some(Either::Right(output));
        }
				// 如果两个future都没准备好，就返回None
        None
    }
}
```

> 因为我们将同一个唤醒器传递给两个 future ，任何一个 future 的进展都会通知我们，我们可以检查其中一个是否完成。
> 

因为这里将同一个waker交给了reactor，所以有事件发生的时候，reactor就会调用wake函数

```rust
fn listen() -> impl Future<Output = ()> {
    poll_fn(|waker| {
        // ...
    })
    .chain(|listener| {
		    // 这里就跟之前不一样了，必须用listen来接收poll_fn的返回值也就是PollFn
        let listen = poll_fn(move |_| match listener.accept() {
            Ok((connection, _)) => {
                // ...
                SCHEDULER.spawn(handle(connection));
                None
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => None,
            Err(e) => panic!("{e}"),
        });

        select(listen, ctrl_c()) // 👈在这里监听这两个future哪个先完成
    })
}
```

但是listen好像永远不会完成，listener将会一直监听是否有新连接。所以设计：一旦收到关闭的信号，等待当前处理中的请求请求执行完毕，30秒后关闭，要么等待30秒，要么所有处理中的请求执行完毕——这个也可以用select

`select(timer, request_counter)` ：两个Future

- 计时器timer，计时30s就关闭程序
    - let timer = spawn_blocking(|| thread::sleep(Duration::from_secs(30)));
    - 新建一个线程来计时
- 计数器request_counter：
    - 为了了解请求全部执行完毕的准确时间，我们需要统计执行中的请求数量
    
    ```rust
    #[derive(Default)]
    struct Counter {
        state: Mutex<(usize, Option<Waker>)>, // usize表示剩余任务数量，Option表示用于唤醒wait_for_zero的唤醒器
    }
    
    impl Counter {
        fn increment(&self) {
            let (count, _) = &mut *self.state.lock().unwrap();
            *count += 1;
        }
    
        fn decrement(&self) {
            let (count, waker) = &mut *self.state.lock().unwrap();
            *count -= 1;
    
            // 已经是最后一个任务了
            if *count == 0 {
                // 唤醒关闭的future
                if let Some(waker) = waker.take() {
                    waker.wake(); // 唤醒
                }
            }
        }
    
        fn wait_for_zero(self: Arc<Self>) -> impl Future<Output = ()> {
            poll_fn(move |waker| {
                match &mut *self.state.lock().unwrap() {
                    // 任务已全部完成
                    (0, _) => Some(()),
                    // 任务未全部完成，保存唤醒器
                    (_, state) => {
                        *state = Some(waker);
                        None
                    }
                }
            })
        }
    }
    ```
    

最终的结果：

```rust
fn graceful_shutdown(tasks: Arc<Counter>) -> impl Future<Output = ()> {
    poll_fn(|waker| {
		    // 设置timer，直到30s后这个Future才会返回Some
        let timer = spawn_blocking(|| thread::sleep(Duration::from_secs(30)));
        // 设置计数器，直到计数为0这个Future才会返回Some
        let request_counter = tasks.wait_for_zero(); // 👈
        select(timer, request_counter)
    }).chain(|_| {
		    // 不管是哪个Future先完成，都关闭程序
        // graceful shutdown process complete, now we actually exit
        println!("Graceful shutdown complete");
        std::process::exit(0)
    })
}
```

Select是怎么被poll的？

Select是一个实现了Future特征的结构体，当它调用chain函数的时候就会被poll。调用chain的时候，调用它的Future1和转换过来的Future2都会被poll，调用chain的那个Future永远都是Future1

# 标准库

## Future Trait

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// poll函数的参数：是一个对waker的简单包装
impl Context<'_> {
    pub fn from_waker(waker: &'a Waker) -> Context<'a>  { /* ... */ }
    pub fn waker(&self) -> &'a Waker  { /* ... */ }
}

// poll函数的返回值：是一个枚举类型
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

## Waker

```rust
impl Waker {
    pub fn wake(self) { /* ... */ }
}
```

## Pinging

```rust
#[derive(Copy, Clone)]
pub struct Pin<P> {
    pointer: P,
}
```

Pin：

- 将 `Future` 在内存中固定到一个位置，防止某些Future位置移动导致对里面数据的引用会被破坏
- 它包裹一个指针，并且能确保该指针指向的数据不会被移动，例如 `Pin<&mut    T>` , `Pin<&T>` , `Pin<Box<T>>` ，都能确保 `T` 不会被移动。

Unpin：

- 它是一个特征，拥有Unpin特征的类型可以被安全移动

## **async/.await**

`async/.await` 是 Rust 内置的语言特性，可以让我们用同步的方式去编写异步的代码。

通过 `async` 标记的语法块会被转换成实现了`Future`特征的状态机。 与同步调用阻塞当前线程不同，当`Future`执行并遇到阻塞时，它会让出当前线程的控制权，这样其它的`Future`就可以在该线程中运行，这种方式完全不会导致当前线程的阻塞。

### async

用于创建一个Future，不使用 poll_fn 来创建 future，而是添加 async 关键字到 fn 的前面

```rust
async fn do_something() {
    println!("go go go !");
}
```

创建一个异步函数，异步函数的返回值是一个 `Future`，若直接调用该函数，不会输出任何结果，因为 `Future` 还未被执行

使用执行器来使用Future：

```rust
// `block_on`会阻塞当前线程直到指定的`Future`执行完成，这种阻塞当前线程以等待任务完成的方式较为简单、粗暴，
// 好在其它运行时的执行器(executor)会提供更加复杂的行为，例如将多个`future`调度到同一个线程上执行。
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // 返回一个Future, 因此不会打印任何输出
    block_on(future); // 执行`Future`并等待其运行完成，此时"hello, world!"会被打印输出
}
```

### .await

在`async fn`函数中使用`.await`可以等待另一个异步调用的完成。但是与`block_on`不同，`.await`并不会阻塞当前的线程，而是异步的等待`Future A`的完成，在等待的过程中，该线程还可以继续执行其它的`Future B`，最终实现了并发处理的效果。

await 在等待另一个 future 完成时返回 Poll:Pending，直到 future 完成

```rust
async fn foo() {
    let one = one().await; // 等待one
    let two = two().await; // 等待two
    assert_eq!(one + 1, two);
}

async fn two() -> usize {
    one().await + 1 // 等待one()执行完毕，将其返回值+1作为two()的返回值
}

async fn one() -> usize {
    1
}
```
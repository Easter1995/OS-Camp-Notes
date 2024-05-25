# Rust Future

type: Post
status: Invisible
date: 2024/05/23
tags: 编程, 课程
category: 操作系统

***异步：***不用等待一个任务执行完成就可以开始执行下一个任务。与多线程不同的是，它并不需要创建额外的线程，此外，所有函数调用都是静态分发的，也没有堆分配。

异步操作意味着当你执行某个操作时，程序不会等待该操作完成，而是继续执行其他操作

![image-20240525111532987](https://cdn.jsdelivr.net/gh/Easter1995/blog-image/202405251115729.png)

***为什么不直接就用线程来实现”同时“执行多个任务呢？***

- 线程开销大
- 创建一个线程需要使用很多系统调用，开销很大
- 操作系统反应可能不及时

# Future

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

## **Leaf futures**

Runtime crate里面的leaf future可以表示像嵌套字一样的资源

```rust
// stream is a **leaf-future**
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

# **I/O密集型 VS CPU密集型任务**

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
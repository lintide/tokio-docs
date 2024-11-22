Bridging with sync code
与同步代码的桥接

In most examples of using Tokio, we mark the main function with #[tokio::main] and make the entire project asynchronous.
在大多数使用 Tokio 的示例中，我们用 #[tokio::main] 标记 main 函数，并使整个项目异步化。

In some cases, you may need to run a small portion of synchronous code. For more information on that, see spawn_blocking.
在某些情况下，你可能需要运行一小部分同步代码。有关更多信息，请参见 spawn_blocking。

In other cases, it may be easier to structure the application as largely synchronous, with smaller or logically distinct asynchronous portions. For instance, a GUI application might want to run the GUI code on the main thread and run a Tokio runtime next to it on another thread.
在其他情况下，将应用程序构建为以同步代码为主，只有较小或逻辑上独立的异步部分可能更容易。例如，GUI 应用程序可能希望在主线程上运行 GUI 代码，并在另一个线程上运行 Tokio 运行时。

This page explains how you can isolate async/await to a small part of your project.
本页解释了如何将 async/await 隔离到项目的一小部分中。

What `#[tokio::main]` expands to
`#[tokio::main]` 展开成什么

The `#[tokio::main]` macro is a macro that replaces your main function with a non-async main function that starts a runtime and then calls your code. For instance, this:
`#[tokio::main]` 宏是一个将你的 main 函数替换为非异步 main 函数的宏，该函数启动运行时然后调用你的代码。例如，这个：

```rust
#[tokio::main]
async fn main() {
    println!("Hello world");
}
```

is turned into this:
会被转换成这样：

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("Hello world");
        })
}
```

by the macro. To use async/await in our own projects, we can do something similar where we leverage the block_on method to enter the asynchronous context where appropriate.
通过这个宏。要在我们自己的项目中使用 async/await，我们可以做类似的事情，在适当的地方利用 block_on 方法进入异步上下文。

A synchronous interface to mini-redis
mini-redis 的同步接口

In this section, we will go through how to build a synchronous interface to mini-redis by storing a Runtime object and using its block_on method. In the following sections, we will discuss some alternate approaches and when you should use each approach.
在本节中，我们将通过存储 Runtime 对象并使用其 block_on 方法来了解如何构建 mini-redis 的同步接口。在接下来的章节中，我们将讨论一些替代方法以及何时应该使用每种方法。

The interface that we will be wrapping is the asynchronous Client type. It has several methods, and we will implement a blocking version of the following methods:
我们要包装的接口是异步 Client 类型。它有几个方法，我们将实现以下方法的阻塞版本：

Client::get
Client::set
Client::set_expires
Client::publish
Client::subscribe
To do this, we introduce a new file called src/clients/blocking_client.rs and initialize it with a wrapper struct around the async Client type:

```rust
use tokio::net::ToSocketAddrs;
use tokio::runtime::Runtime;

pub use crate::clients::client::Message;

/// Established connection with a Redis server.
pub struct BlockingClient {
    /// The asynchronous `Client`.
    inner: crate::clients::Client,

    /// A `current_thread` runtime for executing operations on the
    /// asynchronous client in a blocking manner.
    rt: Runtime,
}

impl BlockingClient {
    pub fn connect<T: ToSocketAddrs>(addr: T) -> crate::Result<BlockingClient> {
        let rt = tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()?;

        // Call the asynchronous connect method using the runtime.
        let inner = rt.block_on(crate::clients::Client::connect(addr))?;

        Ok(BlockingClient { inner, rt })
    }
}
```

Here, we have included the constructor function as our first example of how to execute asynchronous methods in a non-async context. We do this using the block_on method on the Tokio Runtime type, which executes an asynchronous method and returns its result.
这里，我们包含了构造函数作为在非异步上下文中执行异步方法的第一个示例。我们使用 Tokio Runtime 类型上的 block_on 方法来实现这一点，该方法执行异步方法并返回其结果。

One important detail is the use of the current_thread runtime. Usually when using Tokio, you would be using the default multi_thread runtime, which will spawn a bunch of background threads so it can efficiently run many things at the same time. For our use-case, we are only going to be doing one thing at the time, so we won't gain anything by running multiple threads. This makes the current_thread runtime a perfect fit as it doesn't spawn any threads.
一个重要的细节是使用 current_thread 运行时。通常使用 Tokio 时，你会使用默认的 multi_thread 运行时，它会生成一堆后台线程以便能够同时高效地运行多个任务。对于我们的用例，我们一次只做一件事，所以运行多个线程不会带来任何好处。这使得 current_thread 运行时成为完美的选择，因为它不会生成任何线程。

The enable_all call enables the IO and timer drivers on the Tokio runtime. If they are not enabled, the runtime is unable to perform IO or timers.
enable_all 调用启用了 Tokio 运行时上的 IO 和定时器驱动程序。如果不启用它们，运行时将无法执行 IO 或定时器操作。

Because the current_thread runtime does not spawn threads, it only operates when block_on is called. Once block_on returns, all spawned tasks on that runtime will freeze until you call block_on again. Use the multi_threaded runtime if spawned tasks must keep running when not calling block_on.
因为 current_thread 运行时不会生成线程，它只在调用 block_on 时才运行。一旦 block_on 返回，该运行时上的所有生成的任务都将冻结，直到你再次调用 block_on。如果生成的任务在不调用 block_on 时必须继续运行，请使用 multi_threaded 运行时。

Once we have this struct, most of the methods are easy to implement:
一旦我们有了这个结构体，大多数方法都很容易实现：

```rust
use bytes::Bytes;
use std::time::Duration;

impl BlockingClient {
    pub fn get(&mut self, key: &str) -> crate::Result<Option<Bytes>> {
        self.rt.block_on(self.inner.get(key))
    }

    pub fn set(&mut self, key: &str, value: Bytes) -> crate::Result<()> {
        self.rt.block_on(self.inner.set(key, value))
    }

    pub fn set_expires(
        &mut self,
        key: &str,
        value: Bytes,
        expiration: Duration,
    ) -> crate::Result<()> {
        self.rt.block_on(self.inner.set_expires(key, value, expiration))
    }

    pub fn publish(&mut self, channel: &str, message: Bytes) -> crate::Result<u64> {
        self.rt.block_on(self.inner.publish(channel, message))
    }
}
```

The Client::subscribe method is more interesting because it transforms the Client into a Subscriber object. We can implement it in the following manner:
Client::subscribe 方法更有趣，因为它将 Client 转换为 Subscriber 对象。我们可以按以下方式实现它：

```rust
/// A client that has entered pub/sub mode.
///
/// Once clients subscribe to a channel, they may only perform
/// pub/sub related commands. The `BlockingClient` type is
/// transitioned to a `BlockingSubscriber` type in order to
/// prevent non-pub/sub methods from being called.
pub struct BlockingSubscriber {
    /// The asynchronous `Subscriber`.
    inner: crate::clients::Subscriber,

    /// A `current_thread` runtime for executing operations on the
    /// asynchronous client in a blocking manner.
    rt: Runtime,
}

impl BlockingClient {
    pub fn subscribe(self, channels: Vec<String>) -> crate::Result<BlockingSubscriber> {
        let subscriber = self.rt.block_on(self.inner.subscribe(channels))?;
        Ok(BlockingSubscriber {
            inner: subscriber,
            rt: self.rt,
        })
    }
}

impl BlockingSubscriber {
    pub fn get_subscribed(&self) -> &[String] {
        self.inner.get_subscribed()
    }

    pub fn next_message(&mut self) -> crate::Result<Option<Message>> {
        self.rt.block_on(self.inner.next_message())
    }

    pub fn subscribe(&mut self, channels: &[String]) -> crate::Result<()> {
        self.rt.block_on(self.inner.subscribe(channels))
    }

    pub fn unsubscribe(&mut self, channels: &[String]) -> crate::Result<()> {
        self.rt.block_on(self.inner.unsubscribe(channels))
    }
}
```

So, the subscribe method will first use the runtime to transform the asynchronous Client into an asynchronous Subscriber. Then, it will store the resulting Subscriber together with the Runtime and implement the various methods using block_on.
因此，subscribe 方法首先使用运行时将异步 Client 转换为异步 Subscriber。然后，它将生成的 Subscriber 与 Runtime 一起存储，并使用 block_on 实现各种方法。

Note that the asynchronous Subscriber struct has a non-async method called get_subscribed. To handle this, we simply call it directly without involving the runtime.
注意，异步 Subscriber 结构体有一个名为 get_subscribed 的非异步方法。要处理这个，我们只需直接调用它，而不需要涉及运行时。

Other approaches
其他方法

The above section explains the simplest way to implement a synchronous wrapper, but it is not the only way. The approaches are:
上面的部分解释了实现同步包装器的最简单方法，但这不是唯一的方法。这些方法包括：

Create a Runtime and call block_on on the async code.
创建一个 Runtime 并在异步代码上调用 block_on。

Create a Runtime and spawn things on it.
创建一个 Runtime 并在其上生成任务。

Run the Runtime in a separate thread and send messages to it.
在单独的线程中运行 Runtime 并向其发送消息。

Spawning things on a runtime
在运行时上生成任务

The Runtime object has a method called spawn. When you call this method, you create a new background task to run on the runtime. For example:
Runtime 对象有一个名为 spawn 的方法。当你调用这个方法时，你会创建一个新的后台任务在运行时上运行。例如：

[代码块保持不变...]

In the above example, we spawn 10 background tasks on the runtime, then wait for all of them. As an example, this could be a good way of implementing background network requests in a graphical application because network requests are too time consuming to run them on the main GUI thread. Instead, you spawn the request on a Tokio runtime running in the background, and have the task send information back to the GUI code when the request has finished, or even incrementally if you want a progress bar.
在上面的例子中，我们在运行时上生成了 10 个后台任务，然后等待它们全部完成。作为一个例子，这可能是在图形应用程序中实现后台网络请求的好方法，因为网络请求太耗时，不适合在主 GUI 线程上运行。相反，你可以在后台运行的 Tokio 运行时上生成请求，并让任务在请求完成时将信息发送回 GUI 代码，如果你想要进度条，甚至可以逐步发送。

In this example, it is important that the runtime is configured to be a multi_thread runtime. If you change it to be a current_thread runtime, you will find that the time consuming task finishes before any of the background tasks start. This is because background tasks spawned on a current_thread runtime will only execute during calls to block_on as the runtime otherwise doesn't have anywhere to run them.
在这个例子中，将运行时配置为 multi_thread 运行时很重要。如果你将其更改为 current_thread 运行时，你会发现耗时任务在任何后台任务开始之前就完成了。这是因为在 current_thread 运行时上生成的后台任务只会在调用 block_on 期间执行，因为运行时否则没有地方运行它们。

The example waits for the spawned tasks to finish by calling block_on on the JoinHandle returned by the call to spawn, but this isn't the only way to do it. Here are some alternatives:
该示例通过在 spawn 调用返回的 JoinHandle 上调用 block_on 来等待生成的任务完成，但这不是唯一的方法。以下是一些替代方案：

Use a message passing channel such as tokio::sync::mpsc.
使用消息传递通道，如 tokio::sync::mpsc。

Modify a shared value protected by e.g. a Mutex. This can be a good approach for a progress bar in a GUI, where the GUI reads the shared value every frame.
修改由 Mutex 等保护的共享值。这对于 GUI 中的进度条来说是一个很好的方法，GUI 每帧都会读取共享值。

The spawn method is also available on the Handle type. The Handle type can be cloned to get many handles to a runtime, and each Handle can be used to spawn new tasks on the runtime.
spawn 方法也可在 Handle 类型上使用。Handle 类型可以被克隆以获得运行时的多个句柄，每个 Handle 都可以用于在运行时上生成新任务。

Sending messages
发送消息

The third technique is to spawn a runtime and use message passing to communicate with it. This involves a bit more boilerplate than the other two approaches, but it is the most flexible approach. You can find a basic example below:
第三种技术是生成一个运行时并使用消息传递与之通信。这比其他两种方法需要更多的样板代码，但这是最灵活的方法。你可以在下面找到一个基本示例：

[代码块保持不变...]

This example could be configured in many ways. For instance, you could use a Semaphore to limit the number of active tasks, or you could use a channel in the opposite direction to send a response to the spawner. When you spawn a runtime in this way, it is a type of actor.
这个示例可以通过多种方式配置。例如，你可以使用 Semaphore 来限制活动任务的数量，或者可以使用相反方向的通道向生成器发送响应。当你以这种方式生成运行时时，它是一种 actor 类型。
# Flow 概述

`low` 表示的是**一个用于异步计算的数据流**。`Flow` 接口本身只允许收集那些流动的元素，这也就是说每个元素只有到达流的末端时，我们才去处理它们（Flow 的 `collect` 类似于集合的 `forEach`）。

```kotlin
interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}
```

可以看到， `collect` 是 `Flow` 中唯一的成员函数。其它的函数都被定义为扩展函数。这与 `Iterable` 或 `Sequence` 类似，它们都只有 `iterator` 作为成员函数。

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}

interface Sequence<out T> {
    operator fun iterator(): Iterator<T>
}
```

### 流 vs 其他表示数据的方式

对于使用 `RxJava` 或 `Reactor` 的人来说，流的概念应该是他们最熟悉的，但对于其他不熟悉的人，可能需要更好的解释。

假设你需要一个函数返回多个值，如果这些值同时被提供出来，我们会使用像 `List` 和 `Set` 这样的集合。

```kotlin
fun allUsers(): List<User> =
    api.getAllUsers().map { it.toUser() }
```

这里的本质是 `List` 和 `Set` 表示的是一个完全计算的集合。处理这些值需要时间，所以我们需要等待所有的值处理好，然后才能拿到它们。

```kotlin
fun getList(): List<Int> = List(3) {
    Thread.sleep(1000)
    "User$it"
}

fun main() {
    val list = getList()
    println("Function started")
    list.forEach { println(it) }
}
// (3 sec)
// Function started
// User0
// User1
// User2
```

如果元素是一个接一个出现，我们会使用的一种方法是 `Sequence`。

```kotlin
fun getSequence(): Sequence<String> = sequence {
    repeat(3) {
        Thread.sleep(1000)
        yield("User$it")
    }
}
fun main() {
    val list = getSequence()
    println("Function started")
    list.forEach { println(it) }
}
// Function started
// (1 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
```

当计算可能是 CPU 密集型（比如计算复杂的结果）或阻塞的（比如读取文件）时候，序列是一个合适的按需计算的数据流。但是，你必须要知道**序列的终端操作（如 `forEach`）是不会挂起的**，因此序列构建器中的任何挂起都意味着阻塞等待线程来处理这个值。这就是为什么在 `sequence` 构建器的作用域中，除了在 `SequenceScope` 接收者上调用的函数（`yield` 和 `yieldAll`）外，不能使用任何挂起函数。

```kotlin
fun getSequence(): Sequence<String> = sequence {
    repeat(3) {
        delay(1000) // 这里编译错误
        yield("User$it")
    }
}
```

引入这种机制是为了防止序列被误用。例如，有人可能希望使用分页的方式从 Http 端口获取所有的用户列表，直到接收到空白的数据。 即使上面的例子可以通过编译，它也不会是正确的，因为终端操作（如 `forEach`）将阻塞线程而不是挂起线程，这可能会导致意外的线程阻塞。

```kotlin
// 不要这样做，我们应该使用 Flow 来代替 Sequence
fun allUsersSequence(
    api: UserApi
): Sequence<User> = sequence {
    var page = 0
    do {
        val users = api.takePage(page++) // 挂起了，所以编译错误
        yieldAll(users)
    } while (!users.isNullOrEmpty())
}
```

我希望你已经了解到线程阻塞可能是危险的，会导致意想不到的情况，为了更清楚地说明这一点，看一下下面的示例，我们使用 `Sequence`，因此它的 `forEach` 是一个阻塞操作。这就是为什么在同一个线程上启动的协程会等待，一个协程的执行会阻塞另一个协程的执行：

```kotlin
fun getSequence(): Sequence<String> = sequence {
    repeat(3) {
        Thread.sleep(1000)
        // 就算这里能使用 delay(1000) ，结果也还是一样的
        yield("User$it")
    }
}

suspend fun main() {
    withContext(newSingleThreadContext("main")) {
        launch {
            repeat(3) {
                delay(100)
                println("Processing on coroutine")
            }
        }
        val list = getSequence()
        list.forEach { println(it) }
    }
}
// (1 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
```

在这种情况下，我们应该使用 `Flow` 而不是 `Sequence`。它完全支持协程。它的构建器和操作都是可挂起，并且支持结构化并发和适当的异常处理。我们将在下一章中解释这些内容。但现在让我们看看它对这个案例有什么帮助。

```kotlin
fun getFlow(): Flow<String> = flow {
    repeat(3) {
        delay(1000)
        emit("User$it")
    }
}

suspend fun main() {
    withContext(newSingleThreadContext("main")) {
        launch {
            repeat(3) {
                delay(100)
                println("Processing on coroutine")
            }
        }
        val list = getFlow()
        list.collect { println(it) }
    }
}
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (1 - 3 * 0.1 = 0.7 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
```

`Flow` 应该用于需要使用协程的数据流。例如，它可以用于生成一个从 API 页面逐页获取的用户流。例如，如果我们调用 `allUserFlow(api).first()`，我们将获取到第一页；如果我们调用 `allUserFlow(api).toList()` ，我们将获取所有数据；如果我们调用 `allUserFlow(api).find { it.id == id }`，我们将一直拉取页面数据，直到找到我们想要找到的页面。

```kotlin
fun allUsersFlow(
    api: UserApi
): Flow<User> = flow {
    var page = 0
    do {
        val users = api.takePage(page++) // 挂起了
        emitAll(users)
    } while (!users.isNullOrEmpty())
}
```

### Flow 的特性

Flow 的终端操作（如 `collect`）将挂起一个协程，而不是阻塞线程。它们还支持其它协程功能，例如异常的处理。Flow 处理可以被取消，并且可以在外部支持结构化并发。 `flow` 构建器不会挂起，也不需要任何作用域。

下面的示例展示了 `CoroutineName` 上下文如何从集合传递到 `flow` 构建器中的。它还表明，`launch` 的取消也会导致 flow 的处理被取消。

```kotlin
// 注意，该函数不会挂起，而且不需要任何的 CoroutineScope
fun usersFlow(): Flow<String> = flow {
    repeat(3) {
        delay(1000)
        val ctx = currentCoroutineContext()
        val name = ctx[CoroutineName]?.name
        emit("User$it in $name")
    }
}

suspend fun main() {
    val users = usersFlow()
    withContext(CoroutineName("Name")) {
        val job = launch {
            // collect 是挂起的
            users.collect { println(it) }
        }
        launch {
            delay(2100)
            println("I got enough")
            job.cancel()
        }
    }
}
// (1 sec)
// User0 in Name
// (1 sec)
// User1 in Name
// (0.1 sec)
// I got enough
```

### Flow 命名法

* Flow 需要从某个地方开始，它通常从一个流构建器开始，从不同的对象或从某些 helper 函数开始，最重要的选项在下一章中解释
* Flow 上最后一个操作被称为终端操作，这是非常重要的，因为它通常是唯一的挂起函数，或需要协程作用域的操作。典型的终端操作是 `collect`。然而，还有其它终端操作，我会在后面的章节中讲解
* 在开始操作和终端操作之间，我们可能有中间操作，每个操作都以某种方式修改流，我们将在_Flow的生命周期_和_处理Flow_的章节中学习不同的中间操作

![···图片···](https://img-blog.csdnimg.cn/c1f13b89e745414e9a7d050ba2090d97.png)

### 实际用例

实践表明，我们更多时候需要的是 flow，而不是 channel。如果请求数据流，我们通常希望是按需请求的。如果你需要观察某些东西，例如数据库中的更改或者来自 UI 部件的感知，你可能希望每个观察者都能接收到这些事件。当没有人要观察时，你也要停止监听。这就是为什么在所有这些情况下，使用 flow 会比使用 channel 更好（尽管在某些情况下，我们将这两者混合使用）。

flow 最典型的用法包括：

* 接收从 Server 连通通道中发送的消息，如 WebSocket、通知等
* 观察用户的操作，如文本更改或点击
* 接收来自传感器或设备的其他信息的更新，如其位置或方向
* 观察数据库的变化

下面是我们如何使用 Room 库来观察 SQL 数据库的变化：

```kotlin
@Dao
interface MyDao {
@Query("SELECT * FROM somedata_table")
    fun getData(): Flow<List<SomeData>>
}
```

让我们看一些示例，看看如何使用 flow 来处理来自 API 的响应流。首先，假设你实现了聊天功能，其中消息通过 Server 通道和通知发送。将两个数据源作为一个流，将它们合并在一起，然后用该流来更新视图，这是很方便的。另一个例子可能是用它来提供越来越好的响应结果。例如，当我们在 SkyScanner 上搜索最佳航班时，有些报价很快就会到达，但随着时间的推移，会有更多更好的报价达到，因此，你会看到越来越好的结果。这也是使用 flow 的一个很好的例子。

![····图片···](https://img-blog.csdnimg.cn/c92c755668a04b60affe3e0a22ffc1fd.png)

除了这些情况，对于不同的并发处理， flow 也是一个有用的工具。例如，假设你有一个卖家列表，你需要获取每个卖家的报价。我们已经知道可以使用 async 在集合处理中实现这一点：

```kotlin
suspend fun getOffers(
    sellers: List<Seller>
): List<Offer> = coroutineScope {
    sellers
        .map { seller ->
            async { api.requestOffers(seller.id) }
        }
        .flatMap { it.await() }
}
```

上面的方法在很多情况下是正确的，但它有一个缺点：当卖家列表很大时，一次发送这么多请求对我们和服务器都没有什么好处。当然，这可以在服务器中进行限频或限流，但我们也希望在客户端控制它，因此我们可以使用 Flow。在这种情况下，为了将并发调用的数量限制在20个，我们可以使用 `flaotMapMerge`，并将最大并发数 `concurrency` 修改为 20：

```kotlin
suspend fun getOffers(
    sellers: List<Seller>
): List<Offer> = sellers
    .asFlow()
    .flatMapMerge(concurrency = 20) { seller ->
        suspend { api.requestOffers(seller.id) }.asFlow()
    }
    .toList()
```

对 Flow 而不是集合进行操作，可以让我们对并发行为、上下文、异常等进行更多的控制。我们将在下一章中探索这些功能，这就是（以我的经验） flow 最有用的地方。我希望在我们介绍了它的所有不同功能之后，你能清楚的了解这一点。

最后，因为更喜欢响应式编程的风格，一些团队倾向使用响应流而不是挂起函数。这种风格在 Android 上很流行，其中 RxJava 就很主流，但现在 `Flow` 通常被视为更好的选择。

正如你所看到的，flow 有相当多的用例。在一些项目中，它们会被普遍使用，而在另一些项目中，它们只会被偶尔使用。但我希望你能知道它是有用的，值得学习的。

### 总结

在本章中，我们介绍了 `Flow` 的概念。它表示支持协程（不同的序列）的异步数据流。在相当多的用例中，flow 是有用的。

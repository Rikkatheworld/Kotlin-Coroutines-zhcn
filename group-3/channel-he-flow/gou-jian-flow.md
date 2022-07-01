# 构建 Flow

每个 flow 都需要从某个地方启动。有很多方法可以做到这一点，这取决于我们需要什么，在本章中，我们将集中讨论那种最重要的用法。

### Flow 的原始值

创建 flow 的最简单方法是使用 `flowOf` 函数，在该函数中，我们只需定义该 flow 应该具有哪些值（类似于创建列表的 `listOf` 函数）。

```kotlin
suspend fun main() {
    flowOf(1, 2, 3, 4, 5)
        .collect { print(it) } // 12345
}
```

有时，我们可能还需要没有值的 flow。为此，我们可以使用 `emptyFlow` 函数（类似于创建空列表的 `emptyList` 函数）。

```kotlin
suspend fun main() {
    emptyFlow<Int>()
        .collect { print(it) } // 什么都没有
}
```

### 转换器

我们还可以使用 `asFlow` 函数将每个 `Iterable`、`Iterator` 或 `Sequence` 转化成 flow ：

```kotlin
suspend fun main() {
    listOf(1, 2, 3, 4, 5)
        // or setOf(1, 2, 3, 4, 5)
        // or sequenceOf(1, 2, 3, 4, 5)
        .asFlow()
        .collect { print(it) } // 12345
}
```

这些函数产生立即可用的元素流，然后我们可以使用 flow 处理函数来处理这些元素流。

### 将函数转换为流

Flow 经常被用来表示一个延时产生单个值的流（就像 RxJava 中的 Single）。因此，将挂起函数转换为 Flow 是有意义的，函数的结果将是该流中的唯一值。为此，`asFlow` 函数也扩展了函数类型（包裹 `suspend () -> T` 和 `() -> T`），在这里它用于将一个挂起的 lambda 表达式转化为 Flow。

```kotlin
suspend fun main() {
    val function = suspend {
        // 这是挂起的 lambda 表达式
        delay(1000)
        "UserName"
    }

    function.asFlow()
        .collect { println(it) }
}
// (1 sec)
// UserName
```

如果要转换一个常规函数，我们需要先引用它，可以在 Kotlin 中使用 ::。

```kotlin
suspend fun getUserName(): String {
    delay(1000)
    return "UserName"
}

suspend fun main() {
    ::getUserName
        .asFlow()
        .collect { println(it) }
}
// (1 sec)
// UserName
```

### Flow 和响应式流

如果你在你的应用中使用响应流（例如 Reactor、RxJava2.x、RxJava3.X），那么你不需要在你的代码中做出太多的改变。所有的对象，例如 `Flux`，`Flowable` 和 `Obserable` 都实现了 `Publisher` 接口，它可以通过 kotlinx-coroutines-reactive 库中的 `asFlow` 函数转换为 Flow。

```kotlin
suspend fun main() = coroutineScope {
    Flux.range(1, 5).asFlow()
        .collect { print(it) } // 12345

    Flowable.range(1, 5).asFlow()
        .collect { print(it) } // 12345
    
    Observable.range(1, 5).asFlow()
        .collect { print(it) } // 12345
}
```

如果要反过来转换，你则需要特定的库，使用 kotlinx-coroutines-reactor，你可以将 Flow 转化为 `Flux`。使用 kotlinx-coruotines-rx3（或 kotlinx-coroutines-rx2），你可以将 Flow 转换为 `Flowable` 或 `Observable`。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf(1, 2, 3, 4, 5)
    flow.asFlux()
        .doOnNext { print(it) } // 12345
        .subscribe()
    
    flow.asFlowable()
        .subscribe { print(it) } // 12345
    
    flow.asObservable()
        .subscribe { print(it) } // 12345
}
```

### Flow 构建器

主流创建 flow 的方式是使用 flow 构建器。我们已经在前面的章节中使用过了。它的行为类似于创建一个序列的 `sequence` 构建器、或者创建一个 channel 的 `produce` 构建器。我们调用 `flow` 函数来启动一个构建器，并在其 lambda 表达式中使用 `emit` 函数发射出下一个值。我们还可以使用 `emitAll` 来发射出来自 Channel 或 Flow 的所有值（ `emitAll(Flow)` 是 `flow.collect { emit(it) }` 的简写）。

```kotlin
fun makeFlow(): Flow<Int> = flow {
    repeat(3) { num ->
        delay(1000)
        emit(num)
    }
}
suspend fun main() {
    makeFlow()
        .collect { println(it) }
}
// (1 sec)
// 0
// (1 sec)
// 1
// (1 sec)
// 2
```

这个构建器已经在前面的章节中使用过了，在接下来的章节还会用到很多次，所以我们会看到它的很多用法。现在，我来回顾序列构建器一章中的一个例子，flow 构建器用于拉取从网络 API 中逐页请求来的用户流：

```kotlin
fun allUsersFlow(
    api: UserApi
): Flow<User> = flow {
    var page = 0
    do {
        val users = api.takePage(page++) // 挂起
        emitAll(users)
    } while (!users.isNullOrEmpty())
}
```

### 理解 flow 构建器

flow 构建器是创建流的最基本方法。所有其他选项都基于此。

```kotlin
public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}
```

我们只需要理解这个构建器是如何工作的，我们就知道 flow 是如何工作的。flow 构建器的内部其实非常简单：它只创建一个实现 `Flow` 接口的对象，该对象只调用 collect 方法内部的 block 函数：

```kotlin
fun <T> flow(
    block: suspend FlowCollector<T>.() -> Unit
): Flow<T> = object : Flow<T>() {
    override suspend fun collect(collector: FlowCollector<T>) {
        collector.block()
    }
}

interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}

fun interface FlowCollector<in T> {
    suspend fun emit(value: T)
}

public suspend inline fun <T> Flow<T>.collect(
    crossinline action: suspend (value: T) -> Unit
): Unit =
    collect(object : FlowCollector<T> {
        override suspend fun emit(value: T) = action(value)
    })
```

知道了这一点，让我们来分析下面的代码是如何工作的：

```kotlin
fun main() = runBlocking {
    flow { // 1
        emit("A")
        emit("B")
        emit("C")
    }.collect { value -> // 2
        println(value)
    }
}
// A
// B
// C
```

当我们调用 flow 构建器时，我们只是创建了一个对象。但是，当调用了 `collect` 函数，就意味着调用 collector 接口上的 `block` 函数。本例中 `block` 函数是注释1处的 lambda 表达式，它的接收者是 collector，也就是注释2处定义的。当我们使用 lambda 表达式定义一个函数接口（如 FlowCollector）时，这个 lambda 表达式的主体将被用作该函数接口的唯一函数（这个例子中就是 `emit`）的主体。因此， `emmit` 函数的主体是 `println(value)`。因此，当我们调用 `collect` 函数时，我们开始执行注释1处定义的 lambda 表达式。这就是 flow 的工作原理，其它的一切都是建立在这个基础上的。

### channelFlow

Flow 是一种冷的数据流，因此它按需生成值。如果你思考了上面给出的 `allUsersFlow`，那么只有当接收者请求时，下一个用户页才会被拉取。这在某些情况下是需要的，例如，假设我们正在寻找一个特定的用户，如果它在第一页中，我们就不需要请求更多的页了。为了在实践中印证这一点，在下面的示例中，我们使用 Flow 构建器生成下一个元素，请注意，下一页都是惰性请求的。

```kotlin
data class User(val name: String)
interface UserApi {
    suspend fun takePage(pageNumber: Int): List<User>
}

class FakeUserApi : UserApi {
    private val users = List(20) { User("User$it") }
    private val pageSize: Int = 3
    
    override suspend fun takePage(
        pageNumber: Int
    ): List<User> {
        delay(1000) // 挂起
        return users
            .drop(pageSize * pageNumber)
            .take(pageSize)
    }
}

fun allUsersFlow(api: UserApi): Flow<User> = flow {
    var page = 0
    do {
        println("Fetching page $page")
        val users = api.takePage(page++) // 挂起
        emitAll(users.asFlow())
    } while (!users.isNullOrEmpty())
}

suspend fun main() {
    val api = FakeUserApi()
    val users = allUsersFlow(api)
    val user = users
        .first {
            println("Checking $it")
            delay(1000) // 挂起
            it.name == "User3"
        }
    println(user)
}
// Fetching page 0
// (1 sec)
// Checking User(name=User0)
// (1 sec)
// Checking User(name=User1)
// (1 sec)
// Checking User(name=User2)
// (1 sec)
// Fetching page 1
// (1 sec)
// Checking User(name=User3)
// (1 sec)
// User(name=User3)
```

另一方面，我们可能需要在处理元素的同时拉取下页数据。在本例中，这样做可能会导致更多的网络请求，但也可能更快的产生结果。要实现这一目标，我们需要独立的生产和消费。这种独立性是热数据流（如 channel）的典型特征。所以，我们需要混合 channel 和 flow。是的，这是支持的：我们只需要调用 `channelFlow` 函数，它与 Flow 类似，因为它实现了 Flow 接口。这个这个构建器是一个常规函数，它以一个终端操作（如 collect）开始。它也类似于 Channel，因为一旦启动，它将在单独的协程中生成值，而无需等待接收者。因此，拉取下一页和检查用户信息是同时进行的。

```kotlin
fun allUsersFlow(api: UserApi): Flow<User> = channelFlow {
    var page = 0
    do {
        println("Fetching page $page")
        val users = api.takePage(page++) // 挂起
        users?.forEach { send(it) }
    } while (!users.isNullOrEmpty())
}

suspend fun main() {
    val api = FakeUserApi()
    
    val users = allUsersFlow(api)
    val user = users
        .first {
            println("Checking $it")
            delay(1000)
            it.name == "User3"
        }
    println(user)
}
// Fetching page 0
// (1 sec)
// Checking User(name=User0)
// Fetching page 1
// (1 sec)
// Checking User(name=User1)
// Fetching page 2
// (1 sec)
// Checking User(name=User2)
// Fetching page 3
// (1 sec)
// Checking User(name=User3)
// Fetching page 4
// (1 sec)
// User(name=User3)
```

在 channelFlow 的内部，我们在 `ProducerScope<T>` 作用域上操作，`ProduceScope` 与 `produce` 构建器使用的类型相同。它实现了 `CoroutineScope`，因此我们可以使用它来启动新协程。要生成元素，我们要使用 `send` 而不是 `emit`。我们还可以使用 `SendChannel` 函数访问或直接控制 channel。

```kotlin
interface ProducerScope<in E>:
    CoroutineScope, SendChannel<E> {
        val channel: SendChannel<E>
    }
```

`channelFlow` 的一个典型用法是在我们需要独立计算时使用。为了支持这一点， `channelFlow` 创建了一个协程作用域，因此我们可以使用像 `launch` 这样的函数启动协程。下面的代码不适用于flow，因为它不能创建协程构建器所需的作用域。

```kotlin
fun <T> Flow<T>.merge(other: Flow<T>): Flow<T> =
    channelFlow {
        launch {
            collect { send(it) }
        }
        other.collect { send(it) }
    }

fun <T> contextualFlow(): Flow<T> = channelFlow {
    launch(Dispatchers.IO) {
        send(computeIoValue())
    }
    launch(Dispatchers.Default) {
        send(computeCpuValue())
    }
}
```

就像所有其它协程一样， `channelFlow` 会等待，直到它的所有子程序都处于终端状态时才会结束。

### callbackFlow

假设你需要监听事件流，比如用户的点击或其它类型的操作。监听过程应该独立于处理这些事件的过程，因此 `channelFlow` 是一个很好的候选者。但是，还有一个更好的方法： `callbackFlow`。

很长的一段时间捏，`channelFlow` 和 `callbackFlow` 之间没有区别。在1.3.4版本中，引入了一些小的更改，以降低使用回调时出错的可能性。然而，最大的区别还是在于人们如何理解这些函数： `callFlow` 是为了封装回调而存在的。

在 `callbackFlow` 内部，我们仍然在 `ProducerScope<T>` 作用域上操作。下面是一些可能对包装回调有用的函数：

* `awaitClose {...}` —— 一个挂起直到 channel 关闭的函数。一旦 channel 关闭就会调用它的参数。 `awaitClose` 对于 `callbackFlow` 非常重要。看看下面的例子，如果没有 `awaitClose`，协程将在注册回调后立即结束。这对于协程来说是很自然的：它的主体已经结束，并且它没有需要等待的子协程，所以它结束了，我们使用 `awaitClose`（即使它主体是空的）来防止这种情况的发生，并且我们监听元素，直到 channel 以其它方式关闭为止
* `trySendBlocking(value)` —— 类似于 `send`，但是它是阻塞而不是挂起的，所以它可以用于非挂起的函数
* `close()` —— 关闭此 channel
* `cancel(throwable)` —— 关闭 channel 并向 flow 发送一个异常

下面是一个使用 `callbackFlow` 的典型例子：

```kotlin
fun flowFrom(api: CallbackBasedApi): Flow<T> = callbackFlow {
    val callback = object : Callback {
        override fun onNextValue(value: T) {
            try {
                trySendBlocking(value)
            } catch (e: Exception) {
                // 在 channel 中处理异常
            }
        }
        
        override fun onApiError(cause: Throwable) {
            cancel(CancellationException("API Error", cause))
        }
        
        override fun onCompleted() = channel.close()
    }
    
    api.register(callback)
    awaitClose { api.unregister(callback) }
}
```

### 总结

在本章中，我们了解 flow 不同的创建方式。有许多用于启动 flow 的函数，从简单的 `flowOf` 或 `emptyFlow`、转换为 flow，到 flow 构建器。最简单的 flow 构建器只是一个 `flow` 函数。你可以在其中使用 `emit` 函数生成下一个值。还有 `channelFlow` 和 `callbackFlow` 构建器，它们创建具有 Channel 的一些特征的 flow。这些函数都有自己的用法。为了充分利用 Flow 的潜力，了解它们是很有用的。

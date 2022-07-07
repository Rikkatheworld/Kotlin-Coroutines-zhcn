# Flow 生命周期函数

Flow 可以想象成一个管道，请求的值在一个方向上流动，而相应产生的值在另一个方向上流动。当 flow 完成或出现异常时，这些信息也会被传递，并关闭途中的中间步骤。因此，当这些值开始流动时，我们可以监听值、异常或其它特征事件（如开始或完成）。为此，我们使用了 `onEach`、 `onStart`、`onCompletion`、`onEmpty` 和 `catch` 等方法。下面让我逐一解释这些生命周期方法。

### onEach

为了响应每个流动的值，我们使用 `onEach` 函数。

```kotlin
suspend fun main() {
    flowOf(1, 2, 3, 4)
        .onEach { print(it) }
        .collect() // 1234
}
```

`onEach` 的 lambda 表达式是挂起的，元素依次按顺序（顺序）被处理。因此，如果我们在 `onEach` 添加 `delay` 函数，我们将延迟每个值的流动。

```kotlin
suspend fun main() {
    flowOf(1, 2)
        .onEach { delay(1000) }
        .collect { println(it) }
}
// (1 sec)
// 1
// (1 sec)
// 2
```

### onStart

`onStart` 函数设置一个监听器，一旦 flow 启动，就会回调该监听器。需要注意的是， `onStart` 并不会等待第一个元素的响应，而是当我们请求第一个元素时，它就会被调用。

```kotlin
suspend fun main() {
    flowOf(1, 2)
        .onEach { delay(1000) }
        .onStart { println("Before") }
        .collect { println(it) }
}
// Before
// (1 sec)
// 1
// (1 sec)
// 2
```

在 `onStart`（以及在 onCompletion、onEmpty、catch） 中可以发射元素，这些元素将该处往下流动。

```kotlin
suspend fun main() {
    flowOf(1, 2)
        .onEach { delay(1000) }
        .onStart { emit(0) }
        .collect { println(it) }
}
// 0
// (1 sec)
// 1
// (1 sec)
// 2
```

### onCompletion

有几种方法可以完成一个 flow。最常见的是在 flow 构建器完成时（比如发送了最后一个元素），尽管有可能会出现在未捕获异常或者协程取消的情况下。在所有这些情况下，我们都可以使用 `onCompletion` 方法为 flow 的完成添加一个监听器。

```kotlin
suspend fun main() = coroutineScope {
    flowOf(1, 2)
        .onEach { delay(1000) }
        .onCompletion { println("Completed") }
        .collect { println(it) }
}
// (1 sec)
// 1
// (1 sec)
// 2
// Completed

suspend fun main() = coroutineScope {
    val job = launch {
        flowOf(1, 2)
            .onEach { delay(1000) }
            .onCompletion { println("Completed") }
            .collect { println(it) }
    }
    delay(1100)
    job.cancel()
}
// (1 sec)
// 1
// (0.1 sec)
// Completed
```

在 Android 中，我们经常使用 `onStart` 来展示进度条（等待网络响应的指示器），之后我们使用 `onCompletion` 来隐藏它。

```kotlin
fun updateNews() {
    scope.launch {
        newsFlow()
            .onStart { showProgressBar() }
            .onCompletion { hideProgressBar() }
            .collect { view.showNews(it) }
    }
}
```

### onEmpty

flow 可能在不发射任何值的情况下完成，有可能是出现意外状况。对于这种情况，有一个 `onEmpty` 函数，它在 flow 完成但没有发出任何元素时会回调。我们可以使用 `onEmpty` 来发射一些默认值。

```kotlin
suspend fun main() = coroutineScope {
    flow<List<Int>> { delay(1000) }
        .onEmpty { emit(emptyList()) }
        .collect { println(it) }
}
// (1 sec)
// []
```

### catch

在 flow 构建器或处理值的任何时刻，都可能发生异常。这样的异常会向下流动，关闭途中每个处理步骤；然而，它是可以被捕获和管理的。为此，我们可以使用 `catch` 方法。这个监听器接收异常作为参数，并允许你执行恢复操作。

```kotlin
class MyError : Throwable("My error")

val flow = flow {
   emit(1)
   emit(2)
   throw MyError()
}

suspend fun main(): Unit {
   flow.onEach { println("Got $it") }
       .catch { println("Caught $it") }
       .collect { println("Collected $it") }
}
// Got 1
// Collected 1
// Got 2
// Collected 2
// Caught MyError: My error
```

在上面的例子中，`onEach` 没有对异常做出响应。同样的情况也发生在其他功能上，如 `map`、 `filter` 等。只有 `onCompletion` 的处理才会被调用。

`catch` 方法通过捕获来阻止异常的传播。虽然前面的步骤已经完成了，但是 `catch` 仍然可以发射出新的值，并保持 flow 的其余部分处于活跃状态。

```kotlin
val flow = flow {
    emit("Message1")
    throw MyError()
}

suspend fun main(): Unit {
    flow.catch { emit("Error") }
        .collect { println("Collected $it") }
}
// Collected Message1
// Collected Error
```

`catch` 只会对上游定义的函数中抛出的异常做出响应（可以想象，当下流还有异常时，仍需要捕获异常）。

![···图片···](https://img-blog.csdnimg.cn/05a75c716f3c48d09c2a40eb608b0d0d.png)

在 Android 中，我们经常使用 `catch` 来展示 flow 中发生的异常：

```kotlin
fun updateNews() {
    scope.launch {
        newsFlow()
            .catch { view.handleError(it) }
            .onStart { showProgressBar() }
            .onCompletion { hideProgressBar() }
            .collect { view.showNews(it) }
    }
}
```

我们也可以使用 `catch` 来发出默认数据以显示在屏幕上，比如空列表。

```kotlin
fun updateNews() {
    scope.launch {
        newsFlow()
            .catch {
                view.handleError(it)
                emit(emptyList())
            }
            .onStart { showProgressBar() }
            .onCompletion { hideProgressBar() }
            .collect { view.showNews(it) }
    }
}
```

### 未捕获的异常

flow 中出现未捕获的异常则会立即取消该 flow，并且 `collect` 会重新抛出此异常。这种行为是挂起函数的典型行为， `coroutineScope` 也有同样的行为，典型的应对方法是使用 try-catch 块在 flow 的外部捕获异常：

```kotlin
val flow = flow {
    emit("Message1")
    throw MyError()
}

suspend fun main(): Unit {
    try {
        flow.collect { println("Collected $it") }
    } catch (e: MyError) {
        println("Caught")
    }
}
// Collected Message1
// Caught
```

请注意，使用 `catch` 并不能防止终端操作中出现异常（因为 `catch` 不能用最后一个操作之后）。因此，如果 `collect` 中有一个异常，它将不会捕获，而是将抛出一个错误。

```kotlin
val flow = flow {
    emit("Message1")
    emit("Message2")
}

suspend fun main(): Unit {
    flow.onStart { println("Before") }
        .catch { println("Caught $it") }
        .collect { throw MyError() }
}
// Before
// Exception in thread "..." MyError: My error
```

因此，通常的做法是将逻辑层从 `collect` 移动到 `onEach`，并将其放在 `catch` 之前。当我们怀疑 `collect` 的逻辑可能会出现异常时，这样做会特别有用。如果我们将逻辑操作从 `collect` 中挪走，就可以确定 `catch` 能捕获所有的异常。

```kotlin
val flow = flow {
    emit("Message1")
    emit("Message2")
}
suspend fun main(): Unit {
    flow.onStart { println("Before") }
        .onEach { throw MyError() }
        .catch { println("Caught $it") }
        .collect()
}
// Before
// Caught MyError: My error
```

### flowOn

传给 flow 操作（如 `onEach`、`onStart`、`onCompletion` 等）的 lambda 表达式及其构建器（如 `flow {..}` 或 `channelFlow{..}`）都是挂起的。挂起函数需要有一个上下文，并且应该与它们的父协程相关联（结构化并发）。因此，你可能想知道这些函数的上下文都来自哪里。答案是：从调用的 `collect` 的上下文中而来。

```kotlin
fun usersFlow(): Flow<String> = flow {
    repeat(2) {
        val ctx = currentCoroutineContext()
        val name = ctx[CoroutineName]?.name
        emit("User$it in $name")
    }
}

suspend fun main() {
    val users = usersFlow()

    withContext(CoroutineName("Name1")) {
        users.collect { println(it) }
    }

    withContext(CoroutineName("Name2")) {
        users.collect { println(it) }
    }
}
// User0 in Name1
// User1 in Name1
// User0 in Name2
// User1 in Name2
```

这段代码是如何工作的？ 终端操作会调用来自上游的元素，从而提供协程上下文。然而，它也可以通过 `flowOn` 函数进行修改（修改协程上下文）：

```kotlin
suspend fun present(place: String, message: String) {
    val ctx = coroutineContext
    val name = ctx[CoroutineName]?.name
    println("[$name] $message on $place")
}

fun messagesFlow(): Flow<String> = flow {
    present("flow builder", "Message")
    emit("Message")
}

suspend fun main() {
    val users = messagesFlow()
    withContext(CoroutineName("Name1")) {
        users
            .flowOn(CoroutineName("Name3"))
            .onEach { present("onEach", it) }
            .flowOn(CoroutineName("Name2"))
            .collect { present("collect", it) }
    }
}
// [Name3] Message on flow builder
// [Name2] Message on onEach
// [Name1] Message on collect
```

请记住， `flowOn` 只适用于 flow 中位于上游的函数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/18505fd62d494a7293cf87c5e5f0f834.png)

### launchIn

`collect` 是一个挂起函数，它会挂起一个协程直到 flow 完成。我们通常会使用 `launch` 构建器对其进行包装，以便 flow 处理可以在另一个协程上启动。为了优化这种情况，有一个 `launchIn` 函数，它会在作为参数传递的作用域上调用 `collect`。

```kotlin
fun <T> Flow<T>.launchIn(scope: CoroutineScope): Job =
    scope.launch { collect() }
```

`launchIn` 通常用来在一个单独的协程中启动一个 flow。

```kotlin
suspend fun main(): Unit = coroutineScope {
    flowOf("User1", "User2")
        .onStart { println("Users:") }
        .onEach { println(it) }
        .launchIn(this)
}
// Users:
// User1
// User2
```

### 总结

在本章中，我们学习了不同的 flow 功能。现在我们知道如何在 flow 开始时、结束时或者在每个元素上执行某些操作；我们还知道如何捕获异常，以及如何在新的协程中启动 flow。这些都是被广泛使用的基本工具，特别是在 Android 开发中。例如，下面是一段 Android 中使用 flow 的代码：

```kotlin
fun updateNews() {
    newsFlow()
        .onStart { showProgressBar() }
        .onCompletion { hideProgressBar() }
        .onEach { view.showNews(it) }
        .catch { view.handleError(it) }
        .launchIn(viewModelScope)
}
```

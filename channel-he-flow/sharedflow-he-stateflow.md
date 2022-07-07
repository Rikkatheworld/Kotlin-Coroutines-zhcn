# SharedFlow 和 StateFlow

Flow 是典型的冷数据流，所以它的值是按需计算的。然而在某些情况下，我们希望多个接收者订阅一个会更改的数据源。这就是我们使用 `SharedFlow` 的地方，它在概念上类似于邮件列表。我们还有 `StateFlow`，它近似与一个可观察对象。让我们一个个了解它们。

### SharedFlow

让我们从 `MutableSharedFlow` 开始，它就像一个广播通道：每个人都可以发送（发射）信息，信息会被被每个正在监听（收集）的协程接收。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val mutableSharedFlow =
        MutableSharedFlow<String>(replay = 0)
    // 或者 MutableSharedFlow<String>()
    
    launch {
        mutableSharedFlow.collect {
            println("#1 received $it")
        }
    }
    
    launch {
        mutableSharedFlow.collect {
            println("#2 received $it")
        }
    }
    delay(1000)
    mutableSharedFlow.emit("Message1")
    mutableSharedFlow.emit("Message2")
}

// (1 sec)
// #1 received Message1
// #2 received Message1
// #1 received Message2
// #2 received Message2
// (program never ends)
```

上面的程序永远不会结束，因为 `coroutineScope` 会等待它里面用 `launch` 启动的协程结束，而这些协程一直监听 `MutableSharedFlow`，显然， `MutableSharedFlow` 是不可关闭的，所以解决这个问题的唯一方法是取消整个作用域。

`MutableSharedFlow` 也可以持续发送信息。如果我们设置 `replay` 参数（默认设置为0），它会缓存最新的 n 个值，如果协程现在开始订阅，它将首先接收这些值。这个缓存也可以用 `resetReplayCache` 函数重置。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>(
        replay = 2, 
    )
    mutableSharedFlow.emit("Message1")
    mutableSharedFlow.emit("Message2")
    mutableSharedFlow.emit("Message3")

    println(mutableSharedFlow.replayCache)
    // [Message2, Message3]

    launch {
        mutableSharedFlow.collect {
            println("#1 received $it")
        }
        // #1 received Message2
        // #1 received Message3
    }
    delay(100)
    mutableSharedFlow.resetReplayCache()
    println(mutableSharedFlow.replayCache) // []
}
```

`MutableSharedFlow` 在概念上类似于 RxJava 的 `Subject`。当 `replay` 参数设置为0时，它类似于 `PuiblishSubject`。当 `replay` 是1时，它类似于 `BehaviorSubject`。当 `replay` 是 `Int.MAX_VALUE` 时，它类似于 `ReplaySubject`。

在 Kotlin 中，我们希望在用于监听的接口和用于修改的接口之间有一些区别。例如，我们已经看到了 `SendCahnnel` 、 `ReceiveChannel` 和 `Channel` 的区别。同样的情况也适用于这里。 `MutableSharedFlow` 继承自 `SharedFlow` 和 `FlowCollector`，前者继承自 `Flow` ，用于订阅，而 `FlowCollector` 则用于发射值。

```kotlin
interface MutableSharedFlow<T> :
    SharedFlow<T>, FlowCollector<T> {

    fun tryEmit(value: T): Boolean
    val subscriptionCount: StateFlow<Int>
    fun resetReplayCache()
}

interface SharedFlow<out T> : Flow<T> {
    val replayCache: List<T>
}

interface FlowCollector<in T> {
   suspend fun emit(value: T)
}
```

这些接口通常只用于暴露函数、发射或收集函数。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>()
    val sharedFlow: SharedFlow<String> = mutableSharedFlow
    val collector: FlowCollector<String> = mutableSharedFlow

    launch {
        mutableSharedFlow.collect {
            println("#1 received $it")
        }
    }

    launch {
        sharedFlow.collect {
            println("#2 received $it")
        }
    }

    delay(1000)
    mutableSharedFlow.emit("Message1")
    collector.emit("Message2")
}
// (1 sec)
// #1 received Message1
// #2 received Message1
// #1 received Message2
// #2 received Message2
```

以下是 Android 上的典型用法：

```kotlin
class UserProfileViewModel {
    private val _userChanges =
        MutableSharedFlow<UserChange>()
    val userChanges: SharedFlow<UserChange> = _userChanges

    fun onCreate() {
        viewModelScope.launch {
            userChanges.collect(::applyUserChange)
        }
    }
    
    fun onNameChanged(newName: String) {
        // ...
        _userChanges.emit(NameChange(newName))
    }
    
    fun onPublicKeyChanged(newPublicKey: String) {
        // ...
        _userChanges.emit(PublicKeyChange(newPublicKey))
    }
}
```

### shareIn

Flow 通常用于观察更改行为，如用户操作、数据库修改或新消息出现。我们已经知道了如何处理这些事件的方法，譬如已经学习了如何将多个 flow 合并为一个 flow。但是如果多个订阅者对这些更改感兴趣，或者我们想把一个 flow 变成多个 flow，该怎么解决呢？答案是使用 `SharedFlow`，将一个 flow 转换成 `SharedFlow`，最简单的方法是使用 `sharIn` 函数。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf("A", "B", "C")
        .onEach { delay(1000) }

    val sharedFlow: SharedFlow<String> = flow.shareIn(
        scope = this,
        started = SharingStarted.Eagerly,
        // replay = 0 (default)
    )
   
    delay(500)
    launch {
        sharedFlow.collect { println("#1 $it") }
    }

    delay(1000)
    launch {
        sharedFlow.collect { println("#2 $it") }
    }
    
    delay(1000)
    launch {
        sharedFlow.collect { println("#3 $it") }
    }
}
// (1 sec)
// #1 A
// (1 sec)
// #1 B
// #2 B
// (1 sec)
// #1 C
// #2 C
// #3 C
```

`shareIn` 函数创建了一个 `SharedFlow`，并从它的 flow 上发射元素。因为我们需要启动一个协程来收集这些 flow 上的元素， 所以 `shareIn` 的第一个参数是协程作用域。 第三个参数是 `replay`， 默认值为0。第二个参数很有趣： `started` 决定流上的数据合适被发送。支持下面选项：

* `SharingStated.Eagerly` —— 立即发送数据。注意，如果你有一个有限的 `relay` 值，你会失去一些你订阅之前发出的值（如果你的 `replay` 设置为0，你将失去所有那些值）。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf("A", "B", "C")
    val sharedFlow: SharedFlow<String> = flow.shareIn(
        scope = this,
        started = SharingStarted.Eagerly,
    )

    delay(100)
    launch {
        sharedFlow.collect { println("#1 $it") }
    }
    print("Done")
}
// (0.1 sec)
// Done
```

* `SharingStarted.Lazily` —— 当第一个订阅者观察时才开始发送数据。这保证了第一个订阅者能获得所有发射的值，而后续订阅者只保证获得最新的 `replay` 数量的值。即使所有订阅者都消失了，上游仍然是激活的，只有最新的 `replay` 数量的数据在没有订阅者的情况下才会被缓存

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow1 = flowOf("A", "B", "C")
    val flow2 = flowOf("D")
        .onEach { delay(1000) }

    val sharedFlow = merge(flow1, flow2).shareIn(
        scope = this,
        started = SharingStarted.Lazily,
    )

    delay(100)
    launch {
        sharedFlow.collect { println("#1 $it") }
    }

    delay(1000)
    launch {
        sharedFlow.collect { println("#2 $it") }
    }
}
// (0.1 sec)
// #1 A
// #1 B
// #1 C
// (1 sec)
// #2 D
// #1 D
```

* `WhileSubscribed()` - 当第一个订阅者出现时，flow 发射数据；当最后一个订阅者消失时，该 flow 将停止。如果我们的 `SharedFlow` 停止时出现了一个新的订阅者，它将会再次启动。而添加新的订阅者有额外的可选配置参数： `stopTimeoutMulls`（在最后一个订阅者消失后保留多长时间，默认为 0） 和 `replayExpirationMills` （停止后保存 `replay` 的缓存数量多久，默认为 `Long.MAX_VALUE`）

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf("A", "B", "C", "D")
        .onStart { println("Started") }
        .onCompletion { println("Finished") }
        .onEach { delay(1000) }
    
    val sharedFlow = flow.shareIn(
        scope = this,
        started = SharingStarted.WhileSubscribed(),
    )

    delay(3000)
    launch {
        println("#1 ${sharedFlow.first()}")
    }
    launch {
        println("#2 ${sharedFlow.take(2).toList()}")
    }

    delay(3000)
    launch {
        println("#3 ${sharedFlow.first()}")
    }
}
// (3 sec)
// Started
// (1 sec)
// #1 A
// (1 sec)
// #2 [A, B]
// Finished
// (1 sec)
// Started
// (1 sec)
// #3 A
// Finished
```

* 也可以通过实现 `ShareingStated` 接口来自定义一个策略

当多个服务对相同的更改感兴趣时，使用 `sharedIn` 非常方便。假设你需要观察数据库中位置信息是如何随时间变化的，下面就是 DTO（数据传输对象）在 Android 的 Room 上实现的：

```kotlin
@Dao
interface LocationDao {
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertLocation(location: Location)
    
    @Query("DELETE FROM location_table")
    suspend fun deleteLocations()
    
    @Query("SELECT * FROM location_table ORDER BY time")
    fun observeLocations(): Flow<List<Location>>
}
```

问题是，如果多个服务需要依赖于这些位置信息，那么让每个服务单独观察数据库并不是最优的。相反，我们可以创建一个服务来监听这些更改，并将它们共享到 `SharedFlow` 中。这就是我们使用 `shareIn` 的地方。但是我们应该如何配置它们呢？你需要自己做决定。你希望你的订阅者在订阅时立即收到最新的位置列表吗？ 如果是，则设置 `replay` 为1。如果你只想对更改做出反应，则设置为 0 即可。`started` 应该设置成什么呢？ `WhileSubscribed` 看起来适合这个场景。

```kotlin
class LocationService(
    private val locationDao: LocationDao,
    private val scope: CoroutineScope
) {
    private val locations = locationDao.observeLocations()
        .shareIn(
            scope = scope,
            started = SharingStarted.WhileSubscribed(),
        )

    fun observeLocations(): Flow<List<Location>> = locations
}
```

小心！我们不要为每个调用都创建一个新的 `SharedFlow`。你只需要创建一个，并将其作为一个属性存储即可。

### StateFlow

`StateFlow` 是 `SharedFlow` 的一个衍生概念。它的工作原理和设置了 `replay` 为1的 `SharedFlow` 类似。它总是存储一个值，可以使用 `value` 属性来访问该值。

```kotlin
interface StateFlow<out T> : SharedFlow<T> {
    val value: T
}

interface MutableStateFlow<T> :
    StateFlow<T>, MutableSharedFlow<T> {
    override var value: T
        fun compareAndSet(expect: T, update: T): Boolean
}
```

初始值需要传递给构造函数。我们都使用 `value` 属性来访问和设置该值。正如你所看到的， `MutableStateFlow` 就像一个数据的可观察容器。

```kotlin
suspend fun main() = coroutineScope {
    val state = MutableStateFlow("A")
    println(state.value) // A
    launch {
        state.collect { println("Value changed to $it") }
        // Value changed to A
    }
   
    delay(1000)
    state.value = "B" // Value changed to B
    
    delay(1000)
    launch {
        state.collect { println("and now it is $it") }
        // and now it is B
    }

    delay(1000)
    state.value = "C" // Value changed to C and now it is C
}
```

在 Android 上， `StateFlow` 被用作 `LiveData` 的现成替代品。首先，它完全支持协程，其次，它有一个初始值，所以它不需要为空。因此， `StateFlow` 经常用于表示 `ViewModel` 的状态。这个状态被观察着，并在此基础上显示和更新一个视图。

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {
    private val _uiState =
        MutableStateFlow<NewsState>(LoadingNews)
    val uiState: StateFlow<NewsState> = _uiState
    
    fun onCreate() {
        scope.launch {
            _uiState.value =
                NewsLoaded(newsRepository.getNews())
        }
    }
}
```

### stateIn

`stateIn` 是一个将 `Flow<T>` 转换为 `StateFlow<T>` 的函数。只能在作用域上去调用它，但它是一个挂起函数。请记住， `StateFlow` 始终都需要一个值，因此，如果你没有指定它，那么你需要等待直到第一个值被计算出来。

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B", "C")
        .onEach { delay(1000) }
        .onEach { println("Produced $it") }
    val stateFlow: StateFlow<String> = flow.stateIn(this)
    
    println("Listening")
    println(stateFlow.value)
    stateFlow.collect { println("Received $it") }
}
// (1 sec)
// Produced A
// Listening
// A
// Received A
// (1 sec)
// Produced B
// Received B
// (1 sec)
// Produced C
// Received C
```

`stateIn` 的第二种变体不是挂起的，但它需要初始值和启动模式。此模式具有与 `shareIn` 相同的选项（如上所述）。

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B")
        .onEach { delay(1000) }
        .onEach { println("Produced $it") }

    val stateFlow: StateFlow<String> = flow.stateIn(
        scope = this,
        started = SharingStarted.Lazily,
        initialValue = "Empty"
    )

    println(stateFlow.value)
    
    delay(2000)
    stateFlow.collect { println("Received $it") }
}
// Empty
// (2 sec)
// Received Empty
// (1 sec)
// Produced A
// Received A
// (1 sec)
// Produced B
// Received B
```

当我们需要订阅来自单个更改源的数据流时，我们通常会使用 `stateIn`。在这个过程中，可以处理这些更改。

```kotlin
class LocationsViewModel(
    private val locationService: LocationService
) : ViewModel() {
    private val location = locationService.observeLocations()
        .map { it.toLocationsDisplay() }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.Lazily,
            initialValue = emptyList(),
        )

    // ...
}
```

### 总结

在本章中，我们学习了 `SharedFlow` 和 `StateFlow`，这两个东西对于 Android 开发者来说都是特别重要的，因为它们通常被用作 MVVM 模式的一部分。记住它们并考虑使用它们，特别是如果你在 Android 开发中使用 view model 的时候。

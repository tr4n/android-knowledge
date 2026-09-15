# Bài 05 — Asynchronous Flow: Khái niệm Cốt lõi & Vận hành (Flows)

> **Tài liệu gốc:** [Flows — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-flow.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Nắm vững toàn bộ kiến trúc đường ống Flow (Emitter - Operator - Collector), luồng nguội (Cold Flows), kỹ thuật viết toán tử tùy biến, nguyên tắc bảo toàn ngữ cảnh với `flowOn`, xử lý ngoại lệ và Exception Transparency, cơ chế hủy Flow và toán tử `.take()`, phát dữ liệu đồng thời với `channelFlow`, luồng nóng (Hot Flows: `SharedFlow` & `StateFlow`), chuyển đổi Cold sang Hot với `shareIn`/`stateIn`, và xử lý ngoại lệ trong Hot Flows.

---

## 1. Giới thiệu về Flows (Introduction)

Một **Flow** đại diện cho một luồng tuần tự các giá trị được sản sinh một cách bất đồng bộ (**sequential stream of values that can be produced asynchronously**). Khác với một hàm suspend thông thường vốn chỉ trả về một giá trị duy nhất, bạn có thể sử dụng flows để làm việc với nhiều giá trị tuần tự theo thời gian.

Bạn có thể dùng flows để xây dựng các **đường ống xử lý (flow pipelines)** giúp tải dữ liệu theo tiến trình (progressively), phản ứng với dòng sự kiện (event streams) và mô hình hóa các API dạng đăng ký theo dõi (subscription-style APIs).

### Các vai trò trong Đường ống Flow (Flow Pipeline Roles):
- **Emitter (Bên phát)**: Sản xuất và đẩy các giá trị vào luồng.
- **Intermediate operators (Toán tử trung gian - Tùy chọn)**: Tiêu thụ giá trị từ luồng, áp dụng một thao tác xử lý và trả về một luồng mới.
- **Collector (Bên thu thập)**: Tiêu thụ các giá trị cuối cùng từ luồng.

```kotlin
import kotlinx.coroutines.flow.*

//sampleStart
suspend fun main() {
    // Emitter sản xuất các giá trị số hex
    flowOf(0x4B, 0x6F, 0x74, 0x6C, 0x69, 0x6E)
        // Toán tử trung gian tiêu thụ giá trị, biến đổi Char và trả về flow mới
        .map { value -> value.toChar() }
        // Collector tiêu thụ các giá trị đã được biến đổi
        .collect { updatedValue ->
            println("Say '$updatedValue'!")
        }
}
//sampleEnd
```

**Kết quả in ra Console:**
```text
Say 'K'!
Say 'o'!
Say 't'!
Say 'l'!
Say 'i'!
Say 'n'!
```

Trong một flow, các giá trị luôn di chuyển từ bên phát về phía bên thu thập, từ **thượng nguồn (upstream)** xuống **hạ nguồn (downstream)**.

### Phân loại Flow trong Kotlin:
- [**Cold flows (Luồng nguội)**](#2-luong-nguoi-cold-flows): Có tính chất lười (lazy). Chỉ bắt đầu sản xuất giá trị khi được thu thập (`collect()`). Mỗi collector kích hoạt một lần thực thi độc lập riêng biệt từ đầu.
- [**Hot flows (Luồng nóng)**](#5-luong-nong-hot-flows-sharedflow--stateflow): Phát dữ liệu độc lập với các collector và chia sẻ chung một dòng giá trị cho tất cả các collector (subscriber).

> [!TIP]
> Bạn có thể sử dụng thư viện [Turbine](https://github.com/cashapp/turbine) để viết kiểm thử đơn vị (Unit Test) cho Kotlin Flows, giúp đơn giản hóa việc assert các giá trị phát ra cũng như các trường hợp hoàn thành và thất bại.

---

## 2. Luồng Nguội (Cold Flows)

Tương tự như `Sequence` trong Kotlin Standard Library, **Cold Flow có tính chất lười (lazy)**.
Khối mã bên trong bộ tạo Cold Flow **sẽ không hề chạy** cho đến khi có một collector tiến hành thu thập nó. Mỗi collector mới sẽ bắt đầu một phiên thực thi hoàn toàn mới của flow đó.

### 2.1 Khởi tạo Cold Flow
Để tạo một cold flow, sử dụng hàm builder [`flow()`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html). Bên trong khối mã, sử dụng hàm [`emit()`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow-collector/emit.html) để phát giá trị tới collector:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
fun main() {
    // Tạo một flow
    val pageFlow = flow {
        for (page in 1..3) {
            println("Loading page $page...")

            // Phát từng trang khi đã nạp xong
            emit("Page $page")
        }
    }
    println("Creating a cold flow doesn't run it!")
}
//sampleEnd
```

**Kết quả Console:**
```text
Creating a cold flow doesn't run it!
```

*Phân tích*: Hàm `flow()` trả về một `Flow<T>` nhưng không hề thực thi khối mã bên trong nó. Một cold flow giống như một công thức nấu ăn: Nó chỉ định nghĩa cách tạo ra giá trị, nhưng chỉ thực sự sản xuất khi bạn gọi `collect()`.

Các hàm tạo cold flow tiện ích khác:
- [`flowOf(...)`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-of.html): Tạo flow từ các giá trị được truyền trực tiếp (`flowOf("Page 1", "Page 2", "Page 3")`).
- [`.asFlow()`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/as-flow.html): Chuyển đổi một iterable (như khoảng `(1..3).asFlow()`) thành flow.

---

### 2.2 Thu thập Cold Flow (Collect a Cold Flow)
Để kích hoạt và nhận dữ liệu từ một cold flow, gọi hàm [`collect()`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect.html):

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

//sampleStart
suspend fun main() {
    withContext(Dispatchers.Default) {
        val pageFlow = flow {
            for (page in 1..3) {
                println("Loading page $page...")
                emit("Page $page")
            }
        }
        // Thu thập flow với một lambda nhận từng trang được phát ra
        pageFlow.collect { page ->
            println("Processing $page...")
            delay(100.milliseconds)
            println("Done processing $page.")
        }
    }
}
//sampleEnd
```

**Tính độc lập giữa các Collectors:**  
Mỗi lần gọi `collect()` sẽ chạy lại toàn bộ cold flow từ đầu. Nếu nhiều collector cùng thu thập một cold flow, mỗi collector kích hoạt một phiên chạy riêng biệt:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

//sampleStart
suspend fun main() {
    val pageFlow = flow {
        val coroutineName = currentCoroutineContext()[CoroutineName]?.name

        println("Starting emissions in $coroutineName")
        for (page in 1..3) {
            println("Loading page $page in $coroutineName")
            emit("Page $page")
        }
        println("Done emitting in $coroutineName")
    }

    withContext(Dispatchers.Default) {
        // Collector xử lý chậm
        launch(CoroutineName("a slow coroutine")) {
            pageFlow.collect {
                println("Processing $it slowly")
                delay(100.milliseconds)
                println("Done processing $it slowly")
            }
        }

        // Collector xử lý nhanh
        launch(CoroutineName("a fast coroutine")) {
            pageFlow.collect {
                println("Processing $it quickly")
                delay(10.milliseconds)
                println("Done processing $it quickly")
            }
        }
    }
}
//sampleEnd
```

---

### 2.3 Các Toán tử Trung gian & Tự Viết Custom Operator

Các toán tử trung gian nhận flow thượng nguồn và trả về flow hạ nguồn. Chúng mang tính chất lười (cold) — flow trả về không chạy cho đến khi được collect.

Bạn có thể tự định nghĩa toán tử trung gian tùy biến cho riêng mình bằng cách dùng `flow { collect { emit(...) } }`:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
// Triển khai tùy biến đơn giản của toán tử .map() mặc định
fun <T, R> Flow<T>.myMap(transform: suspend (value: T) -> R): Flow<R> = flow {
    // Thu thập các giá trị từ flow thượng nguồn
    this@myMap.collect { value ->
        // Biến đổi từng giá trị thu được và phát giá trị mới xuống hạ nguồn
        emit(transform(value))
    }
}

suspend fun main() {
    flowOf(1, 2, 3).myMap { 2 * it }.collect {
        println("Collecting $it")
    }
}
//sampleEnd
```

#### Gọi hàm `suspend` bên trong `flow()` builder:
Không giống như `Sequence`, bạn hoàn toàn có thể gọi các hàm suspend khác bên trong `flow()`:
```kotlin
suspend fun loadPage(): Int {
    delay(100)
    return 3
}

suspend fun main() {
    flow {
        emit(loadPage()) // Hợp lệ!
    }.collect { println(it) }
}
```

> [!CAUTION]
> **Quy tắc Bất biến của Flow (Flow Invariant)**:
> Một hàm builder `flow { ... }` **bắt buộc phải phát giá trị từ cùng một coroutine context nơi nó đang chạy**.
> Bạn **KHÔNG ĐƯỢC PHÉP** khởi chạy một coroutine con khác gọi `emit()`, và **KHÔNG ĐƯỢC PHÉP** thay đổi context bằng `withContext()` bên trong khối `flow`:

```kotlin
// ❌ LỖI NGHIÊM TRỌNG: Sẽ ném ngoại lệ IllegalStateException khi chạy:
flow {
    withContext(Dispatchers.IO) {
        emit('a') // BỊ CẤM TUYỆT ĐỐI!
    }
}
```

#### Thay đổi Context của Cold Flow với toán tử `.flowOn()`:
Nếu bạn cần phần thượng nguồn (upstream) chạy trên một Dispatcher khác (ví dụ: đọc file trên IO thread trong khi collector chạy trên Main thread), hãy sử dụng toán tử **`.flowOn()`**:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
suspend fun main() {
    withContext(Dispatchers.Default + CoroutineName("downstream")) {
        flow {
            val coroutineName = currentCoroutineContext()[CoroutineName]?.name
            println("Emitting '1' in $coroutineName")
            emit(1)
        // Thay đổi coroutine context của luồng thượng nguồn
        }.flowOn(Dispatchers.IO + CoroutineName("upstream"))
            .collect {
            val coroutineName = currentCoroutineContext()[CoroutineName]?.name
            println("Collecting '$it' in $coroutineName")
        }
    }
}
//sampleEnd
```

**Kết quả in ra Console:**
```text
Emitting '1' in upstream
Collecting '1' in downstream
```

Toán tử `.flowOn()` **bảo toàn ngữ cảnh (context-preserving)**: Nó chỉ đổi context cho phần thượng nguồn bên trên nó mà vẫn giữ nguyên context của bên gọi `collect()` ở hạ nguồn.

---

## 3. Xử lý Ngoại lệ trong Flow (Handle Exceptions in Flows)

Cả bên phát (emitter) lẫn bên thu thập (collector) đều có thể ném ngoại lệ.

### 3.1 Ngoại lệ ở Hạ nguồn (Collector Exception)
Nếu collector ném ngoại lệ, ngoại lệ sẽ lan truyền ngược lên thượng nguồn và được ném ra bên ngoài hàm `collect()`. Bạn có thể bọc lời gọi `collect()` trong khối `try-catch`:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class MyFlowException(message: String) : Exception(message)

//sampleStart
suspend fun main() {
    val myFlow = flow {
        try {
            emit('a')
        } catch (e: MyFlowException) {
            println("Collector threw $e")
            // Luôn rethrow ngoại lệ của hạ nguồn để bảo đảm Exception Transparency!
            throw e
        }
    }
    try {
        myFlow.collect {
            throw MyFlowException("Can't process '$it'!")
        }
    } catch (e: MyFlowException) {
        println("Flow collection failed with $e")
        throw e
    }
}
//sampleEnd
```

> [!IMPORTANT]
> Khi bạn bắt ngoại lệ do collector ném ra bên trong hàm tạo `flow { ... }`, hãy luôn **ném lại (rethrow)** nó. Điều này bảo toàn nguyên lý **Exception Transparency** và để cho bên gọi `collect()` tự quyết định cách xử lý.

---

### 3.2 Sử dụng Toán tử `.catch()` Xử lý Ngoại lệ Thượng nguồn

Toán tử **`.catch()`** can thiệp và xử lý các ngoại lệ xảy ra ở phần **thượng nguồn** trước khi chúng tới được collector. Bạn có thể sử dụng `emit()` bên trong `.catch` để phát ra một giá trị dự phòng (fallback):

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
suspend fun main() {
    flow {
        emit("a")
        emit("b")
        throw UnsupportedOperationException("I am tired of listing letters")
    }.catch { upstreamException ->
        println("Upstream completed with $upstreamException!")
        // Phát một giá trị fallback xuống hạ nguồn
        emit("Upstream terminated with an exception!")
    }.collect {
        println("Got '$it'")
    }
}
//sampleEnd
```

**Kết quả Console:**
```text
Got 'a'
Got 'b'
Upstream completed with java.lang.UnsupportedOperationException: I am tired of listing letters!
Got 'Upstream terminated with an exception!'
```

#### Xử lý lỗi dự kiến vs lỗi không mong muốn:
```kotlin
fun loadBlob(url: String) = flow {
    emit(LoadingState.Started)
    // Tải dữ liệu...
    if (networkFailed) throw IOException("Failed to load!")
    emit(LoadingState.Done)
}.catch { e ->
    if (e is IOException) {
        // Xử lý ngoại lệ dự kiến: phát trạng thái Failed
        emit(LoadingState.Failed)
    } else {
        // Ném lại các ngoại lệ không mong muốn
        throw e
    }
}
```

> [!WARNING]
> **Toán tử `.catch()` KHÔNG BẮT các ngoại lệ do chính Collector ném ra!**  
> Vì lambda của `collect()` chạy sau `.catch()`, nếu bạn muốn `.catch()` xử lý cả logic xử lý từng phần tử, hãy chuyển logic đó lên thượng nguồn bằng toán tử **`.onEach()`** đặt trước `.catch()`:
> ```kotlin
> flowOf('a', 'o', '5', 'c')
>     .onEach { 
>         require(!it.isDigit()) { "Digits are not allowed!" } 
>         println("Got '$it'")
>     }
>     .catch { e -> println("Caught an exception: $e") }
>     .collect()
> ```

---

### 3.3 Thử lại Luồng Thượng nguồn với `.retry()`

Đối với các lỗi có thể hồi phục (như mất mạng tạm thời), hãy sử dụng toán tử **`.retry()`**:

```kotlin
fun loadBlob(url: String) = flow {
    // Logic tải dữ liệu có thể ném IOException
    emit(data)
}.retry(3) { e ->
    if (e is IOException) {
        delay(1000) // Chờ 1 giây rồi thử lại
        true // Cho phép retry (tối đa 3 lần)
    } else {
        false // Dừng retry và ném ngoại lệ
    }
}
```

---

## 4. Hủy bỏ Flow (Flow Cancellation)

Quá trình thu thập Flow gắn liền với coroutine triệu gọi hàm `collect()`. Khi coroutine đó bị hủy, việc thu thập sẽ dừng lại và luồng thượng nguồn cũng sẽ bị hủy:

```kotlin
val job = launch {
    myFlow.collect { println(it) }
}
delay(100)
job.cancel() // Hủy coroutine -> Flow tự động dừng thu thập
```

### Hủy Thượng nguồn từ phía Collector (Cơ chế của `.take()`)
Bên thu thập cũng có thể chủ động hủy luồng thượng nguồn bằng cách ném một **`CancellationException`**. Dưới đây là cách toán tử `.take()` được triển khai bên dưới:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

//sampleStart
// Triển khai tùy biến đơn giản của toán tử .take() mặc định
fun <T> Flow<T>.myTake(count: Int): Flow<T> = flow {
    require(count > 0)
    val cancellationException = CancellationException()
    var elementsRemaining = count
    try {
        this@myTake.collect {
            emit(it)
            --elementsRemaining
            if (elementsRemaining == 0) {
                // Hủy luồng thượng nguồn sau khi đã đủ số phần tử yêu cầu
                throw cancellationException
            }
        }
    } catch (e: Throwable) {
        if (e === cancellationException) {
            // Nuốt CancellationException do chính mình ném ra để hoàn thành flow an toàn
        } else {
            throw e
        }
    }
}

suspend fun main() {
    (0..1000).asFlow().myTake(3).collect {
        println("Got $it")
    }
}
//sampleEnd
```

**Kết quả in ra Console:**
```text
Got 0
Got 1
Got 2
```

---

## 5. Phát Dữ liệu Đồng thời với `channelFlow()`

Hàm `flow()` chuẩn rất gọn và hiệu quả cho các luồng phát giá trị từ một coroutine duy nhất. Nếu bạn cần **phát các giá trị từ nhiều coroutine đồng thời vào cùng một flow**, hãy sử dụng **[`channelFlow()`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/channel-flow.html)**.

Bên trong `channelFlow`, bạn sử dụng hàm **`send()`** (thay vì `emit()`) để đẩy dữ liệu:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

//sampleStart
// Triển khai tùy biến đơn giản của toán tử .merge() mặc định
fun <T> Flow<T>.myMerge(other: Flow<T>): Flow<T> = channelFlow {
    // CoroutineScope và SendChannel sẵn sàng làm receiver tại đây
    launch {
        this@myMerge.collect { send(it) }
    }
    launch {
        other.collect { send(it) }
    }
}

suspend fun main() {
    val flow1 = (0..3).asFlow().onEach { delay(20.milliseconds) }
    val flow2 = (6..9).asFlow().onEach { delay(50.milliseconds) }
    flow1.myMerge(flow2).collect { println(it) }
}
//sampleEnd
```

### Quản lý Bộ đệm trong `channelFlow`:
`channelFlow` sử dụng một buffered channel với dung lượng mặc định là 64 phần tử. Bạn có thể tùy chỉnh kích thước bộ đệm bằng toán tử `.buffer()`:
- `.buffer(0)`: Xóa bỏ bộ đệm, bên phát `send()` sẽ phải chờ bên nhận xử lý xong từng phần tử.
- Mặc định: Bên phát gửi nhanh cho tới khi đầy 64 phần tử rồi mới tạm dừng nhường bên nhận.

---

## 6. Luồng Nóng (Hot Flows): `SharedFlow` & `StateFlow`

**Hot flows** là các luồng chia sẻ phát dữ liệu độc lập với việc có collector hay không. Chúng tiếp tục phát ngay cả khi không có ai lắng nghe, và nhiều collector có thể cùng thu thập các giá trị từ một luồng đã hoạt động sẵn. Collector của một hot flow được gọi là một **người đăng ký (subscriber)**.

Kotlin cung cấp 2 loại hot flow:
1. **`SharedFlow`**: Phát quảng bá (broadcast) các giá trị sự kiện tới nhiều subscribers.
2. **`StateFlow`**: Một dạng `SharedFlow` chuyên biệt luôn lưu giữ **một giá trị trạng thái mới nhất** và phát cập nhật khi giá trị này thay đổi.

---

### 6.1 Tạo một `SharedFlow`
Khởi tạo bằng hàm `MutableSharedFlow()`. Để bảo đảm tính đóng gói, hãy lưu trữ nó trong một private backing property và cung cấp ra ngoài một `SharedFlow` chỉ đọc bằng **`.asSharedFlow()`**:

```kotlin
data class Message(val senderId: Int, val text: String)

const val MESSAGES_TO_REMEMBER = 10

class Chatroom {
    // Lưu trữ trong private backing property
    private val _messages = MutableSharedFlow<Message>(
        replay = MESSAGES_TO_REMEMBER // Phát lại 10 tin nhắn gần nhất cho subscriber mới
    )

    // Cung cấp SharedFlow chỉ đọc ra bên ngoài
    val messages: SharedFlow<Message>
        get() = _messages.asSharedFlow()

    suspend fun sendMessageToEveryone(message: Message) {
        _messages.emit(message)
    }
}
```

> [!NOTE]
> Thu thập hot flow **sẽ không bao giờ tự kết thúc**. Bạn phải chủ động hủy coroutine đang gọi `collect()` khi không còn cần thiết.

#### Khởi chạy subscriber với `CoroutineStart.UNDISPATCHED`:
Trong các kịch bản khởi chạy đồng thời, việc sử dụng `launch(start = CoroutineStart.UNDISPATCHED)` đảm bảo subscriber bắt đầu lắng nghe ngay lập tức trước khi bên phát gọi `emit()`, tránh bị bỏ lỡ các thông điệp đầu tiên nếu bộ đệm replay quá nhỏ.

#### Cú pháp Kotlin 2.0+: Explicit Backing Fields
```kotlin
class Chatroom {
    // Khai báo SharedFlow chỉ đọc kèm backing field có thể chỉnh sửa
    val messages: SharedFlow<Message>
        field = MutableSharedFlow<Message>(replay = MESSAGES_TO_REMEMBER)

    suspend fun sendMessageToEveryone(message: Message) {
        messages.emit(message) // Tự động smart cast sang MutableSharedFlow bên trong class!
    }
}
```

---

### 6.2 Tạo một `StateFlow`

[`StateFlow`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/) luôn luôn chứa một giá trị trạng thái hiện tại. Thuộc tính **`.value`** cho phép đọc và ghi giá trị một cách an toàn luồng (thread-safe):

```kotlin
val countState = MutableStateFlow(0) // Giá trị ban đầu là 0
countState.value = 1
println(countState.value) // 1
```

> [!WARNING]
> Việc gán `state.value = newValue` là thread-safe, nhưng việc cập nhật giá trị mới dựa trên giá trị cũ (ví dụ: `state.value = state.value + 1`) **KHÔNG PHẢI là thao tác nguyên tử (atomic)**!
> Khi có nhiều coroutine cùng cập nhật, hãy luôn sử dụng hàm **`.update { ... }`**:
> ```kotlin
> class Post(val id: Long) {
>     private val _numberOfLikes = MutableStateFlow(0)
>     val numberOfLikes = _numberOfLikes.asStateFlow()
> 
>     fun like() {
>         // Cập nhật nguyên tử (Atomic CAS) chống mất mát dữ liệu
>         _numberOfLikes.update { it + 1 }
>     }
> }
> ```

#### Lưu trữ Trạng thái Tích lũy (Accumulated State):
Thay vì phát từng tin nhắn chat dưới dạng event bằng `SharedFlow<Message>`, bạn có thể lưu toàn bộ lịch sử trò chuyện trong một `StateFlow<List<Message>>`:
```kotlin
class Chatroom {
    private val _messageHistory = MutableStateFlow<List<Message>>(emptyList())
    val messageHistory: StateFlow<List<Message>> = _messageHistory.asStateFlow()

    suspend fun sendMessage(message: Message) {
        _messageHistory.update { it + message }
    }
}
```

---

### 6.3 Chuyển đổi Cold Flows sang Hot Flows (`shareIn` & `stateIn`)

Khi nhiều subscriber cùng cần dữ liệu từ một cold flow đắt đỏ, hãy biến nó thành hot flow để chỉ chạy phép tính thượng nguồn **đúng 1 lần duy nhất**:

#### 1. Chuyển sang SharedFlow với `.shareIn()`
```kotlin
val sharedMessages: SharedFlow<String> = myColdFlow
    .map { serialize(it) }
    .shareIn(
        scope = customScope,
        started = SharingStarted.Eagerly, // Bắt đầu thu thập ngay lập tức
        replay = 0
    )
```

#### 2. Chuyển sang StateFlow với `.stateIn()`
`stateIn()` yêu cầu một giá trị khởi tạo vì StateFlow luôn phải có giá trị hiện tại:
```kotlin
val state: StateFlow<Data?> = myColdFlow
    .stateIn(
        scope = customScope,
        started = SharingStarted.WhileSubscribed(5000), // Dừng phát nếu không còn subscriber sau 5s
        initialValue = null
    )
```

### 6.4 Hủy Hot Flows & Xử lý Ngoại lệ trong Hot Flows
- **Hủy Hot Flow**: Việc hủy subscriber chỉ dừng việc lắng nghe của người đó. Để hủy bản thân hot flow, bạn phải **hủy scope quản lý việc chia sẻ** (`customScope.cancel()`).
- **Ngoại lệ trong Hot Flows**: Nếu luồng thượng nguồn ném ngoại lệ trong `shareIn` hoặc `stateIn`, coroutine chia sẻ sẽ bị hủy. Để khắc phục, hãy **đặt toán tử `.retry()` ngay trước `.shareIn()` hoặc `.stateIn()`** để phục hồi trước khi ngoại lệ làm sập coroutine chia sẻ:

```kotlin
val stateFlow = myFlow
    .retry(retries = 5) // Tự động phục hồi lỗi trước khi tới stateIn!
    .stateIn(this@launch)
```

---

## 📝 Bảng Thuật ngữ & Tóm tắt nhanh

| Thuật ngữ | Khái niệm kỹ thuật tương ứng |
| :--- | :--- |
| **Cold Flow** | Luồng dữ liệu lười, mỗi collector kích hoạt một phiên tính toán riêng từ đầu. |
| **Hot Flow** | Luồng chủ động phát dữ liệu độc lập, đa phát tới nhiều subscribers cùng lúc. |
| **`flowOn`** | Đổi CoroutineDispatcher cho phần thượng nguồn mà không vi phạm tính bảo toàn ngữ cảnh. |
| **Exception Transparency** | `.catch` chỉ can thiệp ngoại lệ phía thượng nguồn, không bao giờ nuốt lỗi của collector hạ nguồn. |
| **`channelFlow`** | Cho phép phát dữ liệu đồng thời từ nhiều coroutines song song thông qua `send()`. |
| **`StateFlow`** | Hot flow đại diện trạng thái, luôn có giá trị khởi tạo và cơ chế loại bỏ dữ liệu trùng lặp (`conflation`). |
| **`SharedFlow`** | Hot flow phù hợp cho luồng phát sự kiện với bộ đệm phát lại (`replay`) tùy chỉnh. |
| **`update { }`** | Cập nhật nguyên tử CAS cho `MutableStateFlow` tránh xung đột đa luồng. |

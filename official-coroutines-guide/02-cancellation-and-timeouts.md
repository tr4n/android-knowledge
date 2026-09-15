# Bài 02 — Hủy bỏ và Giới hạn Thời gian (Cancellation and Timeouts)

> **Tài liệu gốc:** [Cancellation and timeouts — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-cancellation.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Nắm vững cơ chế hủy coroutine qua đối tượng `Job`, nguyên lý Structured Concurrency trong việc lan truyền hủy bỏ, tính hợp tác (cooperative cancellation), các điểm tạm dừng (suspension points), hàm `yield()`, ngắt mã blocking với `runInterruptible`, nguyên lý Prompt Cancellation, xử lý dọn dẹp tài nguyên với `try/finally`, khối không thể hủy `withContext(NonCancellable)`, và xử lý giới hạn thời gian (Timeout).

---

## 1. Hủy Coroutine (Cancel Coroutines)

Việc hủy bỏ (Cancellation) cho phép bạn yêu cầu dừng một coroutine trước khi nó tự hoàn thành. Cơ chế này giúp ngăn chặn các tác vụ không còn cần thiết tiếp tục tiêu tốn tài nguyên, ví dụ:
- Người dùng đóng cửa sổ hoặc điều hướng rời khỏi màn hình giao diện (UI) trong khi coroutine vẫn đang tải dữ liệu.
- Dừng các coroutine chạy dài thực hiện công việc lặp lại định kỳ (gửi tín hiệu heartbeat, chạy tác vụ lên lịch, cập nhật đồng hồ hiển thị).
- Giải phóng sớm tài nguyên phần cứng và bộ nhớ trước khi đối tượng bị dọn dẹp (disposal).

Cơ chế hủy hoạt động thông qua đối tượng đại diện [`Job`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/), đại diện cho vòng đời của một coroutine và các mối quan hệ phân cấp cha-con.

### 1.1 Kích hoạt Hủy với hàm `.cancel()`
Một coroutine bị hủy khi hàm `cancel()` được gọi trên đối tượng `Job` của nó:

```kotlin
import kotlinx.coroutines.*

suspend fun main() {
    withContext(Dispatchers.Default) {
        // Sử dụng làm tín hiệu báo rằng coroutine đã bắt đầu chạy
        val childStarted = CompletableDeferred<Unit>()
        
        val childJob: Job = launch {
            println("The coroutine has started")

            // Hoàn thành CompletableDeferred để phát tín hiệu
            childStarted.complete(Unit)
            try {
                // Tạm dừng vô thời hạn cho tới khi bị hủy
                awaitCancellation()
            } catch (e: CancellationException) {
                println("The coroutine was canceled: $e")
              
                // Luôn luôn ném lại (rethrow) CancellationException!
                throw e
            }
            println("This line will never be executed")
        }
      
        // Chờ coroutine khởi động trước khi thực hiện hủy
        childStarted.await()

        // Hủy coroutine, khiến awaitCancellation() ném ra CancellationException
        childJob.cancel()
    }
    // Các coroutine builder như withContext() hay coroutineScope()
    // luôn chờ tất cả các coroutine con hoàn tất, ngay cả khi chúng bị hủy
    println("All coroutines have completed")
}
```

**Kết quả Console:**
```text
The coroutine has started
The coroutine was canceled: kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelling}@...
All coroutines have completed
```

Trong ví dụ này:
- [`CompletableDeferred`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/) được sử dụng làm tín hiệu xác nhận coroutine con đã thực sự bắt đầu trước khi gọi lệnh hủy.
- Hàm `awaitCancellation()` tạm dừng coroutine vô hạn cho tới khi nhận lệnh hủy (tương đương `delay(Duration.INFINITE)`).
- Vì `Deferred` kế thừa từ `Job`, cơ chế hủy hoạt động hoàn toàn tương tự đối với các coroutine được tạo bởi `async()`:
  ```kotlin
  val deferred = async { /* ... */ }
  deferred.cancel()
  ```

> [!WARNING]
> Việc bắt ngoại lệ `CancellationException` mà không ném lại (rethrow) có thể phá vỡ sự lan truyền hủy của hệ thống. Nếu bạn bắt buộc phải `catch`, hãy luôn ném lại nó để sự kiện hủy được lan truyền chính xác trong cây phân cấp coroutine!

---

### 1.2 Sự Lan truyền Hủy bỏ (Cancellation Propagation)

Nguyên lý **Structured Concurrency** bảo đảm rằng khi bạn hủy một coroutine cha, **tất cả các coroutine con của nó sẽ tự động bị hủy theo chuỗi**. Điều này ngăn ngừa hoàn toàn tình trạng các coroutine con âm thầm tiếp tục chạy ngầm vô ích:

```kotlin
import kotlinx.coroutines.*

suspend fun main() {
    withContext(Dispatchers.Default) {
        // Tín hiệu báo rằng các coroutine con đã được khởi chạy
        val childrenLaunched = CompletableDeferred<Unit>()

        // Khởi chạy 2 coroutine con bên trong parentJob
        val parentJob = launch {
            launch {
                println("Child coroutine 1 has started running")
                try {
                    awaitCancellation()
                } finally {
                    println("Child coroutine 1 has been canceled")
                }
            }
            launch {
                println("Child coroutine 2 has started running")
                try {
                    awaitCancellation()
                } finally {
                    println("Child coroutine 2 has been canceled")
                }
            }
            // Hoàn thành tín hiệu khởi chạy
            childrenLaunched.complete(Unit)
        }

        // Chờ coroutine cha khởi chạy xong các con
        childrenLaunched.await()

        // Hủy coroutine cha -> Cả 2 coroutine con tự động bị hủy theo!
        parentJob.cancel()
    }
}
```

**Kết quả Console:**
```text
Child coroutine 1 has started running
Child coroutine 2 has started running
Child coroutine 1 has been canceled
Child coroutine 2 has been canceled
```

---

## 2. Tính Hợp tác của Việc Hủy (Make Coroutines React to Cancellation)

Trong Kotlin, việc hủy bỏ coroutine mang **tính chất hợp tác (cooperative)**. 

> [!IMPORTANT]
> Một coroutine chỉ phản hồi lại lệnh hủy khi nó chịu hợp tác bằng cách:
> 1. Đạt tới một **điểm tạm dừng (suspension point)**, HOẶC
> 2. Chủ động **kiểm tra trạng thái hủy một cách tường minh**.

### 2.1 Điểm Tạm dừng và Việc Hủy (Suspension Points and Cancellation)
Khi một coroutine bị hủy, nó tiếp tục chạy cho tới khi chạm đến một điểm trong mã nguồn mà nó có thể tạm dừng (gọi là **suspension point**). Tại điểm này, hàm suspend kiểm tra xem coroutine đã bị hủy hay chưa; nếu đã bị hủy, nó sẽ dừng lại và ném `CancellationException`.

Dưới đây là ví dụ minh họa các hàm suspend phổ biến trong `kotlinx.coroutines` đều có sẵn điểm kiểm tra hủy:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.channels.Channel
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.Duration

suspend fun main() {
    withContext(Dispatchers.Default) {
        val childJobs = listOf(
            launch {
                // Tạm dừng chờ hủy
                awaitCancellation()
            },
            launch {
                // Tạm dừng vô hạn
                delay(Duration.INFINITE)
            },
            launch {
                val channel = Channel<Int>()
                // Tạm dừng chờ phần tử không bao giờ gửi đến
                channel.receive()
            },
            launch {
                val deferred = CompletableDeferred<Int>()
                // Tạm dừng chờ kết quả không bao giờ hoàn tất
                deferred.await()
            },
            launch {
                val mutex = Mutex(locked = true)
                // Tạm dừng chờ mở khóa mutex bị lock vô thời hạn
                mutex.lock()
            }
        )
        
        // Cho phép các coroutine con có đủ thời gian khởi động và rơi vào suspension point
        delay(100.milliseconds)
        
        // Hủy toàn bộ các coroutine con
        childJobs.forEach { it.cancel() }
    }
    println("All child jobs completed!")
}
```

**Kết quả Console:**
```text
All child jobs completed!
```

> [!TIP]
> Tất cả các hàm suspend trong thư viện `kotlinx.coroutines` đều hợp tác với cơ chế hủy vì bên dưới chúng sử dụng [`suspendCancellableCoroutine()`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html). Ngược lại, các hàm suspend tùy biến tự viết bằng `suspendCoroutine()` của thư viện chuẩn stdlib sẽ **không** phản hồi lại lệnh hủy!

---

### 2.2 Hàm Suspend `yield()`
Nếu một coroutine thực thi tính toán CPU liên tục mà không có hàm suspend nào, các coroutine khác sẽ không có cơ hội được chạy trên thread đó, và coroutine đó cũng **không thể dừng lại khi bị cancel**.

Trong các vòng lặp tính toán nặng, hãy gọi định kỳ hàm **`yield()`**:
- `yield()` nhường thread hiện tại cho các coroutine khác có cơ hội chạy.
- `yield()` kiểm tra trạng thái hủy; nếu coroutine đã bị hủy, nó sẽ lập tức ném `CancellationException`.

```kotlin
import kotlinx.coroutines.*

fun main() {
    // runBlocking sử dụng thread hiện tại để chạy toàn bộ coroutines
    runBlocking {
        val coroutineCount = 5
        repeat(coroutineCount) { coroutineIndex ->
            launch {
                val id = coroutineIndex + 1
                repeat(5) { iterationIndex ->
                    val iteration = iterationIndex + 1
                    // Tạm dừng nhường thread cho các coroutine khác cùng chạy
                    // Nếu không có dòng này, các coroutine sẽ chạy tuần tự!
                    yield()
                    println("$id * $iteration = ${id * iteration}")
                }
            }
        }
    }
}
```

---

### 2.3 Kiểm tra Hủy Tường minh (`isActive` & `ensureActive`)
Nếu bạn không muốn gọi `yield()`, bạn có thể kiểm tra hủy thủ công:
- **Thuộc tính `isActive`**: Trả về `false` khi coroutine đã nhận tín hiệu hủy.
  ```kotlin
  while (isActive) {
      // Tiếp tục tính toán...
  }
  ```
- **Hàm `ensureActive()`**: Lập tức ném `CancellationException` nếu coroutine không còn active:
  ```kotlin
  for (item in largeDataset) {
      ensureActive() // Dừng ngay lập tức nếu đã bị cancel
      processItem(item)
  }
  ```

---

### 2.4 Ngắt Mã Blocking Khi Coroutine Bị Hủy (`runInterruptible`)
Trên máy ảo JVM, một số hàm chặn luồng như `Thread.sleep()` hoặc `BlockingQueue.take()` sẽ khóa cứng thread. Khi bạn gọi chúng từ coroutine, lệnh hủy coroutine thông thường **không thể ngắt (interrupt) thread JVM đó**.

Để ngắt thread khi coroutine bị hủy, hãy bọc mã blocking bên trong hàm **`runInterruptible()`**:

```kotlin
import kotlinx.coroutines.*

suspend fun main() {
    withContext(Dispatchers.Default) {
        val childStarted = CompletableDeferred<Unit>()
        val childJob = launch {
            try {
                // Lệnh hủy coroutine sẽ kích hoạt Thread.interrupt() trên luồng này
                runInterruptible {
                    childStarted.complete(Unit)
                    try {
                        // Khóa thread hiện tại trong thời gian cực dài
                        Thread.sleep(Long.MAX_VALUE)
                    } catch (e: InterruptedException) {
                        println("Thread interrupted (Java): $e")
                        throw e
                    }
                }
            } catch (e: CancellationException) {
                println("Coroutine canceled (Kotlin): $e")
                throw e
            }
        }
        childStarted.await()

        // Hủy coroutine và ngắt thread đang chạy Thread.sleep()
        childJob.cancel()
    }
}
```

**Kết quả Console:**
```text
Thread interrupted (Java): java.lang.InterruptedException: sleep interrupted
Coroutine canceled (Kotlin): kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelling}@...
```

---

## 3. Xử lý An toàn Giá trị Khi Hủy Coroutine (Handle Values Safely)

### 3.1 Nguyên lý Hủy Ngay Tức Thì (Prompt Cancellation)
Khi một coroutine đang tạm dừng bị hủy, nó sẽ **tiếp tục (resumes) bằng một `CancellationException` thay vì trả về giá trị kết quả**, ngay cả khi dữ liệu đó đã sẵn sàng tính toán xong.

Hành vi này được gọi là **Prompt Cancellation**. Nó ngăn ngừa việc mã nguồn tiếp tục chạy vô nghĩa trong scope đã bị hủy (chẳng hạn như cập nhật giao diện lên một màn hình đã bị người dùng đóng).

Hãy xem ví dụ màn hình giao diện người dùng (UI):

```kotlin
// Định nghĩa một scope chạy trên UI thread
class ScreenWithButtons(private val scope: CoroutineScope) {
    fun loadAndUpdateButtons(filename: String) {
        scope.launch {
            // withContext() tự động kiểm tra hủy trước khi vào khối và sau khi khối trả về
            val buttonNames = withContext(Dispatchers.IO) {
                // Thao tác đọc file blocking
                readLines(filename)
            }
            
            // Lời gọi updateUi() này an toàn tuyệt đối:
            // Vì withContext() sẽ không trả về nếu coroutine đã bị hủy trước đó
            updateUi(buttonNames)
        }
    }

    private fun updateUi(buttonNames: List<String>) {
        // Cập nhật danh sách nút bấm lên giao diện
    }

    fun leaveScreen() {
        // Người dùng bấm Back rời màn hình -> Hủy scope ngay
        scope.cancel()
    }
}
```

### 3.2 Đóng và Dọn dẹp Tài nguyên với `try/finally`
Mặc dù Prompt Cancellation rất an toàn cho UI, nó có thể làm đứt gãy luồng xử lý nếu bạn đang nắm giữ một tài nguyên cần đóng (như `AutoCloseable`, `BufferedReader`, `Socket`).

Để bảo đảm tài nguyên luôn được giải phóng, hãy đặt mã dọn dẹp bên trong khối **`finally`**:

```kotlin
import java.nio.file.*
import java.nio.charset.*
import kotlinx.coroutines.*
import java.io.*

class ScreenWithFileContents(private val scope: CoroutineScope) {
    fun displayFile(path: Path) {
        scope.launch {
            // Lưu reader vào biến để khối finally có thể đóng nó
            var reader: BufferedReader? = null
            
            try {
                withContext(Dispatchers.IO) {
                    reader = Files.newBufferedReader(path, Charset.forName("US-ASCII"))
                }
                // Sử dụng reader
                updateUi(reader!!)
            } finally {
                // Đảm bảo reader luôn luôn được đóng ngay cả khi coroutine bị hủy
                reader?.close()
                println("Tài nguyên reader đã được giải phóng an toàn!")
            }
        }
    }

    private suspend fun updateUi(reader: BufferedReader) {
        while (true) {
            val line = withContext(Dispatchers.IO) { reader.readLine() } ?: break
            addOneLineToUi(line)
        }
    }

    private fun addOneLineToUi(line: String) {}

    fun leaveScreen() {
        scope.cancel()
    }
}
```

---

### 3.3 Chạy Khối Không thể Hủy: `withContext(NonCancellable)`

> [!CAUTION]
> **Tuyệt đối tránh sử dụng `NonCancellable` với các builder như `.launch()` hoặc `.async()`**. Làm như vậy sẽ phá vỡ Structured Concurrency vì cắt đứt mối quan hệ cha-con.
> Chỉ truyền `NonCancellable` làm đối số cho hàm **`withContext()`**!

`NonCancellable` cực kỳ hữu ích khi bạn cần thực hiện một thao tác dọn dẹp có chứa hàm `suspend` (ví dụ: gọi hàm đóng dịch vụ bất đồng bộ `shutdownServiceAndWait()`) bên trong khối `finally` của một coroutine đã bị hủy:

```kotlin
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

val serviceStarted = CompletableDeferred<Unit>()

fun startService() {
    println("Starting the service...")
    serviceStarted.complete(Unit)
}

suspend fun shutdownServiceAndWait() {
    println("Shutting down...")
    delay(100.milliseconds) // Thao tác suspend dọn dẹp
    println("Successfully shut down!")
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        val childJob = launch {
            startService()
            try {
                awaitCancellation()
            } finally {
                withContext(NonCancellable) {
                    // Nếu không có withContext(NonCancellable),
                    // hàm suspend này sẽ bị hủy ngay lập tức và không thể hoàn thành!
                    shutdownServiceAndWait()
                }
            }
        }
        serviceStarted.await()
        childJob.cancel()
    }
    println("Exiting the program")
}
```

**Kết quả Console:**
```text
Starting the service...
Shutting down...
Successfully shut down!
Exiting the program
```

---

## 4. Giới hạn Thời gian (Timeout) với `withTimeoutOrNull`

Hạn định thời gian (Timeout) cho phép bạn tự động hủy coroutine sau một khoảng thời lượng quy định.

Ví dụ: Nếu yêu cầu tải dữ liệu quá lâu, bạn có thể hủy nó và chuyển sang dùng dữ liệu lưu tạm (cache) cục bộ:

```kotlin
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun slowOperation(): String {
    try {
        delay(300.milliseconds)
        return "A"
    } catch (e: CancellationException) {
        println("The slow operation has been canceled: $e")
        throw e
    }
}

suspend fun fastOperation(): String {
    try {
        delay(15.milliseconds)
        return "B"
    } catch (e: CancellationException) {
        println("The fast operation has been canceled: $e")
        throw e
    }
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        // Thao tác chậm: timeout 100ms trong khi hàm mất 300ms -> Trả về null
        val slow = withTimeoutOrNull(100.milliseconds) {
            slowOperation()
        }
        println("The slow operation finished with $slow")

        // Thao tác nhanh: timeout 100ms trong khi hàm chỉ mất 15ms -> Trả về "B"
        val fast = withTimeoutOrNull(100.milliseconds) {
            fastOperation()
        }
        println("The fast operation finished with $fast")
    }
}
```

**Kết quả Console:**
```text
The slow operation has been canceled: kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 100 ms
The slow operation finished with null
The fast operation finished with B
```

---

## 📝 Bảng Thuật ngữ & Tóm tắt nhanh

| Thuật ngữ | Khái niệm kỹ thuật |
| :--- | :--- |
| **Cooperative Cancellation** | Hủy có hợp tác: Coroutine chỉ dừng lại tại suspension point hoặc khi chủ động kiểm tra trạng thái hủy. |
| **Prompt Cancellation** | Coroutine đang suspend khi bị cancel sẽ resume với `CancellationException`, ngăn chặn việc dùng dữ liệu cũ. |
| **`yield()`** | Hàm suspend tạm nhường thread và kiểm tra hủy ngay tức thì cho các vòng lặp tính toán nặng. |
| **`runInterruptible`** | Bọc mã blocking JVM để kích hoạt `Thread.interrupt()` khi coroutine bị hủy. |
| **`NonCancellable`** | Chỉ dùng với `withContext(NonCancellable)` để chạy các hàm suspend dọn dẹp trong khối `finally`. |
| **`withTimeoutOrNull`** | Hủy tác vụ khi quá thời hạn quy định và trả về `null` an toàn thay vì ném ngoại lệ. |

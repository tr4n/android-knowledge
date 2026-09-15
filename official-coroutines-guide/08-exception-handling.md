# Bài 08 — Xử lý Ngoại lệ trong Coroutine (Coroutine Exceptions Handling)

> **Tài liệu gốc:** [Coroutine exceptions handling — Kotlin Documentation](https://kotlinlang.org/docs/exception-handling.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Nắm vững cơ chế lan truyền ngoại lệ (Exception Propagation) hai chiều; phân biệt hành vi ngoại lệ giữa `launch` và `async`; vai trò và phạm vi hoạt động của `CoroutineExceptionHandler`; xử lý hủy bỏ và ngoại lệ (Cancellation & Exceptions); cơ chế gộp ngoại lệ bị đè nén (Exceptions Aggregation); và kỹ thuật cô lập lỗi một chiều với `SupervisorJob` và `supervisorScope`.

---

## 1. Cơ chế Lan truyền Ngoại lệ (Exception Propagation)

Chúng ta đã biết rằng một coroutine bị hủy sẽ ném ra `CancellationException` tại các điểm tạm ngưng (suspension points), và ngoại lệ này được bộ máy quản lý coroutine bỏ qua mà không làm sập ứng dụng. Phần này sẽ xem xét điều gì sẽ xảy ra khi một ngoại lệ khác (không phải `CancellationException`) phát sinh trong lúc coroutine đang chạy, hoặc khi nhiều coroutine con cùng ném ra lỗi.

Các coroutine builder được chia thành hai nhóm riêng biệt về cơ chế lan truyền ngoại lệ:
1. **Tự động lan truyền ngoại lệ (Propagating exceptions automatically)**: Điển hình là **`launch`**. Khi gặp ngoại lệ chưa bắt (uncaught exception), builder này sẽ ném ra ngay lập tức (tương tự như `Thread.uncaughtExceptionHandler` trong Java).
2. **Ủy quyền cho người dùng tự xử lý (Exposing exceptions to users)**: Điển hình là **`async`** và **`produce`** (trong Channel). Các builder này không ném lỗi ra ngay mà đóng gói ngoại lệ vào đối tượng kết quả (`Deferred` hoặc `ReceiveChannel`), trông đợi người dùng sẽ tiêu thụ ngoại lệ đó thông qua lệnh **`await()`** hoặc **`receive()`**.

Khi các builder trên được dùng để tạo một **coroutine gốc (root coroutine)** — tức là coroutine không phải là con của bất kỳ coroutine nào khác:

```kotlin
import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val job = GlobalScope.launch { // Root coroutine tạo bằng launch
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Sẽ được in ra console bởi Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    
    val deferred = GlobalScope.async { // Root coroutine tạo bằng async
        println("Throwing exception from async")
        throw ArithmeticException() // Không in gì cả, đợi người dùng gọi await()
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

> [!NOTE]
> `GlobalScope` là một API nhạy cảm (`@DelicateCoroutinesApi`) có thể gây ra những phản ứng phụ nguy hiểm. Việc tạo một root coroutine cho toàn bộ vòng đời ứng dụng là một trong số rất ít trường hợp sử dụng hợp lệ của `GlobalScope`.

**Kết quả Console Output (kèm cờ `-Dkotlinx.coroutines.debug`):**
```none
Throwing exception from launch
Exception in thread "DefaultDispatcher-worker-1 @coroutine#2" java.lang.IndexOutOfBoundsException
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

---

## 2. Trình Xử lý Ngoại lệ: `CoroutineExceptionHandler`

Bạn có thể tùy biến hành vi mặc định (in ngoại lệ chưa bắt ra console) bằng cách cài đặt phần tử ngữ cảnh **`CoroutineExceptionHandler`**.

Cài đặt `CoroutineExceptionHandler` trên một **root coroutine** đóng vai trò như một khối `catch` tổng quát cho root coroutine đó và **tất cả các coroutine con của nó**, tương tự như `Thread.setUncaughtExceptionHandler` trong Java.

> [!CAUTION]
> **Bạn KHÔNG THỂ phục hồi (recover) coroutine từ `CoroutineExceptionHandler`!**  
> Khi handler này được gọi, coroutine tương ứng **đã hoàn thành trong trạng thái lỗi (completed exceptionally)**. Thông thường handler chỉ được dùng để:
> - Ghi log lỗi (crash reporting/logging).
> - Hiển thị thông báo lỗi lên giao diện.
> - Kết thúc hoặc khởi động lại một tiến trình.

### Quy tắc Hoạt động Khắt khe của `CoroutineExceptionHandler`:
1. `CoroutineExceptionHandler` **chỉ được kích hoạt đối với các ngoại lệ chưa bắt (uncaught exceptions)**.
2. **Tất cả coroutine con** đều ủy quyền xử lý ngoại lệ lên coroutine cha, và cha ủy quyền tiếp lên cha của nó cho đến root. Do đó, bất kỳ `CoroutineExceptionHandler` nào được cài đặt ở ngữ cảnh của **coroutine con đều bị BỎ QUA HOÀN TOÀN**!
3. Builder **`async` luôn tự bắt tất cả ngoại lệ** và thể hiện chúng trong đối tượng `Deferred`, nên việc cài `CoroutineExceptionHandler` vào `async` cũng **hoàn toàn vô tác dụng**:

```kotlin
import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    
    val job = GlobalScope.launch(handler) { // Root coroutine chạy trong GlobalScope
        throw AssertionError()
    }
    
    val deferred = GlobalScope.async(handler) { // Cũng là root, nhưng là async
        throw ArithmeticException() // Sẽ không in gì ra handler, chờ gọi await()
    }
    
    joinAll(job, deferred)
}
```

**Kết quả Console Output:**
```none
CoroutineExceptionHandler got java.lang.AssertionError
```

---

## 3. Hủy bỏ và Ngoại lệ (Cancellation and Exceptions)

Quá trình hủy bỏ coroutine gắn liền mật thiết với ngoại lệ. Coroutine sử dụng `CancellationException` nội bộ cho việc hủy; ngoại lệ này được tất cả các handler bỏ qua.

Khi một coroutine bị hủy bằng lệnh `Job.cancel()`, nó sẽ kết thúc nhưng **KHÔNG làm hủy coroutine cha của nó**:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Child is cancelled")
            }
        }
        yield()
        println("Cancelling child")
        child.cancel()
        child.join()
        yield()
        println("Parent is not cancelled")
    }
    job.join()
}
```

**Kết quả Console Output:**
```none
Cancelling child
Child is cancelled
Parent is not cancelled
```

### Lan truyền Ngoại lệ Không Phải Hủy bỏ (Non-Cancellation Exceptions)
Nếu một coroutine gặp phải một ngoại lệ **khác với `CancellationException`**, nó sẽ **hủy ngay lập tức coroutine cha của nó** bằng chính ngoại lệ đó! Hành vi này mang tính cốt lõi của **Đồng thời Có cấu trúc (Structured Concurrency)** và không thể bị ghi đè.

> [!NOTE]
> Coroutine cha chỉ thực sự chuyển giao ngoại lệ cho handler xử lý sau khi **TẤT CẢ các coroutine con khác của nó đã kết thúc hoàn toàn**.

Hãy xem ví dụ dưới đây: Coroutine con thứ 2 ném lỗi, khiến coroutine con thứ 1 bị hủy; tuy nhiên coroutine con thứ 1 có khối dọn dẹp `NonCancellable` mất 100ms. Handler chỉ được gọi sau khi con thứ 1 dọn dẹp xong:

```kotlin
import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    val job = GlobalScope.launch(handler) {
        launch { // Coroutine con thứ nhất
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // Coroutine con thứ hai
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
}
```

**Kết quả Console Output:**
```none
Second child throws an exception
Children are cancelled, but exception is not handled until all children terminate
The first child finished its non cancellable block
CoroutineExceptionHandler got java.lang.ArithmeticException
```

---

## 4. Cơ chế Gộp Ngoại lệ (Exceptions Aggregation)

Khi nhiều coroutine con của cùng một cha đồng thời thất bại với các ngoại lệ khác nhau, nguyên tắc tổng quát là: **"Ngoại lệ đầu tiên sẽ chiến thắng (The first exception wins)"**.

Ngoại lệ đầu tiên xảy ra sẽ được chuyển tới handler để xử lý. Tất cả các ngoại lệ phát sinh tiếp theo sau đó sẽ được **đính kèm vào ngoại lệ đầu tiên dưới dạng ngoại lệ bị đè nén (`suppressed exceptions`)**:

```kotlin
import kotlinx.coroutines.*
import java.io.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) // Sẽ bị hủy khi coroutine anh em ném IOException
            } finally {
                throw ArithmeticException() // Ngoại lệ thứ hai phát sinh trong khối dọn dẹp
            }
        }
        launch {
            delay(100)
            throw IOException() // Ngoại lệ đầu tiên phát sinh
        }
        delay(Long.MAX_VALUE)
    }
    job.join()  
}
```

**Kết quả Console Output:**
```none
CoroutineExceptionHandler got java.io.IOException with suppressed [java.lang.ArithmeticException]
```

### Tính Trong suốt của CancellationException (Unwrapping)
Các ngoại lệ hủy bỏ mang tính chất trong suốt và được tự động bóc tách (unwrapped) theo mặc định:

```kotlin
import kotlinx.coroutines.*
import java.io.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        val innerJob = launch {
            launch {
                launch {
                    throw IOException() // Ngoại lệ gốc ban đầu
                }
            }
        }
        try {
            innerJob.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e // Ném lại CancellationException, nhưng handler vẫn nhận được đúng IOException gốc!
        }
    }
    job.join()
}
```

**Kết quả Console Output:**
```none
Rethrowing CancellationException with original cause
CoroutineExceptionHandler got java.io.IOException
```

---

## 5. Cơ chế Giám sát (Supervision)

Như chúng ta đã biết, theo mặc định sự cố lỗi trong coroutine là một **mối quan hệ hai chiều (bidirectional)**: con chết $\rightarrow$ cha chết $\rightarrow$ các con khác bị chết theo.

Tuy nhiên, trong thực tế có rất nhiều kịch bản đòi hỏi **sự lan truyền lỗi chỉ theo MỘT CHIỀU (unidirectional failure propagation)**:
1. **Thành phần Giao diện UI**: Nếu một tác vụ con trên màn hình (như tải ảnh đại diện) bị lỗi mạng, ta không thể để toàn bộ màn hình UI bị crash hoặc đóng lại. Nhưng nếu người dùng đóng màn hình (Job cha bị hủy), ta bắt buộc phải hủy toàn bộ các tác vụ con đang chạy ngầm.
2. **Máy chủ (Server Process)**: Một server quản lý nhiều kết nối người dùng cần giám sát từng tác vụ xử lý; lỗi ở một kết nối không được phép làm sập toàn bộ máy chủ.

---

### 5.1 Giám sát bằng `SupervisorJob`

`SupervisorJob` hoạt động tương tự như một `Job` thông thường, với ngoại lệ duy nhất là: **Sự cố lỗi hoặc việc hủy bỏ của một coroutine con KHÔNG lan truyền lên `SupervisorJob` và KHÔNG làm ảnh hưởng đến các coroutine con khác**:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // Khởi chạy con thứ nhất -- lỗi của nó không làm sập supervisor
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("The first child is failing")
            throw AssertionError("The first child is cancelled")
        }
        // Khởi chạy con thứ hai
        val secondChild = launch {
            firstChild.join()
            // Việc hủy con thứ nhất KHÔNG lan truyền sang con thứ hai:
            println("The first child is cancelled: ${firstChild.isCancelled}, but the second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // Nhưng nếu supervisor bị hủy thì tất cả các con vẫn sẽ bị hủy theo:
                println("The second child is cancelled because the supervisor was cancelled")
            }
        }
        // Đợi con thứ nhất thất bại hoàn tất
        firstChild.join()
        println("Cancelling the supervisor")
        supervisor.cancel() // Hủy cha thì con thứ hai mới bị hủy
        secondChild.join()
    }
}
```

**Kết quả Console Output:**
```none
The first child is failing
The first child is cancelled: true, but the second one is still active
Cancelling the supervisor
The second child is cancelled because the supervisor was cancelled
```

---

### 5.2 Khối Phạm vi Giám sát: `supervisorScope`

Thay vì tạo `SupervisorJob` thủ công, đối với đồng thời có cấu trúc cục bộ, chúng ta sử dụng **`supervisorScope`** (thay thế cho `coroutineScope`).

`supervisorScope` chỉ lan truyền việc hủy bỏ theo **một chiều**:
- Nếu một tác vụ con bên trong thất bại, các tác vụ con khác trong scope vẫn chạy bình thường.
- Nhưng nếu chính khối `supervisorScope` gặp ngoại lệ, tất cả các tác vụ con bên trong nó sẽ bị hủy.
- Giống như `coroutineScope`, nó luôn chờ tất cả các coroutine con kết thúc trước khi hoàn thành:

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("The child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("The child is cancelled")
                }
            }
            // Nhường CPU cho child chạy một chút
            yield()
            println("Throwing an exception from the scope")
            throw AssertionError() // Lỗi từ chính scope sẽ hủy child
        }
    } catch(e: AssertionError) {
        println("Caught an assertion error")
    }
}
```

**Kết quả Console Output:**
```none
The child is sleeping
Throwing an exception from the scope
The child is cancelled
Caught an assertion error
```

---

### 5.3 Xử lý Ngoại lệ trong các Coroutine Giám sát (Exceptions in Supervised Coroutines)

Đây là điểm khác biệt sống còn giữa Job thông thường và Supervisor Job:

> [!IMPORTANT]
> **Trong `supervisorScope`, mỗi coroutine con phải tự chịu trách nhiệm xử lý ngoại lệ của chính nó!**  
> Do lỗi không lan truyền lên coroutine cha, các coroutine được khởi chạy trực tiếp bên trong `supervisorScope` **ĐƯỢC PHÉP và BẮT BUỘC sử dụng `CoroutineExceptionHandler` cài đặt trong chính ngữ cảnh của nó** (tương tự như cách một root coroutine hoạt động):

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    supervisorScope {
        // Cài đặt handler trực tiếp vào coroutine con bên trong supervisorScope:
        val child = launch(handler) {
            println("The child throws an exception")
            throw AssertionError()
        }
        println("The scope is completing")
    }
    println("The scope is completed")
}
```

**Kết quả Console Output:**
```none
The scope is completing
The child throws an exception
CoroutineExceptionHandler got java.lang.AssertionError
The scope is completed
```

> [!NOTE]
> Hãy quan sát kết quả: Dòng `The scope is completed` vẫn được in ra bình thường! Khối `supervisorScope` không hề bị sập bởi lỗi của `child`, và handler đã bắt thành công `AssertionError`.

---

## 6. Bảng Tổng kết & Phân định Trách nhiệm

| Tình huống / Cơ chế | `coroutineScope` / `Job` thông thường | `supervisorScope` / `SupervisorJob` |
| :--- | :--- | :--- |
| **Hướng lan truyền lỗi** | Hai chiều (Bidirectional): Con lỗi $\rightarrow$ Cha chết $\rightarrow$ Toàn bộ con khác chết. | Một chiều (Unidirectional): Con lỗi $\rightarrow$ Cô lập; Cha chết $\rightarrow$ Hủy tất cả con. |
| **Vị trí `CoroutineExceptionHandler`** | Chỉ có tác dụng ở **Root Coroutine**; cài ở coroutine con bị bỏ qua hoàn toàn. | Có tác dụng ở **ngay từng coroutine con trực tiếp** bên trong supervisor scope. |
| **Hành vi `async` khi có lỗi** | Ngoại lệ ném ra khi gọi `await()`; nhưng đồng thời làm sập luôn toàn bộ cây cha con ngay lập tức! | Ngoại lệ chỉ ném ra khi gọi `await()`; không làm ảnh hưởng đến cha hay các nhánh anh em khác. |
| **Kịch bản khuyến nghị** | Tính toán song song mà các kết quả phụ thuộc lẫn nhau (hỏng 1 việc thì hủy toàn bộ). | Giao diện người dùng Android (UI), Server xử lý request độc lập, tác vụ nền định kỳ. |

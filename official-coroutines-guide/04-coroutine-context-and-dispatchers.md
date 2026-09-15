# Bài 04 — Ngữ cảnh Coroutine và Bộ điều phối (Coroutine Context and Dispatchers)

> **Tài liệu gốc:** [Coroutine context and dispatchers — Kotlin Documentation](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Hiểu sâu về `CoroutineContext`, các loại `CoroutineDispatcher`, cơ chế Unconfined vs Confined, công cụ gỡ lỗi Coroutine Debugger trên IntelliJ IDEA và logging `-Dkotlinx.coroutines.debug`, nhảy thread an toàn với `withContext`, truy xuất `Job` trong context, quan hệ phân cấp cha-con, trách nhiệm bảo bọc của coroutine cha, đặt tên coroutine, quản lý lifecycle với `CoroutineScope`, và đồng bộ dữ liệu `ThreadLocal`.

---

## 1. Bộ điều phối và Luồng (Dispatchers and Threads)

Mọi coroutine luôn thực thi trong một ngữ cảnh được biểu diễn bởi một giá trị thuộc kiểu [`CoroutineContext`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/), được định nghĩa ngay trong thư viện chuẩn của Kotlin.

Ngữ cảnh coroutine là một tập hợp các phần tử cấu hình khác nhau. Các phần tử chủ đạo nhất bao gồm [`Job`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) của coroutine và bộ điều phối [`CoroutineDispatcher`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html).

- **`CoroutineDispatcher`** quyết định thread hoặc các thread nào sẽ được sử dụng để thực thi coroutine tương ứng.
- Bộ điều phối có thể giới hạn coroutine chạy trên một thread cụ thể, phân phối nó vào một thread pool, hoặc để nó chạy không giới hạn (`unconfined`).
- Tất cả các coroutine builder như `launch` và `async` đều chấp nhận một tham số `CoroutineContext` tùy chọn để bạn chỉ định rõ ràng dispatcher:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch { // Ngữ cảnh của coroutine cha (main runBlocking)
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // Không giới hạn -- sẽ chạy ngay trên main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // Được phân phối tới DefaultDispatcher 
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    @OptIn(DelicateCoroutinesApi::class)
    launch(newSingleThreadContext("MyOwnThread")) { // Tạo một thread mới riêng biệt
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

**Kết quả in ra Console (thứ tự có thể thay đổi):**
```text
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

### Phân tích chi tiết:
1. Khi `launch { ... }` được gọi không kèm tham số, nó **thừa kế ngữ cảnh (và dispatcher)** từ `CoroutineScope` mà nó được sinh ra. Ở đây là coroutine chính `runBlocking` đang chạy trên thread `main`.
2. `Dispatchers.Default` là bộ điều phối mặc định được sử dụng khi không có dispatcher nào được chỉ định rõ trong scope. Nó sử dụng một pool các thread chạy nền dùng chung.
3. `newSingleThreadContext` tạo riêng một thread chuyên dụng cho coroutine chạy. 
   > [!WARNING]
   > Một thread chuyên dụng là một tài nguyên rất đắt đỏ của hệ điều hành. Trong ứng dụng thực tế, nó phải được giải phóng khi không còn cần thiết bằng hàm `close()`, hoặc được lưu trong một biến cấp cao nhất (top-level) và tái sử dụng xuyên suốt vòng đời ứng dụng.

---

## 2. Bộ điều phối Giới hạn vs Không giới hạn (Unconfined vs Confined Dispatcher)

Bộ điều phối [`Dispatchers.Unconfined`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html) khởi chạy coroutine ngay trên thread của bên gọi, nhưng **chỉ cho tới điểm suspension đầu tiên**.
- Sau khi hàm suspend hoàn tất, nó tiếp tục thực thi coroutine trên thread do chính hàm suspend đó quyết định.
- `Unconfined` phù hợp cho các coroutine không tiêu tốn CPU cũng như không cập nhật dữ liệu chia sẻ bị ràng buộc vào một thread cố định (như cập nhật UI).

Ngược lại, các coroutine mặc định thừa kế dispatcher từ `CoroutineScope` bên ngoài. Cụ thể, dispatcher của `runBlocking` bị giới hạn trong thread của bên triệu gọi với cơ chế lập lịch FIFO có thể dự đoán được:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch(Dispatchers.Unconfined) { // Không giới hạn -- bắt đầu trên main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // Kế thừa từ main runBlocking
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

**Kết quả in ra Console:**
```text
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

*Phân tích*: Coroutine kế thừa ngữ cảnh từ `runBlocking` tiếp tục chạy trên thread `main`, trong khi coroutine `Unconfined` sau khi gọi `delay()` được resume trên thread của bộ thực thi mặc định (`DefaultExecutor`) mà hàm `delay` sử dụng.

> [!NOTE]
> Bộ điều phối `Unconfined` là một cơ chế nâng cao chỉ hữu ích trong một số trường hợp góc (corner cases) khi việc phân phối coroutine để thực thi sau đó là không cần thiết hoặc gây ra các hiệu ứng phụ không mong muốn. Không nên sử dụng `Dispatchers.Unconfined` trong mã nguồn thông thường.

---

## 3. Gỡ lỗi Coroutines và Threads (Debugging Coroutines and Threads)

Vì coroutine có thể tạm dừng trên thread này và tiếp tục trên thread khác, việc theo dõi những gì một coroutine đang làm, ở đâu và vào thời điểm nào có thể rất khó khăn nếu không có công cụ hỗ trợ.

### 3.1 Gỡ lỗi với IntelliJ IDEA (Debugging with IDEA)
Công cụ **Coroutine Debugger** tích hợp trong Kotlin plugin của IntelliJ IDEA đơn giản hóa việc theo dõi coroutines:
- Cửa sổ công cụ **Debug** chứa tab **Coroutines**.
- Trong tab này, bạn có thể xem thông tin chi tiết về cả các coroutine đang chạy (`RUNNING`) lẫn các coroutine đang tạm dừng (`SUSPENDED`), được nhóm lại theo từng Dispatcher.
- Xem toàn bộ giá trị biến cục bộ và biến capture tại các điểm suspension.
- Xem toàn bộ stack khởi tạo coroutine (creation stack) và call stack bên trong coroutine.
- Lấy báo cáo snapshot toàn cảnh bằng cách nhấp chuột phải và chọn **Get Coroutines Dump**.

### 3.2 Gỡ lỗi bằng cách Ghi Log (Debugging Using Logging)
Một phương pháp khác để gỡ lỗi khi không dùng Coroutine Debugger là in tên thread trong mỗi câu lệnh log. Hãy thêm tùy chọn máy ảo JVM: **`-Dkotlinx.coroutines.debug`** khi chạy:

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
//sampleStart
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
//sampleEnd    
}
```

**Kết quả in ra Console:**
```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

Hàm `log` in tên thread trong dấu ngoặc vuông kèm theo **định danh coroutine** được gán tự động (`@coroutine#1`, `@coroutine#2`, `@coroutine#3`), giúp bạn phân biệt chính xác coroutine nào đang thực thi mã.

---

## 4. Chuyển đổi giữa các Luồng (Jumping Between Threads)

Hãy chạy đoạn mã sau với tùy chọn JVM `-Dkotlinx.coroutines.debug`:

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() {
    newSingleThreadContext("Ctx1").use { ctx1 ->
        newSingleThreadContext("Ctx2").use { ctx2 ->
            runBlocking(ctx1) {
                log("Started in ctx1")
                withContext(ctx2) {
                    log("Working in ctx2")
                }
                log("Back to ctx1")
            }
        }
    }
}
```

**Kết quả in ra Console:**
```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

Ví dụ trên minh họa hai kỹ thuật:
1. `runBlocking` với một ngữ cảnh được chỉ định rõ ràng (`ctx1`).
2. Hàm `withContext` tạm dừng coroutine hiện tại và chuyển sang ngữ cảnh mới (`ctx2`). Sau khi khối mã trong `withContext` kết thúc, quyền thực thi tự động trở về dispatcher ban đầu.
3. Sử dụng hàm `.use` từ Kotlin Standard Library để bảo đảm thread chuyên dụng tạo bởi `newSingleThreadContext` được tự động đóng (`close`) an toàn.

---

## 5. Job trong Ngữ cảnh (Job in the Context)

Đối tượng `Job` của coroutine là một phần trong ngữ cảnh của nó, và có thể được truy xuất thông qua biểu thức **`coroutineContext[Job]`**:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    println("My job is ${coroutineContext[Job]}")
//sampleEnd    
}
```

**Kết quả Console ở chế độ Debug:**
```text
My job is "coroutine#1":BlockingCoroutine{Active}@6d311334
```

> [!TIP]
> Thuộc tính `isActive` trong `CoroutineScope` thực chất là một cách viết ngắn gọn tiện lợi cho biểu thức: `coroutineContext[Job]?.isActive == true`.

---

## 6. Các Coroutine Con (Children of a Coroutine)

Khi một coroutine được khởi chạy bên trong `CoroutineScope` của một coroutine khác, nó sẽ thừa kế ngữ cảnh thông qua `CoroutineScope.coroutineContext`, và `Job` của coroutine mới sẽ trở thành **con (child)** của Job cha. Khi coroutine cha bị hủy, tất cả các coroutine con của nó sẽ bị hủy đệ quy.

Tuy nhiên, mối quan hệ cha - con này có thể bị ghi đè một cách tường minh theo 2 cách:
1. Khi một **scope khác** được chỉ định rõ ràng khi khởi chạy (ví dụ: `GlobalScope.launch`), nó không thừa kế `Job` từ scope cha.
2. Khi một **đối tượng `Job` khác** được truyền vào làm context cho coroutine mới (như ví dụ bên dưới), nó sẽ ghi đè Job của scope cha:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    // Khởi chạy một coroutine cha để xử lý yêu cầu
    val request = launch {
        // Sinh ra job1 với một Job() độc lập mới
        launch(Job()) { 
            println("job1: I run in my own Job and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // job2 thừa kế context của cha
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
    }
    delay(500)
    request.cancel() // Hủy yêu cầu cha
    println("main: Who has survived request cancellation?")
    delay(1000) // Trì hoãn main thread 1 giây để theo dõi kết quả
//sampleEnd
}
```

**Kết quả in ra Console:**
```text
job1: I run in my own Job and execute independently!
job2: I am a child of the request coroutine
main: Who has survived request cancellation?
job1: I am not affected by cancellation of the request
```

Trong cả hai trường hợp ghi đè, coroutine được tạo không còn bị ràng buộc vào scope ban đầu và hoạt động hoàn toàn độc lập.

---

## 7. Trách nhiệm của Coroutine Cha (Parental Responsibilities)

Một coroutine cha **luôn luôn kiên nhẫn chờ đợi toàn bộ các coroutine con của nó hoàn thành**. Coroutine cha không cần phải theo dõi danh sách các con hoặc gọi `Job.join()` trên từng con một:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    val request = launch {
        repeat(3) { i -> // Khởi tạo 3 coroutine con
            launch {
                delay((i + 1) * 200L) // Delay 200ms, 400ms, 600ms
                println("Coroutine $i is done")
            }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    request.join() // Chờ toàn bộ request và các con của nó hoàn tất
    println("Now processing of the request is complete")
//sampleEnd
}
```

**Kết quả in ra Console:**
```text
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Now processing of the request is complete
```

---

## 8. Đặt Tên Coroutine để Gỡ lỗi (Naming Coroutines for Debugging)

Khi một coroutine gắn liền với việc xử lý một yêu cầu cụ thể, việc đặt tên rõ ràng cho nó sẽ giúp ích rất nhiều cho việc gỡ lỗi. Phần tử ngữ cảnh [`CoroutineName`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-name/index.html) phục vụ mục đích tương tự như tên thread:

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking(CoroutineName("main")) {
//sampleStart
    log("Started main coroutine")
    // Chạy hai phép tính toán nền
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        6
    }
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        7
    }
    log("The answer for v1 * v2 = ${v1.await() * v2.await()}")
//sampleEnd    
}
```

**Kết quả in ra Console:**
```text
[main @main#1] Started main coroutine
[main @v1coroutine#2] Computing v1
[main @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 * v2 = 42
```

---

## 9. Kết hợp các Phần tử Ngữ cảnh với Toán tử `+`

Đôi khi chúng ta cần định nghĩa nhiều phần tử cho cùng một ngữ cảnh coroutine. Bạn có thể sử dụng toán tử **`+`** để gộp chúng:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch(Dispatchers.Default + CoroutineName("test")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

**Kết quả in ra Console với `-Dkotlinx.coroutines.debug`:**
```text
I'm working in thread DefaultDispatcher-worker-1 @test#2
```

---

## 10. Quản lý Vòng đời với `CoroutineScope`

Hãy tổng hợp kiến thức về context, children và jobs. Giả sử ứng dụng của chúng ta có một đối tượng có vòng đời (như một `Activity` trên Android), nhưng bản thân đối tượng đó không phải là một coroutine. Chúng ta khởi chạy nhiều coroutine để tải dữ liệu, chạy hoạt họa, v.v. Toàn bộ coroutine này phải bị hủy khi Activity bị tiêu hủy để tránh rò rỉ bộ nhớ.

Thư viện `kotlinx.coroutines` đóng gói cơ chế này thông qua interface **`CoroutineScope`**:
- `CoroutineScope()`: Hàm factory tạo một scope mục đích tổng quát.
- `MainScope()`: Hàm factory tạo scope cho ứng dụng UI sử dụng `Dispatchers.Main` làm dispatcher mặc định.

```kotlin
import kotlinx.coroutines.*

class Activity {
    private val mainScope = CoroutineScope(Dispatchers.Default) // Sử dụng Default cho mục đích test
    
    fun destroy() {
        mainScope.cancel() // Hủy toàn bộ coroutines
    }

    fun doSomething() {
        // Khởi chạy 10 coroutines chạy với thời gian khác nhau
        repeat(10) { i ->
            mainScope.launch {
                delay((i + 1) * 200L)
                println("Coroutine $i is done")
            }
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val activity = Activity()
    activity.doSomething()
    println("Launched coroutines")
    delay(500L)
    println("Destroying activity!")
    activity.destroy() // Hủy toàn bộ coroutine trong scope
    delay(1000) // Chờ để xác nhận trực quan rằng các coroutine sau đã dừng
//sampleEnd    
}
```

**Kết quả in ra Console:**
```text
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying activity!
```

Chỉ có 2 coroutine đầu tiên kịp in kết quả. Toàn bộ các coroutine còn lại đã bị hủy an toàn bằng một lời gọi duy nhất `mainScope.cancel()` trong `Activity.destroy()`.

---

## 11. Dữ liệu Thread-local trong Coroutines (`asContextElement`)

Đối với biến [`ThreadLocal`](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html), hàm mở rộng **`asContextElement`** giữ giá trị của `ThreadLocal` tương ứng và khôi phục nó mỗi khi coroutine chuyển đổi context:

```kotlin
import kotlinx.coroutines.*

val threadLocal = ThreadLocal<String?>() // Khai báo biến thread-local

fun main() = runBlocking<Unit> {
//sampleStart
    threadLocal.set("main")
    println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
        println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
        yield()
        println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    }
    job.join()
    println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
//sampleEnd    
}
```

**Kết quả Console:**
```text
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'launch'
After yield, current thread: Thread[DefaultDispatcher-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

> [!IMPORTANT]
> **Hạn chế then chốt của `ThreadLocal`**: Khi một biến thread-local bị thay đổi giá trị (`mutate`) bên trong coroutine, giá trị mới sẽ không được tự động lan truyền ngược lại bên ngoài và có thể bị mất ở điểm suspension tiếp theo. Hãy sử dụng `withContext` để cập nhật giá trị của thread-local, hoặc sử dụng `ThreadContextElement` nếu tích hợp sâu với các thư viện như logging MDC hoặc transaction context.

---

## 📝 Bảng Thuật ngữ & Tóm tắt nhanh

| Thuật ngữ | Khái niệm kỹ thuật tương ứng |
| :--- | :--- |
| **`CoroutineContext`** | Tập hợp các phần tử cấu hình môi trường thực thi của coroutine. |
| **`CoroutineDispatcher`** | Phân phối coroutine tới thread pool tương ứng (`Default`, `IO`, `Main`, `Unconfined`). |
| **`newSingleThreadContext`** | Tạo thread riêng, tài nguyên đắt đỏ cần giải phóng bằng `.close()` hoặc `.use`. |
| **`withContext`** | Chuyển đổi ngữ cảnh và dispatcher an toàn và tự động quay về sau khi hoàn thành. |
| **Parent-Child Hierarchy** | Mối quan hệ phân cấp: Cha hủy thì con hủy; Cha luôn chờ toàn bộ con hoàn tất. |
| **`asContextElement()`** | Duy trì tính nhất quán của dữ liệu `ThreadLocal` khi coroutine chuyển đổi qua lại giữa các thread. |

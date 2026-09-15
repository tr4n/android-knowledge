# Bài 03 — Kết hợp các Hàm Suspend (Composing Suspending Functions)

> **Tài liệu gốc:** [Composing suspending functions — Kotlin Documentation](https://kotlinlang.org/docs/composing-suspending-functions.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Nắm vững hành vi thực thi tuần tự mặc định của coroutine, tính toán đồng thời song song với `async`, chế độ khởi chạy lười (`CoroutineStart.LAZY`), lý do JetBrains khuyến cáo bài trừ phong cách "Async-style functions", và cách áp dụng Structured Concurrency với `async` khi có ngoại lệ phát sinh.

---

## 1. Tuần tự theo Mặc định (Sequential by Default)

Giả sử chúng ta có hai hàm `suspend` độc lập được định nghĩa ở một nơi nào đó thực hiện các tác vụ hữu ích (ví dụ như gọi dịch vụ mạng từ xa hoặc tính toán nặng). Ở đây, chúng ta giả lập mỗi hàm mất 1 giây để hoàn thành:

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // Giả lập đang làm việc gì đó hữu ích
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // Giả lập đang làm việc gì đó hữu ích
    return 29
}
```

Chúng ta sẽ làm gì nếu cần gọi chúng **tuần tự (sequentially)** — đầu tiên gọi `doSomethingUsefulOne`, **sau đó** gọi `doSomethingUsefulTwo`, và tính tổng kết quả của chúng? Trong thực tế, chúng ta làm điều này nếu kết quả của hàm thứ nhất quyết định xem có cần gọi hàm thứ hai hay không, hoặc quyết định cách thức gọi hàm thứ hai.

Chúng ta chỉ cần sử dụng lời gọi tuần tự thông thường, bởi vì mã nguồn bên trong một coroutine, tương tự như mã thông thường, **luôn tuần tự theo mặc định**:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

**Kết quả in ra Console:**
```text
The answer is 42
Completed in 2017 ms
```

---

## 2. Tính toán Đồng thời với `async` (Concurrent Using `async`)

Điều gì xảy ra nếu **hoàn toàn không có sự phụ thuộc** giữa các lần gọi `doSomethingUsefulOne` và `doSomethingUsefulTwo`, và chúng ta muốn nhận kết quả nhanh hơn bằng cách thực hiện cả hai **đồng thời (concurrently)**? Đây chính là lúc **`async`** trợ giúp.

- **Về mặt ý niệm**: `async` tương tự như `launch`. Nó khởi động một coroutine riêng biệt — một tiến trình siêu nhẹ hoạt động đồng thời với tất cả các coroutine khác.
- **Điểm khác biệt**:
  - `launch` trả về một [`Job`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) và **không mang theo giá trị kết quả nào**.
  - `async` trả về một [`Deferred<T>`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html) — một non-blocking future siêu nhẹ đại diện cho một lời hứa sẽ cung cấp kết quả sau này.
- Bạn sử dụng phương thức **`.await()`** trên giá trị deferred để nhận kết quả cuối cùng. Đồng thời, `Deferred` cũng là một `Job`, vì vậy bạn có thể hủy (`cancel`) nó nếu cần thiết.

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

**Kết quả in ra Console:**
```text
The answer is 42
Completed in 1017 ms
```

> [!NOTE]
> Đoạn mã trên chạy nhanh gấp đôi (khoảng 1000 ms so với 2000 ms ban đầu) vì hai coroutine được thực thi đồng thời. Hãy lưu ý rằng tính đồng thời với coroutines luôn mang tính tường minh (explicit).

---

## 3. Khởi chạy Lười với `async` (Lazily Started Async)

Tùy chọn khác, bạn có thể biến `async` thành phép tính lười (lazy) bằng cách thiết lập tham số `start` thành [`CoroutineStart.LAZY`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-l-a-z-y/index.html).

Ở chế độ này, coroutine **chỉ bắt đầu chạy khi kết quả của nó thực sự được yêu cầu** bởi phương thức [`await`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html), hoặc nếu hàm [`start`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/start.html) của `Job` được gọi:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // Có thể thực hiện một số tính toán khác tại đây...
        one.start() // Kích hoạt coroutine thứ nhất
        two.start() // Kích hoạt coroutine thứ hai
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

**Kết quả in ra Console:**
```text
The answer is 42
Completed in 1017 ms
```

### Phân tích quan trọng về tính tuần tự ngoài ý muốn:
Ở ví dụ trên, hai coroutine được khai báo nhưng chưa chạy ngay. Quyền kiểm soát thời điểm bắt đầu được trao cho lập trình viên bằng cách gọi `start()`. Chúng ta kích hoạt `one.start()`, rồi kích hoạt `two.start()`, sau đó gọi `await()` chờ cả hai hoàn thành.

> [!WARNING]
> Lưu ý rằng nếu chúng ta **chỉ gọi `await()` trong lệnh `println` mà không gọi `start()` trước đó** trên từng coroutine riêng lẻ, mã nguồn sẽ quay trở về **hành vi tuần tự (mất 2000 ms)**! 
> Lý do: `one.await()` sẽ kích hoạt `one` và tạm dừng chờ `one` hoàn tất trước khi dòng code đi tiếp đến `two.await()`. Đây không phải là mục đích của lazy. Mục đích của `async(start = CoroutineStart.LAZY)` là đóng vai trò thay thế cho hàm `lazy` chuẩn của Kotlin khi việc khởi tạo giá trị có liên quan đến các hàm suspend.

---

## 4. Cảnh báo: Các Hàm theo Phong cách Async (Async-style Functions)

> [!NOTE]
> Phong cách lập trình với các hàm async được trình bày ở đây **chỉ nhằm mục đích minh họa**, vì đây là phong cách phổ biến trong các ngôn ngữ lập trình khác. Việc sử dụng phong cách này với Kotlin coroutines bị **khuyến cáo bài trừ mạnh mẽ (STRONGLY DISCOURAGED)** vì những lý do được giải thích dưới đây.

Chúng ta có thể định nghĩa các hàm phong cách async gọi `doSomethingUsefulOne` và `doSomethingUsefulTwo` một cách bất đồng bộ bằng builder `async` kết hợp với `GlobalScope` để thoát khỏi Structured Concurrency:

```kotlin
// Kiểu trả về của somethingUsefulOneAsync là Deferred<Int>
@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// Kiểu trả về của somethingUsefulTwoAsync là Deferred<Int>
@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

Lưu ý rằng các hàm `xxxAsync` này **KHÔNG PHẢI là hàm `suspend`**. Chúng có thể được gọi từ bất kỳ đâu:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

// Lưu ý: Chúng ta không có `runBlocking` ở bên phải của `main`
fun main() {
    val time = measureTimeMillis {
        // Chúng ta có thể khởi tạo các tác vụ async bên ngoài coroutine
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // Nhưng việc chờ kết quả phải thông qua suspend hoặc blocking.
        // Ở đây ta dùng runBlocking để khóa main thread trong khi chờ
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}

@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}
```

### Tại sao phong cách này bị bài trừ?
Hãy cân nhắc điều gì sẽ xảy ra nếu giữa dòng `val one = somethingUsefulOneAsync()` và biểu thức `one.await()` xuất hiện một lỗi logic trong mã nguồn, khiến chương trình ném ra ngoại lệ và thao tác bị hủy bỏ:
- Thông thường, một trình xử lý lỗi toàn cục có thể bắt ngoại lệ này, ghi log báo cáo lỗi cho lập trình viên, và chương trình vẫn có thể tiếp tục thực hiện các tác vụ khác.
- **TUY NHIÊN**: Tác vụ `somethingUsefulOneAsync` **vẫn âm thầm tiếp tục chạy ngầm trong nền**, mặc dù thao tác khởi tạo ra nó đã bị hủy bỏ! Điều này gây lãng phí bộ nhớ, CPU và rò rỉ tác vụ nền. Vấn đề này hoàn toàn không xảy ra nếu sử dụng Structured Concurrency.

---

## 5. Structured Concurrency với `async`

Hãy tái cấu trúc ví dụ trên thành một hàm chạy `doSomethingUsefulOne` và `doSomethingUsefulTwo` đồng thời và trả về tổng kết quả của chúng. Vì `async` là một extension function trên `CoroutineScope`, chúng ta sử dụng hàm **`coroutineScope`** để cung cấp phạm vi cần thiết:

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```

Bằng cách này, nếu có sự cố xảy ra bên trong thân hàm `concurrentSum` và nó ném ra ngoại lệ, **tất cả các coroutine đã được khởi chạy trong scope của nó sẽ tự động bị hủy**.

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}
```

**Kết quả Console:**
```text
The answer is 42
Completed in 1017 ms
```

### Lan truyền Hủy bỏ khi Xảy ra Thất bại:
Sự hủy bỏ luôn được lan truyền qua cây phân cấp coroutine:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // Giả lập tác vụ tính toán rất lâu
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```

**Kết quả in ra Console:**
```text
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

> [!NOTE]
> Hãy quan sát cách mà cả coroutine `async` thứ nhất lẫn coroutine cha đang chờ đợi đều bị hủy ngay khi một trong các coroutine con (ở đây là `two`) thất bại. Không có bất kỳ tác vụ nào bị rò rỉ ngầm!

---

## 📝 Bảng Thuật ngữ & Tóm tắt nhanh

| Thuật ngữ | Khái niệm kỹ thuật tương ứng |
| :--- | :--- |
| **Sequential by default** | Mặc định các hàm suspend và dòng mã trong coroutine chạy tuần tự theo thứ tự. |
| **`async { }`** | Coroutine builder trả về `Deferred<T>`, phục vụ tính toán đồng thời song song. |
| **`Deferred<T>`** | Đại diện cho lời hứa kết quả kiểu `T` trong tương lai, lấy kết quả bằng `.await()`. |
| **`CoroutineStart.LAZY`** | Chế độ khởi chạy trễ: Chỉ kích hoạt khi gọi `.start()` hoặc `.await()`. |
| **Async-style functions** | Phong cách hàm trả về `Deferred` dùng `GlobalScope` bị phản đối vì làm mất tính an toàn của Structured Concurrency. |
| **Structured Concurrency** | Cơ chế tự động hủy toàn bộ các nhánh con song song khi một nhánh con gặp ngoại lệ. |

# Bài 09 — Trạng thái Khả biến Dùng chung & Đồng thời (Shared Mutable State and Concurrency)

> **Tài liệu gốc:** [Shared mutable state and concurrency — Kotlin Documentation](https://kotlinlang.org/docs/shared-mutable-state-and-concurrency.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Hiểu rõ bản chất xung đột dữ liệu (Race Conditions) khi thực thi coroutine song song trên bộ điều phối đa luồng; tại sao chú thích `@Volatile` không giải quyết được vấn đề; cách áp dụng cấu trúc dữ liệu nguyên tử (Atomic Data Structures); kỹ thuật giam luồng (Thread Confinement) hạt mịn và hạt thô; và giải pháp khóa loại trừ tương hỗ phi ngăn chặn với `Mutex.withLock`.

---

## 1. Vấn đề Xung đột Trạng thái (The Problem)

Các coroutine có thể được thực thi song song hoàn toàn bằng cách sử dụng các bộ điều phối đa luồng (multi-threaded dispatchers) như `Dispatchers.Default`. Điều này kéo theo tất cả các vấn đề kinh điển của tính song song (parallelism), trong đó bài toán lớn nhất là **đồng bộ hóa quyền truy cập vào trạng thái khả biến dùng chung (shared mutable state)**.

Một số giải pháp trong thế giới coroutine tương tự như các giải pháp truyền thống của đa luồng Java, nhưng một số giải pháp khác lại mang tính đặc thù riêng biệt.

Hãy xem xét một hàm hỗ trợ `massiveRun` khởi chạy 100 coroutine, mỗi coroutine lặp lại một hành động 1.000 lần (tổng cộng 100.000 lần thực thi), đồng thời đo tổng thời gian chạy:

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // Số lượng coroutine cần khởi chạy
    val k = 1000 // Số lần mỗi coroutine lặp lại hành động
    val time = measureTimeMillis {
        coroutineScope { // Phạm vi quản lý các coroutine con
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

Bắt đầu với một biến đếm `counter` thông thường và tăng giá trị đồng thời bằng `Dispatchers.Default`:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*    

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope { 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

**Kết quả Console Output mẫu:**
```none
Completed 100000 actions in 27 ms
Counter = 72348
```

> [!CAUTION]
> Biến `counter` hầu như không bao giờ đạt giá trị `100000`, bởi vì 100 coroutine đồng thời thực hiện phép tăng `counter++` từ nhiều thread khác nhau của thread pool mà không có bất kỳ cơ chế đồng bộ hóa nào, gây ra hiện tượng **mất mát dữ liệu (lost updates / race condition)**.

---

## 2. `@Volatile` Không Giúp Ích Gì Trong Tình Huống Này (Volatiles Are of No Help)

Có một quan niệm sai lầm rất phổ biến rằng việc gắn chú thích `@Volatile` vào biến sẽ giải quyết được vấn đề tranh chấp dữ liệu. Hãy thử kiểm chứng:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope { 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

@Volatile // Trong Kotlin, `volatile` là một annotation
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

**Kết quả Console Output mẫu:**
```none
Completed 100000 actions in 35 ms
Counter = 81254
```

> [!NOTE]
> Đoạn mã này chạy chậm hơn nhưng kết quả cuối cùng vẫn **sai lệch và không đạt được 100000**.  
> **Lý do kỹ thuật**: `@Volatile` chỉ đảm bảo tính tuyến tính (linearizable / atomic) cho các thao tác **đọc đơn lẻ** và **ghi đơn lẻ** trên biến bộ nhớ (xóa cache CPU, ép đọc/ghi từ RAM chính). Tuy nhiên, phép toán `counter++` không phải là một thao tác đơn lẻ, mà là một chuỗi hành động bao gồm 3 bước:
> 1. Đọc giá trị hiện tại của `counter` vào thanh ghi CPU.
> 2. Tăng giá trị thanh ghi lên 1.
> 3. Ghi giá trị mới ngược trở lại biến.  
> `@Volatile` hoàn toàn bất lực trong việc đảm bảo tính nguyên tử (atomicity) cho cả cụm hành động tổng hợp này.

---

## 3. Cấu trúc Dữ liệu An toàn Đa luồng (Thread-safe Data Structures)

Giải pháp tổng quát hoạt động hiệu quả cho cả thread truyền thống lẫn coroutine là sử dụng một cấu trúc dữ liệu an toàn đa luồng (thread-safe, synchronized, hoặc atomic). 

Đối với bài toán biến đếm đơn giản, chúng ta có thể sử dụng lớp **`AtomicInteger`** với hàm nguyên tử `incrementAndGet()` (sử dụng lệnh phần cứng CPU Compare-And-Swap CAS):

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.atomic.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope { 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val counter = AtomicInteger()

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
```

**Kết quả Console Output:**
```none
Completed 100000 actions in 18 ms
Counter = 100000
```

> [!TIP]
> Đây là giải pháp **có tốc độ thực thi nhanh nhất** cho bài toán biến đếm cụ thể này. Nó hoạt động tốt cho các biến đếm, bộ sưu tập (concurrent collections), hàng đợi và các cấu trúc dữ liệu chuẩn. Tuy nhiên, nó khó mở rộng (scale) cho các đối tượng trạng thái phức tạp hoặc các thao tác nghiệp vụ phức tạp chưa có sẵn cài đặt thread-safe.

---

## 4. Giam Luồng Hạt Mịn (Thread Confinement Fine-grained)

**Giam luồng (Thread confinement)** là phương pháp tiếp cận trong đó **mọi quyền truy cập vào một trạng thái dùng chung nhất định đều bị giới hạn (confined) nghiêm ngặt trong một thread duy nhất**.

Kỹ thuật này được áp dụng rộng rãi trong các ứng dụng giao diện (UI) như Android/Swing, nơi toàn bộ trạng thái UI chỉ được phép truy cập và sửa đổi trên duy nhất Main/UI Thread. Với Coroutines, ta có thể dễ dàng áp dụng bằng cách chuyển sang một ngữ cảnh đơn luồng (single-threaded context).

Trong ví dụ dưới đây, ta bọc riêng từng thao tác `counter++` vào `withContext(counterContext)`:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope { 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

@OptIn(DelicateCoroutinesApi::class)
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // Giam từng thao tác tăng biến vào ngữ cảnh đơn luồng
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

**Kết quả Console Output mẫu:**
```none
Completed 100000 actions in 1024 ms
Counter = 100000
```

> [!WARNING]
> Mặc dù kết quả in ra chính xác là `100000`, đoạn mã này chạy **cực kỳ chậm chạp** (mất hơn 1000 ms so với 18 ms của `AtomicInteger`)!  
> Lý do là vì nó thực hiện giam luồng ở mức **hạt mịn (fine-grained)**: Mỗi một lần tăng trong số 100.000 lần, coroutine đều phải thực hiện thao tác chuyển luồng (thread context switch) từ `Dispatchers.Default` sang `CounterContext` và ngược lại. Chi phí chuyển ngữ cảnh liên tục này làm hiệu năng sụt giảm nghiêm trọng.

---

## 5. Giam Luồng Hạt Thô (Thread Confinement Coarse-grained)

Trong thực tế, kỹ thuật giam luồng được áp dụng trên các **khối xử lý lớn (large chunks)**, ví dụ giam toàn bộ luồng nghiệp vụ cập nhật dữ liệu vào một thread duy nhất thay vì chuyển đổi qua lại ở từng câu lệnh con.

Dưới đây là phiên bản giam luồng ở mức **hạt thô (coarse-grained)**: Toàn bộ quá trình chạy `massiveRun` được đặt trực tiếp bên trong `withContext(counterContext)`:

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope { 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

@OptIn(DelicateCoroutinesApi::class)
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    // Giam toàn bộ khối thực thi lớn vào ngữ cảnh đơn luồng ngay từ đầu
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

**Kết quả Console Output:**
```none
Completed 100000 actions in 26 ms
Counter = 100000
```

> [!NOTE]
> Đoạn mã giờ đây chạy **nhanh hơn rất nhiều** (chỉ mất ~26 ms) và cho ra kết quả hoàn toàn chính xác, bởi vì không còn chi phí chuyển đổi ngữ cảnh thread giữa các lần tăng biến đếm.

---

## 6. Khóa Loại trừ Tương hỗ: `Mutex` (Mutual Exclusion)

Giải pháp loại trừ tương hỗ là bảo vệ tất cả các thao tác sửa đổi trạng thái dùng chung bằng một **vùng tranh chấp trọng yếu (critical section)** sao cho không bao giờ có hai tác vụ nào được thực thi vùng này đồng thời.

Trong thế giới đa luồng truyền thống của Java, bạn thường dùng từ khóa `synchronized` hoặc lớp `ReentrantLock`. Nhưng cả hai cơ chế này đều **gây block (chặn) thread**, đi ngược lại triết lý non-blocking của Coroutines.

Giải pháp thay thế chuẩn mực trong Kotlin Coroutines là **`Mutex`**. 
- `Mutex` có hai hàm cơ bản là `lock()` và `unlock()` để phân định ranh giới của critical section.
- **Điểm khác biệt cốt lõi:** **`Mutex.lock()` là một suspending function**, nó tạm ngưng coroutine mà **KHÔNG hề làm block thread** đang chạy! Thread được giải phóng để làm công việc khác trong lúc chờ ổ khóa mở ra.

Để an toàn và thuận tiện, bạn nên sử dụng hàm mở rộng **`withLock`**, tương đương với mẫu thiết kế `mutex.lock(); try { ... } finally { mutex.unlock() }`:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope { 
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // Bảo vệ từng lần tăng bằng khóa phi ngăn chặn withLock
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

**Kết quả Console Output:**
```none
Completed 100000 actions in 280 ms
Counter = 100000
```

> [!NOTE]
> Việc đặt khóa trong ví dụ trên là ở mức hạt mịn nên vẫn phải trả giá về mặt hiệu năng (~280 ms), nhưng `Mutex` là sự lựa chọn tuyệt vời cho các tình huống bạn bắt buộc phải sửa đổi trạng thái dùng chung định kỳ mà trạng thái đó không thể giam cố định vào một thread tự nhiên nào.

---

## 7. Bảng Ma trận Quyết định Chiến lược Đồng bộ Hóa

| Phương pháp | Hiệu năng | Độ phức tạp mã nguồn | Khi nào nên áp dụng? |
| :--- | :--- | :--- | :--- |
| **`@Volatile`** | Nhanh | Rất thấp | **KHÔNG DÙNG** cho các thao tác kép (`++`, check-then-act). Chỉ dùng làm cờ nhị phân một chiều (`flag = true`). |
| **Atomic Classes (`AtomicInteger`, CAS)** | Rất cao | Thấp | Dành cho các biến đếm đơn, cờ trạng thái, hoặc tham chiếu đối tượng đơn lẻ (`AtomicReference`). |
| **Thread Confinement (Hạt thô)** | Cao | Trung bình | Kiến trúc UI Android (MVI/MVVM đẩy hết logic cập nhật UI State lên Main Thread); Actor Model. |
| **`Mutex.withLock`** | Trung bình | Thấp | Bảo vệ vùng tài nguyên nhạy cảm (ghi file, cập nhật cache in-memory, gọi API token refresh) giữa các coroutine bất đồng bộ. |

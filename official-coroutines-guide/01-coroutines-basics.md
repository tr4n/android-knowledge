# Bài 01 — Cơ bản về Coroutines (Coroutines Basics)

> **Tài liệu gốc:** [Coroutines basics — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-basics.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Nắm vững khái niệm coroutine, hàm tạm dừng (`suspend`), quy trình từng bước khởi tạo coroutine, nguyên lý Structured Concurrency, các coroutine builder (`launch`, `async`, `runBlocking`), bộ điều phối Dispatchers, và so sánh chi tiết bộ nhớ giữa Coroutines và JVM Threads.

---

## 1. Khái niệm Coroutine là gì?

Để xây dựng các ứng dụng có thể thực thi nhiều tác vụ cùng một lúc — một khái niệm được gọi là **tính đồng thời (concurrency)** — Kotlin sử dụng **coroutines**.

- Một **coroutine** là một phép tính có thể tạm dừng (**suspendable computation**), cho phép bạn viết mã xử lý đồng thời theo phong cách tuần tự, rõ ràng và tuyến tính.
- Coroutine có thể chạy đồng thời với các coroutines khác và có khả năng chạy song song (**parallel**) trên các lõi CPU khác nhau.
- Trên máy ảo JVM và trong Kotlin/Native, toàn bộ mã đồng thời (bao gồm cả coroutines) đều chạy trên các **threads** do hệ điều hành quản lý.
- **Điểm khác biệt cốt lõi**: Coroutine có thể **tạm dừng (suspend) việc thực thi thay vì khóa cứng (block) một thread**. Điều này cho phép một coroutine tạm dừng chờ dữ liệu tải về, trong khi một coroutine khác có thể tận dụng chính thread đó để tiếp tục chạy, đảm bảo hiệu suất sử dụng tài nguyên tối đa.

```
So sánh Mô hình Concurrency và Parallelism:
Thread 1: ───[Coroutine A]───► [Coroutine B (chạy trên Thread 1 khi A suspend)]───►
Thread 2: ───[Coroutine C]───► [Chạy song song thực sự trên lõi CPU khác]────────►
```

---

## 2. Hàm Tạm dừng (Suspending Functions)

Khối xây dựng nguyên tử cơ bản nhất của coroutines là **hàm tạm dừng (suspending function)**. Nó cho phép một thao tác đang chạy tạm dừng lại và tiếp tục sau đó mà không làm thay đổi cấu trúc mã nguồn tuần tự của bạn.

Để khai báo một hàm tạm dừng, bạn sử dụng từ khóa `suspend`:

```kotlin
suspend fun greet() {
    println("Hello world from a suspending function")
}
```

### Quy tắc triệu gọi:
Bạn chỉ có thể gọi một hàm `suspend` từ bên trong một hàm `suspend` khác. Để gọi các hàm tạm dừng tại điểm khởi đầu (entry point) của một ứng dụng Kotlin, hãy đánh dấu hàm `main()` với từ khóa `suspend`:

```kotlin
suspend fun main() {
    showUserInfo()
}

suspend fun showUserInfo() {
    println("Loading user...")
    greet()
    println("User: John Smith")
}

suspend fun greet() {
    println("Hello world from a suspending function")
}
```

**Kết quả in ra Console:**
```text
Loading user...
Hello world from a suspending function
User: John Smith
```

> [!NOTE]
> Ví dụ này chưa áp dụng tính đồng thời, nhưng bằng cách đánh dấu các hàm với từ khóa `suspend`, bạn cho phép chúng gọi các hàm suspend khác và khởi chạy mã đồng thời bên trong.
> Trong khi từ khóa `suspend` là một phần cốt lõi của ngôn ngữ Kotlin, hầu hết các tính năng coroutine nâng cao đều được cung cấp thông qua thư viện **`kotlinx.coroutines`**.

---

## 3. Thêm Thư viện `kotlinx.coroutines` vào Dự án

Tùy thuộc vào công cụ build của bạn, hãy khai báo dependency tương ứng:

```kotlin
// build.gradle.kts (Kotlin DSL)
repositories {
    mavenCentral()
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
}
```

```groovy
// build.gradle (Groovy DSL)
repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0'
}
```

```xml
<!-- pom.xml (Maven) -->
<dependencies>
    <dependency>
        <groupId>org.jetbrains.kotlinx</groupId>
        <artifactId>kotlinx-coroutines-core</artifactId>
        <version>1.11.0</version>
    </dependency>
</dependencies>
```

---

## 4. Tạo các Coroutine Đầu tiên (Create Your First Coroutines)

> [!NOTE]
> Các ví dụ trong phần này sử dụng biểu thức tường minh `this` với các hàm tạo coroutine `CoroutineScope.launch()` và `CoroutineScope.async()`. Các coroutine builder này là **hàm mở rộng (extension functions)** trên `CoroutineScope`, và biểu thức `this` trỏ trực tiếp đến `CoroutineScope` hiện tại với tư cách là receiver.

Để tạo một coroutine trong Kotlin, bạn cần 4 thành phần:
1. Một **hàm tạm dừng (suspending function)**.
2. Một **phạm vi coroutine (coroutine scope)** để nó chạy bên trong, ví dụ bên trong hàm `withContext()`.
3. Một **hàm tạo coroutine (coroutine builder)** như `CoroutineScope.launch()` để khởi chạy nó.
4. Một **bộ điều phối (dispatcher)** để điều khiển thread nào sẽ được sử dụng.

### Quy trình 6 bước xây dựng mã đồng thời đa luồng:

#### Bước 1: Import thư viện coroutines
```kotlin
import kotlinx.coroutines.*
```

#### Bước 2: Đánh dấu các hàm có thể dừng và tiếp tục bằng từ khóa `suspend`
```kotlin
suspend fun greet() {
    println("The greet() on the thread: ${Thread.currentThread().name}")
}

suspend fun main() {}
```

> [!NOTE]
> Mặc dù bạn có thể đánh dấu hàm `main()` là `suspend` trong hầu hết các dự án hiện đại, điều này có thể không khả thi khi tích hợp với mã cũ hoặc các framework chuyên biệt. Trong trường hợp đó, hãy kiểm tra tài liệu framework xem có hỗ trợ hàm suspend không; nếu không, hãy sử dụng **`runBlocking()`** để gọi chúng bằng cách khóa thread hiện tại.

#### Bước 3: Thêm hàm `delay()` để mô phỏng tác vụ tạm dừng
Hàm `delay()` mô phỏng một tác vụ tạm dừng như lấy dữ liệu từ mạng hoặc ghi vào cơ sở dữ liệu:
```kotlin
suspend fun greet() {
    println("The greet() on the thread: ${Thread.currentThread().name}")
    delay(1000L)
}
```

#### Bước 4: Sử dụng `withContext(Dispatchers.Default)` làm điểm vào đa luồng
```kotlin
suspend fun main() {
    withContext(Dispatchers.Default) {
        // Thêm các coroutine builders tại đây
    }
}
```

> [!NOTE]
> Hàm suspend `withContext()` thường được dùng để chuyển đổi ngữ cảnh (context switching), nhưng trong ví dụ này, nó cũng định nghĩa một điểm vào non-blocking cho mã đồng thời. Nó sử dụng bộ điều phối `Dispatchers.Default` để chạy mã trên một thread pool dùng chung. Mặc định, pool này sử dụng số lượng thread tương đương số lõi CPU có sẵn (tối thiểu là 2).
> Các coroutine khởi chạy trong khối `withContext()` chia sẻ cùng một coroutine scope, đảm bảo nguyên lý Structured Concurrency.

#### Bước 5: Sử dụng `CoroutineScope.launch()` để khởi động coroutine
```kotlin
suspend fun main() {
    withContext(Dispatchers.Default) { // this: CoroutineScope
        this.launch { greet() }
        println("The withContext() on the thread: ${Thread.currentThread().name}")
    }
}
```

#### Bước 6: Kết hợp toàn bộ các thành phần để chạy nhiều coroutines đồng thời
```kotlin
// Imports thư viện coroutines
import kotlinx.coroutines.*

// Imports kotlin.time.Duration để biểu thị thời gian bằng giây
import kotlin.time.Duration.Companion.seconds

// Định nghĩa một hàm suspend
suspend fun greet() {
    println("The greet() on the thread: ${Thread.currentThread().name}")
    // Tạm dừng 1 giây và giải phóng thread
    delay(1.seconds) 
    // Hàm delay() mô phỏng một API call tạm dừng (ví dụ: gọi API mạng)
}

suspend fun main() {
    // Chạy mã bên trong khối này trên thread pool dùng chung
    withContext(Dispatchers.Default) { // this: CoroutineScope
        this.launch() {
            greet()
        }

        // Khởi chạy một coroutine khác song song
        this.launch() {
            println("The CoroutineScope.launch() on the thread: ${Thread.currentThread().name}")
            delay(1.seconds)
        }

        println("The withContext() on the thread: ${Thread.currentThread().name}")
    }
}
```

**Kết quả Console mẫu:**
```text
The withContext() on the thread: DefaultDispatcher-worker-1
The CoroutineScope.launch() on the thread: DefaultDispatcher-worker-3
The greet() on the thread: DefaultDispatcher-worker-2
```

Hãy thử chạy ví dụ nhiều lần: Bạn sẽ thấy thứ tự in ra và tên của các thread có thể thay đổi giữa các lần chạy, bởi vì hệ điều hành là bên quyết định thời điểm các thread được cấp phát CPU.

> [!TIP]
> Bạn có thể hiển thị tên định danh của coroutine bên cạnh tên thread trong log bằng cách thêm tùy chọn máy ảo JVM: **`-Dkotlinx.coroutines.debug`**.

---

## 5. Coroutine Scope và Tính Đồng thời có Cấu trúc (Structured Concurrency)

Khi bạn chạy nhiều coroutine trong một ứng dụng, bạn cần một phương thức để quản lý chúng theo nhóm. Kotlin coroutines dựa trên nguyên lý **Structured Concurrency (Tính đồng thời có cấu trúc)**.

- Theo nguyên lý này, các coroutine tạo thành một **cây phân cấp (tree hierarchy)** giữa các tác vụ cha và con với các vòng đời được liên kết chặt chẽ.
- **Vòng đời của coroutine**: Chuỗi các trạng thái từ khi khởi tạo cho đến khi hoàn thành, thất bại hoặc bị hủy.
- **Nguyên tắc cha - con**:
  - Coroutine cha luôn kiên nhẫn **chờ đợi tất cả các coroutine con của nó hoàn tất** trước khi bản thân nó kết thúc.
  - Nếu coroutine cha bị lỗi hoặc bị hủy, **tất cả các coroutine con của nó sẽ tự động bị hủy đệ quy**.
  - Việc duy trì mối liên kết có cấu trúc này giúp cơ chế xử lý lỗi và hủy bỏ trở nên an toàn và có thể dự đoán được.

Để đảm bảo Structured Concurrency, các coroutine mới chỉ có thể được tạo ra bên trong một [`CoroutineScope`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/).

---

### 5.1 Tạo Coroutine Scope với hàm `coroutineScope()`

Để tạo một coroutine scope mới kế thừa ngữ cảnh hiện tại, hãy sử dụng hàm **`coroutineScope()`**. Hàm này tạo ra một coroutine gốc (root) cho cây phân cấp coroutine con:

```kotlin
import kotlin.time.Duration.Companion.seconds
import kotlinx.coroutines.*

suspend fun main() {
    // Root của cây phân cấp coroutine
    coroutineScope { // this: CoroutineScope
        this.launch {
            this.launch {
                delay(2.seconds)
                println("Child of the enclosing coroutine completed")
            }
            println("Child coroutine 1 completed")
        }
        this.launch {
            delay(1.seconds)
            println("Child coroutine 2 completed")
        }
    }
    // Dòng này CHỈ chạy sau khi TẤT CẢ các con trong coroutineScope đã hoàn thành
    println("Coroutine scope completed")
}
```

**Kết quả Console:**
```text
Child coroutine 1 completed
Child coroutine 2 completed
Child of the enclosing coroutine completed
Coroutine scope completed
```

*Phân tích*: Vì không chỉ định dispatcher nào, các coroutine builder `launch` trong khối `coroutineScope()` sẽ kế thừa ngữ cảnh hiện tại. Nếu context hiện tại chưa có dispatcher, `launch` sẽ mặc định sử dụng `Dispatchers.Default`.

---

### 5.2 Tách Coroutine Builders ra khỏi Scope (Extraction)

Trong thực tế, bạn thường muốn tách các đoạn mã gọi builder (như `launch`) thành các hàm riêng:

```kotlin
suspend fun main() {
    coroutineScope { // this: CoroutineScope
        this.launch { println("1") }
        this.launch { println("2") }
    } 
}
```

> [!TIP]
> Bạn hoàn toàn có thể viết ngắn gọn `launch` thay vì viết tường minh `this.launch`. Các ví dụ sử dụng `this.launch` để nhấn mạnh rằng `launch` là một hàm mở rộng trên `CoroutineScope`.

Để tách logic gọi `launch` ra một hàm riêng, hàm đó **bắt buộc phải khai báo `CoroutineScope` làm receiver**:

```kotlin
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        launchAll()
    }
}

fun CoroutineScope.launchAll() { // this: CoroutineScope
    // Gọi .launch() trực tiếp trên CoroutineScope
    this.launch { println("1") }
    this.launch { println("2") } 
}
```

Nếu bạn cố tình gọi `launch` trong một hàm thông thường mà không khai báo receiver là `CoroutineScope`, trình biên dịch sẽ báo lỗi:
```kotlin
/*
fun launchAll() {
    // Lỗi biên dịch: 'this' không được định nghĩa, Unresolved reference: launch
    this.launch { println("1") }
    this.launch { println("2") }
}
*/
```

> [!IMPORTANT]
> **Tại sao hàm `launchAll()` không cần từ khóa `suspend`?**  
> Bởi vì `launchAll()` chỉ đơn thuần khởi động các coroutine bên trong `CoroutineScope` hiện tại và **trả về ngay lập tức**. Nó không hề tạm dừng (pause) luồng thực thi. Chỉ nên đánh dấu một hàm với từ khóa `suspend` khi bên trong nó thực sự có các thao tác tạm dừng và tiếp tục trước khi trả về kết quả!

---

## 6. Các Hàm Khởi tạo Coroutine (Coroutine Builder Functions)

Một hàm tạo coroutine (coroutine builder function) là hàm nhận vào một lambda `suspend` định nghĩa khối mã cần chạy.

Mỗi builder yêu cầu một `CoroutineScope` để hoạt động (có thể là scope sẵn có hoặc scope được tạo bởi các hàm trợ giúp như `coroutineScope()`, `runBlocking()`, hoặc `withContext()`).

### 6.1 `CoroutineScope.launch()`
- **Đặc tính**: Khởi chạy một coroutine mới bên trong scope mà không làm khóa phần còn lại của scope.
- **Mục đích**: Chạy một tác vụ song song khi bạn **không cần lấy giá trị kết quả** hoặc không muốn chờ đợi nó.

```kotlin
import kotlin.time.Duration.Companion.milliseconds
import kotlinx.coroutines.*

suspend fun main() {
    withContext(Dispatchers.Default) {
        performBackgroundWork()
    }
}

suspend fun performBackgroundWork() = coroutineScope { // this: CoroutineScope
    // Khởi chạy coroutine chạy nền không khóa scope
    this.launch {
        delay(100.milliseconds)
        println("Sending notification in background")
    }

    // Coroutine chính tiếp tục chạy ngay trong khi coroutine trên đang delay
    println("Scope continues")
}
```

**Kết quả Console:**
```text
Scope continues
Sending notification in background
```

> [!TIP]
> Hàm `CoroutineScope.launch()` trả về một đối tượng đại diện [`Job`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/). Bạn có thể sử dụng đối tượng này để kiểm tra trạng thái hoặc gọi `job.join()` để chờ nó hoàn thành.

---

### 6.2 `CoroutineScope.async()`
- **Đặc tính**: Bắt đầu một phép tính toán đồng thời bên trong scope và trả về một đối tượng đại diện [`Deferred<T>`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/).
- **Nhận kết quả**: Sử dụng hàm suspend **`.await()`** để tạm dừng luồng cho đến khi kết quả tính toán sẵn sàng.

```kotlin
import kotlin.time.Duration.Companion.milliseconds
import kotlinx.coroutines.*

suspend fun main() = withContext(Dispatchers.Default) { // this: CoroutineScope
    // Bắt đầu tải trang thứ nhất
    val firstPage = this.async {
        delay(50.milliseconds)
        "First page"
    }

    // Bắt đầu tải song song trang thứ hai
    val secondPage = this.async {
        delay(100.milliseconds)
        "Second page"
    }

    // Chờ cả hai kết quả và so sánh chúng
    val pagesAreEqual = firstPage.await() == secondPage.await()
    println("Pages are equal: $pagesAreEqual")
}
```

**Kết quả Console:**
```text
Pages are equal: false
```

---

### 6.3 `runBlocking()`
Hàm `runBlocking()` tạo ra một coroutine scope và **khóa cứng (blocks) thread hiện tại** cho đến khi toàn bộ coroutines khởi chạy trong scope đó hoàn thành.

> [!CAUTION]
> **Chỉ sử dụng `runBlocking()` khi không còn lựa chọn nào khác** để làm cầu nối gọi mã `suspend` từ mã đồng bộ thông thường (non-suspending code), ví dụ như khi tích hợp với thư viện bên thứ ba không thể sửa đổi:

```kotlin
import kotlin.time.Duration.Companion.milliseconds
import kotlinx.coroutines.*

// Một interface của bên thứ ba không hỗ trợ suspend mà bạn không thể sửa đổi
interface Repository {
    fun readItem(): Int
}

object MyRepository : Repository {
    override fun readItem(): Int {
        // Cầu nối từ hàm thông thường sang hàm suspend
        return runBlocking {
            myReadItem()
        }
    }
}

suspend fun myReadItem(): Int {
    delay(100.milliseconds)
    return 4
}
```

---

## 7. Bộ điều phối Coroutine (Coroutine Dispatchers)

Một **coroutine dispatcher** điều khiển thread hoặc thread pool nào sẽ được sử dụng để thực thi coroutine.
- Coroutine **không bị trói buộc vĩnh viễn vào một thread duy nhất**. Chúng có thể tạm dừng trên thread này và tiếp tục chạy trên thread khác tùy thuộc vào dispatcher.
- Điều này cho phép bạn chạy đồng thời hàng triệu coroutine mà không cần phải cấp phát một thread riêng biệt cho từng coroutine.

> [!TIP]
> Ngay cả khi một coroutine tạm dừng trên thread này và tiếp tục trên một thread khác, **toàn bộ các giá trị biến được ghi trước khi coroutine suspend được bảo đảm tính hiển thị bộ nhớ (memory visibility guarantee)** và luôn sẵn sàng cho coroutine sử dụng khi resume.

### Cơ chế thừa kế Dispatcher:
- Bạn không bắt buộc phải chỉ định dispatcher cho mọi coroutine. Mặc định, các coroutine con sẽ **thừa kế dispatcher từ scope cha**.
- Nếu trong ngữ cảnh không có dispatcher nào được chỉ định, các coroutine builder sẽ mặc định dùng **`Dispatchers.Default`** (chạy trên thread pool chia sẻ tối ưu cho CPU).

### Chỉ định Dispatcher cho Builder:
```kotlin
suspend fun runWithDispatcher() = coroutineScope { // this: CoroutineScope
    this.launch(Dispatchers.Default) {
        println("Running on ${Thread.currentThread().name}")
    }
}
```

### Sử dụng `withContext(Dispatchers.Default)` để tính toán song song:
```kotlin
import kotlin.time.Duration.Companion.milliseconds
import kotlinx.coroutines.*

suspend fun main() = withContext(Dispatchers.Default) { // this: CoroutineScope
    println("Running withContext block on ${Thread.currentThread().name}")

    val one = this.async {
        println("First calculation starting on ${Thread.currentThread().name}")
        val sum = (1L..500_000L).sum()
        delay(200L)
        println("First calculation done on ${Thread.currentThread().name}")
        sum
    }

    val two = this.async {
        println("Second calculation starting on ${Thread.currentThread().name}")
        val sum = (500_001L..1_000_000L).sum()
        println("Second calculation done on ${Thread.currentThread().name}")
        sum
    }

    // Chờ cả hai phép tính và in ra tổng
    println("Combined total: ${one.await() + two.await()}")
}
```

**Kết quả Console mẫu:**
```text
Running withContext block on DefaultDispatcher-worker-1
First calculation starting on DefaultDispatcher-worker-2
Second calculation starting on DefaultDispatcher-worker-3
Second calculation done on DefaultDispatcher-worker-3
First calculation done on DefaultDispatcher-worker-2
Combined total: 500000500000
```

---

## 8. So sánh Coroutines và JVM Threads

Mặc dù coroutines là các phép tính có thể tạm dừng chạy đồng thời tương tự như Threads trên máy ảo JVM, cơ chế hoạt động bên dưới của chúng hoàn toàn khác nhau:

- **JVM Thread**: Do hệ điều hành quản lý. Khi bạn tạo một thread, hệ điều hành cấp phát bộ nhớ riêng cho Stack của nó (thường từ 1MB đến 2MB) và sử dụng nhân hệ điều hành (kernel) để chuyển đổi giữa các thread. Do đó, JVM thông thường chỉ có thể xử lý vài nghìn threads cùng lúc trước khi cạn kiệt tài nguyên.
- **Coroutine**: Không bị ràng buộc với một thread cụ thể. Khi coroutine tạm dừng, thread bên dưới không hề bị khóa và hoàn toàn rảnh rỗi để thực thi các tác vụ khác. Điều này giúp coroutine nhẹ hơn thread hàng nghìn lần và cho phép chạy hàng triệu coroutines trong cùng một tiến trình.

```
So sánh Mức Tiêu Thụ Bộ Nhớ:
50,000 OS Threads  ───► Có thể ngốn tới ~100 GB RAM (Gây OutOfMemoryError ngay lập tức)
50,000 Coroutines  ───► Chỉ tốn khoảng ~500 MB RAM (Chạy mượt mà, hoàn thành an toàn)
```

### Thử nghiệm 1: 50,000 Coroutines
```kotlin
import kotlin.time.Duration.Companion.seconds
import kotlinx.coroutines.*

suspend fun main() {
    withContext(Dispatchers.Default) {
        // Khởi chạy 50,000 coroutines, mỗi coroutine chờ 5 giây rồi in một dấu chấm
        printPeriods()
    }
}

suspend fun printPeriods() = coroutineScope { // this: CoroutineScope
    repeat(50_000) {
        this.launch {
            delay(5.seconds)
            print(".")
        }
    }
}
```
*Kết quả*: Chương trình chạy mượt mà, in ra 50,000 dấu chấm và tiêu tốn chỉ khoảng ~500 MB RAM.

### Thử nghiệm 2: 50,000 JVM Threads
```kotlin
import kotlin.concurrent.thread

fun main() {
    repeat(50_000) {
        thread {
            Thread.sleep(5000L)
            print(".")
        }
    }
}
```
*Kết quả*: Tùy thuộc vào hệ điều hành và dung lượng RAM của máy, đoạn mã trên sẽ ném ra ngoại lệ **`OutOfMemoryError`** hoặc làm chậm toàn bộ hệ thống vì hệ điều hành không thể cấp phát thêm native thread!

---

## 9. Các Bước Tiếp theo (What's Next)

- [**Cancellation and timeouts (Hủy bỏ và Giới hạn thời gian)**](02-cancellation-and-timeouts.md): Tìm hiểu cách hủy coroutine an toàn và xử lý giới hạn thời gian thực thi.
- [**Composing suspending functions (Kết hợp các hàm Suspend)**](03-composing-suspending-functions.md): Khám phá các phương pháp kết hợp nhiều hàm tạm dừng đồng thời.
- [**Coroutine context and dispatchers (Ngữ cảnh và Bộ điều phối)**](04-coroutine-context-and-dispatchers.md): Đào sâu vào quản lý thread và context.
- [**Flows (Asynchronous Flow)**](05-flows-core.md): Học cách trả về một chuỗi nhiều giá trị được tính toán bất đồng bộ theo thời gian.

---

## 💡 Phân tích Chuyên sâu (Under The Hood: CPS & State Machine)

Làm sao một hàm `suspend` có thể tạm dừng mà không block thread?

Khi trình biên dịch Kotlin gặp từ khóa `suspend`, nó áp dụng kỹ thuật biến đổi mang tên **Continuation-Passing Style (CPS)**:
1. **Thêm tham số `Continuation`**: Mọi hàm `suspend fun doWork(): String` sau khi biên dịch sang bytecode thực chất trở thành:
   ```java
   Object doWork(Continuation<? super String> completion)
   ```
2. **Tạo State Machine nội bộ**: Toàn bộ thân hàm được chia tách thành một switch-case state machine tương ứng với các điểm suspension point (`label = 0`, `label = 1`, ...).
3. **Khi tạm dừng**: Hàm lưu trạng thái biến cục bộ vào đối tượng `Continuation` và trả về một hằng số đặc biệt `COROUTINE_SUSPENDED`. Thread hiện tại được giải phóng hoàn toàn để phục vụ tác vụ khác.
4. **Khi tiếp tục**: Khi có dữ liệu trả về, phương thức `continuation.resumeWith(result)` được kích hoạt, đưa công việc trở lại Dispatcher để chạy tiếp case tiếp theo trong state machine.

---

## 📝 Bảng Thuật ngữ & Tóm tắt nhanh

| Thuật ngữ | Khái niệm kỹ thuật tương ứng |
| :--- | :--- |
| **Suspending Function** | Hàm có thể tạm dừng nhường thread và tiếp tục sau đó mà không làm nghẽn luồng. |
| **Structured Concurrency** | Nguyên lý bảo đảm coroutine con luôn được kiểm soát và bao bọc bởi coroutine cha. |
| **`launch`** | Khởi chạy coroutine kiểu bắn và quên (fire-and-forget), trả về `Job`. |
| **`async`** | Khởi chạy coroutine tính toán đồng thời, trả về `Deferred<T>` để lấy kết quả bằng `.await()`. |
| **`runBlocking`** | Khóa thread hiện tại cho đến khi coroutine hoàn thành, chỉ dùng cho bridge hoặc Unit Test. |
| **Memory Footprint** | Coroutine siêu nhẹ (~vài KB) so với JVM Thread (~1MB stack), cho phép chạy 50k coroutine tốn ~500MB thay vì ~100GB. |

# Bài 07 — Kênh Giao tiếp (Channels & Pipelines)

> **Tài liệu gốc:** [Channels — Kotlin Documentation](https://kotlinlang.org/docs/channels.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Hiểu rõ cơ chế truyền dòng giá trị (stream of values) giữa các coroutine bằng `Channel`; phân biệt Channel với `BlockingQueue`; kỹ thuật đóng kênh và duyệt phần tử; xây dựng producer bằng hàm `produce`; kiến trúc Pipeline (áp dụng thuật toán sàng số nguyên tố); các mô hình phân phối tải Fan-out & Fan-in; các loại dung lượng đệm (Buffered channels, Rendezvous); tính công bằng (Fairness) trong Channel; và kênh phát nhịp Ticker Channels.

---

## 1. Giới thiệu về Channels (Channel Basics)

Trong khi các giá trị `Deferred` (trả về từ `async`) cung cấp phương thức tiện lợi để truyền **một giá trị đơn lẻ** giữa các coroutine, thì **Channel** cung cấp giải pháp truyền **một dòng các giá trị (stream of values)**.

Về mặt khái niệm, một `Channel` rất giống với hàng đợi chặn `BlockingQueue` trong lập trình đa luồng truyền thống của Java. Điểm khác biệt cốt lõi là:
- Thay vì phương thức `put()` gây block thread, Channel sử dụng hàm tạm ngưng (suspending) **`send()`**.
- Thay vì phương thức `take()` gây block thread, Channel sử dụng hàm tạm ngưng **`receive()`**.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        // Đây có thể là phép tính toán nặng tiêu tốn CPU hoặc logic bất đồng bộ,
        // ở đây chúng ta gửi 5 số chính phương:
        for (x in 1..5) channel.send(x * x)
    }
    // In ra 5 số nguyên nhận được từ kênh:
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

**Kết quả Console Output:**
```none
1
4
9
16
25
Done!
```

---

## 2. Đóng Kênh & Duyệt qua Kênh (Closing and Iteration over Channels)

Không giống như các hàng đợi (queue) thông thường, một Channel có thể được **đóng lại (`close()`)** để báo hiệu rằng không còn phần tử nào được gửi đến nữa. 

Ở phía bên nhận (receiver), bạn có thể sử dụng vòng lặp `for` thông thường rất thuận tiện để nhận lần lượt các phần tử từ channel.

> [!NOTE]
> Về mặt khái niệm, hành động `close()` tương đương với việc gửi một "thẻ bài đóng kênh" (special close token) đặc biệt vào channel. Vòng lặp `for` sẽ dừng lại ngay khi nhận được thẻ bài này, đảm bảo rằng **tất cả các phần tử được gửi đi trước thời điểm gọi `close()` đều được nhận đầy đủ**:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // Báo hiệu đã gửi xong toàn bộ dữ liệu
    }
    // In các giá trị nhận được bằng vòng lặp `for` (cho đến khi kênh bị đóng)
    for (y in channel) println(y)
    println("Done!")
}
```

**Kết quả Console Output:**
```none
1
4
9
16
25
Done!
```

---

## 3. Xây dựng Kênh Sản xuất Dữ liệu (Building Channel Producers)

Mô hình trong đó một coroutine sản xuất ra một chuỗi phần tử là rất phổ biến — đây chính là một phần của mô hình **Người sản xuất - Người tiêu thụ (Producer-Consumer pattern)** thường thấy trong mã nguồn đồng thời.

Bạn hoàn toàn có thể đóng gói logic sản xuất đó vào một hàm nhận `Channel` làm tham số, nhưng điều này đi ngược lại nguyên lý lập trình thông thường rằng kết quả nên được trả về từ hàm.

Kotlin Coroutines cung cấp một coroutine builder tiện lợi mang tên **`produce`** giúp bạn dễ dàng viết mã ở phía producer, cùng với hàm mở rộng **`consumeEach`** thay thế cho vòng lặp `for` ở phía consumer:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

// Hàm mở rộng trên CoroutineScope trả về một ReceiveChannel
fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

fun main() = runBlocking {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

**Kết quả Console Output:**
```none
1
4
9
16
25
Done!
```

---

## 4. Đường ống Xử lý (Pipelines)

Một **Pipeline** là mô hình trong đó một coroutine sản xuất một dòng giá trị (có thể là vô tận):

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // Dòng số nguyên vô tận bắt đầu từ 1
}
```

Và một hoặc nhiều coroutine khác sẽ tiêu thụ dòng giá trị đó, thực hiện một số bước xử lý, rồi tiếp tục sản xuất ra một dòng kết quả mới. Trong ví dụ dưới đây, các con số được bình phương:

```kotlin
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

Hàm `main` khởi động và kết nối toàn bộ đường ống pipeline lại với nhau:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    val numbers = produceNumbers() // Sản xuất các số nguyên từ 1 trở đi
    val squares = square(numbers)  // Bình phương các số nguyên nhận được
    repeat(5) {
        println(squares.receive()) // In ra 5 số đầu tiên
    }
    println("Done!")
    coroutineContext.cancelChildren() // Hủy tất cả coroutine con để chương trình kết thúc
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // Dòng số nguyên vô tận bắt đầu từ 1
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

**Kết quả Console Output:**
```none
1
4
9
16
25
Done!
```

> [!NOTE]
> Tất cả các hàm tạo coroutine trên đều được định nghĩa là hàm mở rộng trên `CoroutineScope`, giúp chúng ta tận dụng triệt để cơ chế **Đồng thời Có cấu trúc (Structured Concurrency)** nhằm đảm bảo không có coroutine rác nào bị bỏ sót hay chạy ngầm vô hạn trong ứng dụng.

---

## 5. Sàng Số Nguyên Tố bằng Pipeline (Prime Numbers with Pipeline)

Hãy đẩy mô hình Pipeline lên mức độ kinh điển với ví dụ tạo ra chuỗi các số nguyên tố bằng một pipeline các coroutine (dựa trên thuật toán Sàng Eratosthenes).

Chúng ta bắt đầu bằng một chuỗi số vô tận:

```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // Dòng số nguyên vô tận bắt đầu từ start
}
```

Tầng tiếp theo của pipeline sẽ lọc dòng số đi vào, loại bỏ toàn bộ các số chia hết cho số nguyên tố đã cho:

```kotlin
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

Bây giờ ta xây dựng pipeline bằng cách bắt đầu dòng số từ 2, lấy số nguyên tố hiện tại từ channel, và khởi chạy một tầng pipeline mới cho mỗi số nguyên tố tìm thấy:

```text
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ...
```

Chương trình hoàn chỉnh in ra 10 số nguyên tố đầu tiên:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    var cur = numbersFrom(2)
    repeat(10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren() // Hủy tất cả coroutine con để hàm main kết thúc
}

fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++)
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

**Kết quả Console Output:**
```none
2
3
5
7
11
13
17
19
23
29
```

> [!TIP]
> Bạn có thể xây dựng pipeline tương tự bằng builder `iterator` trong thư viện chuẩn Kotlin (thay `produce` bằng `iterator`, `send` bằng `yield`, `receive` bằng `next`, `ReceiveChannel` bằng `Iterator`).  
> Tuy nhiên, lợi thế vượt trội của pipeline sử dụng Channel là nó **có thể tận dụng tối đa nhiều lõi CPU đồng thời** nếu chạy trên ngữ cảnh `Dispatchers.Default`, và bên trong có thể gọi các tác vụ bất đồng bộ bất kỳ (như gọi Network API), điều mà `sequence`/`iterator` đồng bộ không thể làm được.

---

## 6. Mô hình Phân phối Tải Fan-out (Fan-out)

Nhiều coroutine có thể cùng đọc (receive) từ một Channel duy nhất để **phân chia khối lượng công việc** giữa chúng.

Giả sử một producer phát ra các số nguyên đều đặn (10 số mỗi giây):

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) {
        send(x++)
        delay(100) // Đợi 100ms
    }
}
```

Chúng ta khởi tạo nhiều processor coroutine để cùng nhận dữ liệu:

```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

Chạy thử nghiệm với 5 processor:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // Hủy producer, đóng channel và kết thúc các processor
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) {
        send(x++)
        delay(100)
    }
}

fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

**Kết quả Console Output mẫu:**
```none
Processor #2 received 1
Processor #4 received 2
Processor #0 received 3
Processor #1 received 4
Processor #3 received 5
Processor #2 received 6
Processor #4 received 7
Processor #0 received 8
Processor #1 received 9
Processor #3 received 10
```

> [!IMPORTANT]
> **Sự khác biệt quan trọng giữa vòng lặp `for` và `consumeEach` trong Fan-out:**  
> Trong hàm `launchProcessor`, chúng ta bắt buộc phải duyệt channel bằng vòng lặp **`for`**. Cú pháp `for` này an toàn tuyệt đối khi sử dụng từ nhiều coroutine cùng lúc. Nếu một processor gặp sự cố lỗi hoặc bị hủy, các processor khác vẫn tiếp tục hoạt động và đọc channel bình thường.  
> Ngược lại, nếu bạn dùng `consumeEach`, khi hàm kết thúc (dù bình thường hay do lỗi), nó sẽ **hủy luôn Channel gốc bên dưới**, làm sập toàn bộ các coroutine tiêu thụ khác!

---

## 7. Mô hình Gom Tải Fan-in (Fan-in)

Nhiều coroutine có thể cùng gửi dữ liệu vào **cùng một Channel**.

Ví dụ, hàm tạm ngưng `sendString` liên tục gửi một chuỗi định kỳ vào channel:

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

Khởi chạy đồng thời 2 coroutine gửi chuỗi khác nhau vào cùng một channel:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    val channel = Channel<String>()
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(6) { // Nhận 6 phần tử đầu tiên
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // Hủy các coroutine con để main kết thúc
}

suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

**Kết quả Console Output:**
```none
foo
foo
BAR!
foo
foo
BAR!
```

---

## 8. Kênh Có Bộ Đệm (Buffered Channels)

Các kênh được giới thiệu từ đầu bài đến giờ đều là **kênh không đệm (unbuffered channels)** hay còn gọi là **kênh gặp gỡ (rendezvous)**. Khi không có đệm:
- Phần tử chỉ được truyền khi bên gửi và bên nhận gặp nhau tại cùng thời điểm.
- Nếu `send()` được gọi trước, nó sẽ bị tạm ngưng cho đến khi có bên gọi `receive()`.
- Nếu `receive()` được gọi trước, nó sẽ bị tạm ngưng cho đến khi có bên gọi `send()`.

Cả hàm khởi tạo `Channel()` lẫn builder `produce` đều cho phép truyền tham số `capacity` để xác định **kích thước bộ đệm**:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    val channel = Channel<Int>(4) // Tạo kênh có bộ đệm chứa tối đa 4 phần tử
    val sender = launch {
        repeat(10) {
            println("Sending $it")
            channel.send(it) // Sẽ bị tạm ngưng khi bộ đệm đầy
        }
    }
    // Chưa nhận gì cả... tạm dừng chờ một lúc...
    delay(1000)
    sender.cancel()
}
```

**Kết quả Console Output:**
```none
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

> [!NOTE]
> Bên gửi in ra chữ `"Sending"` đúng **5 lần**: 4 phần tử đầu tiên (0, 1, 2, 3) được nạp thành công vào bộ đệm, và đến phần tử thứ 5 (số 4) thì bộ đệm đầy nên lệnh `channel.send(4)` lập tức bị tạm ngưng (suspend).

### Các loại dung lượng đệm đặc biệt trong Kotlin:
- **`Channel.RENDEZVOUS` (mặc định = 0)**: Không có đệm, bên gửi và nhận phải gặp nhau trực tiếp.
- **`Channel.BUFFERED`**: Dung lượng đệm mặc định (thường là 64 phần tử, có thể cấu hình qua thuộc tính hệ thống JVM).
- **`Channel.UNLIMITED`**: Bộ đệm danh sách liên kết vô hạn (có nguy cơ OutOfMemory nếu bên gửi nhanh hơn bên nhận liên tục).
- **`Channel.CONFLATED`**: Dung lượng đệm bằng 1. Giá trị mới nhất sẽ ghi đè lên giá trị cũ; bên gửi không bao giờ bị tạm ngưng.

---

## 9. Tính Công Bằng trong Channel (Channels Are Fair)

Các thao tác `send` và `receive` đối với Channel luôn đảm bảo **tính công bằng (fairness)** theo thứ tự gọi giữa nhiều coroutine. Chúng được phục vụ theo cơ chế **vào trước ra trước (FIFO)**: coroutine nào gọi `receive()` trước sẽ là coroutine đầu tiên nhận được phần tử.

Xem ví dụ trận đấu bóng bàn giữa hai coroutine `"ping"` và `"pong"` cùng chia sẻ một channel chiếc bàn (`table`):

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

data class Ball(var hits: Int)

fun main() = runBlocking {
    val table = Channel<Ball>() // Chiếc bàn chung
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // Phát quả bóng vào bàn
    delay(1000) // Trận đấu diễn ra trong 1 giây
    coroutineContext.cancelChildren() // Hết giờ, dừng trận đấu
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // Nhận bóng từ bàn
        ball.hits++
        println("$name $ball")
        delay(300) // Đỡ bóng và phản xạ mất 300ms
        table.send(ball) // Đánh bóng trả lại bàn
    }
}
```

**Kết quả Console Output:**
```none
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
```

> [!NOTE]
> Mặc dù coroutine `"ping"` sau khi đánh bóng xong sẽ ngay lập tức quay lại vòng lặp để đợi nhận bóng tiếp, nhưng quả bóng luôn được chuyển cho coroutine `"pong"`, bởi vì `"pong"` đã xếp hàng chờ trước đó.

---

## 10. Kênh Phát Nhịp (Ticker Channels)

**Ticker channel** là một kênh rendezvous đặc biệt phát ra giá trị `Unit` sau mỗi khoảng thời gian trễ nhất định kể từ lần tiêu thụ trước. 

Nó là khối xây dựng nền tảng để tạo ra các pipeline xử lý theo cửa sổ thời gian (windowing) hoặc kết hợp với biểu thức `select` để thực hiện hành động định kỳ ("on tick").

Để tạo kênh này, sử dụng hàm `ticker()`. Để dừng kênh, gọi hàm `cancel()`:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    // Tạo ticker channel: phát nhịp mỗi 200ms, không trễ lúc khởi đầu
    val tickerChannel = ticker(delayMillis = 200, initialDelayMillis = 0)
    
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // Nhận ngay lập tức

    nextElement = withTimeoutOrNull(100) { tickerChannel.receive() } // Các phần tử sau cần 200ms
    println("Next element is not ready in 100 ms: $nextElement")

    nextElement = withTimeoutOrNull(120) { tickerChannel.receive() }
    println("Next element is ready in 200 ms: $nextElement")

    // Giả lập người tiêu thụ bị dừng trễ lớn (300ms)
    println("Consumer pauses for 300ms")
    delay(300)
    
    // Phần tử tiếp theo có sẵn ngay lập tức vì đã quá chu kỳ 200ms
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    
    // Khoảng dừng giữa các lần nhận được tính bù trừ nên phần tử sau đó đến nhanh hơn
    nextElement = withTimeoutOrNull(120) { tickerChannel.receive() }
    println("Next element is ready in 100ms after consumer pause in 300ms: $nextElement")

    tickerChannel.cancel() // Hủy kênh khi không còn nhu cầu sử dụng
}
```

**Kết quả Console Output:**
```none
Initial element is available immediately: kotlin.Unit
Next element is not ready in 100 ms: null
Next element is ready in 200 ms: kotlin.Unit
Consumer pauses for 300ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 100ms after consumer pause in 300ms: kotlin.Unit
```

### Các Chế độ Phát Nhịp (`TickerMode`):
- **`TickerMode.FIXED_PERIOD` (mặc định)**: Ticker theo dõi khoảng dừng của bên nhận và tự động điều chỉnh độ trễ của phần tử tiếp theo để cố gắng duy trì một tần suất phát cố định (fixed rate).
- **`TickerMode.FIXED_DELAY`**: Duy trì khoảng thời gian trễ cố định tuyệt đối giữa thời điểm phần tử trước được tiêu thụ và thời điểm phần tử sau xuất hiện.

---

## 11. Bảng So sánh Tổng hợp

| Khái niệm / API | Đặc điểm Kỹ thuật | Kịch bản Áp dụng |
| :--- | :--- | :--- |
| **`Channel<T>()`** | Hàng đợi truyền tin non-blocking giữa các coroutine. | Giao tiếp giữa 2 hoặc nhiều coroutine độc lập. |
| **`produce { }`** | Coroutine builder tạo producer, tự động đóng kênh khi coroutine kết thúc. | Tạo dòng dữ liệu đầu vào cho Pipeline. |
| **`consumeEach { }`** | Duyệt và tự động hủy channel khi kết thúc. | Dành cho Consumer duy nhất (đơn quyền sở hữu). |
| **`for (item in channel)`** | Duyệt an toàn không hủy channel. | Bắt buộc sử dụng trong **Fan-out** đa luồng tiêu thụ. |
| **Fan-out** | 1 Producer $\rightarrow$ Nhiều Consumer chia sẻ tải. | Xử lý các tác vụ nặng ngốn CPU hoặc Network song song. |
| **Fan-in** | Nhiều Producer $\rightarrow$ 1 Consumer gom dữ liệu. | Gom log, gom sự kiện người dùng từ nhiều nguồn về một xử lý tập trung. |
| **`ticker()`** | Kênh phát nhịp định kỳ với cơ chế bù trừ thời gian. | Tạo nhịp đập heartbeat, timeout định kỳ, xử lý theo cửa sổ (windowing). |

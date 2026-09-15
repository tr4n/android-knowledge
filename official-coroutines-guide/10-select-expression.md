# Bài 10 — Biểu thức Chọn Lựa: `select` (Select Expression)

> **Tài liệu gốc:** [Select expression (experimental) — Kotlin Documentation](https://kotlinlang.org/docs/select-expression.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Hiểu rõ cơ chế chờ đợi đồng thời nhiều tác vụ bất đồng bộ và chọn kết quả đầu tiên sẵn sàng với `select`; các mệnh đề chọn nhận từ Channel (`onReceive`, `onReceiveCatching`); chọn gửi vào Channel (`onSend`); chọn kết quả `Deferred` hoàn thành trước (`onAwait`); tính thiên vị (bias) trong `select`; và kỹ thuật chuyển mạch luồng dữ liệu (Switch over a channel of deferred values).

---

## 1. Giới thiệu về Biểu thức `select`

Biểu thức `select` cho phép bạn **chờ đợi đồng thời nhiều hàm suspending cùng một lúc** và **lựa chọn (select) tác vụ đầu tiên có sẵn dữ liệu** để xử lý.

> [!NOTE]
> Biểu thức `select` là một tính năng thực nghiệm (experimental) trong `kotlinx.coroutines`. API của nó có thể tiếp tục hoàn thiện trong các bản cập nhật tiếp theo.

---

## 2. Chọn Nhận Dữ liệu từ Channels (Selecting from Channels)

Giả sử chúng ta có hai hàm sản xuất chuỗi bất đồng bộ:
- `fizz`: Cứ mỗi 500 ms lại phát ra chuỗi `"Fizz"`.
- `buzz`: Cứ mỗi 1000 ms lại phát ra chuỗi `"Buzz!"`.

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) { // Gửi "Fizz" mỗi 500 ms
        delay(500)
        send("Fizz")
    }
}

fun CoroutineScope.buzz() = produce<String> {
    while (true) { // Gửi "Buzz!" mỗi 1000 ms
        delay(1000)
        send("Buzz!")
    }
}
```

Nếu dùng hàm suspending `receive()` thông thường, chúng ta chỉ có thể chọn đọc từ channel này **hoặc** channel kia theo thứ tự tuần tự. Nhưng với biểu thức `select`, chúng ta có thể **lắng nghe cả hai kênh cùng lúc** thông qua mệnh đề **`onReceive`**:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.fizz() = produce<String> {
    while (true) {
        delay(500)
        send("Fizz")
    }
}

fun CoroutineScope.buzz() = produce<String> {
    while (true) {
        delay(1000)
        send("Buzz!")
    }
}

suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> { // <Unit> nghĩa là biểu thức select này không trả về giá trị
        fizz.onReceive { value ->  // Mệnh đề select thứ nhất
            println("fizz -> '$value'")
        }
        buzz.onReceive { value ->  // Mệnh đề select thứ hai
            println("buzz -> '$value'")
        }
    }
}

fun main() = runBlocking<Unit> {
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    coroutineContext.cancelChildren() // Hủy hai coroutine fizz và buzz
}
```

**Kết quả Console Output:**
```none
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
fizz -> 'Fizz'
buzz -> 'Buzz!'
fizz -> 'Fizz'
fizz -> 'Fizz'
```

---

## 3. Chọn Lựa Khi Kênh Đóng: `onReceiveCatching` (Selecting on Close)

Mệnh đề `onReceive` trong `select` sẽ bị thất bại và ném ra ngoại lệ nếu kênh bị đóng (`ClosedReceiveChannelException`).

Để xử lý việc kênh bị đóng một cách an toàn và thanh thoát, chúng ta sử dụng mệnh đề **`onReceiveCatching`**. Đồng thời, ví dụ dưới đây cũng minh họa rằng `select` là một **biểu thức (expression)** có thể trả về giá trị:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveCatching { it ->
            val value = it.getOrNull()
            if (value != null) {
                "a -> '$value'"
            } else {
                "Channel 'a' is closed"
            }
        }
        b.onReceiveCatching { it ->
            val value = it.getOrNull()
            if (value != null) {
                "b -> '$value'"
            } else {
                "Channel 'b' is closed"
            }
        }
    }
    
fun main() = runBlocking<Unit> {
    val a = produce<String> {
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String> {
        repeat(4) { send("World $it") }
    }
    repeat(8) { // In 8 kết quả đầu tiên
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()  
}
```

**Kết quả Console Output:**
```none
a -> 'Hello 0'
a -> 'Hello 1'
b -> 'World 0'
a -> 'Hello 2'
a -> 'Hello 3'
b -> 'World 1'
Channel 'a' is closed
Channel 'a' is closed
```

### Hai Quan sát Kỹ thuật Quan trọng:
1. **`select` có tính thiên vị (Biased towards the first clause)**: Khi có nhiều mệnh đề cùng sẵn sàng tại cùng một thời điểm, **mệnh đề đứng đầu tiên trong khối `select` sẽ luôn được ưu tiên lựa chọn**. Ở ví dụ trên, kênh `a` nằm ở dòng đầu nên luôn thắng thế; chỉ khi `a` bị tạm ngưng trên lệnh `send()` (vì là kênh unbuffered) thì kênh `b` mới có cơ hội được chọn.
2. **`onReceiveCatching` được chọn ngay lập tức khi kênh đã đóng**: Nếu kênh đã ở trạng thái đóng, mệnh đề `onReceiveCatching` sẽ được kích hoạt ngay với kết quả `ChannelResult.closed()`.

---

## 4. Chọn Kênh để Gửi Dữ liệu: `onSend` (Selecting to Send)

Biểu thức `select` cung cấp mệnh đề **`onSend`**, kết hợp với tính thiên vị của `select` để giải quyết các bài toán định tuyến dữ liệu rất thông minh: **Gửi vào kênh chính; nếu bên tiêu thụ kênh chính quá chậm thì tự động chuyển hướng gửi sang kênh phụ (side channel)**:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) { // Sản xuất 10 số từ 1 đến 10
        delay(100)       // Mỗi 100 ms
        select<Unit> {
            onSend(num) {}      // Ưu tiên gửi vào kênh chính
            side.onSend(num) {} // Nếu kênh chính bận thì gửi sang kênh phụ
        }
    }
}

fun main() = runBlocking<Unit> {
    val side = Channel<Int>() // Cấp phát kênh phụ
    launch { // Bên tiêu thụ kênh phụ cực nhanh
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach { 
        println("Consuming $it")
        delay(250) // Bên tiêu thụ kênh chính xử lý chậm (250 ms)
    }
    println("Done consuming")
    coroutineContext.cancelChildren()  
}
```

**Kết quả Console Output:**
```none
Consuming 1
Side channel has 2
Side channel has 3
Consuming 4
Side channel has 5
Side channel has 6
Consuming 7
Side channel has 8
Side channel has 9
Consuming 10
Done consuming
```

> [!NOTE]
> Nhờ mệnh đề `onSend(num)` đầu tiên, khi kênh chính sẵn sàng thì số 1 được chuyển tới kênh chính. Trong 250 ms kênh chính đang bận, số 2 và 3 được tự động chuyển hướng sang `side channel`. Đến số 4, kênh chính đã rảnh nên lại nhận được số 4. Mô hình này giúp loại bỏ hoàn toàn hiện tượng nghẽn luồng (blocking) ở phía producer.

---

## 5. Chọn Kết quả `Deferred` Nhanh Nhất: `onAwait` (Selecting Deferred Values)

Các giá trị bất đồng bộ `Deferred` (sinh ra từ `async`) có thể được lựa chọn bằng mệnh đề **`onAwait`**.

Ví dụ dưới đây khởi chạy 12 tác vụ `async` với thời gian trễ ngẫu nhiên. Biểu thức `select` sẽ **chờ tác vụ đầu tiên hoàn thành**, trả về kết quả và kiểm tra xem có bao nhiêu coroutine còn lại vẫn đang chạy:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.*
import java.util.*
    
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}

fun CoroutineScope.asyncStringsList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}

fun main() = runBlocking<Unit> {
    val list = asyncStringsList()
    
    // Vì select là một Kotlin DSL, bạn có thể duyệt vòng lặp để đăng ký mệnh đề:
    val result = select<String> {
        list.withIndex().forEach { (index, deferred) ->
            deferred.onAwait { answer ->
                "Deferred $index produced answer '$answer'"
            }
        }
    }
    println(result)
    val countActive = list.count { it.isActive }
    println("$countActive coroutines are still active")
}
```

**Kết quả Console Output mẫu:**
```none
Deferred 4 produced answer 'Waited for 128 ms'
11 coroutines are still active
```

---

## 6. Chuyển Mạch Kênh Deferred (Switch Over a Channel of Deferred Values)

Một ứng dụng nâng cao của `select` là kết hợp đồng thời cả **`onReceiveCatching`** và **`onAwait`** trong cùng một khối: Tiêu thụ một kênh chứa các đối tượng `Deferred`, chờ kết quả của deferred hiện tại, **nhưng nếu có một deferred mới xuất hiện trước khi deferred cũ tính toán xong, thì lập tức bỏ qua deferred cũ và chuyển sang chờ cái mới** (tương tự như toán tử `switchMap` trong ReactiveX hoặc `flatMapLatest` trong Flow):

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*
    
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive() // Bắt đầu với deferred đầu tiên nhận được
    while (isActive) {
        val next = select<Deferred<String>?> { // Trả về deferred tiếp theo hoặc null
            input.onReceiveCatching { update ->
                update.getOrNull()
            }
            current.onAwait { value ->
                send(value) // Phát giá trị mà deferred hiện tại vừa hoàn thành
                input.receiveCatching().getOrNull() // Lấy deferred tiếp theo từ input
            }
        }
        if (next == null) {
            println("Channel was closed")
            break
        } else {
            current = next
        }
    }
}

fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}

fun main() = runBlocking<Unit> {
    val chan = Channel<Deferred<String>>()
    launch {
        for (s in switchMapDeferreds(chan)) 
            println(s)
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200) // Đủ thời gian cho "BEGIN" hoàn thành
    chan.send(asyncString("Slow", 500))
    delay(100) // Không đủ thời gian cho "Slow" (cần 500ms), gửi đè "Replace" vào ngay!
    chan.send(asyncString("Replace", 100))
    delay(500)
    chan.send(asyncString("END", 500))
    delay(1000)
    chan.close()
    delay(500)
}
```

**Kết quả Console Output:**
```none
BEGIN
Replace
END
Channel was closed
```

> [!NOTE]
> Giá trị `"Slow"` đã hoàn toàn bị bỏ qua và không xuất hiện trong kết quả đầu ra, bởi vì mệnh đề `input.onReceiveCatching` đã kích hoạt trước khi `current.onAwait` kịp hoàn tất, chuyển mạch ngay sang chuỗi `"Replace"`.

---

## 7. Bảng Tổng hợp Mệnh đề trong `select`

| Mệnh đề (Clause) | Áp dụng cho | Hành vi Kích hoạt |
| :--- | :--- | :--- |
| **`onReceive`** | `ReceiveChannel<T>` | Được chọn khi kênh có sẵn phần tử. Ném lỗi nếu kênh đã đóng. |
| **`onReceiveCatching`** | `ReceiveChannel<T>` | Được chọn khi kênh có phần tử **HOẶC** khi kênh đã đóng (`ChannelResult.closed`). |
| **`onSend`** | `SendChannel<T>` | Được chọn khi bộ đệm của kênh còn chỗ trống để gửi giá trị. |
| **`onAwait`** | `Deferred<T>` | Được chọn ngay khi tác vụ bất đồng bộ tính toán xong kết quả. |

# Bài 06 — Các Toán tử trong Flow (Flow Operators)

> **Tài liệu gốc:** [Flow operators — Kotlin Documentation](https://kotlinlang.org/docs/coroutines-flow-operators.html)  
> **Thư viện áp dụng:** `kotlinx.coroutines:1.11.0`  
> **Mục tiêu:** Nắm vững toàn bộ hệ thống toán tử trong Kotlin Flow theo chuẩn Kotlin 2.x: toán tử trung gian (biến đổi, lọc, xử lý đồng thời/backpressure, kết hợp, vòng đời), toán tử kết thúc (`collect`, `collectLatest`, `first`, `toList`, `reduce`/`fold`), thu thập trong một `CoroutineScope` cụ thể (`launchIn`), cơ chế dung hợp toán tử (operator fusion), và cách tự xây dựng các toán tử tùy biến (custom operators).

---

## 1. Tổng quan về Toán tử trong Flow (Flow Operators Overview)

Các toán tử trong Flow cho phép bạn biến đổi và xử lý các giá trị trong một đường ống luồng (flow pipeline). Kotlin cung cấp hai loại toán tử chính:

- **Toán tử Trung gian (Intermediate Operators)**: Trả về một luồng hạ nguồn (downstream flow) mới tiêu thụ các giá trị từ các luồng thượng nguồn (upstream flows) và áp dụng một thao tác xử lý lên chúng. Bạn có thể kết nối (chain) nhiều toán tử trung gian liên tiếp để xây dựng một luồng xử lý trước khi thu thập kết quả cuối cùng. Các toán tử này mang tính chất **nguội (cold)** — chúng không tự thực thi cho tới khi gặp một toán tử kết thúc.
- **Toán tử Kết thúc (Terminal Operators)**: Kích hoạt quá trình thực thi của đường ống luồng bằng cách thu thập (collect) luồng thượng nguồn. Chúng có thể tiêu thụ các giá trị được phát ra, trả về một kết quả tổng hợp dựa trên các giá trị đã thu thập, hoặc khởi chạy việc thu thập luồng trong một `CoroutineScope` cụ thể.

Mặc dù thư viện `kotlinx.coroutines` cung cấp một tập hợp phong phú các toán tử dựng sẵn, bạn cũng hoàn toàn có thể tự định nghĩa các toán tử tùy biến (custom operators) khi cần các hành vi chuyên biệt.

> [!TIP]
> Các phần dưới đây sẽ trình bày song song cả mã nguồn triển khai tùy biến mẫu (custom implementation) cùng với các toán tử dựng sẵn tương ứng của Kotlin để giúp bạn hiểu sâu sắc nguyên lý hoạt động bên dưới (Under The Hood).

---

## 2. Toán tử Trung gian (Intermediate Operators)

Toán tử trung gian nhận vào một luồng thượng nguồn và trả về một luồng hạ nguồn mới. Chúng có thể được phân loại theo mục đích sử dụng như sau:

1. **Toán tử Biến đổi (Transforming operators)**: Biến đổi giá trị trước khi phát tiếp xuống hạ nguồn.
2. **Toán tử Lọc và Giới hạn Kích thước (Filtering and size-limiting operators)**: Kiểm soát phần tử nào được phép đi tiếp xuống hạ nguồn.
3. **Toán tử Xử lý Đồng thời (Concurrent processing operators)**: Tách biệt tiến trình phát dữ liệu (emission) khỏi tiến trình thu thập (collection) bằng bộ đệm (buffer).
4. **Toán tử Kết hợp (Combining operators)**: Thu thập dữ liệu từ nhiều luồng thượng nguồn và phát vào một luồng hạ nguồn duy nhất.
5. **Toán tử Vòng đời (Lifecycle operators)**: Chạy các hành động phản hồi lại các sự kiện cụ thể trong suốt vòng đời thu thập luồng.

---

### 2.1 Toán tử Biến đổi Dữ liệu (Transforming Operators)

Toán tử biến đổi làm thay đổi các giá trị được phát ra bởi luồng thượng nguồn. Bạn có thể sử dụng chúng để chuyển đổi kiểu dữ liệu, bỏ qua giá trị, hoặc phát thêm nhiều giá trị xuống hạ nguồn.

> [!NOTE]
> Các toán tử biến đổi chấp nhận các suspending lambda, do đó bên trong lambda bạn có thể gọi bất kỳ hàm `suspend` nào trong khi xử lý từng giá trị phát ra. Chúng vẫn xử lý tuần tự (sequential) trừ khi đường ống sử dụng toán tử giới thiệu tính đồng thời (concurrency).

#### Toán tử Tổng quát `.transform()`
Toán tử `.transform()` là toán tử biến đổi tổng quát nhất, đóng vai trò nền tảng để xây dựng nên các toán tử chuyên biệt hơn như `.map()` hay `.filter()`.

Với `.transform()`, bạn có quyền tự do gọi `emit()` 0 lần, 1 lần hoặc nhiều lần cho mỗi phần tử đầu vào.

Ví dụ dưới đây sử dụng `.transform()` để phát ra mỗi giá trị thượng nguồn với số lần lặp lại tương ứng bằng chính giá trị đó:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của toán tử .transform() mặc định
inline fun <T, R> Flow<T>.myTransform(
    // Nhận một suspending lambda có khả năng phát giá trị xuống hạ nguồn
    crossinline transform: suspend FlowCollector<R>.(value: T) -> Unit
): Flow<R> = flow {
    // Thu thập giá trị từ luồng thượng nguồn (upstream)
    this@myTransform.collect { value ->
        // Áp dụng phép biến đổi và phát giá trị vào luồng hạ nguồn (downstream)
        this@flow.transform(value)
    }
}

// Sử dụng toán tử .transform() mặc định
suspend fun main() = withContext(Dispatchers.Default) {
    val flow = (0..4).asFlow().transform { value ->
        // Phát mỗi giá trị lặp lại số lần bằng chính giá trị đó
        repeat(value) {
            emit(value)
        }
    }
    println(flow.toList())
    // [1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
}
```

**Kết quả Console Output:**
```none
[1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
```

#### Toán tử `.map()`
Bạn có thể sử dụng toán tử `.map()` để biến đổi mỗi giá trị thượng nguồn thành một giá trị hạ nguồn (ánh xạ 1-1).

Ví dụ nhân mỗi phần tử với 4:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của toán tử .map() mặc định
inline fun <T, R> Flow<T>.myMap(
    crossinline transform: suspend (value: T) -> R
): Flow<R> = transform { value ->
    emit(transform(value))
}

suspend fun main() = withContext(Dispatchers.Default) {
    // Nhân mỗi giá trị thượng nguồn với 4
    val flow = (0..4).asFlow().map { it * 4 }
    println(flow.toList())
    // [0, 4, 8, 12, 16]
}
```

**Kết quả Console Output:**
```none
[0, 4, 8, 12, 16]
```

#### Toán tử `.filter()`
Để chỉ phát các giá trị thượng nguồn thỏa mãn một điều kiện nhất định (predicate), sử dụng toán tử `.filter()`.

Ví dụ chỉ giữ lại các giá trị chia cho 3 dư 1:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của toán tử .filter() mặc định
inline fun <T> Flow<T>.myFilter(
    crossinline predicate: suspend (value: T) -> Boolean
): Flow<T> = transform { value ->
    // Chỉ phát giá trị khi thỏa mãn điều kiện
    if (predicate(value))
        emit(value)
}

suspend fun main() = withContext(Dispatchers.Default) {
    // Chỉ phát các số chia cho 3 có số dư là 1
    val flow = (0..10).asFlow().filter { it % 3 == 1 }
    println(flow.toList())
    // [1, 4, 7, 10]
}
```

**Kết quả Console Output:**
```none
[1, 4, 7, 10]
```

#### Toán tử `.mapNotNull()`
Một số toán tử có thể kết hợp hành vi của cả `.map()` và `.filter()`, bằng cách vừa biến đổi vừa chỉ phát kết quả không null.

Ví dụ chuyển chuỗi sang kiểu `Double` và tự động loại bỏ các chuỗi không hợp lệ:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của toán tử .mapNotNull() mặc định
inline fun <T, R: Any> Flow<T>.myMapNotNull(
    crossinline transform: suspend (value: T) -> R?
): Flow<R> = transform { value ->
    transform(value)?.let { transformed ->
        emit(transformed)
    }
}

suspend fun main() = withContext(Dispatchers.Default) {
    // Chuyển đổi mỗi chuỗi thành Double và bỏ qua các giá trị không thể chuyển đổi
    val flow = flowOf("1.2", "10", "11", "error", "0.000")
        .mapNotNull { it.toDoubleOrNull() }
    
    println(flow.toList())
    // [1.2, 10.0, 11.0, 0.0]
}
```

**Kết quả Console Output:**
```none
[1.2, 10.0, 11.0, 0.0]
```

---

### 2.2 Toán tử Lọc & Giới hạn Kích thước (Filtering and Size-limiting Operators)

Các toán tử này kiểm soát những giá trị nào được tiếp tục đi xuống hạ nguồn: loại bỏ phần tử trùng lặp liên tiếp, bỏ qua các phần tử đầu tiên, hoặc hủy việc thu thập sau một số lượng phần tử nhất định.

#### Toán tử `.distinctUntilChanged()`
Toán tử `.distinctUntilChanged()` lọc bỏ các giá trị trùng lặp xuất hiện **liên tiếp** nhau. Nó chỉ phát một giá trị nếu giá trị đó khác với giá trị vừa phát trước đó:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của .distinctUntilChanged()
fun <T> Flow<T>.myDistinctUntilChanged(): Flow<T> = flow {
    var lastEmitted: Any? = Any() // Giá trị khởi tạo không bằng bất kỳ giá trị nào khác
    this@myDistinctUntilChanged.collect { value ->
        if (lastEmitted != value) {
            this@flow.emit(value)
            lastEmitted = value
        }
    }
}

suspend fun main() = withContext(Dispatchers.Default) {
    // Loại bỏ các giá trị trùng lặp liên tiếp
    val flow = flowOf(1, 2, 3, 3, 3, 4, 5, 5, 1).distinctUntilChanged()
    println(flow.toList())
    // [1, 2, 3, 4, 5, 1]
}
```

**Kết quả Console Output:**
```none
[1, 2, 3, 4, 5, 1]
```

> [!NOTE]
> Số `1` ở cuối danh sách vẫn được phát vì nó không đứng liền kề với số `1` ở đầu danh sách.

#### Toán tử `.drop()`
Toán tử `.drop(count)` bỏ qua `count` phần tử đầu tiên được phát bởi luồng thượng nguồn và phát toàn bộ các phần tử còn lại xuống hạ nguồn:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của .drop()
fun <T> Flow<T>.myDrop(count: Int): Flow<T> = flow {
    require(count >= 0)
    var elementsAlreadyDropped = 0
    this@myDrop.collect { value ->
        if (elementsAlreadyDropped == count) {
            this@flow.emit(value)
        } else {
            ++elementsAlreadyDropped
        }
    }
}

suspend fun main() = withContext(Dispatchers.Default) {
    // Bỏ qua 2 giá trị đầu tiên từ upstream
    val flow = flowOf(1, 2, 3, 4, 5).drop(2)
    println(flow.toList())
    // [3, 4, 5]
}
```

**Kết quả Console Output:**
```none
[3, 4, 5]
```

#### Toán tử `.take()`
Để hủy việc thu thập sau một số lượng phần tử cố định, sử dụng toán tử `.take(count)`. Cơ chế hủy của nó dựa trên việc ném ra một `CancellationException` nội bộ khi đã lấy đủ số phần tử:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Bản triển khai tùy biến thu gọn của .take()
fun <T> Flow<T>.myTake(count: Int): Flow<T> = flow {
    require(count > 0)
    val cancellationException = CancellationException()
    var elementsRemaining = count
    try {
        this@myTake.collect {
            emit(it)
            --elementsRemaining
            if (elementsRemaining == 0) {
                // Hủy luồng thượng nguồn sau khi đã nhận đủ số lượng phần tử yêu cầu
                throw cancellationException
            }
        }
    } catch (e: Throwable) {
        if (e === cancellationException) {
            // Xử lý ngoại lệ hủy luồng, kết thúc bình thường
        } else {
            // Ném lại nếu là ngoại lệ bất ngờ khác
            throw e
        }
    }
}

suspend fun main() = withContext(Dispatchers.Default) {
    // Chỉ thu thập đúng 3 phần tử đầu tiên từ upstream (dù upstream có 1001 phần tử)
    val flow = (0..1000).asFlow().take(3)

    println(flow.toList())
    // [0, 1, 2]
}
```

**Kết quả Console Output:**
```none
[0, 1, 2]
```

---

### 2.3 Toán tử Xử lý Đồng thời & Áp lực Ngược (Concurrent Processing Operators)

Mặc định, một đường ống Flow xử lý các giá trị hoàn toàn **tuần tự (sequentially)**: luồng thượng nguồn phát ra một giá trị, bên thu thập xử lý xong giá trị đó, rồi luồng thượng nguồn mới phát tiếp giá trị tiếp theo.

Để luồng thượng nguồn có thể chạy đồng thời với bên thu thập hạ nguồn, chúng ta cần đưa vào một **bộ đệm (buffer)**. Bộ đệm này lưu giữ các giá trị mà thượng nguồn đã phát ra nhưng bên thu thập vẫn chưa kịp xử lý.

#### Toán tử `.buffer()` & Cấu hình Kích thước Đệm
Toán tử `.buffer()` cho phép cấu hình dung lượng bộ đệm (capacity) và hành vi khi bộ đệm bị đầy:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun main() = withContext(Dispatchers.Default) {
    flow {
        repeat(10) {
            emit(it)
            println("Emitted $it!")
        }
    }
        // Cho phép thượng nguồn phát trước tối đa 4 giá trị so với tốc độ của bên thu thập
        .buffer(4)
        .collect {
            println("Processed $it!")
            delay(20.milliseconds)
        }
}
```

#### Xử lý Khi Tràn Bộ Đệm: `onBufferOverflow`
Khi bên thu thập chậm hơn luồng phát, đường ống cần cơ chế xử lý các phần tử dôi dư.
- Mặc định, bên thu thập áp dụng **áp lực ngược (backpressure)**: luồng phát sẽ bị tạm ngưng (suspend) khi bộ đệm đầy và chỉ tiếp tục chạy khi bên thu thập giải phóng khoảng trống.
- Để **bỏ rơi phần tử (drop values)** thay vì ngưng luồng phát, bạn truyền tham số `onBufferOverflow`:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.BufferOverflow
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun main() = withContext(Dispatchers.Default) {
    flow {
        repeat(10) {
            emit(it)
            println("Emitted $it!")
        }
    }
        // Lưu trữ tối đa 4 giá trị; khi đệm đầy, bỏ qua phần tử cũ nhất (DROP_OLDEST)
        .buffer(4, onBufferOverflow = BufferOverflow.DROP_OLDEST)
        .collect { value ->
            println("Processed $value!")
            delay(20.milliseconds)
        }
}
```

#### Toán tử `.conflate()`
Toán tử `.conflate()` là cú pháp viết tắt của `.buffer(1, onBufferOverflow = BufferOverflow.DROP_OLDEST)`. Nó được sử dụng khi bạn chỉ quan tâm đến giá trị mới nhất và sẵn sàng bỏ qua các giá trị trung gian phát ra trong lúc giá trị trước đó đang được thu thập:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun main() = withContext(Dispatchers.Default) {
    flow {
        repeat(10) {
            emit(it)
            println("Emitted $it!")
        }
    }.conflate().collect {
        println("Processed $it!")
        delay(20.milliseconds)
    }
}
```

> [!IMPORTANT]
> **Khác biệt cốt lõi giữa `conflate()` và `collectLatest()`:**  
> - `.conflate()` chỉ tác động lên việc **lựa chọn phần tử trong bộ đệm** để chuyển tới collector: nếu collector đang bận xử lý, tác vụ xử lý đó **vẫn tiếp tục chạy bình thường đến hết**. Khi xong việc, collector sẽ nhảy cóc tới giá trị mới nhất.
> - `collectLatest()` sẽ **hủy bỏ ngay lập tức (cancel)** tác vụ xử lý đang dở dang của phần tử trước đó ngay thời điểm phần tử mới xuất hiện!

#### Toán tử `.flowOn()` và Dung Hợp Toán Tử (Operator Fusion)
Trong các ví dụ trước, `.buffer()` và `.conflate()` chạy luồng thượng nguồn đồng thời trong một coroutine riêng biệt nhưng **không làm thay đổi `CoroutineContext`** của nó.

Để chạy luồng thượng nguồn trong một context khác (ví dụ đổi Dispatcher sang `Dispatchers.IO`), ta dùng `.flowOn()`. Khi `.flowOn()` làm thay đổi dispatcher, nó sẽ thu thập luồng thượng nguồn trong một coroutine riêng biệt và tạo một bộ đệm trung gian giữa luồng phát thượng nguồn và luồng thu thập hạ nguồn:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun main() = withContext(Dispatchers.Default) {
    flow {
        repeat(10) {
            emit(it)
        }
        println("Finished emitting!")
    }.flowOn(Dispatchers.IO).collect {
        println("Received $it!")
        delay(10.milliseconds)
    }
}
```

#### Tối ưu với Operator Fusion
Để vừa chỉ định CoroutineContext vừa cấu hình cơ chế đệm cho luồng thượng nguồn, bạn có thể kết hợp `.flowOn()` cùng với `.buffer()` hoặc `.conflate()`. Khi sử dụng cùng nhau, các toán tử này sẽ thực hiện cơ chế **dung hợp toán tử (operator fusion)** và dùng chung một bộ đệm duy nhất thay vì tạo nhiều tầng đệm lồng nhau.

Ví dụ dưới đây đọc tín hiệu từ cảm biến nhiệt độ nhà thông minh trên `Dispatchers.IO` và áp dụng `.conflate()` để chỉ gửi dữ liệu nhiệt độ mới nhất:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.random.Random
import kotlin.time.Duration.Companion.milliseconds
import kotlin.math.round

fun awaitSensorSignal(): SensorSignal {
    Thread.sleep(10)
    val reading = round(Random.nextDouble(25.0, 100.0) * 100.0) / 100.0
    println("Measured $reading as the temperature")
    return SensorSignal(temperatureCelsius = reading)
}

data class SensorSignal(
    val temperatureCelsius: Double
)

suspend fun sendLatestTemperature(temperatureCelsius: Double) {
    println("Starting to send $temperatureCelsius...")
    delay(50.milliseconds)
    println("Sent $temperatureCelsius.")
}

suspend fun main() = withContext(Dispatchers.Default) {
    val smartHomeTemperatureFlow = flow {
        while (true) {
            val signal = awaitSensorSignal()
            emit(signal.temperatureCelsius)
            println("Emitted $signal")
        }
    }
        // Chạy thượng nguồn trên Dispatchers.IO
        .flowOn(Dispatchers.IO)
        // Giữ giá trị mới nhất trong đệm và bỏ qua các giá trị cũ (Operator Fusion xảy ra tại đây!)
        .conflate()
        // Chỉ lấy 2 giá trị đầu tiên
        .take(2)
        .collect { temperature ->
            println("Received $temperature!")
            sendLatestTemperature(temperature)
        }
}
```

---

### 2.4 Toán tử Kết hợp Nhiều Luồng (Combining Operators)

Toán tử kết hợp tiêu thụ dữ liệu từ nhiều luồng thượng nguồn và gom thành một luồng hạ nguồn duy nhất. Ba toán tử kết hợp thông dụng nhất trong Kotlin Flow là `.zip()`, `.combine()` và `merge()`.

---

#### 1. Toán tử `.zip()`: Ghép Cặp 1-1 Theo Thứ Tự

Toán tử `.zip()` ghép cặp các phần tử tương ứng của hai luồng theo thứ tự tuần tự: phần tử thứ nhất của luồng 1 ghép với phần tử thứ nhất của luồng 2, phần tử thứ hai ghép với phần tử thứ hai, v.v.

- **Đồng bộ từng cặp (Lockstep):** Nếu một luồng phát nhanh hơn, nó bắt buộc phải đợi luồng còn lại phát ra phần tử cùng số thứ tự trước khi lambda kết hợp được thực thi.
- **Điều kiện kết thúc:** Luồng kết quả sẽ **hoàn thành ngay khi một trong hai luồng kết thúc** (các phần tử dư thừa của luồng dài hơn sẽ bị bỏ qua).

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.TimeSource

suspend fun main() = withContext(Dispatchers.Default) {
    // Phát ra một nhịp sau mỗi 100ms
    val tickerFlow = flow {
        while (true) {
            emit(Unit)
            delay(100.milliseconds)
        }
    }

    val start = TimeSource.Monotonic.markNow()
    tickerFlow
        // Ghép mỗi nhịp ticker với một con số tiếp theo
        .zip(flowOf(1, 2, 3)) { _, value ->
            value
        }.collect {
            println("${start.elapsedNow()}: received $it")
        }
}
```

**Kết quả Console Output:**
```none
104.2ms: received 1
207.8ms: received 2
311.5ms: received 3
```

---

#### 2. Toán tử `.combine()`: Kết hợp Giá trị Mới nhất

Toán tử `.combine()` phát ra một giá trị mới bất cứ khi nào **bất kỳ luồng thượng nguồn nào phát ra giá trị mới**, sử dụng giá trị mới nhất hiện tại (latest value) của từng luồng:

- **Khởi đầu:** Cần đợi tất cả các luồng phát ra **ít nhất 1 giá trị đầu tiên** thì `.combine()` mới bắt đầu phát giá trị kết hợp ban đầu.
- **Cập nhật độc lập:** Sau đó, bất cứ khi nào một luồng có giá trị mới, `.combine()` lập tức lấy giá trị mới đó ghép với giá trị mới nhất gần nhất của luồng kia để phát tiếp.
- **Điều kiện kết thúc:** Luồng kết quả chỉ kết thúc khi **toàn bộ các luồng tham gia đều đã hoàn thành**.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

enum class Theme {
    Dark,
    Light,
}

data class UiState(
    val messages: List<String>,
    val theme: Theme,
)

val messagesFlow = MutableStateFlow(
    listOf(
        "Hello!",
        "Is anyone here?",
    )
)

val themeFlow = MutableStateFlow(
    Theme.Light
)

// Kết hợp giá trị mới nhất từ cả hai StateFlow
val uiStateFlow = combine(messagesFlow, themeFlow) { messages, theme ->
    UiState(messages, theme)
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        // Dùng UNDISPATCHED để coroutine lắng nghe đăng ký trước khi sự kiện cập nhật đầu tiên xảy ra
        val uiUpdateJob = launch(start = CoroutineStart.UNDISPATCHED) {
            uiStateFlow.collect {
                // Vẽ giao diện UI
                println(it)
            }
        }
        messagesFlow.update { messages -> messages + "I'll be back!" }
        delay(100.milliseconds)
        
        themeFlow.value = Theme.Dark
        delay(100.milliseconds)
        
        uiUpdateJob.cancel()
    }
}
```

**Kết quả Console Output:**
```none
UiState(messages=[Hello!, Is anyone here?], theme=Light)
UiState(messages=[Hello!, Is anyone here?, I'll be back!], theme=Light)
UiState(messages=[Hello!, Is anyone here?, I'll be back!], theme=Dark)
```

---

#### 3. Toán tử `merge()`: Trộn Luồng Đa Nguồn Đồng Thời

Khác với `.zip()` và `.combine()` (vốn gom các giá trị lại thành một đối tượng mới qua một lambda biến đổi), hàm `merge()` nhận nhiều luồng có **cùng kiểu dữ liệu** và **trộn chúng vào một luồng duy nhất**.

- **Không ghép cặp (No aggregation):** Giá trị của bất kỳ luồng nào đến trước sẽ được phát ngay lập tức xuống hạ nguồn mà không cần chờ đợi luồng khác.
- **Bảo toàn thứ tự thời gian:** Phản ánh đúng thời điểm phát sinh thực tế của các sự kiện từ nhiều nguồn khác nhau.
- **Điều kiện kết thúc:** Luồng kết quả kết thúc khi **tất cả các luồng con đều đã kết thúc**.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

sealed interface UiEvent {
    object ClickEvent : UiEvent { override fun toString() = "ClickEvent" }
    object RightClickEvent : UiEvent { override fun toString() = "RightClickEvent" }
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        val clickFlow = MutableSharedFlow<UiEvent>()
        val rightClickFlow = MutableSharedFlow<UiEvent>()

        coroutineScope {
            val collectJob = launch(start = CoroutineStart.UNDISPATCHED) {
                // Thu thập đồng thời cả hai luồng sự kiện và đẩy xuống hạ nguồn
                merge(clickFlow, rightClickFlow).collect {
                    println("Observed an event: $it")
                }
            }
            clickFlow.emit(UiEvent.ClickEvent)
            delay(100.milliseconds)
            
            clickFlow.emit(UiEvent.ClickEvent)
            delay(100.milliseconds)
            
            rightClickFlow.emit(UiEvent.RightClickEvent)
            delay(100.milliseconds)
            
            collectJob.cancel()
        }
    }
}
```

**Kết quả Console Output:**
```none
Observed an event: ClickEvent
Observed an event: ClickEvent
Observed an event: RightClickEvent
```

---

#### 4. Bảng So Sánh Chuyên Sâu: `zip` vs `combine` vs `merge`

Để lựa chọn chính xác toán tử phù hợp cho bài toán thực tế, hãy xem xét sơ đồ trục thời gian (Timeline) và bảng đối chiếu dưới đây:

##### Sơ đồ Trục Thời Gian (Marble / Timeline Diagram)

Giả sử chúng ta có 2 luồng:
- **`Flow A`**: phát các số `1`, `2`, `3`
- **`Flow B`**: phát các chữ cái `"A"`, `"B"`, `"C"`, `"D"`

```
Thời gian (ms) ──► 0ms    100ms   150ms   200ms   300ms   450ms   600ms
Flow A (mỗi 100ms): ───────1───────────────2───────3| (kết thúc ở 300ms)
Flow B (mỗi 150ms): ───────────────A───────────────B───────C───────D| (kết thúc ở 600ms)
─────────────────────────────────────────────────────────────────────────────────
.zip(A, B)        : ──────────────(1,A)───────────(2,B)───(3,C)| (D bị bỏ qua vì Flow A đã kết thúc!)
.combine(A, B)    : ──────────────(1,A)───(2,A)───(3,B)───(3,C)───(3,D)|
merge(A, B)       : ───────1───────A───────2───────3───B───C───────D|
```

##### Bảng Đối Chiếu Chi Tiết

| Tiêu chí | `zip()` | `combine()` | `merge()` |
| :--- | :--- | :--- | :--- |
| **Cơ chế cốt lõi** | Ghép cặp nghiêm ngặt 1-1 theo thứ tự chỉ mục (`index`). | Kết hợp các giá trị **mới nhất gần nhất** của từng luồng. | Trộn nhiều luồng thành một luồng duy nhất không gom cặp. |
| **Hàm kết hợp (Transform)** | Có `(T1, T2) -> R` để tạo ra đối tượng mới. | Có `(T1, T2) -> R` để tạo ra đối tượng mới. | **Không có**. Chỉ phát chuyển tiếp giá trị `T` ban đầu. |
| **Thời điểm phát giá trị** | Chỉ phát khi **cả 2 luồng** đều phát ra phần tử ở cùng số thứ tự. | Phát khi **bất kỳ luồng nào** có giá trị mới (sau khi cả 2 đã có ít nhất 1 giá trị). | Phát **ngay lập tức** khi có bất kỳ phần tử nào xuất hiện ở bất kỳ luồng nào. |
| **Khi một luồng phát nhanh hơn** | Luồng nhanh bị hoãn (suspend) chờ luồng chậm bắt kịp cặp. | Luồng nhanh phát ra giá trị mới, ghép với giá trị cũ nhất của luồng chậm. | Không ảnh hưởng nhau, phần tử đến lúc nào được emit lúc đó. |
| **Điều kiện hoàn thành** | Kết thúc ngay khi **một trong hai luồng kết thúc**. | Kết thúc khi **toàn bộ các luồng** đã hoàn thành. | Kết thúc khi **toàn bộ các luồng** đã hoàn thành. |
| **Kiểu dữ liệu đầu vào** | Có thể khác nhau (`Flow<T1>`, `Flow<T2>`). | Có thể khác nhau (`Flow<T1>`, `Flow<T2>`). | Bắt buộc phải **cùng một kiểu chung** (`Flow<T>`). |
| **Ca sử dụng thực tế (Use Case)** | Ghép cặp dữ liệu có thứ tự 1-1 (ví dụ: danh sách ID bài viết và danh sách Thumbnail tương ứng). | Kết hợp nhiều nguồn trạng thái UI (Form validation: username + password; Filter + Search Query). | Hợp nhất nhiều kênh sự kiện độc lập vào một luồng xử lý chung (Click buttons, WebSocket events, Notifications). |

---

#### 5. Mã Nguồn Demo Đối Chiếu Trực Quan Cả Ba Toán Tử

Dưới đây là chương trình hoàn chỉnh chạy thực nghiệm trực tiếp trên cùng một tập dữ liệu đầu vào có độ trễ thời gian, giúp bạn quan sát chính xác sự khác biệt về kết quả đầu ra:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.TimeSource

fun numbersFlow(): Flow<Int> = flow {
    delay(100.milliseconds)
    emit(1)
    delay(100.milliseconds)
    emit(2)
    delay(100.milliseconds)
    emit(3)
}

fun lettersFlow(): Flow<String> = flow {
    delay(150.milliseconds)
    emit("A")
    delay(150.milliseconds)
    emit("B")
    delay(150.milliseconds)
    emit("C")
    delay(150.milliseconds)
    emit("D")
}

suspend fun main() = coroutineScope {
    println("=== 1. THỬ NGHIỆM VỚI .zip() ===")
    val startZip = TimeSource.Monotonic.markNow()
    numbersFlow()
        .zip(lettersFlow()) { num, letter -> "($num, $letter)" }
        .collect { println("[+${startZip.elapsedNow().inWholeMilliseconds}ms] ZIP: $it") }

    println("\n=== 2. THỬ NGHIỆM VỚI .combine() ===")
    val startCombine = TimeSource.Monotonic.markNow()
    numbersFlow()
        .combine(lettersFlow()) { num, letter -> "($num, $letter)" }
        .collect { println("[+${startCombine.elapsedNow().inWholeMilliseconds}ms] COMBINE: $it") }

    println("\n=== 3. THỬ NGHIỆM VỚI merge() ===")
    val startMerge = TimeSource.Monotonic.markNow()
    // Map cả 2 luồng về cùng kiểu String để có thể merge
    val stringNumbers = numbersFlow().map { "Số $it" }
    val stringLetters = lettersFlow().map { "Chữ $it" }
    
    merge(stringNumbers, stringLetters)
        .collect { println("[+${startMerge.elapsedNow().inWholeMilliseconds}ms] MERGE: $it") }
}
```

**Kết quả Console Output Minh Họa Chuẩn Xác:**

```none
=== 1. THỬ NGHIỆM VỚI .zip() ===
[+155ms] ZIP: (1, A)
[+310ms] ZIP: (2, B)
[+460ms] ZIP: (3, C)

=== 2. THỬ NGHIỆM VỚI .combine() ===
[+154ms] COMBINE: (1, A)
[+206ms] COMBINE: (2, A)
[+312ms] COMBINE: (3, B)
[+464ms] COMBINE: (3, C)
[+615ms] COMBINE: (3, D)

=== 3. THỬ NGHIỆM VỚI merge() ===
[+105ms] MERGE: Số 1
[+157ms] MERGE: Chữ A
[+208ms] MERGE: Số 2
[+310ms] MERGE: Số 3
[+312ms] MERGE: Chữ B
[+463ms] MERGE: Chữ C
[+616ms] MERGE: Chữ D
```

##### Phân Tích Chi Tiết Kết Quả Thực Nghiệm:

1. **Tại sao `zip` chỉ in ra 3 dòng và dừng ở `460ms`?**  
   - `numbersFlow` chỉ có 3 phần tử (1, 2, 3) và kết thúc ở 300ms.  
   - Cặp thứ ba `(3, C)` được ghép khi "C" xuất hiện lúc 450ms.  
   - Do `numbersFlow` đã cạn dữ liệu, `zip` lập tức đóng luồng, khiến phần tử `"D"` phát ra lúc 600ms của `lettersFlow` bị bỏ qua hoàn toàn.

2. **Tại sao `combine` in ra `(2, A)` lúc `206ms`?**  
   - Ở 100ms, số `1` phát ra nhưng chữ cái chưa có gì nên `combine` chưa thể ghép.  
   - Ở 150ms, chữ `"A"` phát ra -> `combine` có đủ 2 bên nên phát ngay `(1, A)`.  
   - Ở 200ms, số `2` phát ra trong khi chữ mới nhất vẫn đang là `"A"` -> `combine` lập tức phát `(2, A)` mà không cần đợi chữ `"B"`!

3. **Tại sao `merge` in ra 7 dòng độc lập?**  
   - `merge` không hề tạo cặp `(x, y)`. Bất kể là số hay chữ, luồng nào đến thời điểm phát là dữ liệu được đẩy thẳng xuống collector theo đúng trình tự thời gian xuất hiện (100ms -> 150ms -> 200ms -> 300ms -> 300ms -> 450ms -> 600ms).

---

### 2.5 Toán tử Vòng đời (Lifecycle Operators)

Toán tử vòng đời nhận vào một suspending lambda và kích hoạt nó tại những thời điểm xác định trong suốt vòng đời thu thập luồng.

#### Toán tử `.onStart()` & `.onEach()`
- `.onStart()`: Chạy lambda trước khi luồng thượng nguồn bắt đầu được thu thập.
- `.onEach()`: Chạy logic side-effect trước khi mỗi giá trị được đẩy xuống hạ nguồn.

> [!NOTE]
> Tương tự như `.onStart()`, đối với **Hot Flows**, bạn có thể dùng toán tử `.onSubscription()` để thực thi mã sau khi một subscriber bắt đầu thu thập nhưng trước khi nó nhận bất kỳ giá trị nào phát ra.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Triển khai tùy biến thu gọn của .onStart()
fun <T> Flow<T>.myOnStart(
    action: suspend FlowCollector<T>.() -> Unit
): Flow<T> = flow {
    this@flow.action()
    this@myOnStart.collect(this@flow)
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        flowOf("Page 1", "Page 2", "Page 3").onStart {
            println("Processing pages!")
        }.onEach {
            println("Emitted $it")
        }.collect {
            println("Collected $it")
        }
    }
}
```

#### Toán tử `.onCompletion()`: Vòng Đời Kết Thúc Luồng

Toán tử `.onCompletion()` đóng vai trò tương tự như khối `finally` trong cấu trúc `try-finally`, được kích hoạt sau khi quá trình thu thập luồng hoàn thành — bất kể luồng kết thúc thành công bình thường, bị gián đoạn do ngoại lệ, hay bị hủy bỏ bởi CoroutineScope.

Lambda của `.onCompletion()` nhận một tham số mở rộng `action: suspend FlowCollector<T>.(cause: Throwable?) -> Unit`. Nó mang hai năng lực đặc biệt:
1. **Kiểm tra nguyên nhân kết thúc (`cause`):**
   - `cause == null`: Luồng kết thúc **thành công bình thường** (clean termination).
   - `cause is CancellationException`: Luồng bị **hủy bỏ** (do downstream gọi `cancel()`, hết thời gian `withTimeout`, hoặc dùng toán tử cắt luồng như `take(n)`).
   - `cause is Throwable` (khác Cancellation): Luồng bị gián đoạn bởi một **ngoại lệ phát sinh ở upstream**.
2. **Khả năng phát thêm giá trị (`emit`):** Khi luồng kết thúc thành công (`cause == null`), bạn có thể gọi `emit(...)` để phát thêm một phần tử cuối cùng (ví dụ: Footer, Sentinel Value) xuống hạ nguồn.

> [!IMPORTANT]
> **Nguyên tắc Exception Transparency:**  
> `.onCompletion()` chỉ là một toán tử quan sát (observer / interceptor), nó **không nuốt ngoại lệ**. Nếu thượng nguồn gặp lỗi, `.onCompletion(cause)` sẽ chạy để bạn dọn dẹp tài nguyên, sau đó ngoại lệ vẫn sẽ tiếp tục được chuyển tiếp xuống hạ nguồn (trừ khi có toán tử `.catch()` xử lý sau đó).

---

##### 1. `onCompletion` Xảy Ra Khi Nào? Phân Tích Giữa Cold Flow và Hot Flow

Sự khác biệt về vòng đời giữa Cold Flow và Hot Flow tạo nên hành vi hoàn toàn khác nhau của `.onCompletion()`:

| Tiêu chí | Cold Flow (Luồng lạnh) | Hot Flow (`StateFlow` / `SharedFlow`) |
| :--- | :--- | :--- |
| **Bản chất luồng** | **Hữu hạn hoặc theo nhu cầu** (Finite / On-demand). | **Vô hạn** (Infinite / Never completes naturally). Luôn tồn tại trong bộ nhớ. |
| **Khi kết thúc tự nhiên** | **CÓ XẢY RA.** Khi khối `flow { ... }` chạy hết dòng mã cuối cùng, hoặc các luồng hữu hạn như `flowOf(1, 2, 3)` cạn phần tử. Khi đó: `cause == null`. | **KHÔNG BAO GIỜ.** Hot Flow đại diện cho một kênh phát sóng liên tục, nó **không bao giờ tự kết thúc**. Khối `onCompletion` sẽ không bao giờ chạy trong điều kiện bình thường! |
| **Khi bị hủy bởi Scope** | **CÓ XẢY RA.** Khi coroutine thu thập (`collect`) bị hủy hoặc timeout. Khi đó: `cause is CancellationException`. | **ĐÂY LÀ TRƯỜNG HỢP DUY NHẤT** mà `onCompletion` trên Hot Flow được kích hoạt! Xảy ra khi `viewModelScope` bị clear, Activity bị destroy, hoặc `repeatOnLifecycle` rời khỏi trạng thái `STARTED`. Khi đó: `cause is CancellationException`. |
| **Khi gặp ngoại lệ** | **CÓ XẢY RA.** Khi producer hoặc toán tử trung gian ném exception. Khi đó: `cause` là exception đó. | **HIẾM GẶP.** `StateFlow` và `SharedFlow` không ném exception trong quá trình emit thông thường. |

```
So sánh thời điểm kích hoạt onCompletion:

[Cold Flow: flowOf(1, 2)] ──► emit(1) ──► emit(2) ──► [HẾT DỮ LIỆU] ──► onCompletion(cause = null) ✅

[Hot Flow: StateFlow]     ──► emit(A) ──► emit(B) ──► emit(C) ──► ... (Chạy mãi mãi, KHÔNG BAO GIỜ onCompletion!)
                                                                        ▲
                                                                        │ Người dùng thoát màn hình (Scope Cancelled)
                                                                        └─► onCompletion(cause = CancellationException) ✅
```

---

##### 2. Các Ca Ứng Dụng Thực Tế (Practical Android Use Cases)

###### Ứng dụng 1: Quản lý Trạng thái Tải dữ liệu (Ẩn ProgressBar / Loading Spinner)
Khi gọi API hoặc truy vấn dữ liệu, ViewModel thường bật cờ `isLoading = true`. Sử dụng `.onCompletion()` đảm bảo `isLoading` luôn được reset về `false` trong **mọi tình huống** (thành công, mất mạng, hoặc người dùng bấm Back hủy giữa chừng):

```kotlin
class ProductViewModel(private val repository: ProductRepository) : ViewModel() {
    fun loadProducts() {
        viewModelScope.launch {
            repository.getProductsFlow()
                .onStart { _uiState.update { it.copy(isLoading = true) } }
                .onCompletion { cause ->
                    // Luôn luôn được gọi: thành công, lỗi mạng, hay bị cancel đều ẩn Loading!
                    _uiState.update { it.copy(isLoading = false) }
                    if (cause != null && cause !is CancellationException) {
                        Log.e("ProductVM", "Lỗi tải sản phẩm: $cause")
                    }
                }
                .catch { e -> emitError(e) }
                .collect { products -> _uiState.update { it.copy(items = products) } }
        }
    }
}
```

###### Ứng dụng 2: Giải phóng Tài nguyên Phần cứng & Kết nối (Resource Cleanup)
Đóng các kết nối Socket, giải phóng camera, dừng theo dõi cảm biến GPS hoặc đóng file stream khi luồng dừng:

```kotlin
fun locationUpdatesFlow(locationManager: LocationManager): Flow<Location> = callbackFlow {
    val callback = object : LocationListener {
        override fun onLocationChanged(loc: Location) { trySend(loc) }
    }
    locationManager.requestLocationUpdates(callback)

    awaitClose {
        // Hoặc dùng .onCompletion { locationManager.removeUpdates(callback) }
        locationManager.removeUpdates(callback)
    }
}
```

###### Ứng dụng 3: Phát Phần tử Báo hiệu Cuối cùng (Footer / Sentinel Emission)
Phát thêm một phần tử đặc biệt xuống UI sau khi toàn bộ dữ liệu đã được tải xong:

```kotlin
flowOf("Page 1", "Page 2", "Page 3")
    .onCompletion { cause ->
        if (cause == null) {
            emit("--- HẾT DANH SÁCH ---")
        }
    }
    .collect { println(it) }
```

---

##### 3. Mã Nguồn Demo Toàn Diện Các Kịch Bản `onCompletion`

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun main() = coroutineScope {
    println("=== KỊCH BẢN 1: Cold Flow hoàn thành tự nhiên ===")
    flowOf(1, 2, 3)
        .onCompletion { cause -> println("-> onCompletion: cause = $cause") }
        .collect { println("Cold item: $it") }

    println("\n=== KỊCH BẢN 2: Cold Flow bị lỗi Upstream ===")
    flow {
        emit("Dữ liệu 1")
        throw IllegalStateException("Lỗi đường truyền mạng!")
    }
    .onCompletion { cause -> println("-> onCompletion bắt được lỗi: ${cause?.message}") }
    .catch { e -> println("-> catch xử lý lỗi an toàn: ${e.message}") }
    .collect { println("Cold item: $it") }

    println("\n=== KỊCH BẢN 3: Hot Flow (StateFlow) bị Cancel khi Scope hủy ===")
    val hotStateFlow = MutableStateFlow("Giá trị ban đầu")
    
    val job = launch {
        hotStateFlow
            .onCompletion { cause -> 
                println("-> Hot Flow onCompletion: cause = ${cause?.javaClass?.simpleName}") 
            }
            .collect { println("Hot item: $it") }
    }

    delay(50.milliseconds)
    hotStateFlow.value = "Giá trị mới sau 50ms"
    delay(50.milliseconds)
    
    // Hủy Coroutine thu thập (tương tự ViewModel bị Clear khi rời màn hình)
    println("Hủy coroutine thu thập Hot Flow...")
    job.cancelAndJoin()
}
```

**Kết quả Console Output Minh Họa:**
```none
=== KỊCH BẢN 1: Cold Flow hoàn thành tự nhiên ===
Cold item: 1
Cold item: 2
Cold item: 3
-> onCompletion: cause = null

=== KỊCH BẢN 2: Cold Flow bị lỗi Upstream ===
Cold item: Dữ liệu 1
-> onCompletion bắt được lỗi: Lỗi đường truyền mạng!
-> catch xử lý lỗi an toàn: Lỗi đường truyền mạng!

=== KỊCH BẢN 3: Hot Flow (StateFlow) bị Cancel khi Scope hủy ===
Hot item: Giá trị ban đầu
Hot item: Giá trị mới sau 50ms
Hủy coroutine thu thập Hot Flow...
-> Hot Flow onCompletion: cause = CancellationException
```

---

#### Toán tử `.onEmpty()`
Toán tử `.onEmpty()` được kích hoạt khi luồng kết thúc mà **không phát ra bất kỳ giá trị nào**:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Triển khai tùy biến thu gọn của .onEmpty()
fun <T> Flow<T>.myOnEmpty(
    action: suspend FlowCollector<T>.() -> Unit
): Flow<T> = flow {
    var emittedSomething = false
    this@myOnEmpty.collect { value ->
        emittedSomething = true
        this@flow.emit(value)
    }
    if (!emittedSomething) {
        action()
    }
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        flowOf("Page 1", "Page 2", "Page 3").onEmpty {
            // Không in gì vì upstream có phát ra giá trị
            println("No pages to load!")
        }.collect()

        flowOf<Int>().onEmpty {
            println("No pages to load!")
            // Sẽ in: No pages to load!
        }.collect()
    }
}
```

---

### 2.6 Toán tử Làm phẳng Luồng (Flattening Operators)

#### 1. Bài toán Luồng Lồng Nhau (`Flow<Flow<T>>`)

Trong quá trình xử lý luồng, việc biến đổi một phần tử thành một luồng dữ liệu khác là trường hợp rất phổ biến. 

Giả sử bạn có một luồng các truy vấn tìm kiếm từ người dùng `Flow<String>`. Với mỗi từ khóa, bạn cần gọi một hàm trả về luồng kết quả từ cơ sở dữ liệu hoặc mạng:
```kotlin
fun searchDatabase(query: String): Flow<SearchResult>
```

Nếu bạn dùng toán tử `.map { query -> searchDatabase(query) }`, kiểu dữ liệu nhận được ở hạ nguồn sẽ là một luồng lồng nhau: **`Flow<Flow<SearchResult>>`**. 

Người tiêu thụ (collector) ở tầng UI không thể dễ dàng làm việc với cấu trúc lồng nhau này. Bạn cần "làm phẳng" (flatten) nó thành một luồng duy nhất: **`Flow<SearchResult>`**.

```
Cơ chế làm phẳng:
Flow<Flow<T>> ──────[Flattening Operator]──────► Flow<T>
(Luồng chứa các luồng con)                       (Một luồng phẳng duy nhất)
```

Đối với Collection đồng bộ, Kotlin chỉ có một hàm `flatMap`. Nhưng với Flow bất đồng bộ, các luồng con có thể phát ra giá trị tại các thời điểm khác nhau với tốc độ khác nhau. Do đó, Kotlin Coroutines cung cấp **3 chiến lược làm phẳng riêng biệt**:
1. **`flatMapConcat`**: Làm phẳng tuần tự (Sequential).
2. **`flatMapMerge`**: Làm phẳng đồng thời (Concurrent).
3. **`flatMapLatest`**: Làm phẳng ưu tiên giá trị mới nhất (Preemptive Cancellation / SwitchMap).

---

#### 2. Chi Tiết Từng Toán Tử Làm Phẳng

##### A. `flatMapConcat` — Nối Luồng Tuần Tự (Sequential Concatenation)

Toán tử `flatMapConcat` thu thập các luồng con một cách **hoàn toàn tuần tự**. Nó mở luồng con đầu tiên, thu thập tất cả các giá trị cho đến khi luồng con đó **kết thúc hoàn toàn**, sau đó mới tiếp tục thu thập luồng con được sinh ra từ phần tử thượng nguồn tiếp theo.

- **Mức độ đồng thời (Concurrency):** Cố định là `1` (không có xử lý song song).
- **Bảo toàn thứ tự:** Tuyệt đối giữ đúng thứ tự xuất hiện của các phần tử.
- **Rủi ro (Pitfall):** Nếu một luồng con là luồng vô hạn (như Hot Flow: `StateFlow`, `SharedFlow`) hoặc chạy rất lâu, nó sẽ **chặn vĩnh viễn (block)** việc thu thập các luồng con phía sau!

```kotlin
flowOf(1, 2, 3)
    .flatMapConcat { id ->
        flow {
            emit("Yêu cầu $id: Bắt đầu")
            delay(100.milliseconds)
            emit("Yêu cầu $id: Kết thúc")
        }
    }
    .collect { println(it) }
```

---

##### B. `flatMapMerge` — Trộn Luồng Đồng Thời (Concurrent Merging)

Toán tử `flatMapMerge` cho phép thu thập **đồng thời (concurrently)** nhiều luồng con cùng một lúc ngay khi các giá trị thượng nguồn phát ra.

- **Tham số `concurrency`:** Nhận vào số lượng luồng con tối đa được phép chạy song song tại một thời điểm (mặc định là `DEFAULT_CONCURRENCY = 16`).
- **Không bảo toàn thứ tự:** Các giá trị từ các luồng con sẽ xen kẽ nhau (interleaved) và được đẩy xuống hạ nguồn ngay khi chúng xuất hiện. Luồng con nào có dữ liệu trả về trước sẽ được phát trước.
- **Ưu thế:** Tối đa hóa thông lượng (throughput) khi cần thực hiện nhiều tác vụ mạng độc lập nhau (ví dụ: tải song song chi tiết của 10 sản phẩm từ 10 ID).

```kotlin
// Cho phép tối đa 4 tác vụ mạng chạy song song cùng lúc
userIdsFlow
    .flatMapMerge(concurrency = 4) { userId ->
        apiService.fetchUserProfileFlow(userId)
    }
    .collect { profile -> renderProfile(profile) }
```

---

##### C. `flatMapLatest` — Ưu Tiên Giá Trị Mới Nhất (Preemptive Cancellation)

Toán tử `flatMapLatest` hoạt động theo cơ chế **hủy bỏ có ưu tiên**: Ngay khi luồng thượng nguồn phát ra một giá trị mới, nó sẽ **hủy bỏ ngay lập tức (cancel)** việc thu thập luồng con trước đó và lập tức chuyển sang thu thập luồng con mới.

- **Tương đương:** Toán tử `switchMap` trong RxJava hoặc LiveData.
- **Vứt bỏ dữ liệu trễ:** Bất kỳ giá trị nào của luồng con cũ chưa kịp phát ra sẽ bị hủy vĩnh viễn.
- **Ứng dụng kinh điển trong Android:**
  - **Thanh tìm kiếm theo thời gian thực (Search-as-you-type):** Khi người dùng gõ "A", API tìm "A" được gọi. Người dùng gõ tiếp "AB", `flatMapLatest` sẽ lập tức hủy request tìm "A" để tránh hiển thị kết quả cũ đè lên kết quả mới và tiết kiệm băng thông.
  - **Chuyển Tab / Chuyển Đổi Profile Người Dùng:** Hủy toàn bộ tiến trình tải dữ liệu của màn hình cũ ngay khi người dùng bấm chọn tab mới.

```kotlin
searchQueryStateFlow
    .debounce(300.milliseconds) // Chờ người dùng dừng gõ 300ms
    .distinctUntilChanged()      // Bỏ qua nếu từ khóa không đổi
    .flatMapLatest { query ->
        // Hủy request tìm kiếm cũ ngay lập tức nếu có query mới!
        searchRepository.searchProductsFlow(query)
    }
    .collect { results -> updateSearchResults(results) }
```

---

#### 3. Bảng Đối Chiếu Toàn Diện 6 Tiêu Chí Giữa 3 Toán Tử

| Tiêu chí | `flatMapConcat` | `flatMapMerge` | `flatMapLatest` |
| :--- | :--- | :--- | :--- |
| **Mô hình thực thi** | Tuần tự nghiêm ngặt (Sequential). | Đồng thời (Concurrent / Parallel). | Hủy bỏ luồng cũ, ưu tiên luồng mới (Preemptive / Switch). |
| **Mức độ đồng thời (Concurrency)** | Cố định `1`. | Mặc định `16` (tùy biến qua tham số). | Luôn chỉ có tối đa `1` luồng con hoạt động tại một thời điểm. |
| **Hành vi khi có giá trị thượng nguồn mới** | Lưu vào hàng đợi, chờ luồng con hiện tại kết thúc mới xử lý. | Khởi chạy ngay luồng con mới song song với các luồng con đang chạy. | **HỦY NGAY LẬP TỨC** luồng con đang chạy để mở luồng con mới. |
| **Thứ tự dữ liệu hạ nguồn** | Bảo toàn tuyệt đối thứ tự ban đầu. | Không bảo toàn (xen kẽ tùy tốc độ phát của từng luồng con). | Chỉ phát dữ liệu của luồng con mới nhất (các dữ liệu trễ của luồng cũ bị vứt bỏ). |
| **Rủi ro lớn nhất** | Bị treo vĩnh viễn nếu luồng con là Hot Flow hoặc không bao giờ kết thúc. | Tốn nhiều bộ nhớ/kết nối mạng nếu số lượng luồng con mở cùng lúc quá lớn. | Dữ liệu của luồng con trước có thể không bao giờ đến được hạ nguồn nếu thượng nguồn phát quá nhanh. |
| **Ca sử dụng chuẩn trong Android** | Xử lý giao dịch có thứ tự (Đọc Local Cache -> Ghi Database -> Đồng bộ tuần tự). | Tải song song nhiều tài nguyên độc lập (Tải đồng thời ảnh/chi tiết từ danh sách ID). | Tìm kiếm gõ phím (Search-as-you-type), Lọc danh mục, Đổi ID người dùng, Chuyển Tab UI. |

---

#### 4. Sơ Đồ Trục Thời Gian (Marble / Timeline Diagram)

Giả sử luồng thượng nguồn phát ra:
- Giá trị `1` tại thời điểm **0ms**
- Giá trị `2` tại thời điểm **150ms**

Mỗi giá trị sinh ra một luồng con phát 2 phần tử: phần tử đầu tiên sau **100ms**, phần tử thứ hai sau thêm **100ms** nữa (tổng 200ms).

```
Thời gian (ms) ──► 0ms    100ms   150ms   200ms   250ms   300ms   350ms   400ms
Upstream Flow     : ──1────────────2──────────────────────────────────────────────|
─────────────────────────────────────────────────────────────────────────────────
flatMapConcat     : ────────1a────────────1b──────────────2a──────────────2b─────|
                    (Chờ luồng 1 chạy xong 200ms mới bắt đầu chạy luồng 2 lúc 200ms!)
─────────────────────────────────────────────────────────────────────────────────
flatMapMerge      : ────────1a────────────1b──────2a──────────────2b─────────────|
                    (Tại 150ms, mở ngay luồng 2 song song với luồng 1!)
─────────────────────────────────────────────────────────────────────────────────
flatMapLatest     : ────────1a─────[CANCEL 1!]────2a──────────────2b─────────────|
                    (Tại 150ms có số 2 -> HỦY LUỒNG 1 NGAY! Phần tử 1b bị VỨT BỎ!)
```

---

#### 5. Mã Nguồn Demo Đối Chiếu Chạy Thực Nghiệm

Dưới đây là chương trình hoàn chỉnh minh họa chính xác sự khác biệt về hành vi và thời gian thực tế giữa 3 toán tử:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.TimeSource

fun upstreamFlow(): Flow<Int> = flow {
    emit(1)
    delay(150.milliseconds) // Sau 150ms phát tiếp số 2
    emit(2)
}

fun innerFlow(id: Int): Flow<String> = flow {
    delay(100.milliseconds)
    emit("[$id] Bước 1")
    delay(100.milliseconds)
    emit("[$id] Bước 2")
}

suspend fun main() = coroutineScope {
    println("=== 1. THỬ NGHIỆM flatMapConcat ===")
    val startConcat = TimeSource.Monotonic.markNow()
    upstreamFlow()
        .flatMapConcat { id -> innerFlow(id) }
        .collect { println("[+${startConcat.elapsedNow().inWholeMilliseconds}ms] Concat: $it") }

    println("\n=== 2. THỬ NGHIỆM flatMapMerge ===")
    val startMerge = TimeSource.Monotonic.markNow()
    upstreamFlow()
        .flatMapMerge { id -> innerFlow(id) }
        .collect { println("[+${startMerge.elapsedNow().inWholeMilliseconds}ms] Merge: $it") }

    println("\n=== 3. THỬ NGHIỆM flatMapLatest ===")
    val startLatest = TimeSource.Monotonic.markNow()
    upstreamFlow()
        .flatMapLatest { id -> innerFlow(id) }
        .collect { println("[+${startLatest.elapsedNow().inWholeMilliseconds}ms] Latest: $it") }
}
```

**Kết quả Console Output Minh Họa Chuẩn Xác:**

```none
=== 1. THỬ NGHIỆM flatMapConcat ===
[+105ms] Concat: [1] Bước 1
[+208ms] Concat: [1] Bước 2
[+312ms] Concat: [2] Bước 1
[+415ms] Concat: [2] Bước 2

=== 2. THỬ NGHIỆM flatMapMerge ===
[+104ms] Merge: [1] Bước 1
[+206ms] Merge: [1] Bước 2
[+258ms] Merge: [2] Bước 1
[+360ms] Merge: [2] Bước 2

=== 3. THỬ NGHIỆM flatMapLatest ===
[+104ms] Latest: [1] Bước 1
[+258ms] Latest: [2] Bước 1
[+361ms] Latest: [2] Bước 2
```

##### Phân Tích Chi Tiết Kết Quả Thực Nghiệm:

1. **Với `flatMapConcat`:**
   - Tại mốc `150ms`, upstream phát số `2`, nhưng vì luồng `1` vẫn chưa chạy xong, số `2` phải nằm chờ trong hàng đợi.
   - Chỉ khi luồng `1` kết thúc lúc `208ms`, luồng `2` mới được khởi chạy. Tổng thời gian hoàn thành là **`~415ms`**.

2. **Với `flatMapMerge`:**
   - Tại mốc `150ms`, ngay khi số `2` phát ra, luồng `2` được mở và chạy song song ngay lập tức với luồng `1`.
   - Kết quả `[2] Bước 1` xuất hiện lúc `~258ms` (tức là 150ms + 100ms delay). Tổng thời gian hoàn thành rút ngắn xuống còn **`~360ms`**.

3. **Với `flatMapLatest`:**
   - Tại mốc `150ms`, số `2` xuất hiện -> luồng `1` **bị hủy ngay lập tức**.
   - Do đó, phần tử `[1] Bước 2` (lẽ ra sẽ xuất hiện ở 200ms) **hoàn toàn biến mất** khỏi kết quả! Luồng `2` bắt đầu chạy từ mốc 150ms và phát các giá trị của riêng nó.

---

## 3. Toán tử Kết thúc (Terminal Operators)

Toán tử kết thúc là các hàm `suspend` kích hoạt quá trình thu thập của luồng.

### 3.1 Toán tử `.collect()` và `.collectLatest()`
- `.collect()` có thể nhận một lambda để xử lý từng phần tử, hoặc gọi không tham số (kết hợp với `.onEach` phía trước).
- `.collectLatest()` hủy bỏ việc xử lý phần tử trước đó nếu có phần tử mới phát sinh trong lúc đang xử lý:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun main() {
    withContext(Dispatchers.Default) {
        flow {
            println("Emitting Page 1")
            emit("Page 1")
            delay(50.milliseconds)
            println("Emitting Page 2 in quick succession")
            emit("Page 2")
            delay(200.milliseconds)
            println("Emitting Page 3")
            emit("Page 3")
        }.flowOn(Dispatchers.IO).collectLatest {
            println("Starting to process $it!")
            try {
                delay(100.milliseconds)
            } catch (e: CancellationException) {
                println("Canceled processing $it.")
                throw e
            }
            println("Done processing!")
        }
    }
}
```

**Kết quả Console Output:**
```none
Emitting Page 1
Starting to process Page 1!
Emitting Page 2 in quick succession
Canceled processing Page 1.
Starting to process Page 2!
Done processing!
Emitting Page 3
Starting to process Page 3!
Done processing!
```

### 3.2 Toán tử Lấy Giá trị & Gom Tập Hợp: `first()`, `toList()`, `toSet()`
- **`.first()`**: Trả về phần tử đầu tiên thỏa điều kiện rồi hủy việc thu thập luồng ngay lập tức.
- **`.toList()` & `.toSet()`**: Thu thập toàn bộ phần tử vào danh sách/tập hợp:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Triển khai tùy biến thu gọn của .toList() sử dụng buildList
suspend fun <T> Flow<T>.myToList(): List<T> = buildList {
    this@myToList.collect { value ->
        add(value)
    }
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        val firstValue = flowOf(1, 2, 3).first()
        println(firstValue) // 1

        val list = flowOf(1, 2, 3).toList()
        println(list) // [1, 2, 3]

        val set = flowOf(1, 2, 2, 3).toSet()
        println(set) // [1, 2, 3]
    }
}
```

### 3.3 Toán tử Tích lũy: `.reduce()` và `.fold()`
- **`.reduce()`**: Lấy phần tử đầu tiên phát ra làm giá trị khởi tạo cho biến tích lũy (accumulator).
- **`.fold()`**: Bắt đầu với một giá trị khởi tạo do bạn chỉ định:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

suspend fun main() {
    withContext(Dispatchers.Default) {
        val reduced = flowOf(1, 2, 3).reduce { accumulator, value ->
            accumulator + value
        }

        val folded = flowOf(1, 2, 3).fold(2) { accumulator, value ->
            accumulator + value
        }

        println(reduced) // 6 (1 + 2 + 3)
        println(folded)  // 8 (2 + 1 + 2 + 3)
    }
}
```

---

## 4. Thu thập Luồng trong một `CoroutineScope` Cụ thể (`.launchIn`)

Khi một màn hình (Screen/Activity/Fragment) hoặc một đối tượng có vòng đời dài cần tiêu thụ dữ liệu từ Flow, việc thu thập phải được gắn kết trực tiếp vào `CoroutineScope` của đối tượng đó. Điều này đảm bảo khi đối tượng bị hủy (scope bị cancel), quá trình thu thập cũng tự động bị hủy theo, ngăn ngừa triệt để hiện tượng rò rỉ bộ nhớ (memory leaks).

Toán tử `.launchIn(scope)` khởi chạy một coroutine riêng trong `scope` để gọi `collect()`, và trả về chính đối tượng `Job` của coroutine đó:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

// Triển khai tùy biến thu gọn của .launchIn()
fun <T> Flow<T>.myLaunchIn(scope: CoroutineScope): Job = scope.launch {
    this@myLaunchIn.collect()
}

data class Coordinate(val x: Int, val y: Int)

class MyScreen(val scope: CoroutineScope) {
    private val _mousePosition = MutableStateFlow<Coordinate>(Coordinate(0, 0))
    val mousePosition get() = _mousePosition.asStateFlow()

    init {
        // Bắt đầu thu thập StateFlow trong CoroutineScope của màn hình
        mousePosition.onEach {
            updateStatusBar()
        }.launchIn(scope)
    }

    fun moveMouse(newCoordinate: Coordinate) {
        _mousePosition.value = newCoordinate
    }

    private fun updateStatusBar() {
        println("Mouse is at ${_mousePosition.value}")
    }
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        val childScope = CoroutineScope(
            currentCoroutineContext() + Job(currentCoroutineContext()[Job])
        )
        val screen = MyScreen(childScope)
        delay(100.milliseconds)
        
        screen.moveMouse(Coordinate(10, 15))
        delay(100.milliseconds)
        
        screen.moveMouse(Coordinate(1, 3))
        delay(100.milliseconds)
        
        // Khi màn hình đóng, cancel scope sẽ dừng toàn bộ quá trình thu thập Flow
        childScope.cancel()
    }
}
```

**Kết quả Console Output:**
```none
Mouse is at Coordinate(x=0, y=0)
Mouse is at Coordinate(x=10, y=15)
Mouse is at Coordinate(x=1, y=3)
```

---

## 5. Tổng kết & So sánh Nhanh

| Nhóm Toán tử | Các Toán tử Tiêu biểu | Đặc điểm Kỹ thuật |
| :--- | :--- | :--- |
| **Biến đổi** | `transform`, `map`, `filter`, `mapNotNull` | Giữ nguyên thứ tự tuần tự; lambda hỗ trợ gọi hàm `suspend`. |
| **Lọc & Giới hạn** | `distinctUntilChanged`, `drop`, `take` | `take` hủy luồng thượng nguồn bằng `CancellationException`. |
| **Đồng thời / Đệm** | `buffer`, `conflate`, `flowOn` | Tách rời emitter và collector; hỗ trợ **Operator Fusion** khi dùng chung. |
| **Kết hợp** | `zip`, `combine`, `merge` | `zip` ghép cặp 1-1; `combine` lấy giá trị mới nhất; `merge` hòa trộn đa nguồn. |
| **Vòng đời** | `onStart`, `onEach`, `onCompletion`, `onEmpty` | Bắt các sự kiện vòng đời; `onCompletion` có thể phát thêm giá trị (`emit`). |
| **Làm phẳng** | `flatMapConcat`, `flatMapMerge`, `flatMapLatest` | Chuyển đổi từ `Flow<Flow<T>>` thành `Flow<T>`. |
| **Kết thúc** | `collect`, `collectLatest`, `first`, `toList`, `reduce`, `launchIn` | Kích hoạt luồng chạy; `launchIn` quản lý vòng đời theo `CoroutineScope`. |

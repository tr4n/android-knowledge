# Bài 05 — Flow Exception Handling: catch, onCompletion, retry & Turbine Testing

> **Module:** 1 — Kotlin Coroutines & Flow Foundation  
> **Prerequisite:** [Bài 04 — Flow trong Android Lifecycle: repeatOnLifecycle](04-flow-lifecycle-android.md)  
> **Official Docs:**
> - [Flow Exception Handling — Kotlin Documentation](https://kotlinlang.org/docs/flow.html#flow-exceptions)
> - [Exceptions in Coroutines — Android Developers](https://developer.android.com/kotlin/coroutines/coroutines-adv#exceptions)
> - [Testing Kotlin Flows with Turbine](https://github.com/cashapp/turbine)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Xử lý ngoại lệ (Exception Handling) trong các luồng dữ liệu bất đồng bộ (Reactive Streams) là một trong những bài toán phức tạp nhất. Khác với lập trình đồng bộ truyền thống nơi mọi thứ xảy ra trên cùng một Call Stack, luồng dữ liệu của Flow có thể trải dài qua nhiều Coroutines, Threads và Dispatchers khác nhau.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   FLOW EXCEPTION HANDLING PRINCIPLES                   │
│                                                                        │
│   1. Exception Transparency (Tính minh bạch ngoại lệ)                  │
│      - Không bao giờ được giấu giếm hoặc nuốt ngoại lệ của downstream. │
│      - Code bên trong emitter flow { } không được try-catch bao bọc    │
│        lệnh emit() nếu nuốt lỗi downstream.                            │
│                                                                        │
│   2. Upstream vs Downstream Exceptions                                 │
│      - Upstream: Lỗi sinh ra từ DataSource, API, Database hoặc các     │
│        toán tử trung gian (map, filter).                               │
│      - Downstream: Lỗi sinh ra từ chính khối xử lý của Terminal        │
│        Operator (collect { }).                                         │
│                                                                        │
│   3. CancellationException is NOT a Failure                            │
│      - Là tín hiệu điều khiển nội bộ (Control Signal) của Coroutine.   │
│      - Không được coi là lỗi, không được log như một Crash!            │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Các thành phần cốt lõi

#### Toán tử `catch { cause -> }`
- Toán tử trung gian dùng để bắt các ngoại lệ xảy ra ở **Upstream**.
- Cho phép phân tích lỗi, thực hiện hành động khắc phục, hoặc phát ra (`emit`) một giá trị fallback thay thế cho Downstream.

#### Toán tử `onCompletion { cause -> }`
- Toán tử được gọi khi Flow hoàn thành việc phát dữ liệu.
- Tham số `cause: Throwable?` cho biết Flow kết thúc bình thường (`cause == null`) hay kết thúc do có lỗi hoặc bị hủy (`cause != null`). Thường dùng để đóng file, ngắt kết nối socket hoặc giải phóng bộ nhớ.

#### `SupervisorJob`
- Một loại `Job` đặc biệt làm thay đổi cơ chế lan truyền lỗi trong Coroutine Hierarchy: Khi một coroutine con bị crash, `SupervisorJob` sẽ **không hủy các coroutine con khác** và không hủy chính nó.

#### `CoroutineExceptionHandler`
- Handler xử lý ở cấp độ Context (`CoroutineContext.Element`) dùng để hứng các uncaught exceptions ở **Root Coroutine** (ngăn ứng dụng bị crash đột ngột).

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Nguyên lý Minh bạch Ngoại lệ (Exception Transparency)

Kotlin Flow đặt ra một nguyên tắc bất biến nghiêm ngặt: **Flow phải hoàn toàn minh bạch đối với ngoại lệ**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      RANH GIỚI CỦA TOÁN TỬ CATCH                       │
│                                                                        │
│   Upstream:                                                            │
│   flow {                                                               │
│       val data = fetchApi() // Giả sử Ném IOException ở đây!           │
│       emit(data)                                                       │
│   }                                                                    │
│   .map { transform(it) }                                               │
│         │                                                              │
│         ▼                                                              │
│   .catch { e ->                                                        │
│       emit(fallbackData) //  CATCH BẮT ĐƯỢC TẤT CẢ LỖI PHÍA TRÊN NÓ!   │
│   }                                                                    │
│         │                                                              │
│         ▼                                                              │
│   Downstream:                                                          │
│   .collect { data ->                                                   │
│       updateUi(data) //  NẾU NÉM NullPointerException Ở ĐÂY:           │
│                      // .catch { } Ở TRÊN CỐ TÌNH KHÔNG BẮT!           │
│                      // Lỗi sẽ bắn thẳng ra ngoài CoroutineScope!      │
│   }                                                                    │
└────────────────────────────────────────────────────────────────────────┘
```

#### Tại sao `catch {}` không bắt lỗi ở `collect {}`?
Nếu `catch` bắt cả lỗi của downstream collector, nó sẽ dẫn đến hiện tượng **Swallowed Bugs** (nuốt bug UI). Collector nghĩ rằng dữ liệu phát ra bị lỗi mạng, trong khi thực tế lỗi nằm ở chính logic render của giao diện! Vì vậy, `catch {}` chỉ bảo vệ **Upstream**, còn lỗi ở downstream phải được giải quyết bằng `try-catch` cục bộ tại collector.

---

### 2.2 Cơ chế hủy bỏ (Cancellation) và `CancellationException`

Khi một CoroutineScope bị hủy (ví dụ ViewModel bị cleared khi user back khỏi màn hình):
1. Scope gửi một tín hiệu `CancellationException` dọc theo cây Job Hierarchy.
2. Mọi suspend function đang chờ (`delay()`, `collect()`, `emit()`) sẽ thức dậy và ném ra `CancellationException`.
3. Toàn bộ các khối `finally` và toán tử `onCompletion` được kích hoạt tuần tự để giải phóng tài nguyên.

```
                    LUỒNG HUỶ COROUTINE CHUẨN MỰC
                    
User thoát màn hình ──► viewModelScope.cancel()
                             │
                             ▼
              Bắn CancellationException vào luồng
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   Toán tử catch { }                Toán tử onCompletion { }
   (Tự động BỎ QUA,                 (Được gọi với cause =
    không xử lý như lỗi)             CancellationException)
            │                                 │
            ▼                                 ▼
   Coroutine kết thúc an toàn       Tài nguyên được dọn dẹp
```

> **Lưu ý sống còn:** Toán tử `catch {}` của Kotlin Flow được thiết kế ngầm định để **bỏ qua `CancellationException`**. Bạn không bao giờ phải lo lắng về việc `catch {}` nuốt mất tín hiệu hủy luồng của hệ thống!

---

### 2.3 Sơ đồ Lan truyền Ngoại lệ: `Job` vs `SupervisorJob`

```
TRƯỜNG HỢP 1: JOB THƯỜNG (Bi-directional Failure)
                    Parent Job
                   (BỊ HUỶ LUÔN!)
                    ▲          │
         Lỗi đẩy lên│          │Hủy lan truyền xuống
                    │          ▼
          Child Job 1 (CRASH)   Child Job 2 (BỊ HUỶ OAN!)


TRƯỜNG HỢP 2: SUPERVISOR JOB (Failure Isolation)
                 Parent SupervisorJob
                 (VẪN SỐNG BÌNH THƯỜNG)
                    ▲          X (Chặn không cho hủy xuống)
         Lỗi đẩy lên│          
                    │          
          Child Job 1 (CRASH)   Child Job 2 (TIẾP TỤC CHẠY!)
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Sai lầm kinh điển: Try-Catch bọc ngoài `collect`

Nhiều lập trình viên có thói quen viết:

```kotlin
// ANTI-PATTERN: VÔ CÙNG NGUY HIỂM TRONG ANDROID
viewModelScope.launch {
    try {
        repository.getDataStream().collect { data ->
            updateUI(data)
        }
    } catch (e: Exception) {
        // CẠM BẪY CHÍ MẠNG: Khối catch này bắt cả CancellationException!
        // Khi ViewModel clear, coroutine này nuốt exception và tiếp tục sống,
        // dẫn đến MEMORY LEAK nặng nề!
        Log.e("App", "Error: ${e.message}")
    }
}
```

#### Giải pháp chuẩn mực với Kotlin Flow:
```kotlin
// CLEAN ARCHITECTURE: An toàn vòng đời 100%
viewModelScope.launch {
    repository.getDataStream()
        .catch { e ->
            // Chỉ bắt Upstream lỗi (mạng, DB), không nuốt CancellationException!
            _uiState.value = UiState.Error(e.message ?: "Lỗi kết nối")
        }
        .collect { data ->
            _uiState.value = UiState.Success(data)
        }
}
```

---

### 3.2 Bảng so sánh chiến lược xử lý lỗi: RxJava vs Kotlin Flow

| Tính năng | RxJava 2 / 3 | Kotlin Flow |
|---|---|---|
| **Bắt lỗi cơ bản** | `onErrorReturn`, `onErrorResumeNext` | `catch { emit(...) }` |
| **Dọn dẹp tài nguyên** | `doFinally`, `doOnTerminate` | `onCompletion { cause -> }` |
| **Thử lại khi có lỗi** | `retry()`, `retryWhen()` phức tạp với Flowable | `retry(retries)`, `retryWhen { cause, attempt -> }` |
| **Tính minh bạch** | Dễ bị che giấu lỗi giữa các tầng | Đảm bảo tuyệt đối bởi **Exception Transparency Rule** |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Mô hình UiState Sealed Interface chuẩn mực

Trong Modern Android, lỗi không được coi là một ngoại lệ làm sập app, mà là một **trạng thái hiển thị giao diện (State)**:

```kotlin
sealed interface CryptoUiState {
    data object Loading : CryptoUiState
    data class Success(val prices: List<CryptoPrice>) : CryptoUiState
    data class Error(val message: String, val canRetry: Boolean = true) : CryptoUiState
}
```

---

### 4.2 Chiến lược Retry với Exponential Backoff & Jitter

Khi gọi API hoặc kết nối WebSocket bị đứt, nếu tất cả hàng ngàn client cùng retry ngay lập tức sau 1 giây, server sẽ bị sập vì quá tải (**Thundering Herd Problem**). Giải pháp là dùng **Exponential Backoff kết hợp độ trễ ngẫu nhiên (Jitter)**:

```kotlin
fun <T> Flow<T>.retryWithExponentialBackoff(
    maxRetries: Long = 3,
    initialDelayMs: Long = 1000,
    factor: Double = 2.0
): Flow<T> = retryWhen { cause, attempt ->
    if (cause is IOException && attempt < maxRetries) {
        val delayTime = (initialDelayMs * Math.pow(factor, attempt.toDouble())).toLong()
        val jitter = (0..200).random() // Thêm độ lệch ngẫu nhiên
        delay(delayTime + jitter)
        true // Tiếp tục retry
    } else {
        false // Dừng retry, đẩy lỗi sang toán tử catch tiếp theo
    }
}
```

---

### 4.3 Cạm bẫy phổ biến (Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Vi phạm Exception Transparency bằng `try-catch` bao quanh `emit`
```kotlin
// SAI (CRASH RUNTIME: IllegalStateException):
fun getNumbers(): Flow<Int> = flow {
    try {
        emit(1) // CẤM bọc emit() bằng try-catch nếu nuốt exception của downstream!
    } catch (e: Exception) {
        println("Nuốt lỗi thành công!")
    }
}

// ĐÚNG:
fun getNumbers(): Flow<Int> = flow {
    emit(1)
} // Để việc bắt lỗi cho các toán tử bên ngoài
```

#### Cạm bẫy 2: Không phân biệt `IOException` (Lỗi mạng tạm thời) và `HttpException` (Lỗi 404/401) khi Retry
Chỉ nên retry các lỗi có khả năng phục hồi (Transient Errors) như mất sóng 4G/Wifi (`IOException`, `SocketTimeoutException`). Tuyệt đối không retry các lỗi xác thực như `401 Unauthorized` hoặc `404 Not Found` vì retry bao nhiêu lần cũng sẽ fail!

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng tính năng **Bảng giá tiền mã hóa thời gian thực (Real-time Crypto Ticker)**:
- Tự động Retry tối đa 3 lần khi mất mạng bằng Exponential Backoff.
- Nếu sau 3 lần vẫn lỗi, fallback hiển thị dữ liệu lưu trong Cache cục bộ.
- Xử lý trạng thái lỗi trực quan trên **Jetpack Compose**.
- Viết bộ **Unit Test toàn diện với Turbine**.

### Bước 1: Data Layer (Retry & Local Fallback Cache)

```kotlin
data class CryptoPrice(val symbol: String, val priceUsd: Double)

interface CryptoRepository {
    fun getLivePriceStream(): Flow<List<CryptoPrice>>
    suspend fun getCachedPrices(): List<CryptoPrice>
}

class CryptoRepositoryImpl(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : CryptoRepository {

    // Bộ nhớ cache giả lập
    private val localCache = listOf(
        CryptoPrice("BTC", 65000.0),
        CryptoPrice("ETH", 3500.0)
    )

    override suspend fun getCachedPrices(): List<CryptoPrice> = localCache

    override fun getLivePriceStream(): Flow<List<CryptoPrice>> = flow {
        // Giả lập phát giá real-time
        var attemptCount = 0
        while (true) {
            attemptCount++
            if (attemptCount in 1..2) {
                // Giả lập lỗi mạng ở 2 lần đầu
                throw IOException("Lỗi kết nối Socket Server")
            }
            // Lần thứ 3 thành công
            emit(listOf(
                CryptoPrice("BTC", 67250.0 + (0..50).random()),
                CryptoPrice("ETH", 3580.0 + (0..10).random())
            ))
            delay(3000)
        }
    }
    .retryWhen { cause, attempt ->
        if (cause is IOException && attempt < 3) {
            val delayMs = 500L * (attempt + 1)
            delay(delayMs) // Backoff
            true
        } else {
            false
        }
    }
    .flowOn(ioDispatcher)
}
```

---

### Bước 2: ViewModel Layer (`catch` & Fallback State)

```kotlin
class CryptoViewModel(
    private val repository: CryptoRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<CryptoUiState>(CryptoUiState.Loading)
    val uiState: StateFlow<CryptoUiState> = _uiState.asStateFlow()

    init {
        startObservingPrices()
    }

    fun startObservingPrices() {
        viewModelScope.launch {
            _uiState.value = CryptoUiState.Loading
            
            repository.getLivePriceStream()
                .catch { cause ->
                    // Khi upstream retry thất bại, lấy dữ liệu cache làm fallback
                    val cached = repository.getCachedPrices()
                    if (cached.isNotEmpty()) {
                        emit(cached) // Phát giá trị fallback
                    } else {
                        _uiState.value = CryptoUiState.Error(
                            message = cause.localizedMessage ?: "Lỗi tải dữ liệu",
                            canRetry = true
                        )
                    }
                }
                .collect { prices ->
                    _uiState.value = CryptoUiState.Success(prices)
                }
        }
    }
}
```

---

### Bước 3: UI Layer (Jetpack Compose với Error Banner & Retry Button)

```kotlin
@Composable
fun CryptoPriceScreen(
    viewModel: CryptoViewModel,
    modifier: Modifier = Modifier
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = { TopAppBar(title = { Text("Crypto Market Live") }) }
    ) { padding ->
        Box(
            modifier = modifier
                .fillMaxSize()
                .padding(padding),
            contentAlignment = Alignment.Center
        ) {
            when (val state = uiState) {
                is CryptoUiState.Loading -> {
                    CircularProgressIndicator()
                }
                is CryptoUiState.Error -> {
                    Column(
                        horizontalAlignment = Alignment.CenterHorizontally,
                        modifier = Modifier.padding(16.dp)
                    ) {
                        Text(
                            text = "⚠️ ${state.message}",
                            color = MaterialTheme.colorScheme.error,
                            style = MaterialTheme.typography.titleMedium
                        )
                        Spacer(modifier = Modifier.height(12.dp))
                        if (state.canRetry) {
                            Button(onClick = { viewModel.startObservingPrices() }) {
                                Text("Thử kết nối lại")
                            }
                        }
                    }
                }
                is CryptoUiState.Success -> {
                    LazyColumn(
                        modifier = Modifier.fillMaxSize().padding(16.dp),
                        verticalArrangement = Arrangement.spacedBy(10.dp)
                    ) {
                        items(state.prices, key = { it.symbol }) { item ->
                            Card(
                                modifier = Modifier.fillMaxWidth(),
                                colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceVariant)
                            ) {
                                Row(
                                    modifier = Modifier.fillMaxWidth().padding(16.dp),
                                    horizontalArrangement = Arrangement.SpaceBetween
                                ) {
                                    Text(text = item.symbol, style = MaterialTheme.typography.headlineSmall)
                                    Text(
                                        text = "$${item.priceUsd}",
                                        style = MaterialTheme.typography.headlineSmall,
                                        color = MaterialTheme.colorScheme.primary
                                    )
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

---

### Bước 4: Viết Unit Test chuyên sâu với `Turbine` & `runTest`

```kotlin
class CryptoViewModelTest {

    private val testDispatcher = StandardTestDispatcher()

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun getLivePriceStream_onError_emitsCachedFallback() = runTest(testDispatcher) {
        // Fake Repository luôn ném lỗi
        val fakeRepo = object : CryptoRepository {
            override fun getLivePriceStream(): Flow<List<CryptoPrice>> = flow {
                throw IOException("Server Dead")
            }
            override suspend fun getCachedPrices() = listOf(CryptoPrice("BTC", 60000.0))
        }

        val viewModel = CryptoViewModel(fakeRepo)

        // Dùng Turbine kiểm tra từng emission của uiState
        viewModel.uiState.test {
            // 1. Trạng thái Loading ban đầu
            assertEquals(CryptoUiState.Loading, awaitItem())

            // 2. Chạy hết coroutines đang chờ
            testScheduler.advanceUntilIdle()

            // 3. Vì repo lỗi nhưng có cache nên UI nhận được trạng thái Success với cache fallback!
            val state = awaitItem() as CryptoUiState.Success
            assertEquals(1, state.prices.size)
            assertEquals("BTC", state.prices.first().symbol)
            assertEquals(60000.0, state.prices.first().priceUsd, 0.0)

            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun flow_terminalException_caughtByTurbine() = runTest {
        val errorFlow = flow<Int> {
            emit(1)
            throw IllegalStateException("Fatal DB Crash")
        }

        // Test trực tiếp Flow ném exception với Turbine awaitError()
        errorFlow.test {
            assertEquals(1, awaitItem())
            val throwable = awaitError()
            assertTrue(throwable is IllegalStateException)
            assertEquals("Fatal DB Crash", throwable.message)
        }
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android

#### Q1: Tại sao `catch {}` không thể bắt được Exception xảy ra bên trong khối `collect {}`? Làm sao để xử lý đúng?
**Trả lời chuẩn bản chất:**
Đó là quy tắc **Exception Transparency**. Toán tử `catch` chỉ bắt các exception xảy ra ở upstream (thượng nguồn). Nếu nó bắt cả exception trong `collect`, code sẽ vi phạm tính độc lập giữa Producer và Consumer.
Để xử lý exception phát sinh trong `collect`, bạn có 2 cách chuẩn mực:
1. Đưa logic đó về phía trước bằng toán tử `onEach { }` rồi mới gọi `catch {}`:
   ```kotlin
   flow
       .onEach { processData(it) } // Logic xử lý được chuyển lên upstream
       .catch { handleException(it) } // Bắt được lỗi của cả flow lẫn onEach!
       .collect()
   ```
2. Bọc khối `collect` bằng `try-catch` truyền thống nhưng **nhớ rethrow `CancellationException`**.

#### Q2: Khác biệt giữa `SupervisorJob` và `supervisorScope { }` là gì?
**Trả lời chuẩn bản chất:**
- `SupervisorJob()` là một đối tượng context thường được truyền vào `CoroutineScope` (như `CoroutineScope(SupervisorJob() + Dispatchers.Main)`).
- `supervisorScope { }` là một suspend function builder tạo ra một scope có tính chất supervisor tạm thời cho các coroutine con bên trong nó. Khi một con trong `supervisorScope` bị fail, nó không làm huỷ các con khác và kết quả của khối `supervisorScope` có thể được bắt bằng try-catch bên ngoài!

#### Q3: Điều gì xảy ra nếu bên trong block `catch { }` lại tiếp tục ném ra một Exception mới?
**Trả lời chuẩn bản chất:**
Nếu bản thân block `catch { e -> ... }` ném ra một exception mới, exception mới đó sẽ lập tức thoát ra khỏi chuỗi Flow và truyền ngược lên Coroutine cha đang chứa lệnh `collect`. Chuỗi Flow bị terminate hoàn toàn.

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Giải pháp xử lý triệt để |
|---|---|---|
| **Crash: `IllegalStateException: Flow exception transparency violated`** | Tự ý dùng `try-catch` bọc lệnh `emit()` bên trong hàm builder `flow { }` và nuốt lỗi của downstream. | Xóa bỏ `try-catch` xung quanh lệnh `emit()`. Hãy để downstream tự quản lý lỗi của họ. |
| **ViewModel bị rò rỉ (Memory Leak) sau khi đóng màn hình** | Dùng `try { ... } catch (e: Exception)` tổng quát bên trong `viewModelScope.launch` nuốt mất `CancellationException`. | Luôn kiểm tra `if (e is CancellationException) throw e` hoặc chuyển sang dùng toán tử `.catch {}`. |
| **Unit Test bị treo vô tận (Test Timeout)** | Test Flow bằng `collect` thông thường mà luồng Flow không bao giờ đóng (Infinite Stream). | Sử dụng thư viện **Turbine** với `awaitItem()` và gọi `cancelAndIgnoreRemainingEvents()` khi kết thúc test. |

---

*Bài trước: [04 — Flow trong Android Lifecycle: repeatOnLifecycle](04-flow-lifecycle-android.md)*  
*Module tiếp theo: [Module 2 — Architecture & ViewModel](../module-2-architecture/06-viewmodel-flow-compose-udf.md)*

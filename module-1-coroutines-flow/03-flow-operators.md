# Bài 03 — Flow Operators: map, flatMapLatest, combine, zip, debounce & Backpressure

> **Module:** 1 — Kotlin Coroutines & Flow Foundation  
> **Prerequisite:** [Bài 02 — Cold Flow vs Hot Flow: StateFlow & SharedFlow](02-cold-flow-vs-hot-flow.md)  
> **Official Docs:**
> - [Kotlin Asynchronous Flow](https://kotlinlang.org/docs/flow.html)
> - [Flow Operators API Reference](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong hệ sinh thái Kotlin Coroutines Flow, một **Operator (Toán tử)** là một hàm biến đổi, lọc, kết hợp hoặc tiêu thụ luồng dữ liệu stream. Operators trong Flow được thiết kế dựa trên triết lý lập trình hàm phản ứng (**Functional Reactive Programming - FRP**), chia làm hai nhóm chính:

```
┌────────────────────────────────────────────────────────────────────────┐
│                         FLOW OPERATORS ECOSYSTEM                       │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 1. Intermediate Operators (Toán tử trung gian - Cold & Lazy)   │   │
│   │    - Trả về một Flow<T> mới                                    │   │
│   │    - KHÔNG kích hoạt việc thực thi stream                      │   │
│   │    - Ví dụ: map, filter, transform, debounce, flatMapLatest   │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Upstream (Dòng chảy từ trên xuống) │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 2. Terminal Operators (Toán tử kết thúc - Suspend functions)   │   │
│   │    - Là các hàm suspend                                        │   │
│   │    - KÍCH HOẠT (trigger) dòng chảy của Cold Flow               │   │
│   │    - Ví dụ: collect, first, toList, reduce, fold, stateIn      │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                   │ Downstream (Điểm tiêu thụ cuối)    │
│                                   ▼                                    │
│                             FlowCollector                              │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Khái niệm Upstream vs Downstream
- **Upstream (Phía thượng nguồn):** Tất cả các toán tử và emitter nằm **trước** (phía trên) toán tử hiện tại.
- **Downstream (Phía hạ nguồn):** Tất cả các toán tử và collector nằm **sau** (phía dưới) toán tử hiện tại.
> **Tính chất bất biến (Immutability):** Mỗi Intermediate Operator không thay đổi Flow gốc mà tạo ra một `Flow` wrapper mới. Toàn bộ chuỗi toán tử chỉ là một bản thiết kế khai báo (**Declarative Pipeline**), chỉ thực sự chạy khi gặp Terminal Operator.

---

### 1.2 Bảng phân loại các nhóm Operators phổ biến

| Nhóm toán tử | Danh sách APIs | Mục đích sử dụng |
|---|---|---|
| **Transforming** | `map`, `transform`, `onEach` | Biến đổi từng phần tử $T \rightarrow R$. `transform` cho phép emit nhiều lần. |
| **Filtering** | `filter`, `take`, `drop`, `distinctUntilChanged` | Lọc bớt phần tử không thỏa mãn hoặc loại bỏ phần tử trùng lặp liên tiếp. |
| **Flattening** | `flatMapConcat`, `flatMapMerge`, `flatMapLatest` | Chuyển đổi một Flow chứa các Flow con (`Flow<Flow<T>>`) thành một luồng phẳng duy nhất (`Flow<T>`). |
| **Combining** | `combine`, `zip`, `merge` | Gộp dữ liệu từ 2 hoặc nhiều Flow độc lập lại với nhau. |
| **Time-based** | `debounce`, `sample`, `timeout` | Kiểm soát tần suất phát dữ liệu dựa trên cửa sổ thời gian (Virtual/Real Time). |
| **Backpressure & Context**| `buffer`, `conflate`, `flowOn` | Quản lý bộ đệm khi tốc độ producer > consumer, và chuyển đổi luồng Thread. |
| **Terminal** | `collect`, `first`, `single`, `toList`, `fold` | Thu thập kết quả, tính toán giá trị cuối cùng. |

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Chuỗi toán tử (Operator Chaining) & Lazy Evaluation

Về mặt bản chất, một toán tử trung gian không làm gì khác ngoài việc bọc (`wrap`) `FlowCollector` của tầng downstream bằng một `FlowCollector` mới để chặn bắt giá trị trước khi chuyển tiếp.

```kotlin
// Khi bạn viết:
flow.map { it * 2 }.collect { println(it) }

// Compiler & Runtime thực thi dưới dạng lồng nhau:
flow.collect(object : FlowCollector<Int> {
    override suspend fun emit(value: Int) {
        val transformed = value * 2 // map logic
        downstreamCollector.emit(transformed) // println logic
    }
})
```
Không có thread trung gian, không có heap allocation dư thừa cho queue — dữ liệu được truyền thẳng qua các lời gọi hàm `suspend` tuần tự.

---

### 2.2 Cơ chế Flattening: `flatMapConcat` vs `flatMapMerge` vs `flatMapLatest`

Khi bạn có một stream mà mỗi phần tử lại sinh ra một stream khác (ví dụ: gõ từ khóa `String` $\rightarrow$ gọi API trả về `Flow<List<Product>>`), bạn cần "làm phẳng" (flatten).

```
Dòng dữ liệu đầu vào: Phát ra [A] rồi sau đó phát ra [B]

1. flatMapConcat (Tuần tự - Sequential):
   [A] ──► Flow_A bắt đầu ──────────► Flow_A kết thúc
                                             │
                                             ▼
   [B] (Chờ Flow_A xong mới chạy) ───────────► Flow_B bắt đầu ──► Flow_B kết thúc
   => Bảo đảm tuyệt đối thứ tự, nhưng chậm nếu Flow con tốn thời gian.

2. flatMapMerge (Đồng thời - Concurrent):
   [A] ──► Flow_A bắt đầu ── emit(A1) ────── emit(A2) ──► Xong
   [B] ─────► Flow_B bắt đầu ── emit(B1) ── emit(B2) ──► Xong
   => Cả 2 Flow chạy song song. Output phát ra xen kẽ tùy tốc độ mạng: A1, B1, A2, B2.

3. flatMapLatest (Hủy tác vụ cũ - Switch to Latest):
   [A] ──► Flow_A chạy ── emit(A1) ──┐ (Bị cancel ngay khi [B] xuất hiện!)
                                     ▼
   [B] ────────────────────────► Flow_B chạy ── emit(B1) ── emit(B2) ──► Xong
   => Flow_A bị HỦY ngay lập tức! Chỉ kết quả của Flow_B (mới nhất) được thu thập.
```

---

### 2.3 Cơ chế Combining: `combine` vs `zip` vs `merge`

```
Flow 1: ─── 1 ────────── 2 ──────────────────── 3 ───►
Flow 2: ─────── "A" ─────────── "B" ─────────────────►

1. combine (Flow1, Flow2) { num, text -> "$num$text" }:
   - Chỉ bắt đầu emit khi TẤT CẢ các luồng đã có ít nhất 1 giá trị.
   - Sau đó, BẤT KỲ luồng nào emit giá trị mới, combine sẽ lấy giá trị MỚI NHẤT của luồng kia:
   Output: ───── "1A" ───── "2A" ── "2B" ──────── "3B" ──►

2. zip (Flow1, Flow2) { num, text -> "$num$text" }:
   - Ghép cặp nghiêm ngặt 1-1 theo chỉ số thứ tự (Index matching).
   - Item 1 của Flow1 CHỜ Item 1 của Flow2. Item 2 chờ Item 2:
   Output: ───── "1A" ────────────── "2B" ──► (Số 3 bị bỏ lại nếu Flow 2 dừng)

3. merge (Flow1, Flow2):
   - Không gộp dữ liệu mà chỉ đơn giản hợp nhất 2 dòng chảy thành 1:
   Output: ─── 1 ── "A" ─── 2 ──── "B" ──────── 3 ───►
```

---

### 2.4 Cơ chế Backpressure & Ranh giới Thread với `flowOn`

Khi Producer phát ra dữ liệu quá nhanh mà Consumer xử lý không kịp, đó là hiện tượng **Backpressure**.

```
MÔ HÌNH MẶC ĐỊNH (Sequential Rendezvous - Không có Buffer):
Producer: ──emit(1)──► [Consumer xử lý 100ms] ──► Producer tiếp tục emit(2)
=> Producer bị SUSPEND chờ Consumer. Tổng thời gian = Time(P) + Time(C).

MÔ HÌNH BUFFER (Tách rời Producer và Consumer):
Producer: ──emit(1)──emit(2)──emit(3)──► [Channel Queue: (1, 2, 3)]
                                                    │
Consumer: ◄─────────────────────────────────────────┘ (Lấy dần ra xử lý)
=> Producer không bị gián đoạn cho đến khi Buffer bị đầy.
```

#### Bản chất của `flowOn`: Context Preservation Invariant
Tài liệu Kotlin quy định: **Flow phát ra dữ liệu trong cùng CoroutineContext với nơi gọi `collect`**. Bạn không được dùng `withContext` bên trong `flow {}`. Thay vào đó, bạn phải dùng toán tử `flowOn`:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              flowOn(IO)                                │
│                                                                        │
│   Upstream (Chạy trên Dispatchers.IO):                                 │
│   flow { emit(loadFromDb()) }                                          │
│         .map { parseData(it) }                                         │
│                    │                                                   │
│                    ▼                                                   │
│          ┌───────────────────┐                                         │
│          │ Channel Buffer    │  ◄── flowOn tự động chèn 1 Channel ngầm │
│          └───────────────────┘                                         │
│                    │                                                   │
│   Downstream (Chạy trên Dispatchers.Main do caller gọi):               │
│                    ▼                                                   │
│         .collect { updateUi(it) }                                      │
└────────────────────────────────────────────────────────────────────────┘
```
> **Nguyên tắc cốt lõi:** `flowOn` **CHỈ** áp dụng cho các toán tử đứng **TRƯỚC** nó (Upstream). Nó hoàn toàn **KHÔNG** ảnh hưởng đến các toán tử đứng sau nó (Downstream).

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Bảng so sánh tiến hóa: RxJava vs Kotlin Flow Operators

| Khía cạnh | RxJava 2 / 3 | Kotlin Coroutines Flow |
|---|---|---|
| **Số lượng operators** | Hơn 150 operators phức tạp (`switchMap`, `flatMapIterable`, `throttleWithTimeout`,...) | Gọn nhẹ, tận dụng trực tiếp các hàm Kotlin chuẩn (`map`, `filter`, `transform`). |
| **Xử lý Concurrency** | Phải nhớ các quy tắc Scheduler (`subscribeOn` vs `observeOn`). Viết sai vị trí là sai luồng. | `flowOn` duy nhất áp dụng cho Upstream; Downstream tuân theo Context của Collector. |
| **Backpressure** | Phải tách riêng 2 kiểu dữ liệu: `Observable` (không backpressure) và `Flowable` (chọn strategy `MISSING`, `DROP`, `LATEST`). | Tích hợp tự nhiên thông qua cơ chế `suspend` của Coroutines. Cấu hình thêm dễ dàng với `buffer()`, `conflate()`. |
| **Hủy bỏ tác vụ (Cancellation)**| Cần lưu `Disposable` và gọi `clear()` / `dispose()`. | Tự động hóa 100% nhờ **Structured Concurrency**. |

---

### 3.2 Bài toán kinh điển: Search-as-you-type & Race Conditions

#### Vấn đề gặp phải (Race Condition):
Khi người dùng gõ từ khóa vào ô tìm kiếm:
1. Gõ ký tự `"A"` $\rightarrow$ Gọi API A (Mạng chập chờn, mất 1500ms).
2. Người dùng gõ tiếp `"Android"` $\rightarrow$ Gọi API Android (Mạng nhanh, trả lời sau 200ms).
3. Kết quả `"Android"` hiển thị lên màn hình.
4. 1300ms sau, API A phản hồi và **ghi đè kết quả của "Android"**! Người dùng đang nhìn chữ "Android" trên thanh search nhưng danh sách bên dưới lại là kết quả của chữ "A"!

#### Cách Flow Operators dập tắt Race Condition triệt để:
```kotlin
searchQueryFlow
    .debounce(300)             // 1. Chờ user dừng gõ 300ms mới xử lý
    .distinctUntilChanged()    // 2. Nếu text không đổi so với lần trước thì bỏ qua
    .filter { it.isNotBlank() }// 3. Không search nếu query rỗng
    .flatMapLatest { query ->  // 4. HỦY ngay lập tức request API cũ nếu query mới xuất hiện!
        searchRepository.searchProducts(query)
    }
    .collect { results ->
        _uiState.value = UiState.Success(results)
    }
```

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Lựa chọn Operator cho từng tình huống

```
Bạn cần làm gì?
│
├── Biến đổi dữ liệu 1-1? ─────────────────────────────────► map { }
├── Biến đổi 1 phần tử thành nhiều phần tử? ──────────────► transform { emit(); emit() }
├── Lọc bỏ các giá trị giống hệt giá trị vừa emit? ────────► distinctUntilChanged()
│
├── Nhận text từ Search Box? ──────────────────────────────► debounce(300) + flatMapLatest
├── Lấy vị trí GPS nhưng chỉ lấy tối đa 1 điểm mỗi 2s? ───► sample(2000)
├── Consumer xử lý chậm, chỉ quan tâm giá trị mới nhất? ────► conflate()
│
├── Gộp giỏ hàng + thông tin giảm giá + profile user? ────► combine(flowA, flowB, flowC)
├── Phát 2 luồng song song không phụ thuộc dữ liệu nhau? ──► merge(flowA, flowB)
└── Bắn cặp đồng bộ theo chỉ số (ví dụ: câu hỏi & đáp án)? ─► zip(questionsFlow, answersFlow)
```

---

### 4.2 Cạm bẫy phổ biến (Common Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Vi phạm Flow Invariant khi gọi `withContext` trong `flow { }`
```kotlin
// SAI (CRASH RUNTIME: IllegalStateException: Flow invariant is violated):
fun getPrices(): Flow<Double> = flow {
    withContext(Dispatchers.IO) { // CẤM: Không được tự ý đổi context bên trong block flow
        val price = fetchFromNetwork()
        emit(price) // Crash tại đây!
    }
}

// ĐÚNG: Sử dụng toán tử flowOn
fun getPrices(): Flow<Double> = flow {
    val price = fetchFromNetwork()
    emit(price)
}.flowOn(Dispatchers.IO) // Đúng chuẩn
```

#### Cạm bẫy 2: `combine` bị "treo" không phát dữ liệu
`combine` chỉ bắt đầu phát ra kết quả đầu tiên khi **TẤT CẢ các Flow tham gia đều đã phát ra ít nhất 1 giá trị**. Nếu một trong các Flow là một Cold Flow chưa bao giờ emit hoặc chờ một event trong tương lai, toàn bộ `combine` sẽ im lặng vô tận!
> **Khắc phục:** Đảm bảo các Flow thành phần luôn có giá trị khởi tạo (ví dụ dùng `onStart { emit(initialValue) }` hoặc chuyển sang `StateFlow`).

#### Cạm bẫy 3: Đặt `flowOn` sai vị trí
```kotlin
// SAI: flowOn(Dispatchers.IO) đặt trước map { } nên map vẫn chạy trên Caller Thread (Main)!
repository.getRawData() // chạy trên IO
    .flowOn(Dispatchers.IO)
    .map { heavyCpuTransformation(it) } // CHẠY TRÊN MAIN THREAD -> Lag giật giao diện!
    .collect { updateUi(it) }

// ĐÚNG: flowOn đặt sau tất cả các bước tính toán nặng cần chuyển luồng
repository.getRawData()
    .map { heavyCpuTransformation(it) }
    .flowOn(Dispatchers.Default) // Áp dụng cho cả getRawData và map
    .collect { updateUi(it) }
```

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng tính năng **Tìm kiếm sản phẩm Real-time (E-Commerce Search)** hoàn chỉnh:
- Giảm tải server bằng `debounce(300ms)`.
- Loại bỏ event trùng lặp bằng `distinctUntilChanged()`.
- Hủy bỏ query cũ ngay khi có query mới bằng `flatMapLatest()`.
- Hiển thị kết quả bằng **Jetpack Compose**.

### Bước 1: Data Layer (Clean Architecture & API)

```kotlin
data class Product(
    val id: String,
    val name: String,
    val price: Double,
    val category: String
)

interface SearchRepository {
    fun searchProducts(query: String): Flow<List<Product>>
}

class SearchRepositoryImpl(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : SearchRepository {

    override fun searchProducts(query: String): Flow<List<Product>> = flow {
        if (query.isBlank()) {
            emit(emptyList())
            return@flow
        }
        // Giả lập độ trễ mạng tìm kiếm (500ms)
        delay(500)
        
        val mockDatabase = listOf(
            Product("1", "Google Pixel 8 Pro", 999.0, "Phones"),
            Product("2", "Samsung Galaxy S24 Ultra", 1299.0, "Phones"),
            Product("3", "MacBook Pro M3", 1999.0, "Laptops"),
            Product("4", "Sony WH-1000XM5", 399.0, "Audio")
        )

        val filtered = mockDatabase.filter { 
            it.name.contains(query, ignoreCase = true) 
        }
        emit(filtered)
    }.flowOn(ioDispatcher) // Bảo đảm truy vấn thực thi trên luồng IO
}
```

---

### Bước 2: ViewModel Layer (Reactive Search Pipeline)

```kotlin
sealed interface SearchUiState {
    data object EmptyQuery : SearchUiState
    data object Loading : SearchUiState
    data class Success(val products: List<Product>) : SearchUiState
    data class Error(val message: String) : SearchUiState
}

@OptIn(ExperimentalCoroutinesApi::class, FlowPreview::class)
class SearchViewModel(
    private val searchRepository: SearchRepository
) : ViewModel() {

    // Input flow nhận query từ UI TextField
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()

    fun onSearchQueryChanged(newQuery: String) {
        _searchQuery.value = newQuery
    }

    // Pipeline biến đổi dữ liệu phản ứng
    val uiState: StateFlow<SearchUiState> = _searchQuery
        .debounce(300L) // Chờ người dùng ngừng gõ 300ms
        .distinctUntilChanged() // Bỏ qua nếu query không thay đổi
        .flatMapLatest { query ->
            if (query.isBlank()) {
                flowOf(SearchUiState.EmptyQuery)
            } else {
                searchRepository.searchProducts(query)
                    .map<List<Product>, SearchUiState> { SearchUiState.Success(it) }
                    .onStart { emit(SearchUiState.Loading) }
                    .catch { emit(SearchUiState.Error("Không thể tải kết quả")) }
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = SearchUiState.EmptyQuery
        )
}
```

---

### Bước 3: UI Layer (Jetpack Compose)

```kotlin
@Composable
fun ProductSearchScreen(
    viewModel: SearchViewModel,
    modifier: Modifier = Modifier
) {
    val query by viewModel.searchQuery.collectAsStateWithLifecycle()
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        // Search Input Bar
        OutlinedTextField(
            value = query,
            onValueChange = viewModel::onSearchQueryChanged,
            modifier = Modifier.fillMaxWidth(),
            placeholder = { Text("Tìm kiếm sản phẩm (Pixel, Galaxy, Sony...)") },
            singleLine = true,
            leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) }
        )

        Spacer(modifier = Modifier.height(16.dp))

        // Dynamic State Rendering
        Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            when (val state = uiState) {
                is SearchUiState.EmptyQuery -> {
                    Text("Nhập từ khóa để bắt đầu tìm kiếm", color = MaterialTheme.colorScheme.outline)
                }
                is SearchUiState.Loading -> {
                    CircularProgressIndicator()
                }
                is SearchUiState.Error -> {
                    Text(text = state.message, color = MaterialTheme.colorScheme.error)
                }
                is SearchUiState.Success -> {
                    if (state.products.isEmpty()) {
                        Text("Không tìm thấy sản phẩm phù hợp.")
                    } else {
                        LazyColumn(
                            modifier = Modifier.fillMaxSize(),
                            verticalArrangement = Arrangement.spacedBy(8.dp)
                        ) {
                            items(state.products, key = { it.id }) { product ->
                                ProductItem(product)
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun ProductItem(product: Product) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column {
                Text(text = product.name, style = MaterialTheme.typography.titleMedium)
                Text(text = product.category, style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.secondary)
            }
            Text(
                text = "$${product.price}",
                style = MaterialTheme.typography.titleLarge,
                color = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

---

### Bước 4: Viết Unit Test chuyên nghiệp với `Turbine`

Thư viện **Turbine** từ CashApp là tiêu chuẩn công nghiệp số 1 để test Flow trong Kotlin.

```kotlin
class SearchViewModelTest {

    private val testDispatcher = StandardTestDispatcher()
    private lateinit var fakeRepository: SearchRepository
    private lateinit var viewModel: SearchViewModel

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        fakeRepository = object : SearchRepository {
            override fun searchProducts(query: String): Flow<List<Product>> = flow {
                if (query == "Pixel") {
                    emit(listOf(Product("1", "Pixel 8", 799.0, "Phones")))
                } else {
                    emit(emptyList())
                }
            }
        }
        viewModel = SearchViewModel(fakeRepository)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun searchQueryPipeline_emitsSuccess_afterDebounce() = runTest(testDispatcher) {
        // Dùng Turbine để test StateFlow uiState
        viewModel.uiState.test {
            // Giá trị ban đầu của StateFlow
            assertEquals(SearchUiState.EmptyQuery, awaitItem())

            // Gõ liên tiếp nhiều ký tự
            viewModel.onSearchQueryChanged("P")
            viewModel.onSearchQueryChanged("Pix")
            viewModel.onSearchQueryChanged("Pixel")

            // Chưa qua 300ms debounce: Không được phát sinh event mới
            expectNoEvents()

            // Tiến thời gian qua 300ms debounce
            testScheduler.advanceTimeBy(301)
            
            // Nhận trạng thái Loading khi bắt đầu gọi repo
            assertEquals(SearchUiState.Loading, awaitItem())

            // Nhận trạng thái Success sau khi repo trả data
            val resultState = awaitItem() as SearchUiState.Success
            assertEquals(1, resultState.products.size)
            assertEquals("Pixel 8", resultState.products.first().name)

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android

#### Q1: Khi nào nên dùng `flatMapLatest`, `flatMapMerge` hay `flatMapConcat`?
**Trả lời chuẩn bản chất:**
- Dùng **`flatMapLatest`**: Cho các luồng thao tác tương tác người dùng UI (Search, Filter, Tab switching). Khi người dùng thực hiện hành động mới, toàn bộ công việc của hành động cũ **phải bị hủy ngay lập tức** để tránh lãng phí tài nguyên và ngăn ngừa race conditions.
- Dùng **`flatMapMerge`**: Khi cần thực hiện đồng thời nhiều tác vụ độc lập mà không quan tâm thứ tự hoàn thành (ví dụ: tải song song avatar của 10 người bạn trong danh bạ). Có thể chỉnh `concurrency` để giới hạn số tác vụ đồng thời.
- Dùng **`flatMapConcat`**: Khi thứ tự thực thi là điều kiện tiên quyết (ví dụ: ghi log tuần tự vào disk, hoặc bước 2 bắt buộc phải chờ dữ liệu từ kết quả bước 1).

#### Q2: Phân biệt sự khác nhau giữa `debounce` và `sample`?
**Trả lời chuẩn bản chất:**
- **`debounce(timeout)`**: Phát ra giá trị nếu và chỉ nếu **không có giá trị mới nào** xuất hiện trong khoảng thời gian `timeout` (chờ yên lặng). Phù hợp với ô tìm kiếm gõ phím.
- **`sample(period)`**: Chia dòng thời gian thành các khung chu kỳ cố định `period` (ví dụ mỗi 1000ms) và chỉ phát ra **giá trị mới nhất trong chu kỳ đó**, bỏ qua các giá trị khác. Phù hợp cho việc theo dõi tọa độ GPS hoặc cảm biến con quay hồi chuyển tốc độ cao (100Hz $\rightarrow$ sample 1Hz).

#### Q3: Toán tử `conflate()` hoạt động thế nào dưới tầng thấp (Under the hood)?
**Trả lời chuẩn bản chất:**
`conflate()` là một shortcut cấu hình sẵn của toán tử `buffer`:
```kotlin
public fun <T> Flow<T>.conflate(): Flow<T> = buffer(capacity = 0, onBufferOverflow = BufferOverflow.DROP_OLDEST)
```
Nó thiết lập buffer capacity bằng 0 và cơ chế overflow là `DROP_OLDEST`. Khi collector đang bận xử lý, emitter sẽ ghi đè giá trị cũ bằng giá trị mới nhất. Khi collector xử lý xong, nó luôn nhận được giá trị mới nhất của stream mà không bao giờ bị nghẽn (tương tự như hành vi conflation của `StateFlow`).

---

### 6.2 Lỗi Runtime/Compile thường gặp & Cách khắc phục

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Crash: `IllegalStateException: Flow invariant is violated`** | Gọi `withContext()` trực tiếp bên trong block `flow { }` để phát dữ liệu ở thread khác. | Xóa `withContext`, sử dụng toán tử `.flowOn(Dispatcher)` ở phía ngoài chuỗi Flow. |
| **`combine` không bao giờ emit giá trị nào** | Một trong các Flow truyền vào `combine` là Cold Flow rỗng hoặc chưa bao giờ emit giá trị đầu tiên. | Đảm bảo mọi Flow thành phần đều có giá trị bằng cách dùng `onStart { emit(defaultVal) }` hoặc dùng `StateFlow`. |
| **Memory Spike khi emit lượng dữ liệu lớn** | Dùng `buffer(Channel.UNLIMITED)` làm bộ nhớ RAM tăng vọt khi Collector xử lý chậm hơn nhiều so với Producer. | Giới hạn dung lượng buffer bằng kích thước cố định (ví dụ `buffer(Channel.CONFLATED)` hoặc `buffer(64, DROP_OLDEST)`). |
| **Recomposition liên tục làm giật lag giao diện** | Luồng StateFlow/Flow phát ra các object giống nhau nhưng không implement `equals()` đúng cách, hoặc thiếu `distinctUntilChanged()`. | Sử dụng `data class` cho Models/UiState và chèn `distinctUntilChanged()` vào pipeline trước khi emit ra UI. |

---

*Bài trước: [02 — Cold Flow vs Hot Flow: StateFlow & SharedFlow](02-cold-flow-vs-hot-flow.md)*  
*Bài tiếp theo: [04 — Flow trong Android Lifecycle: collectAsStateWithLifecycle](04-flow-lifecycle-android.md)*

# Bài 06 — ViewModel + Flow + Compose: Kiến trúc UDF End-to-End

> **Module:** 2 — Architecture & ViewModel  
> **Prerequisite:** Module 1 — Kotlin Coroutines & Flow Foundation (Đặc biệt: Bài 02 & 04)  
> **Official Docs:**
> - [Guide to App Architecture — Android Developers](https://developer.android.com/topic/architecture)
> - [UI Layer Architecture & UDF](https://developer.android.com/topic/architecture/ui-layer)
> - [State Holders and UI State](https://developer.android.com/topic/architecture/ui-layer/stateholders)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Unidirectional Data Flow (UDF - Dòng chảy dữ liệu một chiều)** là một mô hình thiết kế kiến trúc phần mềm trong đó **trạng thái (State) chỉ chảy theo một hướng từ trên xuống dưới (xuống UI)**, và các **sự kiện (Events/Actions) chỉ chảy theo hướng ngược lại từ dưới lên trên (lên ViewModel)**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   UNIDIRECTIONAL DATA FLOW (UDF) CYCLE                 │
│                                                                        │
│                      ┌──────────────────────┐                          │
│                      │      VIEWMODEL       │                          │
│                      │  (UI State Holder)   │                          │
│                      └──────────┬───────────┘                          │
│                                 │                                      │
│               UiState (Flow)    │   ▲   UiAction (User Interaction)    │
│            Dữ liệu chảy XUỐNG   │   │   Sự kiện chảy LÊN               │
│                                 ▼   │                                  │
│                      ┌──────────────┴───────┐                          │
│                      │    JETPACK COMPOSE   │                          │
│                      │      (UI Layer)      │                          │
│                      └──────────────────────┘                          │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Ba trụ cột cốt lõi của tầng UI trong Modern Android

#### 1. `UiState` (Trạng thái giao diện)
- Là một **snapshot bất biến (Immutable Snapshot)** biểu diễn toàn bộ dữ liệu cần thiết để giao diện có thể tự vẽ (render) tại một thời điểm nhất định.
- Bất kỳ thay đổi nào trên UI (nội dung text, màu sắc nút bấm, cờ loading, danh sách items) **đều bắt buộc phải được phản ánh qua `UiState`**.
- Chuẩn hóa bằng `data class` hoặc `sealed interface`.

#### 2. `UiAction` / `UiIntent` (Hành động người dùng)
- Biểu diễn mọi tương tác xuất phát từ phía người dùng hoặc hệ thống (nhấn nút, nhập text, kéo refresh, cuộn danh sách).
- Được định nghĩa dưới dạng các **Sealed Interface / Sealed Class** gửi từ Compose UI lên ViewModel để xử lý.

#### 3. `UiEffect` / `SideEffect` (Hiệu ứng một lần - One-time Events)
- Biểu diễn các sự kiện thoáng qua (**transient events**) chỉ nên xảy ra đúng một lần và không nên được lưu giữ lại trong State dài hạn.
- Ví dụ: Mở màn hình mới (Navigation), hiển thị SnackBar, hiển thị Toast, đóng BottomSheet.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Vòng đời của ViewModel (ViewModel Lifecycle Internals)

Tại sao ViewModel lại có khả năng lưu giữ State mà không bị reset khi người dùng xoay điện thoại (Configuration Change)?

```
                 QUÁ TRÌNH XOAY MÀN HÌNH (CONFIGURATION CHANGE)

Activity cũ (Dọc)                                     Activity mới (Ngang)
┌───────────────────────┐                             ┌───────────────────────┐
│ onDestroy()           │                             │ onCreate()            │
└──────────┬────────────┘                             └───────────▲───────────┘
           │                                                      │
           ▼                                                      │
┌─────────────────────────────────────────────────────────────────┴───────────┐
│ ViewModelStore (Nằm trong NonConfigurationInstances ở cấp độ Platform Host) │
│                                                                             │
│  Map<String, ViewModel>:                                                    │
│  ["UserViewModel"] ──► UserViewModel Instance (VẪN ĐƯỢC GIỮ NGUYÊN TRONG RAM)│
│                        (StateFlow, CoroutineScope, Cache không bị mất!)     │
└─────────────────────────────────────────────────────────────────────────────┘
```

- Khi Activity bị destroy do xoay màn hình, Android Framework không giải phóng `ViewModelStore`. Đối tượng này được lưu lại tạm thời trong `NonConfigurationInstances`.
- Khi Activity mới được tái sinh (`recreate`), nó kết nối lại với chính `ViewModelStore` cũ thông qua `ViewModelProvider`.
- ViewModel chỉ thực sự bị tiêu hủy (`onCleared()`) khi người dùng chủ động **bấm nút Back (Finish Activity)** hoặc Component bị gỡ vĩnh viễn khỏi BackStack!

---

### 2.2 Thread-Safety: Bản chất Atomic Compare-And-Swap (CAS) của `update { }`

Rất nhiều lập trình viên mắc lỗi nghiêm trọng khi cập nhật `MutableStateFlow` bằng phép gán trực tiếp:

```kotlin
// NGUY HIỂM (RACE CONDITION TRONG MÔI TRƯỜNG ĐA LUỒNG):
_uiState.value = _uiState.value.copy(isLoading = true)
```

#### Tại sao lại nguy hiểm?
Nếu có 2 coroutines chạy trên 2 thread khác nhau cùng thực thi dòng lệnh trên tại cùng một mili-giây:
1. Thread A đọc `_uiState.value` (giả sử `count = 1`).
2. Thread B đọc `_uiState.value` (vẫn là `count = 1`).
3. Thread A gán `_uiState.value = count + 1` (giá trị thành `2`).
4. Thread B gán `_uiState.value = count + 1` (giá trị vẫn ghi đè thành `2` $\rightarrow$ Mất dữ liệu của Thread A!).

#### Giải pháp chuẩn mực: `MutableStateFlow.update { }`
Hàm `update` bên dưới mã nguồn Kotlin Coroutines sử dụng vòng lặp **Atomic Compare-And-Swap (CAS)**:

```kotlin
// Mã nguồn bên dưới của kotlinx.coroutines.flow.StateFlow.kt:
public inline fun <T> MutableStateFlow<T>.update(function: (T) -> T) {
    while (true) {
        val prevValue = value
        val nextValue = function(prevValue)
        if (compareAndSet(prevValue, nextValue)) {
            return
        }
    }
}
```

- `compareAndSet(prevValue, nextValue)` là một lệnh vi xử lý phần cứng CPU (Atomic CPU instruction).
- Nếu trong lúc hàm `function` đang tính toán mà một thread khác đã thay đổi `value`, lệnh `compareAndSet` sẽ trả về `false`.
- Vòng lặp `while (true)` lập tức đọc lại giá trị mới nhất `prevValue` và thử lại cho đến khi việc ghi đè thành công 100%!

---

### 2.3 Cơ chế Compose Snapshot System & State Immutability

Tại sao `UiState` bắt buộc phải là **Immutable (`val` trong `data class`)**?

```
Compose Snapshot System theo dõi Recomposition:

1. Mutable Object (SAI LẦM):
   state.items.add(newItem) 
   => Tham chiếu vùng nhớ (Memory Reference) KHÔNG ĐỔI!
   => Compose Snapshot System KHÔNG phát hiện được thay đổi!
   => GIAO DIỆN KHÔNG RECOMPOSE (Không cập nhật)!

2. Immutable Data Class với copy() (CHUẨN MỰC):
   _uiState.update { it.copy(items = it.items + newItem) }
   => Tạo ra một Object Instance mới với địa chỉ vùng nhớ mới!
   => Compose so sánh: oldState !== newState (hoặc !oldState.equals(newState))
   => KÍCH HOẠT RECOMPOSITION CHÍNH XÁC TẠI NƠI CẦN CẬP NHẬT!
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Sự tiến hóa kiến trúc UI trên Android

```
Kỷ nguyên 1: God Activity / God Fragment
- Toàn bộ code network, database, view binding, logic nằm chung trong 1 file 3000 dòng.
- Không thể viết Unit Test. Bất kỳ thay đổi nhỏ nào cũng gây crash dây chuyền.

Kỷ nguyên 2: MVC / MVP (Model - View - Presenter)
- Tách được business logic sang Presenter.
- NHƯỢC ĐIỂM: Interface Hell! Mỗi màn hình cần 2-3 interfaces khổng lồ (`UserView`, `UserPresenter`). Presenter giữ tham chiếu trực tiếp đến View Interface, dễ gây Memory Leak nếu Activity bị xoay máy khi request đang chạy dở.

Kỷ nguyên 3: MVVM sơ khai (Phân mảnh LiveData)
- ViewModel không còn giữ tham chiếu View.
- NHƯỢC ĐIỂM: Phân mảnh trạng thái (State Fragmentation):
  class UserViewModel : ViewModel() {
      val isLoading = MutableLiveData<Boolean>()
      val user = MutableLiveData<User>()
      val errorMessage = MutableLiveData<String>()
      val cartCount = MutableLiveData<Int>()
  }
  => HẬU QUẢ: Không có tính nhất quán (Inconsistent State). Có những thời điểm `isLoading = true` nhưng `user` vẫn hiện, hoặc `errorMessage` hiện đè lên dữ liệu cũ do các luồng LiveData phát độc lập không đồng bộ!

Kỷ nguyên 4 (Hiện đại): UDF với Consolidated State
- Gom toàn bộ trạng thái vào MỘT đối tượng duy nhất: UiState.
- Trạng thái màn hình tại bất kỳ thời điểm nào là một nghiệm xác định duy nhất: UI = f(UiState).
```

---

### 3.2 Bảng so sánh MVVM Phân Mảnh vs Modern UDF

| Tiêu chí | MVVM phân mảnh (Nhiều LiveData/Flow) | Modern UDF (Consolidated UiState) |
|---|---|---|
| **Số lượng stream** | 5-10 streams riêng lẻ | **Duy nhất 1 `StateFlow<UiState>`** |
| **Tính toàn vẹn dữ liệu** | Dễ bị lệch pha (race condition giữa các streams) | Đảm bảo tính nhất quán tuyệt đối 100% |
| **Khả năng Debug / Logging**| Khó theo dõi (mỗi stream đổi 1 lúc) | Cực kỳ dễ dàng (chỉ cần in log mỗi lần `UiState` thay đổi) |
| **Khả năng tái hiện bug** | Rất khó tái hiện các lỗi gián đoạn | Dễ dàng: Chỉ cần inject lại chính xác snapshot `UiState` đó vào Composable Preview |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Phân biệt rạch ròi: `UiState` vs `UiEffect`

```
Sự kiện này thuộc về đâu?
│
├── Dữ liệu cần tồn tại xuyên suốt để vẽ giao diện? ──────────────► UiState (StateFlow)
│   (Danh sách sản phẩm, trạng thái loading, text input, user profile)
│
└── Hành động thoáng qua chỉ chạy đúng 1 lần? ───────────────────► UiEffect (Channel)
    (Hiển thị Toast, SnackBar, Navigate sang màn hình khác, Show Dialog)
```

#### Tại sao KHÔNG ĐƯỢC dùng `StateFlow` hoặc `SingleLiveEvent` cho One-time Events?
Nếu bạn lưu cờ `navigateToDetail = true` trong `StateFlow`:
1. Khi chuyển sang màn hình Detail rồi người dùng bấm Back quay lại màn hình cũ.
2. Composable của màn hình cũ được recompose / collect lại `StateFlow`.
3. Vì `StateFlow` có tính chất Replay = 1, nó sẽ phát lại giá trị `navigateToDetail = true`!
4. **HẬU QUẢ:** Ứng dụng tự động nhảy lại vào màn hình Detail một lần nữa (Navigation Loop Bug)!

#### Giải pháp chuẩn mực của Google: `Channel<UiEffect>`
Sử dụng `Channel<UiEffect>(Channel.BUFFERED)` và expose ra ngoài bằng `.receiveAsFlow()`. Dữ liệu trong Channel chỉ được tiêu thụ đúng một lần duy nhất (One-time delivery).

---

### 4.2 Các nguyên tắc thiết kế ViewModel chuẩn mực

1. **ViewModel KHÔNG BAO GIỜ import Android Framework:**
   - Không import `android.content.Context`.
   - Không import `android.view.View`.
   - Không import `androidx.navigation.NavController`.
   - Không import `@Composable`.
   - *Lý do:* Giữ ViewModel thuần túy Kotlin giúp chạy JVM Unit Test trong 50ms mà không cần khởi động Robolectric hay Emulator!

2. **Luôn đóng gói State nội bộ:**
   ```kotlin
   // CHUẨN MỰC:
   private val _uiState = MutableStateFlow(CheckoutUiState())
   val uiState: StateFlow<CheckoutUiState> = _uiState.asStateFlow()
   ```

3. **Xử lý String Resource trong ViewModel mà không cần Context:**
   Đừng truyền `Context` vào ViewModel chỉ để gọi `context.getString(R.string.error)`. Thay vào đó, đóng gói bằng Sealed Interface:
   ```kotlin
   sealed interface UiText {
       data class DynamicString(val value: String) : UiText
       data class StringResource(@StringRes val resId: Int, val args: List<Any> = emptyList()) : UiText
   }
   ```

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng tính năng **Thanh toán đơn hàng (Checkout Flow)** hoàn chỉnh từ Domain/Data đến ViewModel và Jetpack Compose UI.

### Bước 1: Thiết kế `UiState`, `UiAction`, `UiEffect`

```kotlin
// 1. Immutable State
data class CartItem(val id: String, val name: String, val price: Double)

enum class PaymentMethod { COD, CREDIT_CARD, E_WALLET }

data class CheckoutUiState(
    val items: List<CartItem> = emptyList(),
    val selectedPayment: PaymentMethod = PaymentMethod.COD,
    val discountPercent: Int = 0,
    val isProcessing: Boolean = false,
    val errorMessage: String? = null
) {
    // Computed properties thuận tiện cho UI
    val subtotal: Double get() = items.sumOf { it.price }
    val total: Double get() = subtotal * (1 - discountPercent / 100.0)
}

// 2. User Actions (Đẩy từ UI lên ViewModel)
sealed interface CheckoutUiAction {
    data class ChangePaymentMethod(val method: PaymentMethod) : CheckoutUiAction
    data class ApplyVoucher(val code: String) : CheckoutUiAction
    data object ConfirmPayment : CheckoutUiAction
}

// 3. One-time Effects (Đẩy từ ViewModel sang UI)
sealed interface CheckoutUiEffect {
    data class ShowToast(val message: String) : CheckoutUiEffect
    data class NavigateToSuccessScreen(val orderId: String) : CheckoutUiEffect
}
```

---

### Bước 2: ViewModel Layer (UDF Reducer & Channel Effect)

```kotlin
class CheckoutViewModel(
    private val checkoutRepository: CheckoutRepository,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : ViewModel() {

    private val _uiState = MutableStateFlow(CheckoutUiState())
    val uiState: StateFlow<CheckoutUiState> = _uiState.asStateFlow()

    // Channel buffer để gửi One-time Effects
    private val _effectChannel = Channel<CheckoutUiEffect>(Channel.BUFFERED)
    val effect = _effectChannel.receiveAsFlow()

    init {
        loadInitialCart()
    }

    private fun loadInitialCart() {
        val mockItems = listOf(
            CartItem("1", "Tai nghe Bluetooth Sony", 150.0),
            CartItem("2", "Ốp lưng điện thoại", 20.0)
        )
        _uiState.update { it.copy(items = mockItems) }
    }

    // Single Entry Point xử lý toàn bộ User Actions
    fun onAction(action: CheckoutUiAction) {
        when (action) {
            is CheckoutUiAction.ChangePaymentMethod -> {
                _uiState.update { it.copy(selectedPayment = action.method) }
            }
            is CheckoutUiAction.ApplyVoucher -> {
                applyVoucher(action.code)
            }
            is CheckoutUiAction.ConfirmPayment -> {
                processCheckout()
            }
        }
    }

    private fun applyVoucher(code: String) {
        if (code.equals("SALE50", ignoreCase = true)) {
            _uiState.update { it.copy(discountPercent = 50, errorMessage = null) }
            viewModelScope.launch {
                _effectChannel.send(CheckoutUiEffect.ShowToast("Áp dụng mã giảm giá 50% thành công!"))
            }
        } else {
            _uiState.update { it.copy(errorMessage = "Mã giảm giá không hợp lệ") }
        }
    }

    private fun processCheckout() {
        viewModelScope.launch {
            _uiState.update { it.copy(isProcessing = true, errorMessage = null) }
            
            val result = checkoutRepository.submitOrder(
                items = _uiState.value.items,
                total = _uiState.value.total,
                paymentMethod = _uiState.value.selectedPayment
            )

            result.fold(
                onSuccess = { orderId ->
                    _uiState.update { it.copy(isProcessing = false) }
                    // Gửi Effect chuyển màn hình
                    _effectChannel.send(CheckoutUiEffect.NavigateToSuccessScreen(orderId))
                },
                onFailure = { error ->
                    _uiState.update { 
                        it.copy(isProcessing = false, errorMessage = error.message ?: "Thanh toán thất bại") 
                    }
                }
            )
        }
    }
}
```

---

### Bước 3: UI Layer (Jetpack Compose Screen)

```kotlin
@Composable
fun CheckoutScreen(
    viewModel: CheckoutViewModel,
    onNavigateSuccess: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    // Thu thập One-time Effects an toàn trong Compose
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is CheckoutUiEffect.ShowToast -> {
                    snackbarHostState.showSnackbar(effect.message)
                }
                is CheckoutUiEffect.NavigateToSuccessScreen -> {
                    onNavigateSuccess(effect.orderId)
                }
            }
        }
    }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) },
        topBar = { TopAppBar(title = { Text("Thanh toán đơn hàng") }) }
    ) { padding ->
        Column(
            modifier = modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp)
        ) {
            // Danh sách items
            Text("Sản phẩm:", style = MaterialTheme.typography.titleMedium)
            uiState.items.forEach { item ->
                Row(
                    modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Text(item.name)
                    Text("$${item.price}")
                }
            }

            Divider(modifier = Modifier.padding(vertical = 12.dp))

            // Phương thức thanh toán
            Text("Phương thức thanh toán:", style = MaterialTheme.typography.titleMedium)
            PaymentMethod.values().forEach { method ->
                Row(
                    verticalAlignment = Alignment.CenterVertically,
                    modifier = Modifier
                        .fillMaxWidth()
                        .clickable { viewModel.onAction(CheckoutUiAction.ChangePaymentMethod(method)) }
                ) {
                    RadioButton(
                        selected = (uiState.selectedPayment == method),
                        onClick = { viewModel.onAction(CheckoutUiAction.ChangePaymentMethod(method)) }
                    )
                    Text(text = method.name, modifier = Modifier.padding(start = 8.dp))
                }
            }

            Spacer(modifier = Modifier.height(16.dp))

            // Áp dụng Voucher
            Button(
                onClick = { viewModel.onAction(CheckoutUiAction.ApplyVoucher("SALE50")) },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Áp dụng mã SALE50 (-50%)")
            }

            if (uiState.errorMessage != null) {
                Text(
                    text = uiState.errorMessage!!,
                    color = MaterialTheme.colorScheme.error,
                    modifier = Modifier.padding(top = 8.dp)
                )
            }

            Spacer(modifier = Modifier.weight(1f))

            // Tổng tiền & Submit
            Text(
                text = "Tổng thanh toán: $${uiState.total}",
                style = MaterialTheme.typography.headlineSmall,
                color = MaterialTheme.colorScheme.primary
            )

            Spacer(modifier = Modifier.height(12.dp))

            Button(
                onClick = { viewModel.onAction(CheckoutUiAction.ConfirmPayment) },
                enabled = !uiState.isProcessing,
                modifier = Modifier.fillMaxWidth()
            ) {
                if (uiState.isProcessing) {
                    CircularProgressIndicator(color = MaterialTheme.colorScheme.onPrimary, modifier = Modifier.size(24.dp))
                } else {
                    Text("Xác nhận đặt hàng")
                }
            }
        }
    }
}
```

---

### Bước 4: Viết Unit Test thuần JVM cho ViewModel với `Turbine`

```kotlin
class CheckoutViewModelTest {

    private val testDispatcher = StandardTestDispatcher()
    private lateinit var fakeRepository: CheckoutRepository
    private lateinit var viewModel: CheckoutViewModel

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        fakeRepository = object : CheckoutRepository {
            override suspend fun submitOrder(
                items: List<CartItem>,
                total: Double,
                paymentMethod: PaymentMethod
            ): Result<String> = Result.success("ORD_9999")
        }
        viewModel = CheckoutViewModel(fakeRepository, testDispatcher)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun applyVoucher_validCode_updatesDiscountAndEmitsEffect() = runTest(testDispatcher) {
        // Test State
        viewModel.uiState.test {
            val initialState = awaitItem()
            assertEquals(0, initialState.discountPercent)

            // Khi user apply mã giảm giá
            viewModel.onAction(CheckoutUiAction.ApplyVoucher("SALE50"))

            val updatedState = awaitItem()
            assertEquals(50, updatedState.discountPercent)
            assertEquals(85.0, updatedState.total, 0.0) // (150 + 20) * 0.5 = 85.0
        }

        // Test Effect
        viewModel.effect.test {
            // Hiệu ứng Toast phải được bắn ra
            val effect = awaitItem()
            assertTrue(effect is CheckoutUiEffect.ShowToast)
            assertEquals("Áp dụng mã giảm giá 50% thành công!", (effect as CheckoutUiEffect.ShowToast).message)
        }
    }

    @Test
    fun confirmPayment_success_navigatesToSuccess() = runTest(testDispatcher) {
        viewModel.effect.test {
            viewModel.onAction(CheckoutUiAction.ConfirmPayment)
            testScheduler.advanceUntilIdle()

            val effect = awaitItem()
            assertTrue(effect is CheckoutUiEffect.NavigateToSuccessScreen)
            assertEquals("ORD_9999", (effect as CheckoutUiEffect.NavigateToSuccessScreen).orderId)
        }
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Tại sao lambda của hàm `MutableStateFlow.update { }` có thể bị gọi nhiều hơn 1 lần?
**Trả lời chuẩn bản chất:**
Bởi vì hàm `update` hoạt động theo cơ chế **Optimistic Concurrency Control (Atomic CAS loop)**. Nếu trong quá trình lambda đang tính toán giá trị mới, một thread khác đã hoàn thành việc ghi dữ liệu vào `MutableStateFlow`, lệnh `compareAndSet` sẽ bị fail. Khi đó, vòng lặp `while (true)` sẽ bắt đầu lại từ đầu và gọi lại lambda một lần nữa với giá trị `prevValue` mới nhất. Do đó, **lambda truyền vào `update { }` bắt buộc phải là một Pure Function (không chứa Side-Effect, không gọi API, không ghi file)**!

#### Q2: Làm thế nào để xử lý Form có 10 trường nhập liệu TextFields mà không gây lag Recomposition?
**Trả lời chuẩn bản chất:**
Nếu mỗi lần người dùng gõ 1 ký tự vào 1 TextField mà toàn bộ màn hình 10 trường đều bị Recompose, giao diện sẽ giật lag. Có 2 giải pháp của Senior Architect:
1. **State Hoisting cục bộ:** Chỉ lưu text tạm thời trong `remember { mutableStateOf("") }` tại nội bộ Composable của từng TextField. Chỉ đồng bộ lên ViewModel khi TextField bị mất focus (`onFocusChanged`) hoặc khi người dùng bấm nút Submit.
2. **Derived State & Fine-grained Composables:** Tách mỗi Form Input thành một Composable con độc lập và truyền lambda chuyên biệt để Compose Compiler bỏ qua recomposition ở các trường không thay đổi.

#### Q3: Khi nào nên chia nhỏ `UiState` thành nhiều StateFlow con?
**Trả lời chuẩn bản chất:**
Mặc dù Google khuyên dùng Consolidated `UiState`, trong các màn hình cực kỳ phức tạp (như Dashboard tài chính hoặc màn hình Map có hàng chục lớp overlay thay đổi ở các tần số khác nhau: vị trí GPS cập nhật 1Hz, thông tin tài khoản đổi 1 ngày/lần), việc gộp chung sẽ làm `UiState` biến đổi liên tục. Trong trường hợp đó, bạn có thể tách thành 2-3 `StateFlow` độc lập theo **tần số thay đổi (Rate of change)** để tối ưu hiệu năng recomposition.

---

### 6.2 Lỗi Runtime thường gặp & Cách khắc phục

| Hiện tượng lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Sự kiện Navigation/SnackBar bị kích hoạt lại mỗi khi xoay máy** | Lưu sự kiện one-time trong `StateFlow` hoặc `SingleLiveEvent`. | Chuyển sang sử dụng `Channel<UiEffect>(Channel.BUFFERED)` và collect bằng `LaunchedEffect(Unit)`. |
| **Giao diện không cập nhật dù đã thêm item vào danh sách** | Dùng `MutableList` trong `UiState` và gọi `state.items.add()`. Địa chỉ tham chiếu không đổi nên Compose bỏ qua Recomposition. | Dùng `List` bất biến và tạo list mới với toán tử `+` (`it.copy(items = it.items + newItem)`). |
| **Crash: `Can't access NavController before onAttach`** | Truyền `NavController` trực tiếp vào ViewModel constructor hoặc hàm của ViewModel. | Tuyệt đối không truyền `NavController` vào ViewModel. Hãy gửi `UiEffect.Navigate` ra ngoài để Composable tự gọi `navController.navigate()`. |

---

*Bài trước: [05 — Flow Exception Handling](../module-1-coroutines-flow/05-flow-exception-handling.md)*  
*Bài tiếp theo: [07 — Domain Layer & Use Cases với Flow](07-domain-layer-usecases.md)*

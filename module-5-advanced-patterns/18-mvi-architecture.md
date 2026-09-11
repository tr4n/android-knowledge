# Bài 18 — MVI Architecture với Compose: Model-View-Intent & Finite State Machine

> **Module:** 5 — Advanced Patterns & Integration  
> **Mức độ:** Senior / Staff Android Architect  
> **Prerequisites:** Module 1 (Coroutines & Flow), Module 2 (Architecture & ViewModel UDF), Module 3 (Compose Foundation)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Kiến trúc **MVI (Model-View-Intent)** là một mô hình kiến trúc phần mềm hướng phản ứng (Reactive Architecture), được xây dựng trên nền tảng lý thuyết **Máy trạng thái hữu hạn (Finite State Machine - FSM)** và mô hình toán học **Redux**. MVI mở rộng nguyên lý **Unidirectional Data Flow (UDF)** thành một vòng tròn khép kín, trong đó mọi thay đổi của giao diện người dùng đều là kết quả của một hàm toán học thuần túy (Pure Function).

```
                  ┌──────────────────────┐
                  │         VIEW         │
                  │   (Jetpack Compose)  │
                  └──────────┬───────────┘
                             │
                  User Event │ Intent
                             ▼
                  ┌──────────────────────┐
                  │        INTENT        │
                  │   (Action / Event)   │
                  └──────────┬───────────┘
                             │
                 Process     │ Reducer
                             ▼
                  ┌──────────────────────┐
                  │        MODEL         │
                  │  (Immutable State)   │
                  └──────────┬───────────┘
                             │
                 Render New  │ StateFlow
                 State       │
                             └───────────► (quay lại VIEW)
```

### Các thuật ngữ cốt lõi:
- **Intent (Ý định người dùng / Hành động):** Đại diện cho một sự kiện hoặc ý định hành động xuất phát từ người dùng hoặc hệ thống (ví dụ: `AddItem`, `ApplyVoucher`, `RetryNetwork`). Trong MVI, View không bao giờ trực tiếp gọi các hàm nghiệp vụ của ViewModel; View chỉ gửi (dispatch/emit) một `Intent`.
- **Model (State - Trạng thái):** Đại diện cho toàn bộ trạng thái của UI tại một thời điểm xác định ($S_n$). Model trong MVI bắt buộc phải là **Immutable Data Class**. UI không thể tự do thay đổi bất kỳ thuộc tính nào của Model.
- **View (Giao diện hiển thị):** Trong Jetpack Compose, View là một tập hợp các Composable functions có nhiệm vụ quan sát luồng `StateFlow<UiState>` và chuyển hóa State thành các thành phần đồ họa (Render).
- **Reducer Function:** Một hàm toán học thuần túy nhận vào trạng thái hiện tại ($S_n$) và một Intent ($I$), tính toán và trả về một trạng thái hoàn toàn mới ($S_{n+1}$):
  $$\text{reduce}: (S_n, I) \to S_{n+1}$$
- **Side-Effect (One-time Event):** Các hành động bất đồng bộ hoặc một lần chỉ xảy ra một lần duy nhất mà không được lưu giữ vào trạng thái bền vững của UI (ví dụ: hiển thị Toast, mở SnackBar, Navigation điều hướng màn hình).
- **Deterministic State Machine:** Tính chất xác định tuyệt đối của hệ thống. Với cùng một trạng thái ban đầu $S_0$ và cùng một chuỗi Intent $[I_1, I_2, \dots, I_k]$, hệ thống luôn luôn kết thúc ở chính xác một trạng thái $S_k$, bất kể thời gian hay môi trường thực thi.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Bản chất Toán học của Reducer & State Machine
Trong khoa học máy tính, một **Finite State Machine (FSM)** được định nghĩa bằng bộ 5 phần tử: $(\Sigma, S, s_0, \delta, F)$ trong đó:
- $S$ là tập hợp tất cả các trạng thái có thể có (`UiState`).
- $\Sigma$ là bảng chữ cái các sự kiện đầu vào (`UiIntent`).
- $s_0 \in S$ là trạng thái khởi đầu (`InitialState`).
- $\delta: S \times \Sigma \to S$ là hàm chuyển đổi trạng thái (**Reducer Function**).

Hàm Reducer phải tuyệt đối tuân thủ tính chất **Pure Function**:
1. **Idempotence / Deterministic:** Không phụ thuộc vào bất kỳ trạng thái toàn cục (global mutable variable) hay môi trường bên ngoài. Cùng input luôn sinh ra cùng output.
2. **No Side-Effects:** Không thực hiện I/O (Network call, Database query, SharedPrefs), không khởi chạy Coroutines, không in Log hay thay đổi biến tham chiếu bên ngoài phạm vi hàm.

### 2.2 Cơ chế xếp hàng Intent (Sequential Intent Queue) chống Race Conditions
Trong MVVM thông thường, nếu người dùng click liên tiếp nhiều nút (Intent Storm) hoặc các phản hồi mạng trả về lệch pha, các hàm ViewModel có thể cập nhật `StateFlow` đồng thời và gây ra hiện tượng **Race Condition** hoặc **Lost Update**.

MVI xử lý triệt để bài toán này bằng cách đưa toàn bộ Intent vào một kênh đệm tuần tự:
```
[User Taps] ──► Channel<Intent>(Channel.UNLIMITED) ──► Conflate/Buffer
                                                               │
                                                               ▼
[Actor/Collector Coroutine] ◄── Lần lượt xử lý từng Intent ─────┘
            │
            ├─► 1. Gọi UseCase / Repository (nếu cần xử lý Async)
            ├─► 2. Trả về Internal Mutation (Result)
            └─► 3. Reducer: State(n) + Result ──► State(n+1)
```

### 2.3 Sơ đồ Kiến trúc Toàn diện MVI với Jetpack Compose

```
┌────────────────────────────────────────────────────────────────────────┐
│                              VIEW LAYER                                │
│                                                                        │
│   Composable Screen                                                    │
│   ├── collectAsStateWithLifecycle() ◄────────────┐                     │
│   └── LaunchedEffect(Effect Flow) ◄──────┐       │                     │
│             │                            │       │                     │
│             │ emits                      │       │                     │
│             ▼                            │       │                     │
└───────── CartIntent ─────────────────────┼───────┼─────────────────────┘
              │                            │       │
              ▼                            │       │
┌──────────────────────────────────────────┼───────┼─────────────────────┐
│                          VIEWMODEL (MVI ENGINE)  │                     │
│                                          │       │                     │
│   Intent Channel (Sequential Queue)      │       │                     │
│             │                            │       │                     │
│             ▼                            │       │                     │
│   handleIntent(intent)                   │       │                     │
│             │                            │       │                     │
│             ├── [Async Work] ────► Domain Layer (UseCase/Repository)   │
│             │       │                    │                             │
│             │       ◄── Result ──────────┘                             │
│             │                                                          │
│             ├── [One-time Action] ──────► Channel<Effect>              │
│             │                                    │                     │
│             ▼                                    ▼                     │
│   reduce(currentState, mutation)          receiveAsFlow()              │
│             │                                                          │
│             ▼                                                          │
│   MutableStateFlow<State>.update { } ──► StateFlow<State>              │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.4 So sánh MVI thuần Kotlin vs Orbit-MVI vs Molecule
1. **MVI thuần Kotlin (Vanilla MVI):** Sử dụng Kotlin Coroutines `Channel`, `StateFlow` và `sealed interface`. Hoàn toàn không phụ thuộc vào thư viện ngoài (Zero 3rd-party dependencies), minh bạch tuyệt đối, kiểm soát 100% vòng đời.
2. **Orbit-MVI:** Sử dụng cú pháp DSL (`intent { reduce { ... } }`). Giúp giảm bớt boilerplate, cung cấp sẵn cơ chế xử lý background/main thread và side-effect handling, nhưng tạo ra dependency với bên thứ ba.
3. **CashApp Molecule:** Sử dụng Jetpack Compose Compiler để viết logic ViewModel dưới dạng Composable function (`@Composable fun models(): State`). Hiện đại nhưng đòi hỏi team phải am hiểu sâu về Compose runtime trong tầng Presenter.

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Hạn chế chết người của MVVM truyền thống trong hệ thống lớn

Trong kiến trúc MVVM cổ điển, ViewModel thường sở hữu nhiều biến trạng thái rời rạc và rất nhiều hàm public xử lý logic:

```kotlin
// ❌ ANTI-PATTERN: MVVM với trạng thái phân mảnh và hàm tự do
class LegacyCartViewModel : ViewModel() {
    val items = MutableStateFlow<List<CartItem>>(emptyList())
    val isLoading = MutableStateFlow(false)
    val errorMessage = MutableStateFlow<String?>(null)
    val voucher = MutableStateFlow<Voucher?>(null)
    val totalPrice = MutableStateFlow(0.0)

    fun onSearch(query: String) { /* ... */ }
    fun onApplyVoucher(code: String) { /* ... */ }
    fun onCheckoutClicked() { /* ... */ }
}
```

**Hậu quả:**
1. **Trạng thái bất đồng nhất (Inconsistent / Illegal State):** `isLoading == true` nhưng `items.isNotEmpty()`, đồng thời `errorMessage != null`. Không ai biết giao diện nên render Loading Spinner hay Error Dialog hay Danh sách sản phẩm!
2. **Không thể dự đoán thứ tự gọi hàm (Unpredictable Invocations):** Nếu người dùng nhấn "Apply Voucher" cùng lúc khi tiến trình "Sync Cart" đang chạy ngầm, thứ tự xử lý phụ thuộc hoàn toàn vào độ trễ mạng (Network Latency). Điều này dẫn đến các lỗi **Heisenbugs** (lỗi xuất hiện ngẫu nhiên trong production nhưng không thể tái hiện lại được trong môi trường dev).
3. **Khó viết Unit Test toàn diện:** Bạn phải mock và assert từng biến `StateFlow` riêng biệt sau mỗi lần gọi hàm.

### 3.2 Giá trị vượt trội của MVI
- **Một nguồn chân lý duy nhất (Single Source of Truth - SSOT):** Toàn bộ trạng thái màn hình được cô đọng trong một object duy nhất: `CartState`. Không bao giờ xảy ra tình trạng mâu thuẫn giữa các biến.
- **Tính năng Replay & Time-Travel Debugging:** Mọi hành vi người dùng được ghi lại dưới dạng danh sách các Intent tuần tự. Khi app gặp crash, bạn chỉ cần xuất log chuỗi `[Intent_1, Intent_2, Intent_3]` và đưa vào Reducer là có thể tái hiện chính xác 100% bug tại máy dev.
- **Unit Test cực nhanh và dễ bảo trì:** Hàm Reducer là hàm thuần túy, không cần Mockito, không cần Coroutine Test Rule, có thể test hàng chục kịch bản trạng thái chỉ bằng các câu lệnh `assertEquals(expectedState, reduce(currentState, intent))`.

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Bảng so sánh MVVM (với UDF) vs MVI

| Tiêu chí | MVVM (UDF chuẩn) | MVI (Model-View-Intent) |
|---|---|---|
| **Đầu vào của ViewModel** | Các hàm công khai (`fun onEvent()`) | Luồng Intent duy nhất (`sendIntent(Intent)`) |
| **Đầu ra của ViewModel** | `StateFlow<UiState>` + `Channel<UiEffect>` | `StateFlow<UiState>` + `Channel<UiEffect>` |
| **Biến đổi trạng thái** | `_uiState.update { ... }` phân tán trong các hàm | Tập trung 100% tại hàm `reduce(state, mutation)` |
| **Bản chất logic** | Hướng thao tác (Action-oriented) | Hướng máy trạng thái (State Machine / FSM) |
| **Chi phí triển khai (Boilerplate)** | Vừa phải | Cao hơn (cần định nghĩa Intent, Mutation, Reducer) |
| **Khả năng dự đoán (Predictability)** | Khá cao | Tuyệt đối (Deterministic 100%) |
| **Kịch bản phù hợp** | CRUD, Màn hình đơn giản đến trung bình | E-Commerce Checkout, Banking, Trading, Media Editor |

### 4.2 Do's and Don'ts từ Google & Senior Architects

#### DO:
1. **Sử dụng `sealed interface` cho State, Intent và Effect:** Giúp Kotlin Compiler kiểm tra tính đầy đủ (`when` exhaustive check), đảm bảo không bỏ sót bất kỳ nhánh trạng thái nào.
2. **Đảm bảo Reducer là Pure Function:** Không bao giờ đưa Coroutine Context, Network API call, hay Random values vào trong Reducer.
3. **Tách biệt `Intent` (người dùng muốn) và `Mutation/Result` (kết quả tính toán):**
   - Người dùng gửi: `CartIntent.ApplyVoucher(code)`.
   - Tiến trình async trả về: `CartMutation.VoucherApplied(voucher)` hoặc `CartMutation.VoucherFailed(error)`.
   - Reducer chỉ xử lý `CartMutation` để sinh ra `CartState` mới.

#### DON'T:
1. **KHÔNG dùng MVI cho màn hình tĩnh hoặc CRUD đơn giản:** Màn hình FAQ, About Us, Profile view chỉ đọc sẽ trở nên phức tạp quá mức cần thiết (Over-engineering).
2. **KHÔNG giữ các biến Mutable trong State:** Tránh tuyệt đối `var`, `MutableList`, `ArrayList`. Luôn sử dụng `val` và `ImmutableList` từ thư viện `kotlinx.collections.immutable`.
3. **KHÔNG dispatch Intent từ bên trong Reducer:** Điều này sẽ dẫn đến vòng lặp vô tận (Infinite State Loop) làm tràn bộ nhớ (OutOfMemoryError).

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Chúng ta sẽ triển khai tính năng **Giỏ hàng & Áp dụng Voucher đa tầng (Advanced Shopping Cart)** chuẩn Clean Architecture và MVI thuần Kotlin.

### Bước 1: Khai báo Contract (`UiState`, `UiIntent`, `UiEffect`)

Tạo tệp `CartContract.kt`:

```kotlin
package com.example.mvi.cart

import androidx.compose.runtime.Immutable
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf

// 1. Data Models
@Immutable
data class CartItem(
    val id: String,
    val name: String,
    val unitPrice: Double,
    val quantity: Int
) {
    val subtotal: Double get() = unitPrice * quantity
}

@Immutable
data class Voucher(
    val code: String,
    val discountPercent: Double // Ví dụ: 0.15 là giảm 15%
)

// 2. MVI State: Một đối tượng duy nhất phản ánh toàn bộ màn hình
@Immutable
data class CartState(
    val items: ImmutableList<CartItem> = persistentListOf(),
    val appliedVoucher: Voucher? = null,
    val isLoading: Boolean = false,
    val error: String? = null
) {
    val rawTotal: Double get() = items.sumOf { it.subtotal }
    val discountAmount: Double get() = appliedVoucher?.let { rawTotal * it.discountPercent } ?: 0.0
    val finalTotal: Double get() = (rawTotal - discountAmount).coerceAtLeast(0.0)
}

// 3. User Intents: Các ý định hành động xuất phát từ giao diện
sealed interface CartIntent {
    data object LoadCart : CartIntent
    data class AddItem(val item: CartItem) : CartIntent
    data class RemoveItem(val itemId: String) : CartIntent
    data class UpdateQuantity(val itemId: String, val newQuantity: Int) : CartIntent
    data class ApplyVoucher(val code: String) : CartIntent
    data object ClearCart : CartIntent
    data object ProceedToCheckout : CartIntent
}

// 4. Mutations / Internal Results: Kết quả sau khi xử lý logic hoặc Async Work
sealed interface CartMutation {
    data object Loading : CartMutation
    data class CartLoaded(val items: ImmutableList<CartItem>) : CartMutation
    data class ItemAdded(val item: CartItem) : CartMutation
    data class ItemRemoved(val itemId: String) : CartMutation
    data class QuantityUpdated(val itemId: String, val quantity: Int) : CartMutation
    data class VoucherApplied(val voucher: Voucher) : CartMutation
    data class VoucherFailed(val reason: String) : CartMutation
    data object CartCleared : CartMutation
}

// 5. One-time Side Effects: Sự kiện một lần (Toast, Navigate)
sealed interface CartEffect {
    data class ShowToast(val message: String) : CartEffect
    data class NavigateToCheckout(val finalTotal: Double) : CartEffect
}
```

### Bước 2: Triển khai Base MVI ViewModel

Tạo tệp trừu tượng `MviViewModel.kt` để quản lý Intent Queue và Reducer Loop:

```kotlin
package com.example.mvi.core

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.receiveAsFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

abstract class MviViewModel<INTENT, STATE, EFFECT>(
    initialState: STATE
) : ViewModel() {

    // 1. Quản lý State: StateFlow công khai chỉ đọc
    private val _uiState = MutableStateFlow(initialState)
    val uiState: StateFlow<STATE> = _uiState.asStateFlow()

    // 2. Quản lý One-time Side Effects: Channel công bằng không đệm
    private val _effectChannel = Channel<EFFECT>(Channel.BUFFERED)
    val effect = _effectChannel.receiveAsFlow()

    // 3. Quản lý Intent Queue tuần tự
    private val _intentChannel = Channel<INTENT>(Channel.UNLIMITED)

    init {
        // Lắng nghe và xử lý tuần tự từng Intent (Xóa bỏ hoàn toàn race conditions)
        viewModelScope.launch {
            _intentChannel.receiveAsFlow().collect { intent ->
                handleIntent(intent)
            }
        }
    }

    // Giao diện Composable gọi hàm này để phát Intent
    fun sendIntent(intent: INTENT) {
        viewModelScope.launch {
            _intentChannel.send(intent)
        }
    }

    // Đẩy Side Effect ra UI
    protected fun sendEffect(effect: EFFECT) {
        viewModelScope.launch {
            _effectChannel.send(effect)
        }
    }

    // Cập nhật state thông qua reducer
    protected fun setState(reducer: STATE.() -> STATE) {
        _uiState.update { currentState ->
            currentState.reducer()
        }
    }

    // Bắt buộc lớp con triển khai để xử lý Intent
    protected abstract suspend fun handleIntent(intent: INTENT)
}
```

### Bước 3: Triển khai `CartMviViewModel`

Tạo tệp `CartMviViewModel.kt`:

```kotlin
package com.example.mvi.cart

import androidx.lifecycle.viewModelScope
import com.example.mvi.core.MviViewModel
import kotlinx.collections.immutable.toPersistentList
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

class CartMviViewModel(
    // Trong thực tế, inject UseCases qua Hilt constructor
) : MviViewModel<CartIntent, CartState, CartEffect>(initialState = CartState()) {

    init {
        sendIntent(CartIntent.LoadCart)
    }

    override suspend fun handleIntent(intent: CartIntent) {
        when (intent) {
            is CartIntent.LoadCart -> loadInitialCart()
            is CartIntent.AddItem -> mutate(CartMutation.ItemAdded(intent.item))
            is CartIntent.RemoveItem -> mutate(CartMutation.ItemRemoved(intent.itemId))
            is CartIntent.UpdateQuantity -> {
                if (intent.newQuantity <= 0) {
                    mutate(CartMutation.ItemRemoved(intent.itemId))
                } else {
                    mutate(CartMutation.QuantityUpdated(intent.itemId, intent.newQuantity))
                }
            }
            is CartIntent.ApplyVoucher -> applyVoucherAsync(intent.code)
            is CartIntent.ClearCart -> mutate(CartMutation.CartCleared)
            is CartIntent.ProceedToCheckout -> {
                if (uiState.value.items.isEmpty()) {
                    sendEffect(CartEffect.ShowToast("Giỏ hàng đang trống!"))
                } else {
                    sendEffect(CartEffect.NavigateToCheckout(uiState.value.finalTotal))
                }
            }
        }
    }

    // Xử lý tác vụ bất đồng bộ (Network call giả lập)
    private suspend fun applyVoucherAsync(code: String) {
        mutate(CartMutation.Loading)
        // Giả lập network delay gọi Promo API
        delay(800)
        if (code.equals("VIP15", ignoreCase = true)) {
            mutate(CartMutation.VoucherApplied(Voucher("VIP15", 0.15)))
            sendEffect(CartEffect.ShowToast("Áp dụng mã VIP15 thành công: Giảm 15%!"))
        } else {
            mutate(CartMutation.VoucherFailed("Mã khuyến mãi không tồn tại hoặc đã hết hạn!"))
            sendEffect(CartEffect.ShowToast("Mã voucher không hợp lệ"))
        }
    }

    private suspend fun loadInitialCart() {
        mutate(CartMutation.Loading)
        delay(500) // Giả lập đọc Room DB
        val defaultItems = listOf(
            CartItem(id = "1", name = "Tai nghe chống ồn Sony", unitPrice = 250.0, quantity = 1),
            CartItem(id = "2", name = "Bàn phím cơ không dây", unitPrice = 120.0, quantity = 2)
        ).toPersistentList()
        mutate(CartMutation.CartLoaded(defaultItems))
    }

    // Cầu nối chuyển hóa Mutation thành State thông qua Pure Reducer
    private fun mutate(mutation: CartMutation) {
        setState { reduce(this, mutation) }
    }

    companion object {
        // Pure Reducer Function: S(n) + Mutation -> S(n+1)
        // Hàm này độc lập 100%, có thể đem ra Unit Test trực tiếp mà không cần khởi tạo ViewModel!
        fun reduce(state: CartState, mutation: CartMutation): CartState {
            return when (mutation) {
                is CartMutation.Loading -> state.copy(isLoading = true, error = null)
                is CartMutation.CartLoaded -> state.copy(
                    items = mutation.items,
                    isLoading = false,
                    error = null
                )
                is CartMutation.ItemAdded -> {
                    val existingIndex = state.items.indexOfFirst { it.id == mutation.item.id }
                    val updatedItems = if (existingIndex != -1) {
                        val current = state.items[existingIndex]
                        state.items.set(existingIndex, current.copy(quantity = current.quantity + mutation.item.quantity))
                    } else {
                        state.items.add(mutation.item)
                    }
                    state.copy(items = updatedItems)
                }
                is CartMutation.ItemRemoved -> {
                    val updatedItems = state.items.removeAll { it.id == mutation.itemId }
                    state.copy(items = updatedItems)
                }
                is CartMutation.QuantityUpdated -> {
                    val updatedItems = state.items.map { item ->
                        if (item.id == mutation.itemId) item.copy(quantity = mutation.quantity) else item
                    }.toPersistentList()
                    state.copy(items = updatedItems)
                }
                is CartMutation.VoucherApplied -> state.copy(
                    appliedVoucher = mutation.voucher,
                    isLoading = false,
                    error = null
                )
                is CartMutation.VoucherFailed -> state.copy(
                    isLoading = false,
                    error = mutation.reason
                )
                is CartMutation.CartCleared -> state.copy(
                    items = kotlinx.collections.immutable.persistentListOf(),
                    appliedVoucher = null,
                    error = null
                )
            }
        }
    }
}
```

### Bước 4: Jetpack Compose UI Render

Tạo tệp `CartScreen.kt`:

```kotlin
package com.example.mvi.cart

import android.widget.Toast
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun CartRoute(
    viewModel: CartMviViewModel,
    onNavigateToCheckout: (Double) -> Unit
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    val context = LocalContext.current

    // Lắng nghe One-Time Side Effects an toàn vòng đời
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is CartEffect.ShowToast -> {
                    Toast.makeText(context, effect.message, Toast.LENGTH_SHORT).show()
                }
                is CartEffect.NavigateToCheckout -> {
                    onNavigateToCheckout(effect.finalTotal)
                }
            }
        }
    }

    CartScreen(
        state = state,
        onIntent = viewModel::sendIntent
    )
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CartScreen(
    state: CartState,
    onIntent: (CartIntent) -> Unit
) {
    var voucherInput by remember { mutableStateOf("") }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Giỏ hàng (MVI Architecture)") },
                actions = {
                    TextButton(onClick = { onIntent(CartIntent.ClearCart) }) {
                        Text("Xóa hết", color = MaterialTheme.colorScheme.error)
                    }
                }
            )
        },
        bottomBar = {
            CartBottomBar(
                state = state,
                onCheckout = { onIntent(CartIntent.ProceedToCheckout) }
            )
        }
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            if (state.isLoading) {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            } else if (state.items.isEmpty()) {
                Text(
                    text = "Giỏ hàng của bạn đang trống",
                    modifier = Modifier.align(Alignment.Center),
                    style = MaterialTheme.typography.bodyLarge
                )
            } else {
                LazyColumn(
                    modifier = Modifier.fillMaxSize(),
                    contentPadding = PaddingValues(16.dp),
                    verticalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    items(state.items, key = { it.id }) { item ->
                        CartItemRow(
                            item = item,
                            onIncrease = {
                                onIntent(CartIntent.UpdateQuantity(item.id, item.quantity + 1))
                            },
                            onDecrease = {
                                onIntent(CartIntent.UpdateQuantity(item.id, item.quantity - 1))
                            },
                            onDelete = {
                                onIntent(CartIntent.RemoveItem(item.id))
                            }
                        )
                    }

                    item {
                        Spacer(modifier = Modifier.height(16.dp))
                        Row(
                            modifier = Modifier.fillMaxWidth(),
                            horizontalArrangement = Arrangement.spacedBy(8.dp)
                        ) {
                            OutlinedTextField(
                                value = voucherInput,
                                onValueChange = { voucherInput = it },
                                label = { Text("Nhập voucher (VIP15)") },
                                modifier = Modifier.weight(1f),
                                singleLine = true
                            )
                            Button(
                                onClick = {
                                    onIntent(CartIntent.ApplyVoucher(voucherInput))
                                    voucherInput = ""
                                },
                                modifier = Modifier.align(Alignment.CenterVertically)
                            ) {
                                Text("Áp dụng")
                            }
                        }

                        state.error?.let { err ->
                            Text(
                                text = err,
                                color = MaterialTheme.colorScheme.error,
                                style = MaterialTheme.typography.bodySmall,
                                modifier = Modifier.padding(top = 4.dp)
                            )
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun CartItemRow(
    item: CartItem,
    onIncrease: () -> Unit,
    onDecrease: () -> Unit,
    onDelete: () -> Unit
) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier = Modifier
                .padding(16.dp)
                .fillMaxWidth(),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(item.name, style = MaterialTheme.typography.titleMedium)
                Text(
                    "$${item.unitPrice} x ${item.quantity} = $${item.subtotal}",
                    style = MaterialTheme.typography.bodyMedium
                )
            }
            Row(verticalAlignment = Alignment.CenterVertically) {
                IconButton(onClick = onDecrease) { Text("-", style = MaterialTheme.typography.titleLarge) }
                Text("${item.quantity}", modifier = Modifier.padding(horizontal = 8.dp))
                IconButton(onClick = onIncrease) { Text("+", style = MaterialTheme.typography.titleLarge) }
                IconButton(onClick = onDelete) {
                    Text("🗑", color = MaterialTheme.colorScheme.error)
                }
            }
        }
    }
}

@Composable
fun CartBottomBar(
    state: CartState,
    onCheckout: () -> Unit
) {
    Surface(tonalElevation = 8.dp) {
        Column(modifier = Modifier.padding(16.dp)) {
            state.appliedVoucher?.let { voucher ->
                Text(
                    "Đã áp dụng mã: ${voucher.code} (-${(voucher.discountPercent * 100).toInt()}%)",
                    color = MaterialTheme.colorScheme.primary,
                    style = MaterialTheme.typography.bodySmall
                )
            }
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Column {
                    Text("Tổng thanh toán:", style = MaterialTheme.typography.bodyMedium)
                    Text(
                        "$${state.finalTotal}",
                        style = MaterialTheme.typography.headlineSmall,
                        color = MaterialTheme.colorScheme.primary
                    )
                }
                Button(onClick = onCheckout, enabled = state.items.isNotEmpty()) {
                    Text("Thanh toán")
                }
            }
        }
    }
}
```

### Bước 5: Unit Test Reducer Function thuần túy (No Mock, 100% Deterministic)

Tạo tệp `CartReducerTest.kt` trong thư mục `test`:

```kotlin
package com.example.mvi.cart

import kotlinx.collections.immutable.persistentListOf
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNull
import org.junit.Test

class CartReducerTest {

    @Test
    fun `reduce ItemAdded should append new item when item does not exist`() {
        // Given
        val initialState = CartState()
        val newItem = CartItem(id = "101", name = "MacBook Pro", unitPrice = 2000.0, quantity = 1)
        val mutation = CartMutation.ItemAdded(newItem)

        // When
        val newState = CartMviViewModel.reduce(initialState, mutation)

        // Then
        assertEquals(1, newState.items.size)
        assertEquals("MacBook Pro", newState.items[0].name)
        assertEquals(2000.0, newState.finalTotal, 0.01)
    }

    @Test
    fun `reduce ItemAdded should increase quantity when item already exists`() {
        // Given
        val existingItem = CartItem(id = "101", name = "MacBook Pro", unitPrice = 2000.0, quantity = 1)
        val initialState = CartState(items = persistentListOf(existingItem))
        val mutation = CartMutation.ItemAdded(existingItem.copy(quantity = 2))

        // When
        val newState = CartMviViewModel.reduce(initialState, mutation)

        // Then
        assertEquals(1, newState.items.size)
        assertEquals(3, newState.items[0].quantity)
        assertEquals(6000.0, newState.finalTotal, 0.01)
    }

    @Test
    fun `reduce VoucherApplied should calculate correct discount and final total`() {
        // Given: Giỏ hàng trị giá $1000
        val item = CartItem(id = "1", name = "iPad Pro", unitPrice = 1000.0, quantity = 1)
        val initialState = CartState(items = persistentListOf(item))
        val voucher = Voucher(code = "VIP15", discountPercent = 0.15) // Giảm 15%
        val mutation = CartMutation.VoucherApplied(voucher)

        // When
        val newState = CartMviViewModel.reduce(initialState, mutation)

        // Then
        assertEquals(150.0, newState.discountAmount, 0.01)
        assertEquals(850.0, newState.finalTotal, 0.01)
        assertEquals("VIP15", newState.appliedVoucher?.code)
        assertNull(newState.error)
    }

    @Test
    fun `reduce CartCleared should reset items and voucher but keep safe state`() {
        // Given
        val item = CartItem(id = "1", name = "Mouse", unitPrice = 50.0, quantity = 2)
        val initialState = CartState(
            items = persistentListOf(item),
            appliedVoucher = Voucher("SALE10", 0.10)
        )

        // When
        val newState = CartMviViewModel.reduce(initialState, CartMutation.CartCleared)

        // Then
        assertEquals(0, newState.items.size)
        assertNull(newState.appliedVoucher)
        assertEquals(0.0, newState.finalTotal, 0.01)
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior / Staff Android Architect

#### Câu hỏi 1: MVI có gây ra quá nhiều boilerplate code không? Khi nào sự đánh đổi này là hoàn toàn xứng đáng?
**Trả lời:**
- Đúng, MVI yêu cầu viết thêm các định nghĩa `Intent`, `Mutation`, `Effect` và hàm `reduce()`. Đối với một màn hình CRUD tĩnh chỉ đọc 2-3 trường dữ liệu, MVI là sự phức tạp hóa không cần thiết (Overkill).
- Tuy nhiên, sự đánh đổi này **cực kỳ xứng đáng** khi hệ thống thỏa mãn một trong các điều kiện sau:
  1. **Nhiều luồng tương tác đồng thời:** Màn hình giỏ hàng, đặt cọc chứng khoán, cổng thanh toán ngân hàng (Fintech) nơi dữ liệu thay đổi từ nhiều nguồn: Người dùng nhập + WebSocket socket realtime + Push Notification.
  2. **Yêu cầu bảo mật và truy vết nghiêm ngặt (Audit Trail / Event Sourcing):** Đội ngũ cần lưu vết chính xác chuỗi hành vi dẫn đến lỗi thanh toán để giải trình hoặc tái hiện bug 100%.
  3. **Đội ngũ phát triển lớn (Cross-team):** Tách biệt rạch ròi giữa UI (chỉ dispatch Intent) và Logic State Machine (chỉ bảo trì Reducer).

#### Câu hỏi 2: Làm thế nào để xử lý các tác vụ bất đồng bộ (Async Side-Effects như gọi API, lưu Database) trong MVI mà không phá vỡ tính "Pure Function" của Reducer?
**Trả lời:**
- Giải pháp chuẩn mực là mô hình **Intent $\to$ Async Worker $\to$ Mutation $\to$ Reducer**:
  1. Composable phát ra `CartIntent.ApplyVoucher(code)`.
  2. `handleIntent()` trong ViewModel nhận Intent và khởi chạy Coroutine để gọi UseCase/Repository. Tại thời điểm này, ViewModel có thể dispatch một Mutation trung gian: `CartMutation.Loading`.
  3. Khi API trả về kết quả thành công hoặc thất bại, Coroutine chuyển hóa kết quả thành `CartMutation.VoucherApplied` hoặc `CartMutation.VoucherFailed`.
  4. Reducer chỉ nhận `Mutation` (dữ liệu đã xong xuôi) để tính toán State mới. Bản thân Reducer hoàn toàn không hề biết API hay Network là gì!

#### Câu hỏi 3: Phân biệt MVI với mô hình MVVM có áp dụng UDF (Unidirectional Data Flow)?
**Trả lời:**
- MVI là một trường hợp đặc biệt và khắt khe hơn của MVVM + UDF:
  - Trong MVVM UDF: ViewModel có thể có 10 hàm công khai (`fun search()`, `fun filter()`). UI gọi trực tiếp các hàm này. Các hàm có thể gọi `_uiState.update { ... }` rải rác ở bất cứ đâu.
  - Trong MVI: ViewModel chỉ mở **duy nhất một cổng giao tiếp đầu vào** (`sendIntent(Intent)`). Mọi sự kiện đều được serialize thành object dữ liệu và xếp hàng tuần tự. Việc thay đổi State chỉ được phép diễn ra duy nhất tại **một vị trí trung tâm** (`reduce()`).

### 6.2 Bảng gỡ rối các lỗi thực tế (Troubleshooting Matrix)

| Vấn đề / Lỗi thực tế | Nguyên nhân gốc rễ (Root Cause) | Giải pháp triệt để (Solution) |
|---|---|---|
| **Intent Storm / Lỗi Race Condition khi click liên tục** | Dùng `SharedFlow` không cấu hình buffer khiến Intent bị drop, hoặc gọi coroutine song song không đồng bộ hóa | Sử dụng `Channel<Intent>(Channel.UNLIMITED)` làm hàng đợi tuần tự trong `MviViewModel`. |
| **Recomposition Loop (Vòng lặp vẽ lại vô tận)** | Hàm `reduce()` trả về một `State` mới có chứa đối tượng `List` thông thường, khiến Compose so sánh tham chiếu luôn thấy khác nhau | Sử dụng `@Immutable` annotation cho `CartState` và thay thế `List<T>` bằng `kotlinx.collections.immutable.ImmutableList`. |
| **Side Effect (Toast/Navigation) bị hiển thị lại khi xoay màn hình** | Sử dụng `MutableStateFlow` để lưu trữ Effect thay vì Channel | Sử dụng `Channel<EFFECT>(Channel.BUFFERED).receiveAsFlow()` kết hợp `LaunchedEffect` ở Composable. |
| **Reducer bị nghẽn (UI giật lag / Frame Drops)** | Đặt logic tính toán nặng (Hash, Parse JSON lớn) trực tiếp bên trong Reducer trên Main Thread | Chuyển toàn bộ logic nặng sang Coroutine với `Dispatchers.Default` trước khi đẩy Mutation vào Reducer. |

---

## 7. Tổng kết

MVI biến tầng UI và ViewModel thành một **Hệ thống phản ứng xác định (Deterministic Reactive System)**. Bằng cách kết hợp `Channel<Intent>` tuần tự, hàm `reduce()` thuần túy và Compose `collectAsStateWithLifecycle()`, bạn loại bỏ vĩnh viễn các trạng thái bất đồng nhất và lỗi race conditions trong các ứng dụng quy mô lớn.

*Bài tiếp theo: [Bài 19 — Hilt + Flow + Compose: Dependency Injection End-to-End](19-hilt-flow-compose.md)*

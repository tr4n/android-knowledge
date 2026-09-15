# Bài 04 — StateFlow và SharedFlow trên Android (StateFlow and SharedFlow)

> **Tài liệu tham chiếu gốc:** [StateFlow and SharedFlow — Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)  
> **Áp dụng:** Kotlin 2.0+, `kotlinx.coroutines:1.11.0`, Android Architecture Components  
> **Mục tiêu:** Phân biệt rạch ròi Cold Flow và Hot Flow; nắm vững cơ chế của `StateFlow` (UI state holder) và `SharedFlow` (event broadcaster); chuyển đổi Cold Flow sang Hot Flow bằng `stateIn()` / `shareIn()`; và giải mã lý do tại sao `SharingStarted.WhileSubscribed(5000)` là chuẩn mực vàng khi xử lý xoay màn hình (Configuration Changes) trên Android.

---

## 1. Phân biệt Cold Flow vs Hot Flow

Trong phát triển ứng dụng Android, việc hiểu rõ sự khác biệt giữa Cold Flow và Hot Flow là điều kiện tiên quyết để quản lý trạng thái giao diện chính xác và tối ưu tài nguyên:

| Tiêu chí | Cold Flow (Luồng lạnh) | Hot Flow (Luồng nóng) |
| :--- | :--- | :--- |
| **Khởi tạo & Thực thi** | Lười (Lazy) — Chỉ thực thi khi có ít nhất một bên gọi `collect`. | Tự chủ (Active) — Tồn tại độc lập trong bộ nhớ ngay cả khi chưa có ai lắng nghe. |
| **Số lượng Collector** | Đơn kênh (Unicast) — Mỗi collector mới sẽ kích hoạt toàn bộ khối mã của producer chạy lại từ đầu. | Đa kênh (Multicast) — Phát đồng thời cùng một dữ liệu tới tất cả các collectors đang lắng nghe. |
| **Lưu trữ trạng thái** | Không lưu trạng thái trong bộ nhớ. | Có thể lưu giữ giá trị hiện tại (`StateFlow`) hoặc một hàng đợi các giá trị gần nhất (`SharedFlow`). |
| **Ví dụ đại diện** | `flow { ... }`, câu lệnh truy vấn Room Database. | `StateFlow`, `SharedFlow`, sự kiện click, broadcast events. |

---

## 2. `StateFlow` — Nền tảng Quản lý Trạng thái UI (State Holder)

`StateFlow` là một **Hot Flow** phát ra trạng thái hiện tại và các bản cập nhật trạng thái mới nhất cho các bên thu thập.

### 2.1. Các đặc tính then chốt của `StateFlow`:
1. **Luôn có giá trị ban đầu (Initial Value):** Khi khởi tạo `MutableStateFlow(initialValue)`, bạn bắt buộc phải cung cấp giá trị mặc định.
2. **Đọc giá trị đồng bộ:** Có thể truy cập giá trị tức thì bất kỳ lúc nào qua thuộc tính `.value` mà không cần gọi hàm `suspend`.
3. **Bộ đệm Replay = 1:** Bất kỳ collector nào mới đăng ký lắng nghe đều nhận ngay lập tức giá trị mới nhất gần nhất.
4. **Tự động gộp dữ liệu (Conflation):** Nếu producer phát ra giá trị quá nhanh mà collector chưa kịp xử lý, các giá trị trung gian sẽ bị bỏ qua (conflated), collector sẽ chỉ nhận giá trị cuối cùng.
5. **Kiểm tra trùng lặp (Distinctness):** `StateFlow` sử dụng phép so sánh `Any.equals` (`==`). Nếu bạn gán một giá trị mới giống hệt giá trị cũ (`state.value = oldValue`), nó sẽ **không phát lại** cho các collectors.

### 2.2. Triển khai chuẩn trong ViewModel:

```kotlin
// Định nghĩa State bất biến (Immutable State)
data class UserProfileUiState(
    val isLoading: Boolean = false,
    val username: String = "",
    val email: String = "",
    val errorMessage: String? = null
)

class UserProfileViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    // 1. Biến riêng tư có thể ghi (Mutable)
    private val _uiState = MutableStateFlow(UserProfileUiState(isLoading = true))

    // 2. Phơi bày ra ngoài dưới dạng chỉ đọc (Read-only StateFlow)
    val uiState: StateFlow<UserProfileUiState> = _uiState.asStateFlow()

    init {
        loadUserProfile()
    }

    private fun loadUserProfile() {
        viewModelScope.launch {
            try {
                val user = userRepository.getUser()
                _uiState.value = UserProfileUiState(
                    isLoading = false,
                    username = user.name,
                    email = user.email
                )
            } catch (e: Exception) {
                _uiState.value = UserProfileUiState(
                    isLoading = false,
                    errorMessage = e.message
                )
            }
        }
    }
}
```

### 2.3. So sánh `StateFlow` vs `LiveData`

| Tiêu chí | `StateFlow` | `LiveData` |
| :--- | :--- | :--- |
| **Nền tảng** | Thuần Kotlin (`kotlinx.coroutines`) | Phụ thuộc vào Android SDK (`androidx.lifecycle`) |
| **Kotlin Multiplatform (KMP)** | ✅ Có hỗ trợ (dùng chung cho iOS, Desktop, Web) | ❌ Không hỗ trợ |
| **Giá trị ban đầu** | Bắt buộc phải có | Không bắt buộc (có thể null ban đầu) |
| **Toán tử xử lý** | Đầy đủ hệ sinh thái toán tử mạnh mẽ của Flow (`map`, `filter`, `combine`, ...) | Rất hạn chế (chỉ có `Transformations.map/switchMap`) |
| **Tự nhận biết Vòng đời** | Cần kết hợp với `repeatOnLifecycle` hoặc `collectAsStateWithLifecycle` | Tự động ngừng nhận dữ liệu khi Activity ở background |

---

## 3. `SharedFlow` — Kênh Phát Sự Kiện Đa Điểm (Event Broadcaster)

Trong khi `StateFlow` đại diện cho **trạng thái liên tục** (State), thì `SharedFlow` là giải pháp lý tưởng cho các **sự kiện diễn ra một lần (One-off Events)** như:
- Hiển thị một thanh thông báo tạm thời (Snackbar / Toast).
- Kích hoạt điều hướng màn hình (Navigation events).
- Rung phản hồi xúc giác hoặc phát âm thanh.

### 3.1. Cấu hình tham số của `MutableSharedFlow`:

```kotlin
fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T>
```

- **`replay`:** Số lượng giá trị phát ra trước đó được lưu trữ lại để phát lại ngay cho các collector mới tham gia. Với các sự kiện một lần (Events), tham số này thường được đặt là `0`.
- **`extraBufferCapacity`:** Kích thước bộ đệm bổ sung ngoài bộ nhớ replay. Giúp ngăn chặn việc producer bị tạm dừng (suspend) khi bên thu thập xử lý chậm.
- **`onBufferOverflow`:** Chiến lược khi bộ đệm bị đầy:
  - `BufferOverflow.SUSPEND`: Tạm dừng lệnh `emit()` cho đến khi bộ đệm có chỗ trống.
  - `BufferOverflow.DROP_OLDEST`: Vứt bỏ phần tử cũ nhất trong bộ đệm.
  - `BufferOverflow.DROP_LATEST`: Vứt bỏ phần tử mới nhất vừa được phát ra.

### 3.2. Triển khai phát sự kiện thông báo (UI Event):

```kotlin
sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class NavigateToDetail(val itemId: String) : UiEvent
}

class ProductViewModel : ViewModel() {

    private val _eventFlow = MutableSharedFlow<UiEvent>()
    val eventFlow: SharedFlow<UiEvent> = _eventFlow.asSharedFlow()

    fun onAddToCartClicked(product: Product) {
        viewModelScope.launch {
            // Phát sự kiện 1 lần đến UI
            _eventFlow.emit(UiEvent.ShowSnackbar("Đã thêm ${product.name} vào giỏ hàng!"))
        }
    }
}
```

---

## 4. Chuyển đổi Cold Flow sang Hot Flow: `stateIn` và `shareIn`

Trong thực tế, bạn thường lấy dữ liệu từ một Cold Flow (như Room Database hoặc polling network) và muốn chuyển đổi nó thành một Hot Flow trong ViewModel để phát cho UI.

### 4.1. Toán tử `stateIn`
Chuyển đổi một Cold Flow thành một `StateFlow`.

```kotlin
class TaskViewModel(repository: TaskRepository) : ViewModel() {

    // Chuyển đổi Flow từ Room Database thành StateFlow gắn vào viewModelScope
    val taskUiState: StateFlow<List<Task>> = repository.getAllTasksFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

---

## 5. Giải mã Chiến lược: `SharingStarted.WhileSubscribed(5000)`

Tham số `started` trong `stateIn` và `shareIn` xác định khi nào luồng sản xuất upstream bắt đầu chạy và khi nào nó bị hủy bỏ:

1. **`SharingStarted.Eagerly`:** Khởi chạy ngay lập tức khi class được tạo và không bao giờ dừng lại cho đến khi `scope` bị hủy.
2. **`SharingStarted.Lazily`:** Khởi chạy khi có subscriber đầu tiên xuất hiện và không bao giờ dừng lại.
3. **`SharingStarted.WhileSubscribed(stopTimeoutMillis = 5000)`:** Bắt đầu khi có subscriber đầu tiên. Khi số lượng subscriber giảm về 0, nó sẽ đếm ngược đúng thời gian `stopTimeoutMillis` trước khi hủy bỏ luồng upstream.

### Tại sao con số 5000ms (5 giây) là Tiêu chuẩn Vàng trên Android?

```
Kịch bản Xoay Màn hình (Configuration Change):
Người dùng xoay máy:
T = 0ms   : Activity cũ bị Destroy -> Subscriber count giảm về 0!
            (WhileSubscribed bắt đầu bộ đếm lùi 5000ms)
T = 300ms : Activity mới được tạo lại -> Subscriber mới kết nối vào StateFlow!
            (Bộ đếm lùi bị hủy bỏ! Upstream database/network flow VẪN GIỮ NGUYÊN liên tục, không bị restart!)
```

```
Kịch bản Người dùng Nhấn nút Home (App vào Background):
T = 0s    : UI bị ẩn -> repeatOnLifecycle(STARTED) ngừng thu thập -> Subscriber count = 0!
            (WhileSubscribed bắt đầu đếm lùi)
T = 5s    : Hết 5 giây mà không có UI nào kết nối lại -> DỪNG NGAY LẬP TỨC luồng upstream!
            (Ngắt kết nối mạng, dừng lắng nghe Room/Sensor, tiết kiệm tối đa Pin và CPU!)
```

> [!TIP]
> **Tóm tắt quy tắc:** `WhileSubscribed(5000)` giúp bảo vệ ứng dụng khỏi việc khởi động lại luồng tốn kém trong quá trình xoay màn hình (thường diễn ra dưới 1 giây), nhưng đồng thời giải phóng hoàn toàn tài nguyên nếu ứng dụng thực sự bị đưa vào chế độ nền quá 5 giây.

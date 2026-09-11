# Bài 11 — Compose State: remember, rememberSaveable, State Hoisting & derivedStateOf

> **Module:** 3 — Jetpack Compose Foundation  
> **Prerequisite:** [Bài 10 — Composable Functions: Lifecycle, Slot API & @Composable Contract](10-composable-functions.md)  
> **Official Docs:**
> - [State and Jetpack Compose — Android Developers](https://developer.android.com/develop/ui/compose/state)
> - [Save UI state in Compose](https://developer.android.com/develop/ui/compose/state-saving)
> - [Where to hoist state](https://developer.android.com/develop/ui/compose/state-hoisting)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong Jetpack Compose, **State (Trạng thái)** là bất kỳ giá trị nào có thể biến đổi theo thời gian (ví dụ: chuỗi ký tự trong ô tìm kiếm, cờ bật tắt Dark Mode, số lượng sản phẩm trong giỏ hàng).

Khác với mô hình hướng đối tượng cũ nơi View tự giữ và biến đổi trạng thái của chính nó, Compose tuân theo triết lý hàm toán học phản ứng: **Giao diện người dùng là một hàm thuần túy của State**:

$$\text{UI} = f(\text{State})$$

```
┌────────────────────────────────────────────────────────────────────────┐
│                        COMPOSE STATE SYSTEM                            │
│                                                                        │
│   State<out T> (Interface chỉ đọc - Read-only Observable)              │
│         │                                                              │
│         └── MutableState<T> (Interface ghi nhận - Writable Observable) │
│               ├── .value: T                                            │
│               └── Khởi tạo: mutableStateOf(initialValue)               │
│                                                                        │
│   Công cụ lưu giữ trạng thái:                                          │
│   ├── remember { }          ──► Sống trong Composition                 │
│   ├── rememberSaveable { }  ──► Sống qua Xoay màn hình & Process Death │
│   └── derivedStateOf { }    ──► Tối ưu hóa tính toán phái sinh         │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Khái niệm và thuật ngữ cốt lõi

#### `State<T>` vs `MutableState<T>`
- **`State<T>`**: Interface cung cấp thuộc tính `val value: T`. Compose Snapshot System sẽ tự động theo dõi (track) mọi Composable nào đọc thuộc tính `value` này để kích hoạt Recomposition khi nó biến đổi.
- **`MutableState<T>`**: Kế thừa từ `State<T>`, bổ sung khả năng ghi đè giá trị qua `var value: T`. Thường dùng cú pháp ủy quyền thuộc tính (Property Delegation) `by`:
  ```kotlin
  var count by remember { mutableStateOf(0) }
  ```

#### Stateful vs Stateless Composables
- **Stateful Composable:** Hàm Composable tự khởi tạo và nắm giữ State bên trong thân hàm (thường khó tái sử dụng và khó test).
- **Stateless Composable:** Hàm Composable **không nắm giữ State**, nó chỉ nhận State từ ngoài vào qua tham số và phát ra sự kiện qua các lambda callbacks (dễ tái sử dụng, dễ preview, dễ viết Unit Test).

#### State Hoisting (Kéo trạng thái lên trên)
- Mô hình kiến trúc đưa State từ component con lên component cha (Lowest Common Ancestor) để biến component con thành Stateless.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Snapshot State System: Cơ chế MVCC trong Compose

Tại sao việc thay đổi `mutableState.value` lại khiến giao diện tự động vẽ lại mà không cần bất kỳ callback nào?

Compose Runtime triển khai một hệ thống quản lý bộ nhớ lấy cảm hứng từ các hệ quản trị cơ sở dữ liệu phân tán: **Multi-Version Concurrency Control (MVCC) - Snapshot State System**.

```
                 CƠ CHẾ HOẠT ĐỘNG CỦA SNAPSHOT SYSTEM

1. Thread UI bắt đầu vẽ khung hình (Frame Rendering):
   Snapshot.takeMutableSnapshot() ──► Tạo Snapshot phiên bản V1
          │
          ▼
   Composable đọc state.value ──► Kích hoạt hàm nội bộ readable()
          │
          ▼ Ghi nhận: "Composable Text() đang phụ thuộc vào State A ở phiên bản V1"
          
2. Người dùng tương tác (User Click Event):
   state.value = newValue ──► Kích hoạt hàm nội bộ writable()
          │
          ▼
   Tạo bản ghi mới trong Snapshot V2 ──► Snapshot.sendApplyNotifications()
          │
          ▼
   ĐỐI CHIẾU: Tìm tất cả Composable đã đọc biến này ở V1
          │
          ▼
   KÍCH HOẠT RECOMPOSITION ĐÚNG CÁC COMPOSABLE ĐÓ!
```

- Mọi thao tác đọc `state.value` được đánh chặn bởi hàm `readable()` để ghi vào danh sách quan sát (**Read Tracking**).
- Mọi thao tác ghi `state.value = ...` được ghi nhận qua hàm `writable()`. Khi snapshot được áp dụng (`apply()`), hệ thống gửi thông báo đánh dấu các phạm vi tương ứng bị "bẩn" (**Invalidated**) để lên lịch Recompose ở frame tiếp theo!

---

### 2.2 So sánh vòng đời: `remember` vs `rememberSaveable` vs `ViewModel`

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               PHẠM VI LƯU TRỮ TRẠNG THÁI                               │
├───────────────────────┬────────────────────────┬───────────────────────────────────────┤
│ Công cụ               │ Nơi lưu trữ bên dưới   │ Sống sót qua các sự kiện nào?         │
├───────────────────────┼────────────────────────┼───────────────────────────────────────┤
│ **`remember`**        │ Slot Table trong RAM   │ ✅ Sống qua Recomposition             │
│                       │                        │ ❌ Mất khi Xoay màn hình (Rotate)    │
│                       │                        │ ❌ Mất khi Process Death             │
├───────────────────────┼────────────────────────┼───────────────────────────────────────┤
│ **`rememberSaveable`**│ `SavedStateRegistry`   │ ✅ Sống qua Recomposition             │
│                       │ (Android `Bundle`)     │ ✅ Sống qua Xoay màn hình (Rotate)    │
│                       │                        │ ✅ Sống qua Process Death             │
├───────────────────────┼────────────────────────┼───────────────────────────────────────┤
│ **`ViewModel`**       │ `ViewModelStore`       │ ✅ Sống qua Recomposition             │
│ (StateFlow)           │ (Trong NonConfig RAM)  │ ✅ Sống qua Xoay màn hình (Rotate)    │
│                       │                        │ ❌ Mất khi Process Death (trừ khi     │
│                       │                        │    dùng SavedStateHandle)             │
└───────────────────────┴────────────────────────┴───────────────────────────────────────┘
```

---

### 2.3 Cơ chế bên dưới của `derivedStateOf`

Một trong những tối ưu hóa quan trọng nhất để tránh "thừa thãi Recomposition" (Over-recomposition) là `derivedStateOf`.

```
TÌNH HUỐNG: Theo dõi vị trí cuộn danh sách (LazyListState)
Vị trí cuộn thay đổi liên tục: scrollOffset = 1px ──► 2px ──► 3px ──► ... ──► 500px

TRƯỜNG HỢP 1: KHÔNG DÙNG derivedStateOf (SAI LẦM):
val showButton = listState.firstVisibleItemIndex > 0
=> MỖI KHI cuộn 1 pixel, biến showButton được tính lại
=> KÍCH HOẠT RECOMPOSITION 60 LẦN/GIÂY CHO COMPOSABLE CHA! GÂY LAG DROP FRAME!

TRƯỜNG HỢP 2: DÙNG derivedStateOf (CHUẨN MỰC):
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
=> Snapshot System theo dõi: Dù scrollOffset đổi hàng trăm lần,
   nhưng kết quả Boolean (false -> true) CHỈ ĐỔI ĐÚNG 1 LẦN!
=> CHỈ RECOMPOSE ĐÚNG 1 LẦN DUY NHẤT KHI VƯỢT NGƯỠNG!
```

> **Nguyên tắc vàng:** `derivedStateOf` chỉ phát huy tác dụng khi **Tần số thay đổi của State đầu vào LỚN HƠN RẤT NHIỀU so với tần số thay đổi của kết quả đầu ra** (ví dụ: Scroll Pixel $\rightarrow$ Boolean Threshold, Filtering List $\rightarrow$ Item Count).

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi đau quản lý State trong Android XML View System

1. **State phân tán và không đồng bộ:**
   Trong XML, mỗi View tự giữ trạng thái của mình. Một màn hình Form có 5 Checkbox, 3 EditText, 2 RadioButton nghĩa là có **10 nguồn chân lý khác nhau**. Để thu thập dữ liệu submit, ta phải viết hàng chục dòng `findViewById` để kéo giá trị ra.
2. **Mất sạch dữ liệu khi xoay màn hình:**
   Nếu lập trình viên quên override `onSaveInstanceState(bundle)` và khôi phục trong `onRestoreInstanceState()`, toàn bộ dữ liệu người dùng đang nhập dở sẽ bay màu khi xoay ngang máy.
3. **State Hoisting trong Compose giải quyết triệt để:**
   Tách biệt hoàn toàn giữa việc **Lưu trữ State** và việc **Hiển thị giao diện**. Component UI trở nên hoàn toàn ngây thơ (Dumb UI), chỉ việc nhận dữ liệu vào và hiển thị, không cần biết dữ liệu đến từ đâu!

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Quy tắc State Hoisting chuẩn mực

Khi áp dụng State Hoisting, luôn tuân theo mẫu thiết kế phân tách 2 tham số:
1. **`value: T`**: Dữ liệu truyền từ trên xuống để hiển thị.
2. **`onValueChange: (T) -> Unit`**: Sự kiện đẩy ngược lên khi có thay đổi.

```kotlin
// STATELESS COMPOSABLE: Hoàn hảo cho Reusability và Testing
@Composable
fun SearchInputBar(
    query: String,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier.fillMaxWidth(),
        placeholder = { Text("Tìm kiếm...") }
    )
}
```

---

### 4.2 Tùy biến `Saver` trong `rememberSaveable`

Mặc định `rememberSaveable` chỉ lưu được các kiểu dữ liệu nguyên thủy (Primitives: `Int`, `String`, `Boolean`) hoặc các class có implements `Parcelable` / `Serializable`.

Nếu bạn có một Data Class phức tạp mà không muốn dùng Android `@Parcelize`, hãy viết custom `Saver`:

```kotlin
data class UserDraft(val username: String, val age: Int)

// Tự định nghĩa Saver thông qua mapSaver
val UserDraftSaver = mapSaver(
    save = { mapOf("name" to it.username, "age" to it.age) },
    restore = { UserDraft(it["name"] as String, it["age"] as Int) }
)

// Sử dụng trong Composable:
var draft by rememberSaveable(stateSaver = UserDraftSaver) {
    mutableStateOf(UserDraft("Alex", 25))
}
```

---

### 4.3 Cạm bẫy phổ biến (Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Quên từ khóa `remember` khi khai báo state
```kotlin
// SAI LẦM NGHIÊM TRỌNG:
@Composable
fun Counter() {
    // KHÔNG CÓ REMEMBER: Mỗi khi Recompose, hàm này chạy lại từ đầu
    // và biến count bị reset về 0 ngay lập tức!
    var count by mutableStateOf(0) 

    Button(onClick = { count++ }) {
        Text("Count: $count") // Bấm nút bao nhiêu lần số vẫn là 0!
    }
}

// ĐÚNG:
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

#### Cạm bẫy 2: Dùng `derivedStateOf` bọc các biến tính toán không đọc Compose State
`derivedStateOf` **chỉ hoạt động nếu bên trong lambda của nó có đọc ít nhất một đối tượng `State<T>` của Compose**. Nếu bạn chỉ tính toán các biến thông thường (`val sum = a + b`), việc bọc `derivedStateOf` chỉ làm tăng chi phí cấp phát bộ nhớ vô ích!

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng tính năng **Form Đăng Ký Người Dùng Thông Minh (Smart User Registration)**:
- Bảo toàn dữ liệu khi xoay máy và process death bằng `rememberSaveable` và `mapSaver`.
- State Hoisting hoàn toàn cho các trường nhập liệu.
- Tối ưu hóa kiểm tra tính hợp lệ của Form bằng `derivedStateOf` để ngăn ngừa Recomposition giật lag.

### Bước 1: Khai báo Data Class và Custom Saver

```kotlin
data class RegistrationForm(
    val fullName: String = "",
    val email: String = "",
    val termsAccepted: Boolean = false
)

// Custom Saver giúp lưu trữ an toàn vào Android SavedState Bundle
val RegistrationFormSaver = mapSaver(
    save = { form ->
        mapOf(
            "fullName" to form.fullName,
            "email" to form.email,
            "termsAccepted" to form.termsAccepted
        )
    },
    restore = { map ->
        RegistrationForm(
            fullName = map["fullName"] as String,
            email = map["email"] as String,
            termsAccepted = map["termsAccepted"] as Boolean
        )
    }
)
```

---

### Bước 2: Thiết kế Stateless Input Field (State Hoisting)

```kotlin
@Composable
fun FormInputField(
    label: String,
    value: String,
    onValueChange: (String) -> Unit,
    modifier: Modifier = Modifier,
    isError: Boolean = false,
    errorMessage: String? = null
) {
    Column(modifier = modifier) {
        OutlinedTextField(
            value = value,
            onValueChange = onValueChange,
            label = { Text(label) },
            isError = isError,
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
        )
        if (isError && errorMessage != null) {
            Text(
                text = errorMessage,
                color = MaterialTheme.colorScheme.error,
                style = MaterialTheme.typography.bodySmall,
                modifier = Modifier.padding(start = 4.dp, top = 2.dp)
            )
        }
    }
}
```

---

### Bước 3: Triển khai Stateful Container Form với `derivedStateOf`

```kotlin
@Composable
fun SmartRegistrationScreen(
    onRegisterSuccess: (RegistrationForm) -> Unit,
    modifier: Modifier = Modifier
) {
    // Lưu giữ dữ liệu sống sót qua Xoay màn hình và Process Death!
    var formState by rememberSaveable(stateSaver = RegistrationFormSaver) {
        mutableStateOf(RegistrationForm())
    }

    // TỐI ƯU HÓA: Chỉ tính toán lại và recompose nút Submit khi kết quả Boolean đổi!
    val isFormValid by remember {
        derivedStateOf {
            formState.fullName.trim().length >= 3 &&
            android.util.Patterns.EMAIL_ADDRESS.matcher(formState.email).matches() &&
            formState.termsAccepted
        }
    }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Đăng ký tài khoản") }) }
    ) { padding ->
        Column(
            modifier = modifier
                .fillMaxSize()
                .padding(padding)
                .padding(20.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            // Trường nhập họ tên
            FormInputField(
                label = "Họ và tên",
                value = formState.fullName,
                onValueChange = { newName ->
                    formState = formState.copy(fullName = newName)
                }
            )

            // Trường nhập email
            FormInputField(
                label = "Địa chỉ Email",
                value = formState.email,
                onValueChange = { newEmail ->
                    formState = formState.copy(email = newEmail)
                }
            )

            // Checkbox điều khoản
            Row(
                verticalAlignment = Alignment.CenterVertically,
                modifier = Modifier.fillMaxWidth()
            ) {
                Checkbox(
                    checked = formState.termsAccepted,
                    onCheckedChange = { isChecked ->
                        formState = formState.copy(termsAccepted = isChecked)
                    }
                )
                Spacer(modifier = Modifier.width(8.dp))
                Text(text = "Tôi đồng ý với các điều khoản dịch vụ")
            }

            Spacer(modifier = Modifier.weight(1f))

            // Nút Submit: Chỉ Recompose khi cờ isFormValid đổi trạng thái
            Button(
                onClick = { onRegisterSuccess(formState) },
                enabled = isFormValid,
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Hoàn tất đăng ký")
            }
        }
    }
}
```

---

### Bước 4: Viết Compose UI Test kiểm tra trạng thái State

```kotlin
@RunWith(AndroidJUnit4::class)
class SmartRegistrationScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun submitButton_initiallyDisabled_becomesEnabled_whenFormValid() {
        composeTestRule.setContent {
            SmartRegistrationScreen(onRegisterSuccess = {})
        }

        // Ban đầu nút Submit phải bị Disabled
        composeTestRule.onNodeWithText("Hoàn tất đăng ký").assertIsNotEnabled()

        // Nhập họ tên
        composeTestRule.onNodeWithText("Họ và tên").performTextInput("Nguyen Van A")

        // Nhập email hợp lệ
        composeTestRule.onNodeWithText("Địa chỉ Email").performTextInput("test@example.com")

        // Check vào điều khoản
        composeTestRule.onNode(hasClickAction() and hasAnyAncestor(hasText("Tôi đồng ý với các điều khoản dịch vụ"))).performClick()

        // Nút Submit phải trở thành Enabled!
        composeTestRule.onNodeWithText("Hoàn tất đăng ký").assertIsEnabled()
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Khi nào nên đặt State trong `remember { mutableStateOf() }` và khi nào nên đặt trong `ViewModel` (`StateFlow`)?
**Trả lời chuẩn bản chất:**
- **Đặt trong `remember` / `rememberSaveable`:** Dành cho **UI Element State** (trạng thái cục bộ gắn chặt với giao diện mà các màn hình khác hoặc tầng business logic không cần biết đến). Ví dụ: Composable đang mở rộng (isExpanded) hay thu gọn, vị trí con trỏ trong TextField, tab đang được active.
- **Đặt trong `ViewModel` (`StateFlow`):** Dành cho **Screen UI State** (trạng thái chứa dữ liệu nghiệp vụ cần sống sót qua vòng đời của màn hình, cần gọi UseCase/Repository để truy xuất). Ví dụ: Danh sách giỏ hàng, thông tin tài khoản ngân hàng, trạng thái loading khi tải dữ liệu từ internet.

#### Q2: Điều gì xảy ra nếu dùng `rememberSaveable` lưu một đối tượng quá lớn (> 1MB)?
**Trả lời chuẩn bản chất:**
`rememberSaveable` sử dụng Android `Bundle` và `SavedStateRegistry` bên dưới. Hệ điều hành Android áp dụng giới hạn kích thước giao dịch IPC (Binder Transaction Buffer) là **1MB cho toàn bộ process**. Nếu bạn cố tình lưu một mảng 10,000 items hoặc ảnh Bitmap vào `rememberSaveable`, ứng dụng sẽ sập ngay lập tức với lỗi crash khét tiếng: **`TransactionTooLargeException`**!
> **Quy tắc:** Chỉ lưu ID, text ngắn gọn, hoặc cờ boolean trong `rememberSaveable`. Dữ liệu lớn phải lưu vào Room Database hoặc Disk Cache.

#### Q3: Tại sao gán `state = state` (cùng giá trị) lại không kích hoạt Recomposition?
**Trả lời chuẩn bản chất:**
Bởi vì hàm `mutableStateOf(value, policy)` mặc định sử dụng cơ chế so sánh cấu trúc: `structuralEqualityPolicy()`. Khi bạn gán giá trị mới, Compose sẽ gọi `equals()` giữa giá trị cũ và mới. Nếu `oldValue.equals(newValue)` trả về `true`, Compose Snapshot System sẽ **bỏ qua và không gửi thông báo invalidation**, giúp tránh lãng phí việc vẽ lại màn hình!

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Mất dữ liệu sau khi xoay màn hình điện thoại** | Dùng `remember { mutableStateOf() }` thay vì `rememberSaveable`. | Thay bằng `rememberSaveable`. Nếu là custom object, bổ sung custom `Saver` hoặc `@Parcelize`. |
| **Crash: `IllegalArgumentException: ... cannot be saved using current saveable state registry`** | Cố lưu trữ một Custom Data Class không phải Parcelable vào `rememberSaveable` mà không cung cấp `Saver`. | Viết `mapSaver` hoặc thêm plugin `kotlin-parcelize` và gắn annotation `@Parcelize`. |
| **Giao diện Recompose liên tục không ngừng (100% CPU)** | Đọc và ghi đè `MutableState` trực tiếp bên trong thân hàm `@Composable` mà không bọc trong Event Handler (`onClick`). | Di chuyển việc thay đổi state vào lambda callback hoặc `LaunchedEffect`. |

---

*Bài trước: [10 — Composable Functions: Lifecycle, Slot API & @Composable Contract](10-composable-functions.md)*  
*Bài tiếp theo: [12 — Recomposition & Stability: @Stable, @Immutable](12-recomposition-stability.md)*

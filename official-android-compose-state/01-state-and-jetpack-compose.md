# Bài 01 — State & Jetpack Compose: Bản chất State, Recomposition & State Hoisting

> **Tài liệu tham chiếu chính thức:** [State and Jetpack Compose — Android Developers](https://developer.android.com/develop/ui/compose/state)  
> **Phiên bản áp dụng:** Kotlin 2.0+, Jetpack Compose BOM 2024.06+, Lifecycle 2.8+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong mô hình lập trình giao diện khai báo (Declarative UI) của Jetpack Compose:

- **State (Trạng thái):** Là bất kỳ giá trị nào có khả năng biến đổi theo thời gian trong một ứng dụng (ví dụ: chuỗi ký tự trong ô tìm kiếm `TextField`, trạng thái hộp kiểm `Checkbox`, danh sách bài viết tải về từ API, hoặc trạng thái hiển thị loading spinner).
- **Event (Sự kiện):** Là các tín hiệu đầu vào phát sinh từ bên ngoài hoặc bên trong ứng dụng nhằm thông báo rằng có hành động vừa xảy ra (ví dụ: người dùng nhấn phím, chạm vào nút bấm, cảm biến phát tín hiệu, hoặc phản hồi từ mạng/database).
- **Recomposition (Tái tạo bố cục / Tái thực thi Composition):** Quá trình Compose gọi lại các hàm `@Composable` có khả năng đã thay đổi dữ liệu đầu vào để cập nhật lại cây giao diện (UI tree).

Giao diện người dùng trong Compose là một hàm số toán học thuần túy của State:

$$\text{UI} = f(\text{State})$$

Mô hình điều khiển giữa State và Event tuân theo nguyên lý **Unidirectional Data Flow (UDF - Luồng dữ liệu một chiều)**:

```
                  ┌──────────────────────────────┐
                  │          UI STATE            │
                  │   (Dữ liệu mô tả màn hình)   │
                  └──────────────┬───────────────┘
                                 │
                     State flows DOWN (Hiển thị)
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │       COMPOSABLE UI          │
                  │ (Vẽ giao diện & Nhận tương tác)
                  └──────────────┬───────────────┘
                                 │
                     Events flow UP (Phát sự kiện)
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │        EVENT HANDLER         │
                  │  (Cập nhật State mới: State') │
                  └──────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Tại sao biến thông thường (`var count = 0`) không hoạt động?

Hãy xem xét đoạn mã lỗi kinh điển sau:

```kotlin
@Composable
fun BrokenCounter() {
    var count = 0 // Biến cục bộ thông thường
    Button(onClick = { count++ }) {
        Text("Số lần nhấn: $count")
    }
}
```

Đoạn mã trên gặp hai lỗi chí mạng:
1. **Compose không thể quan sát (Not Observable):** `count` chỉ là một biến nguyên thủy trên stack. Khi giá trị thay đổi, Compose Runtime không có bất kỳ cơ chế nào để biết được biến này vừa bị ghi đè, do đó **không kích hoạt Recomposition**.
2. **Bị reset mỗi khi Recompose:** Kể cả khi hàm này bị recompose bởi một yếu tố bên ngoài khác, biến `var count = 0` sẽ lại được khởi tạo lại về `0`.

### 2.2 `mutableStateOf` và Snapshot Read Tracking

Để Compose nhận biết được sự thay đổi dữ liệu, ta cần bọc giá trị trong kiểu `MutableState<T>` thông qua hàm tạo `mutableStateOf(value)`:

```kotlin
interface State<out T> {
    val value: T
}

interface MutableState<T> : State<T> {
    override var value: T
}
```

Bên dưới `MutableState`, Compose sử dụng hệ thống **Snapshot State System (lấy cảm hứng từ MVCC trong Database)**:
- **Read Tracking:** Khi một hàm `@Composable` đọc thuộc tính `.value` của một `State<T>`, Compose Compiler và Runtime tự động đánh dấu hàm Composable hiện tại là một "Subscriber" (người đăng ký theo dõi) của ô nhớ State đó trong phạm vi Recomposition Scope.
- **Write Tracking:** Khi có thao tác ghi giá trị mới vào `state.value = newValue`, Snapshot System phát tín hiệu invalidate (làm bẩn) đúng các Recomposition Scopes đã từng đọc giá trị đó ở frame trước, lên lịch để chạy lại chúng ở frame tiếp theo!

### 2.3 `remember` và Slot Table

`mutableStateOf` giúp Compose theo dõi biến đổi, nhưng nếu chỉ viết:
```kotlin
val count = mutableStateOf(0) // VẪN SAI!
```
Mỗi khi Recomposition diễn ra, dòng code trên vẫn chạy lại từ đầu và tạo ra một đối tượng `MutableState` mới với giá trị khởi tạo `0`.

Chính vì vậy, ta cần hàm `remember { }`:

```kotlin
val count = remember { mutableStateOf(0) }
```

**Cơ chế lưu trữ bên dưới của `remember`:**
- Compose Runtime lưu trữ cấu trúc giao diện và các đối tượng cần ghi nhớ trong một cấu trúc dữ liệu gọi là **Slot Table** (mảng phẳng liên tục được quản lý theo con trỏ Gap Buffer).
- Khi Composable chạy lần đầu (**Initial Composition**), biểu thức trong lambda `remember { ... }` được tính toán, và kết quả được ghi vào một ô (Slot) trống trong Slot Table.
- Trong các lần **Recomposition** tiếp theo, Compose nhận diện vị trí tương đối trong cây gọi hàm và trả về trực tiếp giá trị đã lưu trong Slot Table mà **không tính toán lại** lambda!
- Khi Composable bị loại bỏ khỏi UI Tree (**Leave the Composition**), slot tương ứng được giải phóng bộ nhớ.

### 2.4 `remember` với Keys: `remember(key1, key2) { ... }`

Hàm `remember` có thể nhận một hoặc nhiều tham số `key`. Khi Composable Recompose:
- Nếu tất cả các `key` **giữ nguyên giá trị** (so sánh bằng `equals`): `remember` trả về kết quả đã lưu trong Slot Table.
- Nếu **bất kỳ key nào thay đổi**: `remember` sẽ vô hiệu hóa ô nhớ cũ, thực thi lại lambda và ghi nhớ giá trị mới!

```kotlin
// Tính toán lại phép băm/chuyển đổi tốn kém CHỈ KHI rawData thay đổi:
val formattedData = remember(rawData) {
    heavyDataProcessing(rawData)
}
```

### 2.5 `rememberSaveable` — Bảo toàn trạng thái qua Thay đổi Cấu hình & Process Death

Mặc dù `remember` giúp lưu giữ State qua các lần Recomposition, nó **không thể sống sót qua các sự kiện vòng đời cấp cao của Android**:
- **Xoay màn hình (Configuration Changes):** Màn hình xoay ngang/dọc, đổi ngôn ngữ, bật Dark Mode khiến `Activity` bị hủy và tạo lại từ đầu. Toàn bộ cây Composition và Slot Table trong RAM bị giải phóng hoàn toàn $\rightarrow$ Dữ liệu trong `remember` bị reset về giá trị ban đầu!
- **Tiến trình bị hệ thống hủy ngầm (Process Death):** Khi ứng dụng ở background, Android OS có thể đơn phương thu hồi RAM của ứng dụng khi thiết bị thiếu bộ nhớ.

Để giải quyết triệt để vấn đề này, Google cung cấp hàm **`rememberSaveable`**:

```kotlin
var text by rememberSaveable { mutableStateOf("") }
```

```
┌────────────────────────────────────────────────────────────────────────┐
│               SO SÁNH BẢN CHẤT: remember VS rememberSaveable            │
├─────────────────────┬──────────────────────┬───────────────────────────┤
│ Đặc tính            │ `remember`           │ `rememberSaveable`        │
├─────────────────────┼──────────────────────┼───────────────────────────┤
│ **Vị trí lưu trữ**  │ Slot Table trong RAM │ `SavedStateRegistry`      │
│                     │ của Composition      │ (cơ chế Android `Bundle`) │
│ **Recomposition**   │ ✅ Sống sót          │ ✅ Sống sót               │
│ **Xoay màn hình**   │ ❌ Mất sạch          │ ✅ Sống sót               │
│ **Process Death**   │ ❌ Mất sạch          │ ✅ Sống sót               │
│ **Kiểu dữ liệu**    │ Bất kỳ Object nào    │ Chỉ các kiểu Bundle-safe   │
│                     │                      │ (hoặc có `@Parcelize`, Saver)│
└─────────────────────┴──────────────────────┴───────────────────────────┘
```

- **Cơ chế hoạt động:** `rememberSaveable` đăng ký một `SavedStateProvider` với `SavedStateRegistryOwner` của Android. Khi Activity chuẩn bị bị hủy, dữ liệu State được tuần tự hóa vào `Bundle`. Khi Activity được tái tạo, Compose đọc lại `Bundle` để khôi phục State.
- **Hỗ trợ mặc định:** Các kiểu dữ liệu nguyên thủy (`Int`, `String`, `Boolean`, `Float`...) và mảng nguyên thủy.
- **Đối tượng tùy biến (Custom Objects):** Với các class tự định nghĩa, cần gắn annotation `@Parcelize` hoặc tự triển khai `listSaver` / `mapSaver` (chi tiết xem tại [Bài 03 — Lưu Trữ & Khôi Phục UI State](03-saving-ui-state.md)).

### 2.6 Tổng hợp các hàm `remember` cốt lõi và thông dụng nhất trong Compose

Trong quá trình phát triển ứng dụng Compose thực tế, Google cung cấp một hệ sinh thái các hàm `remember*` chuyên biệt nhằm phục vụ từng mục đích cụ thể:

| Hàm `remember` | Mục đích sử dụng chính | Vòng đời & Đặc điểm |
| :--- | :--- | :--- |
| **`remember { mutableStateOf(x) }`** | Lưu trữ State hiển thị thông thường của Composable. | Sống trong Composition. Mất khi xoay màn hình. |
| **`rememberSaveable { mutableStateOf(x) }`** | Lưu trữ State cần bảo toàn khi xoay màn hình hoặc app bị kill ngầm (form input, filter selection). | Sống sót qua Configuration Change & Process Death nhờ Android `Bundle`. |
| **`remember(key1, key2) { ... }`** | Ghi nhớ kết quả tính toán nặng hoặc khởi tạo đối tượng tốn kém, chỉ tính lại khi các key đổi. | Cache tính toán dựa trên `equals()` của các key. |
| **`rememberCoroutineScope()`** | Tạo một `CoroutineScope` gắn liền với điểm Composition hiện tại, dùng để kích hoạt các tác vụ bất đồng bộ từ các callback sự kiện (như `onClick`). | Tự động cancel tất cả coroutine khi Composable rời khỏi cây giao diện. |
| **`rememberUpdatedState(value)`** | Bọc một giá trị hoặc lambda callback để các Coroutine/Effect chạy lâu luôn đọc được giá trị mới nhất mà **không làm restart Effect**. | Tránh lỗi Stale State/Stale Callback trong `LaunchedEffect`. |
| **`rememberScrollState()`** | State Holder quản lý trạng thái cuộn của `Modifier.verticalScroll()` hoặc `horizontalScroll()`. | Theo dõi vị trí cuộn hiện tại và cung cấp các hàm cuộn mượt (`animateScrollTo`). |
| **`rememberLazyListState()`** | State Holder chuyên dụng cho danh sách lớn `LazyColumn` / `LazyRow`. | Theo dõi `firstVisibleItemIndex`, `firstVisibleItemScrollOffset` để tối ưu hóa render. |
| **`rememberDrawerState()`** | Quản lý trạng thái Mở/Đóng (`Open` / `Closed`) của ngăn kéo `ModalNavigationDrawer`. | Cung cấp hàm `open()` và `close()` dạng `suspend`. |
| **`rememberSnackbarHostState()`** | Quản lý hàng đợi và hiển thị các thanh thông báo `Snackbar`. | Điều phối hiển thị thông báo với `showSnackbar(...)`. |

#### Ví dụ Thực tế: Kết hợp các hàm `remember` thông dụng

```kotlin
@Composable
fun UserDashboardScreen(
    onNavigateBack: () -> Unit,
    modifier: Modifier = Modifier
) {
    // 1. rememberSaveable: Bảo toàn từ khóa tìm kiếm khi xoay màn hình
    var searchKeyword by rememberSaveable { mutableStateOf("") }

    // 2. rememberCoroutineScope: Kích hoạt coroutine khi bấm nút
    val coroutineScope = rememberCoroutineScope()

    // 3. rememberSnackbarHostState: Điều phối thông báo
    val snackbarHostState = remember { SnackbarHostState() }

    // 4. rememberLazyListState: Theo dõi cuộn danh sách
    val listState = rememberLazyListState()

    // 5. remember(key): Tính toán lọc danh sách CHỈ KHI từ khóa thay đổi
    val filteredCount = remember(searchKeyword) {
        // Giả lập tính toán phức tạp
        searchKeyword.trim().length * 10
    }

    Scaffold(
        snackbarHost = { SnackbarHost(hostState = snackbarHostState) }
    ) { paddingValues ->
        Column(
            modifier = modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            OutlinedTextField(
                value = searchKeyword,
                onValueChange = { searchKeyword = it },
                label = { Text("Tìm kiếm (lưu qua xoay màn hình)") },
                modifier = Modifier.fillMaxWidth().padding(16.dp)
            )

            Button(
                onClick = {
                    // Sử dụng coroutineScope đã remember để hiển thị snackbar
                    coroutineScope.launch {
                        snackbarHostState.showSnackbar("Tìm thấy $filteredCount kết quả phù hợp!")
                    }
                },
                modifier = Modifier.padding(horizontal = 16.dp)
            ) {
                Text("Kiểm tra kết quả")
            }
        }
    }
}
```

### 2.7 Tích hợp các kiểu dữ liệu Observable khác vào Compose State

Compose không bắt buộc bạn phải lưu trữ State gốc dưới dạng `MutableState<T>`. Bạn có thể quản lý State trong kiến trúc bằng **Kotlin Flow**, **LiveData**, hoặc **RxJava**, sau đó chuyển đổi sang `State<T>` trong Composable để Compose có thể đọc được:

| Kiểu Observable nguồn | Thư viện chuyển đổi | Cú pháp chuyển đổi trong Compose | Đặc điểm vòng đời |
| :--- | :--- | :--- | :--- |
| **`StateFlow<T>` / `Flow<T>`** | `lifecycle-runtime-compose` | `flow.collectAsStateWithLifecycle()` | ✅ **Tiêu chuẩn vàng:** Tự dừng collect khi app xuống background (Lifecycle.State.STARTED). |
| **`Flow<T>`** (Cũ) | `androidx.compose.runtime` | `flow.collectAsState()` | ⚠️ Tiếp tục collect kể cả khi app ở background, dễ lãng phí CPU/pin. |
| **`LiveData<T>`** | `runtime-livedata` | `liveData.observeAsState(initial)` | Tự động lắng nghe theo `LifecycleOwner` của Composable. |
| **`Observable<T>` (RxJava2/3)** | `runtime-rxjava2` hoặc `rxjava3` | `observable.subscribeAsState(initial)` | Tự động hủy đăng ký (unsubscribe) khi Composable rời khỏi Composition. |

```kotlin
@Composable
fun ObservablesIntegrationExample(viewModel: MyViewModel) {
    // 1. Chuyển đổi StateFlow sang Compose State chuẩn vòng đời
    val userProfile by viewModel.userProfileFlow.collectAsStateWithLifecycle()

    // 2. Chuyển đổi LiveData sang Compose State
    val networkStatus by viewModel.networkLiveData.observeAsState(initial = "Connected")

    Text("User: ${userProfile.name}, Network: $networkStatus")
}
```

---

## 3. Bài toán & Kiến trúc (Problem Statement & Architecture)

### 3.1 So sánh Imperative UI (View cổ điển) vs Declarative UI (Compose)

| Đặc điểm | Android View System (XML / Cổ điển) | Jetpack Compose (Hiện đại) |
| :--- | :--- | :--- |
| **Bản chất UI** | Mỗi Widget (`TextView`, `EditText`) tự lưu trữ State của chính nó bên trong đối tượng View. | Composable là hàm không trạng thái (Stateless by default), UI chỉ là hình chiếu của State. |
| **Cập nhật UI** | Lệnh mệnh lệnh (Imperative): Gọi thủ công `textView.setText(data)`, `view.setVisibility(GONE)`. | Khai báo phản ứng (Declarative): Thay đổi State $\rightarrow$ Compose tự tính toán Recomposition. |
| **Nguy cơ lỗi** | **Bất đồng bộ trạng thái (State desynchronization):** Dữ liệu trong DB/ViewModel một đằng, dữ liệu hiển thị trên View một nẻo. | **Single Source of Truth (SSOT):** State chỉ nằm ở một nguồn duy nhất, UI luôn phản ánh chính xác nguồn đó. |

### 3.2 3 Cú pháp khai báo State trong Composable

Kotlin cung cấp 3 cú pháp để sử dụng `remember` kết hợp `mutableStateOf`:

```kotlin
// Cách 1: Giữ nguyên đối tượng MutableState<T>
val countState: MutableState<Int> = remember { mutableStateOf(0) }
// Truy cập: countState.value
// Thay đổi: countState.value++

// Cách 2: Sử dụng cú pháp ủy quyền thuộc tính (Property Delegation by) - PHỔ BIẾN NHẤT
var count: Int by remember { mutableStateOf(0) }
// Truy cập: count
// Thay đổi: count++ (yêu cầu import androidx.compose.runtime.getValue / setValue)

// Cách 3: Phân rã cấu trúc (Destructuring)
val (count, setCount) = remember { mutableStateOf(0) }
// Truy cập: count
// Thay đổi: setCount(count + 1)
```

> [!TIP]
> Cách 2 (`by remember { mutableStateOf(...) }`) là tiêu chuẩn quy ước phổ biến nhất trong phát triển ứng dụng Compose vì cú pháp đọc/ghi tự nhiên như biến nguyên thủy.

---

## 4. Thực hành tốt nhất & Cảnh báo (Best Practices & Anti-patterns)

### 4.1 Stateful vs Stateless Composables

- **Stateful Composable:** Là hàm Composable tự khởi tạo và nắm giữ State bên trong (dùng `remember { mutableStateOf(...) }`).
  - *Ưu điểm:* Tiện lợi cho component cấp cao gọi sử dụng mà không cần truyền nhiều tham số.
  - *Nhược điểm:* Khó tái sử dụng, khó viết Preview, khó kiểm thử độc lập vì State bị gắn cứng bên trong hàm.
- **Stateless Composable:** Là hàm Composable **không chứa State nội tại**. Nó chỉ nhận giá trị hiển thị qua tham số và phát sự kiện tương tác qua các lambda callback.
  - *Ưu điểm:* Dễ tái sử dụng tối đa, dễ Preview nhiều trạng thái khác nhau, dễ viết Unit Test và UI Test.

### 4.2 Nguyên lý State Hoisting (Kéo trạng thái lên trên)

**State Hoisting** là mô hình đưa State từ một Composable con lên hàm Composable cha gọi nó để biến component con thành Stateless.

Mẫu hình chuẩn của State Hoisting gồm hai tham số:
1. `value: T` — Giá trị trạng thái hiện tại cần hiển thị.
2. `onValueChange: (T) -> Unit` — Lambda sự kiện phát ra khi người dùng yêu cầu thay đổi giá trị.

```
                  ┌─────────────────────────────────────┐
                  │          PARENT COMPOSABLE          │
                  │   var text by remember { ... }      │
                  └───────────────┬─────────────────────┘
                                  │
                   value = text   │   onValueChange = { text = it }
                   (Truyền xuống) │   (Phát ngược lên)
                                  ▼
                  ┌─────────────────────────────────────┐
                  │         STATELESS CHILD UI          │
                  │  MyTextField(value, onValueChange)  │
                  └─────────────────────────────────────┘
```

**4 Lợi ích cốt lõi của State Hoisting theo Google:**
1. **Single Source of Truth (Nguồn chân lý duy nhất):** Bằng cách nâng state lên, ta loại bỏ việc sao chép state phân tán, tránh sai lệch dữ liệu.
2. **Encapsulated (Đóng gói):** Chỉ Composable cha (hoặc State Holder) mới có quyền sửa đổi State.
3. **Interceptable (Có thể đánh chặn):** Composable cha có thể kiểm duyệt, từ chối hoặc chuyển đổi sự kiện (ví dụ: giới hạn số ký tự nhập vào ô text).
4. **Decoupled (Tách rời):** Composable con trở nên độc lập hoàn toàn với nơi lưu trữ state (có thể đặt trong `remember`, trong `ViewModel`, hay mock trong Unit Test).

### 4.3 3 Quy tắc vàng xác định vị trí kéo State lên (Rules of Thumb)

Theo tài liệu chính thức của Google, khi phân vân không biết nên kéo State lên tầng Composable nào, hãy áp dụng **3 quy tắc ngón tay cái**:

1. **Quy tắc 1 (Vị trí Đọc):** State phải được kéo lên ít nhất là **tổ tiên chung thấp nhất (Lowest Common Ancestor)** của tất cả các Composable có nhu cầu đọc (read) State đó.
2. **Quy tắc 2 (Vị trí Ghi):** State phải được kéo lên ít nhất là **cấp cao nhất (Highest level)** nơi nó có khả năng bị sửa đổi hoặc điều khiển (write/modify).
3. **Quy tắc 3 (Gộp State):** Nếu hai hoặc nhiều biến State luôn luôn **thay đổi cùng nhau** để phản hồi lại cùng một chuỗi sự kiện, chúng phải được kéo lên và đặt cùng một chỗ (hoặc nhóm lại vào một data class duy nhất).

### 4.4 Danh sách Anti-patterns cần tuyệt đối tránh

> [!CAUTION]
> 1. **KHÔNG BAO GIỜ truyền `MutableState<T>` xuống Composable con:**
>    - *Sai:* `fun UserInput(nameState: MutableState<String>)` $\rightarrow$ Phá vỡ tính đóng gói, Composable con có thể tùy ý sửa state ở bất kỳ đâu gây khó debug.
>    - *Đúng:* `fun UserInput(name: String, onNameChange: (String) -> Unit)`
>
> 2. **Tránh biến đổi State bên ngoài các Event Callbacks hoặc Side-Effects:**
>    - *Sai:* Gán `count++` trực tiếp trong thân hàm Composable khi đang render. Điều này kích hoạt Recomposition vô tận (Infinite Recomposition Loop) dẫn đến crash ứng dụng!
>
> 3. **Chuyển đổi Observable State chuẩn vòng đời:**
>    - Khi thu thập Kotlin Flow trong Compose, hãy ưu tiên dùng `collectAsStateWithLifecycle()` (từ thư viện `lifecycle-runtime-compose`) thay vì `collectAsState()` để tự động dừng thu thập khi app xuống background, tránh lãng phí CPU và pin.

---

## 5. Mã nguồn Thực tế (Implementation)

Dưới đây là ví dụ hoàn chỉnh minh họa quá trình tái cấu trúc (refactor) từ một component Stateful cục bộ thành Stateless theo chuẩn State Hoisting, kết hợp cùng ViewModel và Flow.

### 5.1 Component Stateless: Form Tìm Kiếm Tái Sử Dụng

```kotlin
package com.example.compose.state.basics

import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Clear
import androidx.compose.material.icons.filled.Search
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

/**
 * STATELESS COMPOSABLE:
 * - Không sở hữu State nội bộ.
 * - Nhận giá trị hiển thị qua `query`.
 * - Báo cáo tương tác người dùng qua các lambda callbacks.
 */
@Composable
fun SearchBarInput(
    query: String,
    onQueryChange: (String) -> Unit,
    onClearClicked: () -> Unit,
    modifier: Modifier = Modifier,
    placeholderText: String = "Nhập từ khóa tìm kiếm...",
    isEnabled: Boolean = true
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier.fillMaxWidth(),
        enabled = isEnabled,
        placeholder = { Text(placeholderText) },
        leadingIcon = {
            Icon(imageVector = Icons.Default.Search, contentDescription = "Search Icon")
        },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = onClearClicked) {
                    Icon(imageVector = Icons.Default.Clear, contentDescription = "Clear text")
                }
            }
        },
        singleLine = true
    )
}
```

### 5.2 Component Stateful: Nắm giữ State và Điều khiển Logic

```kotlin
/**
 * STATEFUL COMPOSABLE:
 * - Nắm giữ State cục bộ thông qua `remember` và `mutableStateOf`.
 * - Tự chịu trách nhiệm cung cấp state và xử lý event cho `SearchBarInput`.
 */
@Composable
fun StatefulSearchSection(
    onSearchSubmitted: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    // State cục bộ được ghi nhớ qua các lần Recomposition
    var searchQuery by remember { mutableStateOf("") }

    Column(
        modifier = modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        SearchBarInput(
            query = searchQuery,
            onQueryChange = { newQuery ->
                // Cha có quyền đánh chặn (Intercept): Giới hạn tối đa 50 ký tự
                if (newQuery.length <= 50) {
                    searchQuery = newQuery
                }
            },
            onClearClicked = { searchQuery = "" }
        )

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = { onSearchSubmitted(searchQuery) },
            enabled = searchQuery.isNotBlank(),
            modifier = Modifier.align(Alignment.End)
        ) {
            Text("Tìm kiếm")
        }
    }
}
```

---

## 6. Các câu hỏi thực tế thường gặp & Xử lý sự cố (FAQ & Troubleshooting)

### Q1: Sự khác biệt cốt lõi giữa `remember` và `rememberSaveable` là gì?
- **`remember`:** Lưu đối tượng vào **Slot Table** trong bộ nhớ RAM của Compose. Nó chỉ tồn tại chừng nào Composable còn nằm trong Composition Tree. Khi người dùng xoay màn hình (Configuration Change) hoặc khi Activity bị hệ thống hủy (Process Death), đối tượng trong `remember` sẽ bị **mất hoàn toàn**.
- **`rememberSaveable`:** Lưu đối tượng vào Android `SavedStateRegistry` (cơ chế `Bundle`). Do đó, nó **sống sót qua được sự kiện xoay màn hình** và cả **tiến trình bị hủy ngầm (Process Death)**.

### Q2: Tại sao gọi `mutableListOf()` bên trong `remember { mutableStateOf(mutableListOf()) }` lại không kích hoạt Recomposition khi thêm phần tử?
- **Nguyên nhân:** `mutableStateOf` sử dụng chính sách so sánh giá trị mặc định là `structuralEqualityPolicy()` (so sánh qua `==` hoặc tham chiếu instance).
- Khi bạn gọi `list.add("Item")`, nội dung bên trong list thay đổi nhưng **địa chỉ tham chiếu (instance pointer)** của đối tượng `list` vẫn không đổi! Compose Snapshot System không phát hiện được sự thay đổi này nên **không kích hoạt Recomposition**.
- **Giải pháp chuẩn:**
  1. Sử dụng `toMutableStateList()` hoặc `remember { mutableStateListOf<String>() }`.
  2. Hoặc tạo bản sao danh sách mới mỗi lần cập nhật: `list = list + "Item"`.

### Q3: Composable bị "Recomposition Loop" (Recompose vô tận) — Nguyên nhân và cách phát hiện?
- **Nguyên nhân:** Có một thao tác ghi đè State (`state.value = ...`) được đặt trực tiếp trong thân hàm Composable mà không được bao bọc trong `LaunchedEffect` hoặc lambda sự kiện (`onClick`).
- Khi Composable chạy, nó ghi vào State $\rightarrow$ State thay đổi kích hoạt Recomposition $\rightarrow$ Composable chạy lại $\rightarrow$ lại ghi vào State $\rightarrow$ lặp vô tận khiến ứng dụng đơ giật hoặc văng lỗi `IllegalStateException`.
- **Cách phát hiện:** Sử dụng công cụ **Layout Inspector** trong Android Studio để kiểm tra chỉ số Recomposition Count tăng đột biến hàng nghìn lần mỗi giây.

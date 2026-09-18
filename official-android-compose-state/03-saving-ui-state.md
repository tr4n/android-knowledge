# Bài 03 — Lưu Trữ & Khôi Phục UI State: rememberSaveable, Custom Savers & SavedStateHandle

> **Tài liệu tham chiếu chính thức:** [Save UI state in Compose — Android Developers](https://developer.android.com/develop/ui/compose/state-saving)  
> **Phiên bản áp dụng:** Kotlin 2.0+, Compose BOM 2024.06+, Lifecycle 2.8+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong Android, trạng thái giao diện có thể bị mất mát tại **3 cấp độ** khác nhau do vòng đời của hệ thống:

1. **Recomposition (Tái tạo bố cục):** Hàm Composable được thực thi lại để phản ánh State mới.
2. **Configuration Change (Thay đổi cấu hình):** Người dùng xoay màn hình (Landscape $\leftrightarrow$ Portrait), chuyển đổi giao diện Sáng/Tối (Dark Mode), đổi ngôn ngữ hệ thống, hoặc thay đổi kích thước cửa sổ chia đôi màn hình (Multi-window). Lúc này, `Activity` bị hủy hoàn toàn và tạo lại từ đầu.
3. **System-initiated Process Death (Tiến trình bị hệ thống tiêu hủy ngầm):** Khi người dùng chuyển app xuống background (nhấn Home hoặc chuyển sang app khác như mở Camera/OTP), nếu hệ điều hành Android rơi vào tình trạng thiếu hụt RAM, Linux Out-Of-Memory (OOM) Killer sẽ **đơn phương chấm dứt tiến trình của app**. Khi người dùng quay lại, toàn bộ bộ nhớ RAM của app trước đó đã biến mất!

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             3 CẤP ĐỘ MẤT STATE TRONG ANDROID                     │
├───────────────────────┬───────────────────────┬──────────────────────────────────┤
│ Cấp độ sự kiện        │ Điều gì xảy ra?       │ Công cụ bảo vệ dữ liệu tối ưu    │
├───────────────────────┼───────────────────────┼──────────────────────────────────┤
│ **1. Recomposition**  │ Hàm chạy lại          │ `remember { }`                   │
├───────────────────────┼───────────────────────┼──────────────────────────────────┤
│ **2. Config Change**  │ Activity bị hủy & tạo │ `rememberSaveable` HOẶC          │
│                       │ lại trong cùng RAM    │ `ViewModel` (NonConfigInstance)  │
├───────────────────────┼───────────────────────┼──────────────────────────────────┤
│ **3. Process Death**  │ RAM bị giải phóng     │ `rememberSaveable` (UI level)    │
│                       │ triệt để bởi hệ thống │ VÀ `SavedStateHandle` (VM level) │
└───────────────────────┴───────────────────────┴──────────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế bên dưới của `rememberSaveable` và `SavedStateRegistry`

`rememberSaveable` không lưu dữ liệu vào Slot Table đơn thuần như `remember`. Thay vào đó, nó liên kết trực tiếp với cơ chế lưu trữ của Android OS:

1. Khi Composable được gắn vào Composition, `rememberSaveable` đăng ký một `SavedStateProvider` với đối tượng `SavedStateRegistry` của `Activity` hoặc `Fragment` hiện tại.
2. Trước khi `Activity` bị hủy do Config Change hoặc Process Death, hệ thống Android gọi hàm vòng đời `onSaveInstanceState(outState: Bundle)`.
3. `SavedStateRegistry` sẽ yêu cầu tất cả các Composable đã đăng ký tuần tự hóa (serialize) dữ liệu State của mình thành các phần tử bên trong `Bundle`.
4. Khi ứng dụng được tạo lại, `rememberSaveable` đọc lại `Bundle` thông qua một khóa định danh tự động (`auto-generated key` dựa trên vị trí của Composable trong mã nguồn hoặc khóa người dùng tự chỉ định) để khôi phục lại giá trị State ban đầu!

### 2.2 Giới hạn Transaction Buffer & Nguy cơ Crash

> [!WARNING]
> Android `Bundle` lưu dữ liệu trạng thái được truyền qua lại giữa các tiến trình (IPC) thông qua **Binder Transaction Buffer**.
> - Bộ đệm này dùng chung cho toàn bộ thiết bị và có giới hạn rất nhỏ: **chỉ khoảng 500KB đến 1MB**.
> - Nếu bạn cố tình lưu một danh sách hàng trăm sản phẩm hoặc ảnh Bitmap vào `rememberSaveable`, ứng dụng sẽ văng lỗi crash nghiêm trọng: `android.os.TransactionTooLargeException`!

---

## 3. Bài toán & Kiến trúc (Problem Statement & Architecture)

### 3.1 Ma trận sống sót của các giải pháp lưu trữ State

| Giải pháp lưu trữ | Sống qua Recompose? | Sống qua Xoay màn hình? | Sống qua Process Death? | Giới hạn dung lượng lưu trữ |
| :--- | :---: | :---: | :---: | :--- |
| `var x = remember { ... }` | ✅ Có | ❌ Mất | ❌ Mất | Bộ nhớ Heap của JVM |
| `rememberSaveable { ... }` | ✅ Có | ✅ Có | ✅ Có | **Rất nhỏ (< 50KB khuyến nghị)** |
| `ViewModel (StateFlow)` | ✅ Có | ✅ Có | ❌ Mất | Bộ nhớ Heap của JVM |
| `ViewModel + SavedStateHandle` | ✅ Có | ✅ Có | ✅ Có | **Rất nhỏ (< 50KB khuyến nghị)** |
| `Room Database / DataStore` | ✅ Có | ✅ Có | ✅ Có | Dung lượng ổ đĩa Disk |

### 3.2 Chiến lược phối hợp giữa UI Layer và ViewModel Layer

Để đảm bảo trải nghiệm người dùng không bao giờ bị đứt gãy:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              CHIẾN LƯỢC TOÀN DIỆN                      │
│                                                                        │
│ 1. Trạng thái tạm thời của UI (Transient UI Element State):            │
│    - Ô input đang gõ dở, checkbox, scroll offset                       │
│    ──► Dùng `rememberSaveable`                                         │
│                                                                        │
│ 2. Dữ liệu đầu vào hoặc ID truy vấn của Màn hình (Screen Query/ID):    │
│    - userId, searchKeyword, selectedTabId                              │
│    ──► Dùng `SavedStateHandle` trong ViewModel                         │
│                                                                        │
│ 3. Toàn bộ dữ liệu nghiệp vụ lớn (Heavy Business Data):                │
│    - Danh sách 500 items, lịch sử đơn hàng                             │
│    ──► Lưu trong Room Database hoặc nạp lại từ Mạng qua Search ID     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Thực hành tốt nhất & Cảnh báo (Best Practices & Anti-patterns)

### 4.1 Bốn cách lưu dữ liệu trong `rememberSaveable`

Mặc định, `rememberSaveable` chỉ hỗ trợ các kiểu dữ liệu có thể đưa trực tiếp vào `Bundle` (nguyên thủy: `Int`, `String`, `Boolean`, `Float`...). Để lưu các đối tượng tùy chỉnh (Custom Objects), Google cung cấp 4 phương thức:

1. **Thêm Annotation `@Parcelize` (Khuyến nghị số 1):** Yêu cầu đối tượng kế thừa `Parcelable`.
2. **Sử dụng `listSaver`:** Đóng gói đối tượng thành một `List<Any>` rồi giải nén ngược lại khi khôi phục.
3. **Sử dụng `mapSaver`:** Đóng gói đối tượng thành một `Map<String, Any>` (hữu ích khi cấu trúc có nhiều thuộc tính tùy chọn).
4. **Tự viết `Saver` tùy biến:** Triển khai trực tiếp interface `Saver<Original, Saveable>` khi đối tượng thuộc thư viện bên ngoài không thể sửa mã nguồn.

### 4.2 Danh sách Anti-patterns cần tuyệt đối tránh

> [!CAUTION]
> 1. **KHÔNG BAO GIỜ lưu toàn bộ Domain Model vào `rememberSaveable`:**
>    - *Sai:* Lưu toàn bộ đối tượng `User(id, name, avatar, orders, settings...)` vào Bundle.
>    - *Đúng:* Chỉ lưu `userId: String`. Khi app khởi động lại, dùng `userId` để query lại từ Room Database hoặc Network.
>
> 2. **Tránh dùng `rememberSaveable` với `mutableStateListOf` mà không có Saver phù hợp:**
>    - Khi viết `rememberSaveable { mutableStateListOf() }`, nó sẽ chỉ lưu được nếu các phần tử bên trong danh sách có thể đưa vào Bundle.

### 4.3 Khóa tùy biến `key` trong `rememberSaveable(key = ...)`

Theo mặc định, Compose tự động sinh ra một `key` định danh duy nhất cho mỗi hàm `rememberSaveable` dựa trên **vị trí mã nguồn (source code line/column)** của nó.
- **Khi nào bị lỗi?** Nếu bạn gọi `rememberSaveable` bên trong một vòng lặp (`for`/`forEach`) hoặc trong các nhánh điều kiện (`if-else`), vị trí mã nguồn của các item có thể bị trùng lặp hoặc thay đổi thứ tự khi xóa bớt phần tử $\rightarrow$ **Dẫn đến việc khôi phục nhầm dữ liệu của item này sang item khác!**
- **Giải pháp:** Luôn chỉ định tham số `key` tường minh gắn liền với ID duy nhất của đối tượng:
  ```kotlin
  // Chỉ định key duy nhất theo ID của user:
  var input by rememberSaveable(key = "user_input_${user.id}") { 
      mutableStateOf("") 
  }
  ```

---

## 5. Mã nguồn Thực tế (Implementation)

Dưới đây là các ví dụ minh họa đầy đủ các kỹ thuật lưu trữ State chuẩn mực.

### 5.1 Cách 1: Sử dụng `@Parcelize` cho Custom Data Class

```kotlin
package com.example.compose.state.saving

import android.os.Parcelable
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlinx.parcelize.Parcelize

// 1. Khai báo data class hỗ trợ Parcelable
@Parcelize
data class RegistrationDraft(
    val fullName: String = "",
    val email: String = "",
    val step: Int = 1
) : Parcelable

@Composable
fun RegistrationScreen() {
    // 2. rememberSaveable tự động nhận diện Parcelable và lưu vào Bundle!
    var draft by rememberSaveable { mutableStateOf(RegistrationDraft()) }

    Column(modifier = Modifier.padding(16.dp)) {
        Text(text = "Bước đăng ký: ${draft.step}", style = MaterialTheme.typography.titleMedium)

        OutlinedTextField(
            value = draft.fullName,
            onValueChange = { draft = draft.copy(fullName = it) },
            label = { Text("Họ và tên") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        OutlinedTextField(
            value = draft.email,
            onValueChange = { draft = draft.copy(email = it) },
            label = { Text("Địa chỉ Email") },
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

### 5.2 Cách 2: Sử dụng `listSaver` và `mapSaver` cho các class không thể sửa đổi code

#### Ví dụ A: Triển khai với `listSaver` (Gọn nhẹ, truy cập theo chỉ số Index)
```kotlin
import androidx.compose.runtime.saveable.listSaver
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.*

data class CityLocation(val name: String, val latitude: Double, val longitude: Double)

val CityLocationSaver = listSaver<CityLocation, Any>(
    save = { listOf(it.name, it.latitude, it.longitude) },
    restore = { list ->
        CityLocation(
            name = list[0] as String,
            latitude = list[1] as Double,
            longitude = list[2] as Double
        )
    }
)

@Composable
fun LocationPicker() {
    var selectedCity by rememberSaveable(stateSaver = CityLocationSaver) {
        mutableStateOf(CityLocation("Hà Nội", 21.0285, 105.8542))
    }
}
```

#### Ví dụ B: Triển khai với `mapSaver` (Rõ ràng ngữ nghĩa với Key-Value)
```kotlin
import androidx.compose.runtime.saveable.mapSaver
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.*

data class ThirdPartyFilter(val minPrice: Double, val maxPrice: Double, val category: String)

val FilterSaver = run {
    val minPriceKey = "MinPrice"
    val maxPriceKey = "MaxPrice"
    val categoryKey = "Category"

    mapSaver(
        save = { filter: ThirdPartyFilter ->
            mapOf(
                minPriceKey to filter.minPrice,
                maxPriceKey to filter.maxPrice,
                categoryKey to filter.category
            )
        },
        restore = { savedMap ->
            ThirdPartyFilter(
                minPrice = savedMap[minPriceKey] as Double,
                maxPrice = savedMap[maxPriceKey] as Double,
                category = savedMap[categoryKey] as String
            )
        }
    )
}

@Composable
fun FilterComponent() {
    // Truyền stateSaver vào rememberSaveable
    var filter by rememberSaveable(stateSaver = FilterSaver) {
        mutableStateOf(ThirdPartyFilter(minPrice = 0.0, maxPrice = 1000.0, category = "All"))
    }
}
```

### 5.3 Cách 3: Phối hợp `SavedStateHandle` trong ViewModel

Đối với Screen UI State, ViewModel kết hợp cùng `SavedStateHandle` để sống sót qua Process Death:

```kotlin
package com.example.compose.state.saving

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*

class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    companion object {
        private const val KEY_SEARCH_QUERY = "search_query"
    }

    // 1. Đọc và ghi State trực tiếp qua SavedStateHandle dưới dạng StateFlow
    val searchQuery: StateFlow<String> = savedStateHandle.getStateFlow(
        key = KEY_SEARCH_QUERY,
        initialValue = ""
    )

    // 2. Chuyển đổi query thành kết quả tìm kiếm tự động
    val searchResults: StateFlow<List<String>> = searchQuery
        .debounce(300)
        .mapLatest { query ->
            if (query.isBlank()) emptyList()
            else listOf("Kết quả 1 cho $query", "Kết quả 2 cho $query")
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun onQueryChanged(newQuery: String) {
        // Cập nhật giá trị vào SavedStateHandle -> Tự động lưu vào Bundle nếu app bị kill
        savedStateHandle[KEY_SEARCH_QUERY] = newQuery
    }
}
```

---

## 6. Các câu hỏi thực tế thường gặp & Xử lý sự cố (FAQ & Troubleshooting)

### Q1: Làm thế nào để giả lập và kiểm tra Process Death trên thiết bị thật một cách chính xác?
- **Cách làm chuẩn:**
  1. Mở ứng dụng lên màn hình cần test, nhập đầy đủ dữ liệu vào form.
  2. Nhấn nút **Home** để đưa ứng dụng xuống background.
  3. Mở Terminal và chạy lệnh ADB sau:
     ```bash
     adb shell am kill com.example.myapp
     ```
     *(Lệnh này giả lập đúng hành vi hệ thống kill app do hết RAM: giữ lại trạng thái lưu trong SavedStateRegistry).*
  4. Mở lại ứng dụng từ danh sách App gần đây (Recent Apps). Nếu dữ liệu đã nhập vẫn còn nguyên vẹn, bạn đã triển khai khôi phục State thành công!
  5. *(Lưu ý: Không bấm nút Stop màu đỏ trong Android Studio, vì nút đó sẽ kill toàn bộ ứng dụng và xóa sạch cả SavedStateRegistry, không phản ánh đúng Process Death ngoài đời thực).*

### Q2: Sự khác biệt giữa `rememberSaveable(inputs = ...)` và `remember(keys = ...)`?
- Cả hai đều cho phép truyền các "khóa phụ thuộc" (`keys` hoặc `inputs`).
- Khi bất kỳ giá trị nào trong mảng `inputs` thay đổi, `rememberSaveable` sẽ **bỏ qua giá trị đã lưu trước đó** và tính toán lại lambda khởi tạo mới từ đầu.
- Điểm khác biệt: `inputs` trong `rememberSaveable` chỉ hỗ trợ kiểm tra giá trị tại thời điểm Recomposition; còn khi khôi phục từ Process Death, nó sẽ dựa vào `key` định danh duy nhất để kéo dữ liệu từ `Bundle` ra.

### Q3: Tại sao `rememberSaveable { mutableStateOf(user) }` bị crash với lỗi `IllegalArgumentException: ... cannot be saved using the current SaveableStateRegistry`?
- **Nguyên nhân:** Kiểu dữ liệu của đối tượng `user` không phải là kiểu nguyên thủy và cũng chưa được gắn `@Parcelize` hoặc cung cấp `Saver`.
- **Giải pháp:** Gắn `@Parcelize` vào class `User` và kế thừa interface `Parcelable`, hoặc cung cấp `Saver` tùy biến qua tham số `stateSaver`.

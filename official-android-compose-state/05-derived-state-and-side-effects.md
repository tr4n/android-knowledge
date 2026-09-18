# Bài 05 — derivedStateOf, Side Effects & snapshotFlow: Tối Ưu State & Cầu Nối Reactive Flow

> **Tài liệu tham chiếu chính thức:** [Side-effects in Compose & derivedStateOf — Android Developers](https://developer.android.com/develop/ui/compose/side-effects)  
> **Phiên bản áp dụng:** Kotlin 2.0+, Compose BOM 2024.06+, Lifecycle 2.8+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Khi xây dựng giao diện người dùng tương tác phức tạp, ta thường xuyên đối mặt với hai bài toán lớn:
1. **Tính toán trạng thái phái sinh từ một State biến thiên liên tục** mà không làm bùng nổ số lần Recomposition.
2. **Thực thi các tác vụ bất đồng bộ hoặc kết nối với hệ thống bên ngoài** mà vẫn đảm bảo an toàn tuyệt đối theo vòng đời của Composition.

Google cung cấp bộ 3 công cụ chuyên dụng giải quyết triệt để hai bài toán này:

- **`derivedStateOf` (Trạng thái phái sinh):** Một hàm tạo State đặc biệt, nhận vào một lambda tính toán dựa trên một hoặc nhiều đối tượng State khác. Nó đóng vai trò như một **bộ đệm lọc (Buffer Filter)**: Chỉ phát ra thông báo kích hoạt Recomposition khi **kết quả đầu ra** thực sự thay đổi giá trị, bất kể các State đầu vào có thay đổi hàng trăm lần trước đó.
- **Side-Effect (Tác vụ phụ ngoài luồng):** Bất kỳ thao tác nào làm biến đổi trạng thái của ứng dụng hoặc tương tác với hệ thống bên ngoài mà nằm ngoài tầm kiểm soát trực tiếp của hàm Composable (ví dụ: mở Socket mạng, đăng ký cảm biến phần cứng, ghi log Analytics, hiển thị Toast).
- **`snapshotFlow` (Cầu nối State sang Flow):** Hàm chuyển đổi một đối tượng Compose State thành một luồng phản ứng lạnh `Flow<T>`, cho phép tận dụng toàn bộ sức mạnh của các Flow Operators (`debounce`, `filter`, `mapLatest`, `distinctUntilChanged`).

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế bên dưới của `derivedStateOf`

Hãy phân tích cơ chế theo dõi phụ thuộc (Dependency Tracking) của `derivedStateOf`:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      CƠ CHẾ HOẠT ĐỘNG CỦA derivedStateOf               │
│                                                                        │
│ 1. State Nguồn (High-Frequency Input):                                 │
│    listState.firstVisibleItemIndex: 0 ──► 1 ──► 2 ──► 3 ──► 4 ──► 5... │
│    (Thay đổi liên tục mỗi khi cuộn 1 pixel)                           │
│                                                                        │
│ 2. Biểu thức Phái sinh:                                                │
│    derivedStateOf { listState.firstVisibleItemIndex > 0 }              │
│                                                                        │
│ 3. Kết quả đầu ra (Low-Frequency Output):                              │
│    false ──► [TRUE] ──► true ──► true ──► true ──► true...             │
│                 │                                                      │
│                 ▼                                                      │
│    CHỈ PHÁT TÍN HIỆU RECOMPOSITION 1 LẦN DUY NHẤT                      │
│    KHI GIÁ TRỊ ĐẦU RA CHUYỂN TỪ FALSE SANG TRUE!                       │
└────────────────────────────────────────────────────────────────────────┘
```

- Khi Composable lần đầu đọc giá trị `derivedState.value`, `derivedStateOf` tự động ghi nhận danh sách tất cả các đối tượng `State` được đọc bên trong thân lambda của nó.
- Mỗi khi có bất kỳ State nguồn nào thay đổi, `derivedStateOf` tính toán lại biểu thức.
- **Điểm mấu chốt:** Nó so sánh kết quả mới với kết quả cũ bằng phép toán bằng cấu trúc (`==`). Nếu kết quả vẫn bằng nhau, **Snapshot System sẽ triệt tiêu thông báo Recomposition**! Không một Composable nào bị vẽ lại oan uổng.

### 2.2 Cơ chế của `snapshotFlow`

`snapshotFlow` hoạt động bằng cách đăng ký một bộ quan sát với Compose Snapshot Runtime:

```kotlin
fun <T> snapshotFlow(block: () -> T): Flow<T>
```
1. Khởi tạo một `Flow` lạnh.
2. Khi có Collector bắt đầu thu thập (`collect`), `snapshotFlow` chạy lambda `block` lần đầu trong một Snapshot đọc để lấy giá trị ban đầu và xác định các State phụ thuộc.
3. Đăng ký một `Snapshot.registerApplyObserver`. Bất cứ khi nào có một Snapshot mới được ghi đè (`apply()`), nếu một trong các State phụ thuộc bị biến đổi, `snapshotFlow` sẽ chạy lại lambda `block`.
4. Nếu giá trị mới khác với giá trị trước đó (dựa trên `!equals`), nó sẽ phát (`emit`) giá trị mới đó vào luồng Flow!

---

## 3. Bài toán & Kiến trúc: So sánh `remember(key)` vs `derivedStateOf`

Đây là một trong những câu hỏi kiến trúc cốt lõi thường gặp nhất trong Jetpack Compose: **Khi nào dùng `remember(key)` và khi nào dùng `derivedStateOf`?**

```
┌──────────────────────────────────────┬──────────────────────────────────────┐
│       remember(key1, key2) { }       │          derivedStateOf { }          │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ - Tính toán lại khi **key thay đổi**.│ - Tính toán lại khi **State nguồn    │
│ - Tần suất tính toán: Bằng đúng tần  │   bên trong thay đổi**.              │
│   suất thay đổi của key.             │ - Lọc bỏ Recomposition: Chỉ kích    │
│                                      │   hoạt Recompose khi **kết quả đầu ra│
│                                      │   thực sự biến đổi**.                │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

### 3.1 Quy tắc Vàng từ Google
> [!IMPORTANT]
> - Hãy dùng **`derivedStateOf`** khi: Bạn có một State đầu vào thay đổi với tần suất rất cao (High frequency), nhưng kết quả tính toán đầu ra chỉ thay đổi với tần suất thấp (Low frequency).
>   *(Ví dụ: Cuộn vị trí `firstVisibleItemIndex` từ 0 đến 100, nhưng bạn chỉ quan tâm xem nó có `> 0` hay không).*
> - Hãy dùng **`remember(key)`** khi: Bạn muốn lưu lại kết quả của một phép tính tốn kém (như sắp xếp, lọc danh sách) và chỉ tính lại khi danh sách nguồn hoặc tiêu chí lọc thay đổi.

---

## 4. Thực hành tốt nhất & Cảnh báo (Best Practices & Anti-patterns)

### 4.1 Anti-pattern: Lạm dụng `derivedStateOf` sai mục đích

> [!CAUTION]
> **Không bao giờ dùng `derivedStateOf` để nối chuỗi hoặc tính toán đơn giản:**
>
> ```kotlin
> // ❌ SAI LẦM: Tốn CPU và lãng phí RAM!
> var firstName by remember { mutableStateOf("") }
> var lastName by remember { mutableStateOf("") }
> val fullName by remember { 
>     derivedStateOf { "$firstName $lastName" } // CHỐNG CHỈ ĐỊNH!
> }
> ```
> *Tại sao sai?* Mỗi khi `firstName` đổi, `fullName` chắc chắn cũng đổi! Tần suất đầu vào và đầu ra bằng hệt nhau ($1:1$). Sử dụng `derivedStateOf` ở đây chỉ làm tăng thêm chi phí khởi tạo đối tượng Snapshot Observer trong RAM mà không đem lại bất kỳ lợi ích giảm Recomposition nào!
>
> **Cách viết đúng:**
> ```kotlin
> // ✅ ĐÚNG: Tính toán trực tiếp hoặc dùng remember(key) nếu phép tính nặng
> val fullName = "$firstName $lastName"
> ```

### 4.2 Các Side-Effect APIs quan trọng cần kết hợp với State

1. **`LaunchedEffect(key)`:** Chạy một Coroutine an toàn trong phạm vi Composable. Tự động hủy khi Composable rời khỏi cây giao diện hoặc khi `key` thay đổi.
2. **`rememberUpdatedState(latestValue)`:** Giữ tham chiếu mới nhất của một lambda/state trong một Coroutine chạy lâu mà **không làm khởi động lại (restart) Coroutine**.
3. **`DisposableEffect(key)`:** Dành cho các tác vụ cần dọn dẹp tài nguyên (Clean up) khi Composable bị hủy (như hủy đăng ký Listener, BroadcastReceiver).
4. **`produceState(initialValue, ...)`:** Chuyển đổi một nguồn dữ liệu bất đồng bộ bên ngoài (như Flow từ Repository) thành một `State<T>`.

---

## 5. Mã nguồn Thực tế (Implementation)

### 5.1 Nút "Cuộn lên đầu trang" (Scroll-to-top FAB) tối ưu với `derivedStateOf`

```kotlin
package com.example.compose.state.effects

import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.KeyboardArrowUp
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.launch

@Composable
fun ScrollToTopOptimizedScreen() {
    val listState = rememberLazyListState()
    val coroutineScope = rememberCoroutineScope()

    // 1. ÁP DỤNG derivedStateOf:
    // listState.firstVisibleItemIndex thay đổi hàng chục lần khi cuộn,
    // nhưng showButton chỉ phát thông báo Recomposition đúng 1 lần khi chuyển giữa false <-> true!
    val showButton by remember {
        derivedStateOf {
            listState.firstVisibleItemIndex > 0
        }
    }

    Box(modifier = Modifier.fillMaxSize()) {
        LazyColumn(
            state = listState,
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp)
        ) {
            items(100) { index ->
                Text(
                    text = "Dòng dữ liệu thứ #$index",
                    modifier = Modifier.padding(vertical = 12.dp)
                )
                HorizontalDivider()
            }
        }

        // 2. Nút bấm xuất hiện mượt mà
        AnimatedVisibility(
            visible = showButton,
            enter = fadeIn(),
            exit = fadeOut(),
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(24.dp)
        ) {
            FloatingActionButton(
                onClick = {
                    coroutineScope.launch {
                        listState.animateScrollToItem(index = 0)
                    }
                }
            ) {
                Icon(Icons.Default.KeyboardArrowUp, contentDescription = "Scroll to top")
            }
        }
    }
}
```

### 5.2 Cầu nối `snapshotFlow` kết hợp Analytics Logging

Tình huống: Ta muốn ghi log Analytics khi người dùng dừng cuộn và dừng lại ở một trang mới ít nhất 500ms (Debounce), tránh bắn log liên tục khi đang lướt nhanh:

```kotlin
package com.example.compose.state.effects

import androidx.compose.foundation.lazy.LazyListState
import androidx.compose.runtime.*
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.flow.debounce
import kotlinx.coroutines.flow.distinctUntilChanged

@Composable
fun TrackPageScrollAnalytics(
    listState: LazyListState,
    onPageSelected: (Int) -> Unit
) {
    // Sử dụng rememberUpdatedState để giữ callback mới nhất mà không restart Coroutine
    val currentOnPageSelected by rememberUpdatedState(onPageSelected)

    LaunchedEffect(listState) {
        // CHUYỂN ĐỔI COMPOSE STATE SANG FLOW
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged() // Chỉ nhận khi số trang thực sự đổi
            .debounce(500)          // Chờ người dùng dừng tay 500ms
            .collectLatest { settledIndex ->
                currentOnPageSelected(settledIndex)
            }
    }
}
```

---

## 6. Các câu hỏi thực tế thường gặp & Xử lý sự cố (FAQ & Troubleshooting)

### Q1: Tại sao viết `val showButton = remember(listState.firstVisibleItemIndex > 0) { listState.firstVisibleItemIndex > 0 }` lại không tối ưu bằng `derivedStateOf`?
- **Phân tích:** 
  - Trong biểu thức `remember(listState.firstVisibleItemIndex > 0)`, bạn đã **đọc trực tiếp** `listState.firstVisibleItemIndex` ngay trong thân hàm Composable để tính điều kiện key!
  - Việc đọc State này khiến Composable **bị đăng ký Read Observer ở pha Composition**, dẫn đến hàm Composable vẫn bị Recompose liên tục mỗi khi `firstVisibleItemIndex` thay đổi!
  - Với `derivedStateOf`, State chỉ được đọc bên trong lambda của nó; Composable chỉ quan sát giá trị boolean trả về, do đó hoàn toàn không bị Recompose khi State nguồn biến đổi mà kết quả boolean vẫn giữ nguyên!

### Q2: Tại sao `rememberUpdatedState` lại giải quyết được lỗi "Stale State" trong Coroutine?
- **Tình huống:** Bạn khởi chạy một `LaunchedEffect(Unit)` để đếm ngược 10 giây (Splash screen), sau đó gọi callback `onTimeout()`. Nếu người dùng thực hiện một tương tác làm Composable cha Recompose và truyền vào một instance `onTimeout` mới:
  - Nếu không dùng `rememberUpdatedState`: Coroutine trong `LaunchedEffect` đang giữ tham chiếu cũ của lambda tại thời điểm ban đầu $\rightarrow$ Gọi nhầm callback cũ (**Stale Callback**).
  - Nếu truyền `onTimeout` làm key vào `LaunchedEffect(onTimeout)`: Coroutine sẽ bị hủy và chạy lại từ đầu $\rightarrow$ Bộ đếm thời gian bị reset liên tục!
  - `rememberUpdatedState` tạo ra một `State` trung gian bọc lấy lambda. Mỗi lần Recompose, thuộc tính `.value` được cập nhật tham chiếu mới nhất mà **không làm gián đoạn hay khởi động lại Coroutine**!

### Q3: `snapshotFlow` có thể theo dõi biến Kotlin thông thường (`var x = 0`) được không?
- **Tuyệt đối không!** `snapshotFlow` chỉ hoạt động được với các đối tượng thuộc hệ thống Snapshot của Compose (như `mutableStateOf`, `mutableStateListOf`, `LazyListState`...). Nó không thể phát hiện biến đổi trên các biến nguyên thủy hay đối tượng Java/Kotlin thông thường.

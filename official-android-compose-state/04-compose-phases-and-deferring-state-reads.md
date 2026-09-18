# Bài 04 — 3 Pha Compose & Kỹ Thuật Hoãn Đọc State: Composition, Layout, Draw & Tối Ưu 120 FPS

> **Tài liệu tham chiếu chính thức:** [Jetpack Compose phases & Deferring state reads — Android Developers](https://developer.android.com/develop/ui/compose/phases)  
> **Phiên bản áp dụng:** Kotlin 2.0+, Compose BOM 2024.06+, Compose Compiler 2.0+  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Để chuyển đổi các biến State thành các điểm ảnh (pixels) hiển thị trên màn hình thiết bị, Jetpack Compose thực thi một chu trình khép kín gồm **3 pha riêng biệt (3 Phases of Compose)**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        3 PHA VẼ GIAO DIỆN CỦA COMPOSE                  │
│                                                                        │
│  1. COMPOSITION         2. LAYOUT                 3. DRAW              │
│  "Hiển thị cái gì?"     "Đo và đặt ở đâu?"        "Vẽ như thế nào?"   │
│  (What to show)         (Where to place)          (How to render)      │
│         │                      │                         │             │
│         ▼                      ▼                         ▼             │
│  Chạy các hàm           Bao gồm 2 bước:           Vẽ trực tiếp lên     │
│  @Composable để tạo     - Measure: Đo kích thước  Canvas của GPU       │
│  ra cây UI LayoutNode   - Place: Định vị tọa độ                        │
└────────────────────────────────────────────────────────────────────────┘
```

- **Phase-aware State Reading (Đọc State nhận biết theo pha):** Compose Snapshot System theo dõi chính xác **pha nào** đang chạy tại thời điểm một biến `state.value` được đọc. 
- **Quy tắc vàng của hiệu năng:** Khi State thay đổi, Compose **chỉ thực thi lại pha đã đọc State đó** cùng các pha tiếp sau nó, và **bỏ qua hoàn toàn** các pha đi trước!

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Tác động dây chuyền khi đọc State ở từng pha

Chi phí CPU và thời gian xử lý của 3 pha là hoàn toàn khác nhau:

```
NƠI ĐỌC STATE                  CHI PHÍ XỬ LÝ KHI STATE THAY ĐỔI
──────────────────────────────────────────────────────────────────────────────────
1. Đọc ở Composition:   COMPOSITION ──► LAYOUT (Measure + Place) ──► DRAW (NẶNG NHẤT)
                        (Chạy lại toàn bộ Composable function)

2. Đọc ở Layout:        [BỎ QUA COMPOSITION] ──► LAYOUT ──► DRAW
                        (Bỏ qua hoàn toàn Recomposition!)

3. Đọc ở Draw:          [BỎ QUA COMPOSITION] ──► [BỎ QUA LAYOUT] ──► DRAW (NHẸ NHẤT)
                        (Chỉ gửi lệnh vẽ lại trực tiếp lên GPU)
```

1. **Nếu đọc State ở pha Composition:**
   - Composable đọc giá trị trực tiếp: `val offset = scrollState.value`.
   - Khi `scrollState` thay đổi, Compose buộc phải **Recompose** lại toàn bộ hàm Composable $\rightarrow$ chạy lại pha Layout để đo đạc lại cây con $\rightarrow$ vẽ lại ở pha Draw.
2. **Nếu đọc State ở pha Layout:**
   - Đọc State bên trong lambda của `Modifier.layout { }` hoặc `Modifier.offset { }`.
   - Khi State thay đổi, Compose **không Recompose** hàm Composable! Nó nhảy thẳng vào pha Layout để tính toán lại kích thước/tọa độ rồi chuyển qua Draw.
3. **Nếu đọc State ở pha Draw:**
   - Đọc State bên trong lambda của `Modifier.drawWithContent { }` hoặc `Modifier.graphicsLayer { }`.
   - Khi State thay đổi, Compose **bỏ qua cả Recomposition và Layout**! GPU vẽ lại trực tiếp mà không cần đo đạc lại một pixel nào.

### 2.2 Chi tiết 2 bước trong Pha Layout: Đo lường (Measurement) vs Định vị (Placement)

Pha Layout thực chất được chia làm 2 giai đoạn con:
1. **Measurement Phase (`measure()`):** Xác định kích thước chiều rộng và chiều cao (Width, Height) của từng Node.
2. **Placement Phase (`placeRelative()`):** Xác định tọa độ $X, Y$ để đặt Node đó trên màn hình.

> [!NOTE]
> `Modifier.offset { IntOffset(x, y) }` được thực thi **trong bước Placement**.
> Điều này có nghĩa là khi giá trị offset thay đổi, Compose **thậm chí không cần đo lại (Re-measure)** kích thước của phần tử con! Nó chỉ thay đổi tọa độ đặt vị trí trong danh sách lệnh vẽ, giúp tiết kiệm tối đa chu kỳ xử lý của CPU.

---

## 3. Bài toán & Kiến trúc (Problem Statement & Architecture)

### 3.1 Bài toán "Recomposition Thrashing" khi cuộn màn hình

Hãy tưởng tượng bạn xây dựng một hiệu ứng Header trượt hoặc thu nhỏ khi người dùng cuộn danh sách (`scrollState.value`).

Khi người dùng vuốt ngón tay:
- Màn hình 120Hz yêu cầu vẽ 120 khung hình mỗi giây (tương đương mỗi khung hình chỉ có **8.3 mili-giây** để hoàn thành).
- Giá trị `scrollState.value` thay đổi liên tục: `0, 1, 3, 6, 10, 15, 21, ...` (hàng trăm lần trong 1 giây).

#### Cách viết SAI (Đọc State ở pha Composition):
```kotlin
// ❌ CỰC KỲ NGUY HIỂM CHO HIỆU NĂNG
@Composable
fun BadHeader(scrollState: ScrollState) {
    // Đọc state.value ngay trong thân hàm Composable -> Rơi vào pha COMPOSITION
    val offsetDp = (scrollState.value / 2).dp

    Box(
        modifier = Modifier
            .offset(y = offsetDp) // Modifier.offset(Dp) nhận giá trị tĩnh
            .fillMaxWidth()
            .height(200.dp)
    )
}
```

**Hậu quả:**
Hàm `BadHeader` bị Recompose 120 lần/giây! CPU liên tục tái cấu trúc cây giao diện, gây ra hiện tượng rớt khung hình (Frame Drop), giật lag nghiêm trọng (Jank), và làm nóng máy, hao pin.

---

## 4. Thực hành tốt nhất: Kỹ thuật Hoãn Đọc State (Defer State Reads)

Để đạt hiệu năng mượt mà chuẩn 60/120 FPS, quy tắc cốt lõi của Google là: **Hoãn việc đọc giá trị State đến pha muộn nhất có thể (Defer reading state to the latest phase possible)** bằng cách sử dụng **Lambda Modifiers**.

### 4.1 Bảng chuyển đổi Modifier sang dạng Lambda

| Thao tác giao diện | Cách viết cũ (Chậm - Pha Composition) | Cách viết tối ưu (Nhanh - Pha Layout/Draw) |
| :--- | :--- | :--- |
| **Dịch chuyển vị trí (Offset)** | `Modifier.offset(x.dp, y.dp)` | `Modifier.offset { IntOffset(x, y) }` |
| **Độ trong suốt (Alpha)** | `Modifier.alpha(alpha)` | `Modifier.graphicsLayer { this.alpha = alpha }` |
| **Tỉ lệ thu phóng (Scale)** | `Modifier.scale(scale)` | `Modifier.graphicsLayer { scaleX = scale; scaleY = scale }` |
| **Xoay góc (Rotation)** | `Modifier.rotate(degrees)` | `Modifier.graphicsLayer { rotationZ = degrees }` |
| **Vẽ màu nền biến thiên** | `Modifier.background(color)` | `Modifier.drawBehind { drawRect(color) }` |

> [!TIP]
> Bất cứ khi nào một `Modifier` có phiên bản nhận tham số dạng **lambda** `{ ... }`, hãy luôn ưu tiên sử dụng phiên bản lambda nếu giá trị của nó phụ thuộc vào một `State` thay đổi thường xuyên!

### 4.2 Mô hình State Provider Pattern (`() -> T` Lambda)

Khi cần truyền một giá trị State thay đổi với tần suất cao từ Composable cha xuống Composable con, **đừng đọc State ở cha rồi truyền giá trị thô xuống con**:

```kotlin
// ❌ CÁCH VIẾT SAI: Làm Composable cha bị Recompose liên tục!
@Composable
fun ParentScreen(scrollState: ScrollState) {
    // Đọc scrollState.value ở đây -> ParentScreen bị Recompose 120 lần/giây!
    ChildToolbar(offsetY = scrollState.value)
}

// ✅ CÁCH VIẾT TỐI ƯU: Truyền Lambda Provider () -> T
@Composable
fun ParentScreen(scrollState: ScrollState) {
    // Không đọc State ở cha! ParentScreen chỉ Recompose đúng 1 lần duy nhất.
    ChildToolbar(offsetYProvider = { scrollState.value })
}

@Composable
fun ChildToolbar(offsetYProvider: () -> Int) {
    Box(
        modifier = Modifier.offset { 
            // State chỉ được đọc bên trong Placement Phase của Child!
            IntOffset(0, offsetYProvider()) 
        }
    )
}
```

### 4.3 Hiểm họa Vòng lặp Pha (Phase Loop / Infinite Invalidation)

> [!CAUTION]
> **Quy tắc bất biến:** Một pha KHÔNG BAO GIỜ được phép ghi đè (write) vào một State mà chính pha đó hoặc các pha trước nó phụ thuộc vào!
> 
> - **Lỗi kinh điển:** Đọc kích thước của một Node trong pha Layout rồi gán trực tiếp vào một `mutableStateOf` để dùng lại cho chính kích thước đó.
> - **Hậu quả:** Ghi vào State làm pha Layout bị "bẩn" $\rightarrow$ Compose lên lịch Layout lại $\rightarrow$ Layout lại đọc State $\rightarrow$ lại ghi vào State $\rightarrow$ **Tạo ra vòng lặp vô tận (Infinite Layout Loop)** làm ứng dụng treo cứng (ANR) hoặc văng lỗi `IllegalStateException: Reading a state that is being written`.

---

## 5. Mã nguồn Thực tế (Implementation)

Dưới đây là màn hình minh họa thực tế xây dựng một Collapsible Toolbar (Thanh công cụ thu phóng theo thao tác cuộn) đạt tốc độ 120 FPS tuyệt đối nhờ hoãn đọc State sang pha Layout và Draw.

```kotlin
package com.example.compose.state.phases

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.unit.IntOffset
import androidx.compose.ui.unit.dp

@Composable
fun OptimizedCollapsibleToolbarScreen() {
    val listState = rememberLazyListState()

    Box(modifier = Modifier.fillMaxSize()) {
        // 1. Danh sách nội dung bên dưới
        LazyColumn(
            state = listState,
            contentPadding = PaddingValues(top = 200.dp),
            modifier = Modifier.fillMaxSize()
        ) {
            items(100) { index ->
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 8.dp)
                ) {
                    Text(
                        text = "Vật phẩm số: #$index",
                        modifier = Modifier.padding(16.dp)
                    )
                }
            }
        }

        // 2. Header được tối ưu hóa: KHÔNG RECOMPOSE KHI CUỘN!
        CollapsibleHeader(
            scrollOffsetProvider = { listState.firstVisibleItemScrollOffset },
            firstVisibleIndexProvider = { listState.firstVisibleItemIndex }
        )
    }
}

/**
 * Header nhận Lambda providers thay vì nhận giá trị thô,
 * đảm bảo State chỉ được đọc bên trong Pha Layout & Pha Draw!
 */
@Composable
fun CollapsibleHeader(
    scrollOffsetProvider: () -> Int,
    firstVisibleIndexProvider: () -> Int,
    modifier: Modifier = Modifier
) {
    Box(
        modifier = modifier
            .fillMaxWidth()
            .height(200.dp)
            // HOÃN ĐỌC STATE VÀO PHA LAYOUT: Bỏ qua Recomposition!
            .offset {
                val scroll = if (firstVisibleIndexProvider() > 0) 200 else scrollOffsetProvider()
                val collapseOffset = (-scroll).coerceIn(-200, 0)
                IntOffset(x = 0, y = collapseOffset)
            }
            // HOÃN ĐỌC STATE VÀO PHA DRAW: Vẽ trực tiếp lên GPU!
            .graphicsLayer {
                val scroll = if (firstVisibleIndexProvider() > 0) 200 else scrollOffsetProvider()
                // Làm mờ dần từ 1.0 về 0.2 khi cuộn
                alpha = (1f - (scroll / 200f)).coerceIn(0.2f, 1f)
                // Thu nhỏ header nhẹ theo trục Y
                scaleY = (1f - (scroll / 600f)).coerceIn(0.8f, 1f)
            }
            .background(Color(0xFF1E88E5))
            .padding(16.dp)
    ) {
        Text(
            text = "Collapsible Toolbar (120 FPS)",
            style = MaterialTheme.typography.headlineSmall,
            color = Color.White
        )
    }
}
```

---

## 6. Các câu hỏi thực tế thường gặp & Xử lý sự cố (FAQ & Troubleshooting)

### Q1: Tại sao `Modifier.graphicsLayer { }` lại đạt hiệu năng cao nhất trong Compose?
- **Trả lời:** `graphicsLayer` tạo ra một lớp hiển thị riêng biệt trên GPU (RenderNode trên Android 10+ hoặc DisplayList trên các bản Android cũ).
- Khi bạn thay đổi các thuộc tính như `alpha`, `translationX/Y`, `rotation`, `scale` bên trong lambda của `graphicsLayer`:
  1. Compose **không gọi lại Recomposition**.
  2. Compose **không tính toán lại Layout** (không cần đo lại chiều rộng/chiều cao).
  3. Hệ thống chỉ cập nhật ma trận biến đổi (Matrix transform) trực tiếp trong danh sách lệnh vẽ của GPU. Đây là thao tác có chi phí gần như bằng 0 (Zero CPU overhead).

### Q2: Làm sao để kiểm chứng một Composable có đang bị Recomposition thừa thãi hay không?
- **Phương pháp chuẩn:**
  1. Trong Android Studio, mở tab **Layout Inspector**.
  2. Bật cờ **Show Recomposition Counts**.
  3. Thao tác cuộn trên màn hình:
     - Nếu cột **Recomposition Count** của Header liên tục nhảy số: Bạn đang đọc State ở pha Composition!
     - Nếu cột **Recomposition Count** giữ nguyên số 1 (chỉ compose lần đầu), trong khi Header vẫn trượt và mờ dần mượt mà: Bạn đã áp dụng thành công kỹ thuật Hoãn đọc State!

### Q3: Khi nào BẮT BUỘC phải đọc State ở pha Composition mà không thể hoãn?
- **Trả lời:** Khi sự thay đổi của State dẫn đến việc **thay đổi cấu trúc cây giao diện (UI Structural Hierarchy)**.
- Ví dụ:
  ```kotlin
  if (state.isLoading) {
      CircularProgressIndicator() // Node A
  } else {
      ProductContent()            // Node B
  }
  ```
  Trong trường hợp này, vì Compose cần thêm mới hoặc xóa bỏ các Node khỏi cây Composition, ta bắt buộc phải thực thi ở pha Composition. Kỹ thuật hoãn đọc State chỉ áp dụng cho các thuộc tính kích thước, vị trí, độ trong suốt, màu sắc và hoạt ảnh của các Node sẵn có.

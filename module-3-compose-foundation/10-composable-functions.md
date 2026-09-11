# Bài 10 — Composable Functions: Lifecycle, Slot API & @Composable Contract

> **Module:** 3 — Jetpack Compose Foundation  
> **Prerequisite:** Module 1 & 2 hoàn chỉnh  
> **Official Docs:**
> - [Compose Mental Model — Android Developers](https://developer.android.com/develop/ui/compose/mental-model)
> - [Lifecycle of Composables](https://developer.android.com/develop/ui/compose/lifecycle)
> - [CompositionLocal guide](https://developer.android.com/develop/ui/compose/compositionlocal)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Jetpack Compose** là bộ công cụ xây dựng giao diện người dùng hiện đại (Modern UI Toolkit) dạng khai báo (Declarative) cho Android. Trọng tâm của Compose là các **Composable Functions (Hàm khả hợp)** được đánh dấu bằng annotation `@Composable`.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      COMPOSABLE ECOSYSTEM & CONTRACT                   │
│                                                                        │
│   @Composable fun MyScreen()                                           │
│         │                                                              │
│         ▼ Kích hoạt Compose Compiler Plugin                            │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 1. Composition (Giai đoạn khởi tạo cây giao diện)              │   │
│   │    - Thực thi các hàm Composable để dựng Composition Tree      │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Khi State thay đổi                 │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 2. Recomposition (Giai đoạn cập nhật thông minh)               │   │
│   │    - Chỉ thực thi lại những Composable có State biến động      │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Khi node bị gỡ khỏi UI            │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 3. Decomposition / Leave (Giai đoạn giải phóng)                │   │
│   │    - Dọn dẹp tài nguyên và loại bỏ node khỏi bộ nhớ           │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Khái niệm cốt lõi

#### Annotation `@Composable` (Tô màu kiểu dữ liệu - Function Coloring)
- `@Composable` không chỉ là một metadata annotation thông thường, mà nó **làm thay đổi kiểu của hàm ở cấp độ trình biên dịch (Type System Transformation)**:
  ```kotlin
  // Kiểu thực tế của hàm Composable:
  @Composable () -> Unit  khác hoàn toàn với  () -> Unit
  ```
- Tương tự như từ khóa `suspend` trong Coroutines, một hàm `@Composable` **chỉ có thể được gọi từ bên trong một hàm `@Composable` khác**.

#### Composition Tree (Cây thành phần)
- Cấu trúc dữ liệu dạng cây trong bộ nhớ biểu diễn toàn bộ các UI node và quan hệ cha-con được tạo ra sau khi thực thi các hàm `@Composable`.

#### `CompositionLocal` (Dòng chảy dữ liệu môi trường - Ambient Data)
- Công cụ truyền dữ liệu ngầm định xuyên qua cây Composition mà không cần phải truyền tham số thủ công qua từng tầng Composable con (Prop Drilling).
- Ví dụ: `LocalContext.current`, `LocalDensity.current`, `LocalContentColor.current`.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Compose Compiler Plugin: Bytecode Transformation

Khi trình biên dịch Compose Compiler Plugin duyệt qua một hàm `@Composable`, nó thực hiện biến đổi bytecode tinh vi:

#### Mã nguồn Kotlin ban đầu:
```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Xin chào $name")
}
```

#### Mã dịch bytecode (Tương đương sau khi Compiler biến đổi):
Compiler chèn thêm 2 tham số ngầm định: `$composer: Composer` và `$changed: Int`:
```java
public static final void Greeting(String name, Composer $composer, int $changed) {
    $composer = $composer.startRestartGroup(1234567); // Hash ID duy nhất của node
    
    // Kiểm tra bitmask $changed: Nếu tham số 'name' không đổi, SKIP TOÀN BỘ!
    if (($changed & 0b11) == 0b10 && $composer.getSkipping()) {
        $composer.skipToGroupEnd();
    } else {
        // Thực thi vẽ Text
        TextKt.Text("Xin chào " + name, $composer, 0);
    }
    
    ScopeUpdateScope $scope = $composer.endRestartGroup();
    if ($scope != null) {
        $scope.updateScope((Composer nc, int force) -> Greeting(name, nc, $changed | 1));
    }
}
```

- **`$composer: Composer`**: Con trỏ trung tâm chịu trách nhiệm ghi/đọc dữ liệu vào **Slot Table**.
- **`$changed: Int`**: Một bitmask integer chứa trạng thái thay đổi của các tham số. Nhờ bitmask này, Compose biết chính xác tham số nào có giá trị mới để quyết định chạy tiếp hay **Skip (Bỏ qua)** việc thực thi hàm!

---

### 2.2 Vòng đời của Composable (Composable Lifecycle)

Khác hoàn toàn với Activity hay Fragment có hàng chục hàm lifecycle phức tạp (`onCreate`, `onStart`, `onResume`, `onPause`), vòng đời của một Composable cực kỳ tinh gọn:

```
┌────────────────────────────────────────────────────────┐
│                   COMPOSABLE LIFECYCLE                 │
│                                                        │
│   [Enter Composition] ◄── Lần đầu xuất hiện trên UI   │
│           │                                            │
│           ▼                                            │
│   [Recompose 0..N lần] ◄── Chạy lại khi State thay đổi │
│           │                                            │
│           ▼                                            │
│   [Leave Composition] ◄── Bị gỡ khỏi cây giao diện     │
└────────────────────────────────────────────────────────┘
```

> **Lưu ý sống còn:** Một Composable có thể được recompose **với tần suất 60 lần hoặc 120 lần mỗi giây** (theo từng khung hình hiển thị màn hình) hoặc **bị bỏ qua hoàn toàn** nếu State không biến động. Nó cũng có thể được thực thi trên các background threads song song!

---

### 2.3 Cấu trúc dữ liệu bên dưới: Slot Table & Gap Buffer

Làm thế nào Compose lưu trữ trạng thái, con trỏ và cây UI mà không tạo ra hàng triệu objects gây quá tải Garbage Collector? Câu trả lời là: **Slot Table với kỹ thuật Gap Buffer**.

```
MÔ HÌNH GAP BUFFER TRONG BỘ NHỚ CỦA SLOT TABLE:
┌────────────────────────────────────────────────────────────────────────┐
│ [Group: Greeting] [Slot: name] [Slot: Color] [  GAP (VÙNG TRỐNG)  ]   │
│  Slot 0           Slot 1       Slot 2        [  ĐỂ CHÈN DỮ LIỆU   ]   │
│                                              [  KÍCH THƯỚC CO GIÃN]   │
│                                              [                    ]   │
│                                              Slot 100     Slot 101    │
│                                              [Group: Btn] [Slot: Text]│
└────────────────────────────────────────────────────────────────────────┘
```

#### Nguyên lý hoạt động của Gap Buffer:
1. Toàn bộ cây Composition được làm phẳng thành một mảng tuyến tính một chiều (**Flat Array**).
2. Khi người dùng bấm nút làm xuất hiện thêm một Composable mới (ví dụ hiển thị thông báo lỗi bằng lệnh `if (isError)`), Compose chỉ việc **di chuyển vùng trống Gap** đến vị trí cần chèn và ghi dữ liệu vào đó trong thời gian $O(1)$.
3. Kỹ thuật này mượn từ cơ chế quản lý con trỏ soạn thảo văn bản trong các Text Editor kinh điển như Emacs, mang lại tốc độ truy xuất cực nhanh và tối ưu hóa bộ nhớ RAM vượt trội so với cây đối tượng (Object Tree) truyền thống.

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Sự sụp đổ của mô hình XML Imperative UI cũ

| Khía cạnh | XML Imperative UI (View System cũ) | Jetpack Compose (Declarative) |
|---|---|---|
| **Cơ chế hoạt động** | Mệnh lệnh (Imperative): Tìm View bằng ID rồi ép nó đổi trạng thái: `textView.setText()`, `progressBar.setVisibility(GONE)`. | Khai báo (Declarative): Giao diện là hàm của State: $\text{UI} = f(\text{State})$. |
| **Gánh nặng kế thừa** | Class `android.view.View` phình to hơn **30,000 dòng code**, ôm đồm cả vẽ, đo đạc, animation, touch events. | Tách biệt hoàn toàn: Composable chỉ là các hàm Kotlin thuần túy, không kế thừa cha-con cồng kềnh. |
| **Đồng bộ trạng thái** | Thường xuyên bị lệch pha (Desynchronization): Quên ẩn ProgressBar khi dữ liệu đã về, hoặc quên cập nhật text khi biến thay đổi. | Đảm bảo tính nhất quán 100%: Khi State đổi, Compose tự động tính toán diff và vẽ lại đúng node cần đổi. |
| **Tái sử dụng giao diện** | Viết Custom View bằng Java/XML cực kỳ cực nhọc (cần khai báo `attrs.xml`, override 3 constructors, xử lý `onMeasure`, `onDraw`). | Cực kỳ dễ dàng: Chỉ cần tạo một hàm `@Composable` nhận tham số Slot và tái sử dụng ở bất kỳ đâu. |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Quy tắc vàng: Idempotent & Side-Effect Free

Hàm `@Composable` bắt buộc phải có tính chất **Idempotent (Xác định)**: Với cùng một tập tham số đầu vào, nó phải luôn sinh ra cùng một cấu trúc giao diện và **không được phép làm thay đổi bất kỳ trạng thái toàn cục nào bên ngoài thân hàm**.

```kotlin
// SAI LẦM CHÍ MẠNG (ANTI-PATTERN):
@Composable
fun BadCounter() {
    globalCount++ // CẤM: Thay đổi biến toàn cục trong thân Composable!
    
    // CẤM: Gọi hàm sinh ngẫu nhiên hoặc đọc thời gian thực trực tiếp mà không remember!
    val randomId = UUID.randomUUID().toString()
    
    Button(onClick = { }) {
        Text("Count: $globalCount - ID: $randomId")
    }
}

// CHUẨN MỰC:
@Composable
fun GoodCounter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}
```

---

### 4.2 Thiết kế Component với Slot API Pattern

**Slot API** là mẫu thiết kế mạnh mẽ nhất trong Compose giúp tạo ra các UI components có độ tùy biến cực cao. Thay vì truyền hàng chục tham số cờ cấu hình (`showIcon: Boolean`, `iconRes: Int`, `titleColor: Color`), ta cung cấp các "khoảng trống" (Slots) dưới dạng lambda `@Composable () -> Unit`:

```kotlin
@Composable
fun CustomCard(
    modifier: Modifier = Modifier,
    header: @Composable () -> Unit,
    content: @Composable () -> Unit,
    actions: @Composable RowScope.() -> Unit
) {
    Card(modifier = modifier) {
        Column(modifier = Modifier.padding(16.dp)) {
            header()
            Spacer(modifier = Modifier.height(12.dp))
            content()
            Spacer(modifier = Modifier.height(16.dp))
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.End,
                content = actions
            )
        }
    }
}
```

---

### 4.3 Khi nào NÊN và KHÔNG NÊN dùng `CompositionLocal`?

```
BẠN CÓ NÊN DÙNG COMPOSITIONLOCAL?
│
├── Dữ liệu mang tính môi trường toàn cục (Ambient/Theme)? ────────► NÊN DÙNG!
│   (Màu sắc hệ thống, Kích thước font chữ, Spacing, Padding chuẩn, Analytics logger)
│
└── Dữ liệu nghiệp vụ của một tính năng cụ thể (Business State)? ──► CẤM DÙNG!
    (User object, Cart items, cờ loading của form)
    => Vì sẽ biến Component thành "hộp đen ngầm", không biết được dependency đầu vào!
```

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng hệ thống Component thẻ thông tin chuẩn Design System: **`AppSurfaceCard`** ứng dụng toàn diện:
- Slot API (`header`, `content`, `actions`).
- Cung cấp Custom Theme Spacing bằng `CompositionLocal`.
- Hỗ trợ xem trước giao diện với `@Preview`.

### Bước 1: Thiết kế Custom `CompositionLocal` cho Design System

```kotlin
// 1. Data class định nghĩa khoảng cách chuẩn (Spacing Tokens)
data class AppSpacing(
    val default: Dp = 0.dp,
    val extraSmall: Dp = 4.dp,
    val small: Dp = 8.dp,
    val medium: Dp = 16.dp,
    val large: Dp = 24.dp,
    val extraLarge: Dp = 32.dp
)

// 2. Khởi tạo CompositionLocal với giá trị mặc định
val LocalAppSpacing = staticCompositionLocalOf { AppSpacing() }

// 3. Theme Wrapper cung cấp token xuống toàn bộ cây UI
@Composable
fun AppDesignTheme(
    spacing: AppSpacing = AppSpacing(),
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(LocalAppSpacing provides spacing) {
        MaterialTheme(content = content)
    }
}
```

---

### Bước 2: Triển khai Component với Slot API (`AppSurfaceCard`)

```kotlin
@Composable
fun AppSurfaceCard(
    modifier: Modifier = Modifier,
    header: (@Composable () -> Unit)? = null,
    actions: (@Composable RowScope.() -> Unit)? = null,
    content: @Composable () -> Unit
) {
    // Truy xuất spacing từ CompositionLocal thay vì hardcode dp!
    val spacing = LocalAppSpacing.current

    Card(
        modifier = modifier.fillMaxWidth(),
        shape = RoundedCornerShape(spacing.medium),
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceVariant),
        elevation = CardDefaults.cardElevation(defaultElevation = spacing.extraSmall)
    ) {
        Column(modifier = Modifier.padding(spacing.medium)) {
            // Render Header Slot (nếu có)
            if (header != null) {
                header()
                Spacer(modifier = Modifier.height(spacing.small))
            }

            // Render Body Content Slot bắt buộc
            content()

            // Render Actions Slot (nếu có)
            if (actions != null) {
                Spacer(modifier = Modifier.height(spacing.medium))
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.End,
                    verticalAlignment = Alignment.CenterVertically,
                    content = actions
                )
            }
        }
    }
}
```

---

### Bước 3: Ứng dụng Component vào màn hình thực tế

```kotlin
@Composable
fun OrderStatusNotification(
    orderId: String,
    totalAmount: Double,
    onTrackOrder: () -> Unit,
    onCancelOrder: () -> Unit,
    modifier: Modifier = Modifier
) {
    AppSurfaceCard(
        modifier = modifier.padding(16.dp),
        header = {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(
                    text = "Đơn hàng #$orderId",
                    style = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold
                )
                Badge { Text("Đang giao hàng") }
            }
        },
        content = {
            Column {
                Text(
                    text = "Đơn hàng của bạn đang trên đường vận chuyển tới địa chỉ nhà riêng.",
                    style = MaterialTheme.typography.bodyMedium
                )
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = "Tổng tiền: $$totalAmount",
                    style = MaterialTheme.typography.labelLarge,
                    color = MaterialTheme.colorScheme.primary
                )
            }
        },
        actions = {
            OutlinedButton(onClick = onCancelOrder) {
                Text("Hủy đơn")
            }
            Spacer(modifier = Modifier.width(8.dp))
            Button(onClick = onTrackOrder) {
                Text("Theo dõi lộ trình")
            }
        }
    )
}
```

---

### Bước 4: Viết `@Preview` đa cấu hình (Light / Dark Mode)

```kotlin
@Preview(name = "Light Mode", showBackground = true)
@Preview(name = "Dark Mode", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Composable
fun OrderStatusNotificationPreview() {
    AppDesignTheme {
        OrderStatusNotification(
            orderId = "VN-8849",
            totalAmount = 249.99,
            onTrackOrder = {},
            onCancelOrder = {}
        )
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Tại sao một hàm `@Composable` không thể gọi trực tiếp một `suspend` function trong thân hàm?
**Trả lời chuẩn bản chất:**
Bởi vì hàm `@Composable` được thiết kế để chạy đồng bộ và trả về việc ghi vào Slot Table theo từng khung hình vẽ (Frame rendering). Nếu cho phép gọi `suspend` function trực tiếp trong thân hàm, luồng vẽ UI sẽ bị tạm dừng (suspended), dẫn đến việc làm đóng băng việc render khung hình của hệ thống. Để thực thi tác vụ bất đồng bộ hoặc gọi suspend function trong Compose, bắt buộc phải sử dụng các **Effect Handlers** có nhận thức về vòng đời như `LaunchedEffect`.

#### Q2: Phân biệt sự khác nhau giữa `staticCompositionLocalOf` và `compositionLocalOf`?
**Trả lời chuẩn bản chất:**
- **`compositionLocalOf`**: Khi giá trị của nó thay đổi, Compose chỉ recompose **những Composable thực sự đọc giá trị `.current`** của nó (Fine-grained tracking). Phù hợp cho các giá trị có thể thay đổi trong lúc chạy.
- **`staticCompositionLocalOf`**: Compose không theo dõi từng Composable đọc nó. Khi giá trị thay đổi, nó sẽ **buộc toàn bộ cây con bên dưới nó phải Recompose lại từ đầu**! Tuy nhiên, việc đọc nó ít tốn chi phí bộ nhớ hơn. Vì vậy, `staticCompositionLocalOf` là lựa chọn hoàn hảo cho các giá trị gần như tĩnh trong suốt vòng đời ứng dụng (như Theme Colors, Spacing, Typography).

#### Q3: Điều gì xảy ra bên trong Slot Table khi một Composable bị ẩn đi bởi khối lệnh `if (show)`?
**Trả lời chuẩn bản chất:**
Khi điều kiện `if (show)` chuyển thành `false`, con trỏ `$composer` phát hiện rằng nhóm Group tương ứng với Composable đó không còn được gọi nữa. Khối này sẽ rơi vào giai đoạn **Decomposition (Leave Composition)**. Compose sẽ di chuyển con trỏ Gap Buffer qua vùng slot này và đánh dấu nó là trống để sẵn sàng ghi đè. Mọi Coroutine hoặc tài nguyên gắn với node đó (thông qua `DisposableEffect`, `LaunchedEffect`) sẽ lập tức được dọn dẹp và giải phóng.

---

### 6.2 Lỗi Runtime thường gặp & Cách khắc phục

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Crash: `IllegalStateException: CompositionLocal LocalAppSpacing not provided`** | Gọi `LocalAppSpacing.current` ở một Composable nằm ngoài phạm vi bao bọc của `CompositionLocalProvider`. | Bọc màn hình hoặc Root Activity bằng hàm Theme `AppDesignTheme { }`. |
| **Giao diện bị đơ giật do Recomposition liên tục** | Gọi các hàm sinh Side Effect (như khởi tạo Object mới, gửi network log) trực tiếp trong thân hàm `@Composable`. | Đưa toàn bộ Side Effect vào `LaunchedEffect` hoặc `SideEffect`. |
| **Component con không hiển thị gì trên màn hình** | Quên gọi lambda `content()` bên trong hàm custom component sử dụng Slot API. | Đảm bảo vị trí của `content()` được đặt chính xác trong layout Container (`Box`, `Column`, `Row`). |

---

*Bài trước: [09 — Paging 3 + Flow](../module-2-architecture/09-paging3-flow.md)*  
*Bài tiếp theo: [11 — Compose State: remember, rememberSaveable, State Hoisting](11-compose-state.md)*

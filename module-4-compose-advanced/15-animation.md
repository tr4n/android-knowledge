# Bài 15 — Compose Animation: animate*AsState, AnimatedVisibility & Physics Spring

> **Module:** 4 — Jetpack Compose Advanced  
> **Prerequisite:** [Bài 14 — Compose Navigation: Type-Safe, Deep Links & Back Stack](14-navigation.md)  
> **Official Docs:**
> - [Animations in Compose — Android Developers](https://developer.android.com/develop/ui/compose/animation)
> - [Customize animations (AnimationSpec)](https://developer.android.com/develop/ui/compose/animation/customize)
> - [Advanced animation with graphicsLayer](https://developer.android.com/develop/ui/compose/animation/advanced)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong Jetpack Compose, **Animation (Đồ họa chuyển động)** không phải là việc can thiệp sửa đổi các thuộc tính View theo thời gian, mà là **quá trình nội suy giá trị mượt mà giữa các trạng thái (State Transitions)**. 

Bởi vì Compose tuân theo triết lý $\text{UI} = f(\text{State})$, khi State thay đổi, Animation Engine sẽ tính toán các giá trị trung gian ở mỗi khung hình VSYNC (60fps hoặc 120fps) để tạo ra chuyển động liền mạch.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        COMPOSE ANIMATION HIERARCHY                     │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 1. High-Level APIs (Dành cho 90% use cases phổ thông):         │   │
│   │    - animate*AsState: animateFloat, animateColor, animateDp... │   │
│   │    - AnimatedVisibility: Hiển thị / Ẩn nội dung với hiệu ứng   │   │
│   │    - AnimatedContent / Crossfade: Chuyển đổi giữa các layout   │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Xây dựng dựa trên                  │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 2. Low-Level APIs (Kiểm soát chi tiết và đồng bộ đa luồng):    │   │
│   │    - updateTransition: Đồng bộ hóa nhiều thuộc tính cùng lúc   │   │
│   │    - Animatable: Kiểm soát coroutine-driven animation thủ công │   │
│   │    - rememberInfiniteTransition: Hiệu ứng lặp lại vô tận       │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Cấu hình cơ chế chuyển động        │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ 3. Animation Specs (Đặc tả chuyển động):                       │   │
│   │    - spring(): Dựa trên vật lý lò xo (Mặc định & Khuyên dùng)  │   │
│   │    - tween(): Dựa trên thời gian cố định và đường cong Easing  │   │
│   │    - keyframes(): Định nghĩa các mốc thời gian cụ thể          │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Compose Animation Engine & VSYNC Frame Clock

Làm thế nào Compose đảm bảo hoạt ảnh chạy mượt mà ở tần số quét cao (90Hz / 120Hz)?

```
                 COMPOSE FRAME-BY-FRAME ANIMATION RENDER LOOP

Hệ điều hành phát tín hiệu phần cứng: Android VSYNC Pulse (mỗi 8.3ms cho 120Hz)
                              │
                              ▼
           Choreographer ──► MonotonicFrameClock
                              │
                              ▼
    Compose Animation Engine thức dậy tại Frame Time: $nanoTime
                              │
    ┌─────────────────────────┴─────────────────────────┐
    ▼                                                   ▼
1. Tính toán giá trị mới:                           2. Kiểm tra đích đến:
   value = Interpolator(progress)                      targetValue reached?
   velocity = VelocityTracker()                        ├── YES ──► Dừng animation
                              │                        └── NO  ──► Đăng ký VSYNC tiếp
                              ▼
             Gán giá trị vào Snapshot State
                              │
                              ▼
             Render Frame lên màn hình GPU!
```

---

### 2.2 Bản chất Vật lý Lò xo (Spring Physics): Bảo tồn Vận tốc

Tại sao Google khuyến nghị dùng **`spring()`** làm cơ chế animation mặc định thay vì `tween()`?

Trong thế giới thực, một vật thể đang chuyển động sẽ sở hữu **quán tính và vận tốc**. Nếu một chuyển động đang diễn ra mà người dùng đột ngột đổi ý (bấm nút quay ngược lại), mô hình `tween()` thời gian sẽ bị giật cục:

```
SO SÁNH KHI BỊ GIÁN ĐOẠN GIỮA CHỪNG (INTERRUPTED ANIMATION):

1. Tween Animation (Thời gian cố định - SAI LẦM):
   Điểm A ────────────► [Bị gián đoạn ở 50%] ────────────► Điểm B
                              │
                              ▼ Vận tốc bị reset về 0 ngay lập tức!
   Bắt đầu tween mới quay về A ──► GIAO DIỆN BỊ KHỰC, GIẬT CỤC!


2. Spring Physics Animation (Bảo tồn vận tốc - CHUẨN MỰC):
   Điểm A ────────────► [Bị gián đoạn ở 50%] ────────────► Điểm B
                              │
                              ▼ VẬN TỐC HIỆN TẠI ĐƯỢC GIỮ NGUYÊN!
   Lực lò xo kéo ngược về A với gia tốc tự nhiên ──► MƯỢT MÀ NHƯ ĐỜI THỰC!
```

#### Hai tham số cơ học của `spring()`:
- **`dampingRatio` (Hệ số cản):**
  - `DampingRatioHighBouncy` (0.2): Dao động nảy mạnh nhiều lần trước khi dừng.
  - `DampingRatioMediumBouncy` (0.5): Nảy nhẹ nhàng, tự nhiên.
  - `DampingRatioNoBouncy` (1.0 - Mặc định): Không nảy, dừng lại êm ái.
- **`stiffness` (Độ cứng lò xo):**
  - `StiffnessHigh` (10_000): Chuyển động cực nhanh, dứt khoát.
  - `StiffnessMedium` (1_500 - Mặc định): Tốc độ cân bằng cho UI elements.
  - `StiffnessLow` (200): Chuyển động lững lờ, chậm rãi.

---

### 2.3 Tối ưu hóa hiệu năng: Bỏ qua Recomposition với `Modifier.graphicsLayer`

Đây là bí mật hiệu năng quan trọng nhất của Senior Android Architect khi làm Animation:

```
BA GIAI ĐOẠN CỦA MỘT KHUNG HÌNH (COMPOSE PHASES):
┌────────────────────┐    ┌────────────────────┐    ┌────────────────────┐
│ 1. Composition     │───►│ 2. Layout          │───►│ 3. Drawing         │
│ (Xác định vẽ cái gì│    │ (Đo đạc kích thước │    │ (Vẽ điểm ảnh lên   │
│  - TỐN NHIỀU CPU!) │    │  và tọa độ x, y)   │    │  GPU Canvas)       │
└────────────────────┘    └────────────────────┘    └────────────────────┘

CÁCH LÀM TỒN HAO CPU (Recomposition ở mỗi frame):
val alpha by animateFloatAsState(...)
Box(modifier = Modifier.alpha(alpha)) // Đọc State ở Composition Phase!
=> Toàn bộ Composable bị RECOMPOSE 120 LẦN MỖI GIÂY!

CÁCH LÀM TỐI ƯU TUYỆT ĐỐI (GraphicsLayer Render Phase):
val alpha by animateFloatAsState(...)
Box(modifier = Modifier.graphicsLayer { this.alpha = alpha }) // Đọc State ở Draw Phase!
=> BỎ QUA HOÀN TOÀN GIAI ĐOẠN 1 VÀ 2!
=> 0 LẦN RECOMPOSITION! CHỈ VẼ TRỰC TIẾP TRÊN GPU!
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi đau của XML Animation System cũ

1. **Phân mảnh mã nguồn:**
   Trong hệ thống View cũ, ta phải tạo hàng chục file XML rời rạc trong thư mục `res/anim/` (`fade_in.xml`, `slide_up.xml`, `scale.xml`), hoặc viết `ObjectAnimator.ofFloat(view, "translationX", 0f, 100f)` dễ sai tên thuộc tính String.
2. **Trạng thái lấp lửng (Inconsistent State):**
   Nếu View đang chạy animation mà người dùng đổi cấu hình hoặc dữ liệu mạng ập tới, animation có thể bị đứt đoạn, khiến View bị treo ở độ mờ `alpha = 0.5` hoặc biến mất khỏi màn hình.
3. **Compose giải quyết triệt để:**
   Animation được liên kết trực tiếp với State. Bất kể State biến đổi nhanh đến mức nào, Animation Engine sẽ luôn tính toán hướng đi tiếp theo một cách an toàn và tự nhiên.

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Cây quyết định (Decision Tree): Chọn Animation API chuẩn xác

```
BẠN CẦN LÀM HIỆU ỨNG GÌ?
│
├── Thay đổi 1 giá trị đơn lẻ (Màu sắc, kích thước, độ mờ)? ──► animate*AsState
│
├── Hiển thị hoặc ẩn một phần tử trên giao diện? ─────────────► AnimatedVisibility
│   (Kèm theo hiệu ứng mở rộng expandVertically, trượt slideIn, mờ fadeIn)
│
├── Chuyển đổi qua lại giữa các nội dung khác nhau? ──────────► AnimatedContent / Crossfade
│   (Chuyển đổi giữa Loading spinner, Empty state, và Content list)
│
├── Đồng bộ nhiều thuộc tính cùng lúc theo một State chung? ──► updateTransition
│   (Ví dụ: Trạng thái Nút bấm [Idle -> Loading -> Success])
│
└── Hiệu ứng lặp lại vô tận không ngừng? ─────────────────────► rememberInfiniteTransition
    (Hiệu ứng sóng nhạc, vầng hào quang nhấp nháy, Shimmer Loading)
```

---

### 4.2 Cạm bẫy phổ biến (Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Dùng `if (isVisible)` kết hợp với `animateFloatAsState` để ẩn hiện view
```kotlin
// SAI LẦM:
val alpha by animateFloatAsState(if (show) 1f else 0f)
if (show) { // Khi show = false, block này lập tức bị hủy bỏ!
    Text("Hello", modifier = Modifier.alpha(alpha)) // Text biến mất NGAY TỨC THÌ, không kịp mờ dần!
}

// ĐÚNG: Sử dụng AnimatedVisibility
AnimatedVisibility(
    visible = show,
    enter = fadeIn(),
    exit = fadeOut()
) {
    Text("Hello")
}
```

#### Cạm bẫy 2: Dùng `tween()` với thời lượng quá dài cho tương tác chạm
Thời lượng animation tương tác trực tiếp (micro-interactions) chỉ nên kéo dài từ **150ms đến 300ms**. Nếu đặt 1000ms, giao diện sẽ tạo cảm giác chậm chạp và lề mề cho người dùng.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng component tương tác cao cấp: **Interactive Expandable Media Card (Thẻ Trình Phát Nhạc Thu Gọn/Mở Rộng)**:
- Sử dụng `AnimatedVisibility` với custom transitions (`expandVertically` + `fadeIn`).
- Sử dụng `updateTransition` để điều phối màu sắc và kích thước theo trạng thái phát nhạc (`IDLE`, `PLAYING`, `PAUSED`).
- Sử dụng `Modifier.graphicsLayer` để xoay đĩa nhạc 360 độ mà không gây Recomposition.
- Viết Compose UI Test kiểm tra animation với Virtual Clock.

### Bước 1: Khai báo State & Domain Models

```kotlin
enum class PlaybackState { IDLE, PLAYING, PAUSED }

data class SongTrack(
    val title: String,
    val artist: String,
    val durationSeconds: Int,
    val albumCoverRes: Int = 0
)
```

---

### Bước 2: Triển khai Component với `updateTransition` & `graphicsLayer`

```kotlin
@Composable
fun ExpandableMusicPlayerCard(
    track: SongTrack,
    playbackState: PlaybackState,
    onTogglePlay: () -> Unit,
    modifier: Modifier = Modifier
) {
    var isExpanded by remember { mutableStateOf(false) }

    // 1. Quản lý chuyển động đồng bộ bằng updateTransition
    val transition = updateTransition(targetState = playbackState, label = "PlaybackTransition")

    val containerColor by transition.animateColor(
        transitionSpec = { spring(stiffness = Spring.StiffnessLow) },
        label = "ColorAnim"
    ) { state ->
        when (state) {
            PlaybackState.IDLE -> MaterialTheme.colorScheme.surfaceVariant
            PlaybackState.PLAYING -> MaterialTheme.colorScheme.primaryContainer
            PlaybackState.PAUSED -> MaterialTheme.colorScheme.secondaryContainer
        }
    }

    // 2. Hiệu ứng xoay đĩa nhạc vô tận khi đang PLAYING
    val infiniteTransition = rememberInfiniteTransition(label = "DiscRotation")
    val rotationAngle by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 3000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "AngleAnim"
    )

    Card(
        colors = CardDefaults.cardColors(containerColor = containerColor),
        modifier = modifier
            .fillMaxWidth()
            .clickable { isExpanded = !isExpanded }
            .padding(16.dp),
        shape = RoundedCornerShape(16.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.SpaceBetween,
                modifier = Modifier.fillMaxWidth()
            ) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    // TỐI ƯU HÓA GPU: Xoay đĩa nhạc bằng graphicsLayer, không Recompose!
                    Box(
                        modifier = Modifier
                            .size(48.dp)
                            .graphicsLayer {
                                rotationZ = if (playbackState == PlaybackState.PLAYING) rotationAngle else 0f
                            }
                            .background(Color.DarkGray, CircleShape),
                        contentAlignment = Alignment.Center
                    ) {
                        Icon(Icons.Default.PlayArrow, contentDescription = null, tint = Color.White)
                    }

                    Spacer(modifier = Modifier.width(12.dp))

                    Column {
                        Text(text = track.title, style = MaterialTheme.typography.titleMedium, fontWeight = FontWeight.Bold)
                        Text(text = track.artist, style = MaterialTheme.typography.bodySmall)
                    }
                }

                IconButton(onClick = onTogglePlay) {
                    Icon(
                        imageVector = if (playbackState == PlaybackState.PLAYING) Icons.Default.Close else Icons.Default.PlayArrow,
                        contentDescription = "Play/Pause"
                    )
                }
            }

            // 3. Hiệu ứng Mở rộng / Thu gọn chi tiết bằng AnimatedVisibility
            AnimatedVisibility(
                visible = isExpanded,
                enter = fadeIn(animationSpec = spring(stiffness = Spring.StiffnessMedium)) +
                        expandVertically(animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy)),
                exit = fadeOut() + shrinkVertically()
            ) {
                Column(modifier = Modifier.padding(top = 16.dp)) {
                    Divider(color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.2f))
                    Spacer(modifier = Modifier.height(12.dp))
                    Text(
                        text = "Lời bài hát: Đang cập nhật trực tiếp từ hệ thống streaming...",
                        style = MaterialTheme.typography.bodyMedium
                    )
                    Spacer(modifier = Modifier.height(8.dp))
                    Text(
                        text = "Thời lượng: ${track.durationSeconds} giây",
                        style = MaterialTheme.typography.labelSmall
                    )
                }
            }
        }
    }
}
```

---

### Bước 3: Viết Compose UI Test kiểm thử Animation

```kotlin
@RunWith(AndroidJUnit4::class)
class ExpandableMusicPlayerCardTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun card_expandsAndShowsLyrics_onUserClick() {
        val testTrack = SongTrack("Bài Ca Hy Vọng", "Nghệ sĩ A", 210)

        composeTestRule.setContent {
            ExpandableMusicPlayerCard(
                track = testTrack,
                playbackState = PlaybackState.IDLE,
                onTogglePlay = {}
            )
        }

        // Ban đầu phần lời bài hát chưa xuất hiện
        composeTestRule.onNodeWithText("Thời lượng: 210 giây").assertDoesNotExist()

        // Tạm dừng đồng hồ tự động để kiểm soát frame
        composeTestRule.mainClock.autoAdvance = false

        // Bấm vào card để mở rộng
        composeTestRule.onNodeWithText("Bài Ca Hy Vọng").performClick()

        // Tiến thời gian qua 400ms để animation mở rộng hoàn tất
        composeTestRule.mainClock.advanceTimeBy(400L)

        // Lời bài hát phải hiển thị đầy đủ
        composeTestRule.onNodeWithText("Thời lượng: 210 giây").assertIsDisplayed()
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Tại sao `Modifier.graphicsLayer { }` lại tiết kiệm năng lượng và CPU hơn rất nhiều so với việc gọi `Modifier.alpha(value)` khi animate?
**Trả lời chuẩn bản chất:**
Bởi vì Compose chia khung hình thành 3 giai đoạn: **Composition $\rightarrow$ Layout $\rightarrow$ Drawing**.
- Khi bạn dùng `Modifier.alpha(value)`, thuộc tính này nhận giá trị State ở giai đoạn **Composition Phase**. Do đó, mỗi khi `value` thay đổi theo từng mili-giây, toàn bộ Composable này và các con của nó sẽ bị **Recompose** lại từ đầu!
- Khi bạn dùng `Modifier.graphicsLayer { this.alpha = value }`, bạn truyền vào một lambda. Lambda này **chỉ được thực thi ở giai đoạn Drawing Phase**. Compose sẽ bỏ qua hoàn toàn Composition và Layout, và gửi thẳng lệnh biến đổi ma trận (Matrix Transform) xuống GPU Canvas của phần cứng!

#### Q2: Làm thế nào để triển khai Shared Element Transition giữa hai màn hình trong Navigation Compose?
**Trả lời chuẩn bản chất:**
Từ Compose BOM 2024.05+ và Navigation 2.8+, Compose cung cấp API chính thức `SharedTransitionLayout`. Bạn bọc toàn bộ `NavHost` trong `SharedTransitionLayout` và sử dụng hai hàm mở rộng:
1. `sharedElement(rememberSharedContentState(key = "image_$id"), animatedVisibilityScope)`
2. `sharedBounds(...)`
Khi chuyển trang, Compose sẽ tự động tính toán vị trí và kích thước hình học trên cả 2 màn hình để phóng to/thu nhỏ mượt mà giữa các destinations.

#### Q3: Điều gì xảy ra với Coroutine của `Animatable` khi Composable bị gỡ khỏi giao diện?
**Trả lời chuẩn bản chất:**
`Animatable` sử dụng các suspend functions (`snapTo`, `animateTo`). Khi Composable bị gỡ khỏi cây Composition (Leave Composition), CoroutineScope quản lý animation đó (thường là `LaunchedEffect`) sẽ tự động nhận tín hiệu `CancellationException`. Animation Engine sẽ dừng ngay lập tức tại frame hiện tại và giải phóng MonotonicFrameClock.

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Giật lag, sụt giảm khung hình nghiêm trọng khi chạy animation** | Đọc Animation State trong thân hàm Composable làm kích hoạt Recomposition liên tục 120fps. | Chuyển sang sử dụng các modifier dạng lambda như `Modifier.graphicsLayer { ... }`. |
| **Animation không hiển thị chuyển động mà nhảy cóc (snap) tức thì** | Cung cấp cùng một `key` hoặc gán nhầm giá trị trực tiếp thay vì bọc trong `animate*AsState`. | Đảm bảo targetValue thay đổi qua một State phản ứng. |
| **Unit Test bị treo (Test Timeout) không bao giờ dừng** | Chạy `rememberInfiniteTransition` trong Composable Test mà không tạm dừng `mainClock.autoAdvance = false`. | Gọi `composeTestRule.mainClock.autoAdvance = false` trước khi render hoặc không test infinite transition trực tiếp. |

---

*Bài trước: [14 — Compose Navigation: Type-Safe, Deep Links & Back Stack](14-navigation.md)*  
*Bài tiếp theo: [16 — Performance & Lazy Layouts: LazyColumn, Keys, Baseline Profiles](16-performance-lazy-layouts.md)*

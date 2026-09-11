# Bài 16 — Performance & Lazy Layouts: LazyColumn, Keys & Baseline Profiles

> **Module:** 4 — Jetpack Compose Advanced  
> **Prerequisite:** [Bài 15 — Compose Animation: animate*AsState, AnimatedVisibility](15-animation.md)  
> **Official Docs:**
> - [Lazy layouts in Compose — Android Developers](https://developer.android.com/develop/ui/compose/lists)
> - [Improve Compose performance](https://developer.android.com/develop/ui/compose/performance)
> - [Baseline Profiles overview](https://developer.android.com/topic/performance/baselineprofiles/overview)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong các ứng dụng di động thực tế, việc hiển thị các danh sách lớn (hàng trăm đến hàng chục nghìn phần tử) là yêu cầu phổ biến nhất. Để đảm bảo ứng dụng không bị sập vì tràn bộ nhớ (**OutOfMemoryError**) và luôn duy trì tốc độ khung hình chuẩn **60fps / 120fps**, Compose cung cấp hệ sinh thái **Lazy Layouts**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        LAZY LAYOUTS ECOSYSTEM                          │
│                                                                        │
│   1. Layout Components:                                                │
│      - LazyColumn (Tương đương RecyclerView dọc)                       │
│      - LazyRow (Tương đương RecyclerView ngang)                        │
│      - LazyVerticalGrid / LazyHorizontalGrid (Dạng lưới chia cột)      │
│      - LazyVerticalStaggeredGrid (Dạng lưới so le soắn ốc - Pinterest) │
│                                                                        │
│   2. Tối ưu hóa hiệu năng cấp mã nguồn (Source-level Optimization):     │
│      - itemKey: Định danh duy nhất và ổn định cho từng item            │
│      - contentType: Phân loại kiểu bố cục để tối ưu tái sử dụng slot   │
│      - Subcomposition: Trì hoãn việc đo đạc và compose đến khi cần     │
│                                                                        │
│   3. Tối ưu hóa hiệu năng cấp hệ thống (System-level Optimization):    │
│      - Baseline Profiles: Biên dịch AOT (Ahead-of-Time) mã máy CPU     │
│      - Android Macrobenchmark: Đo lường tốc độ khởi động và cuộn thực  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế Subcomposition & Item Prefetch

Tại sao `LazyColumn` lại "lười biếng" (Lazy) và tiết kiệm tài nguyên hơn một `Column` thông thường?

```
                 CƠ CHẾ SUBCOMPOSITION TRONG LAZY LAYOUT
                 
Màn hình thiết bị (Viewport): Chỉ nhìn thấy được Item 3, 4, 5
┌────────────────────────────────────────────────────────────────────────┐
│ [Item 1] ◄── ĐÃ CUỘN QUA: Bị đưa vào vùng DECOMPOSITION (Giải phóng)   │
│ [Item 2] ◄── ĐÃ CUỘN QUA: Bị đưa vào vùng DECOMPOSITION (Giải phóng)   │
├────────────────────────────────────────────────────────────────────────┤
│ === RANH GIỚI VIEWPORT HIỂN THỊ TRÊN MÀN HÌNH ===                      │
│ [Item 3] ──► Đang nằm trong Composition & Đang vẽ lên GPU              │
│ [Item 4] ──► Đang nằm trong Composition & Đang vẽ lên GPU              │
│ [Item 5] ──► Đang nằm trong Composition & Đang vẽ lên GPU              │
│ === RANH GIỚI VIEWPORT HIỂN THỊ TRÊN MÀN HÌNH ===                      │
├────────────────────────────────────────────────────────────────────────┤
│ [Item 6] ◄── PREFETCH SLOT: Compose âm thầm đo đạc trước ở luồng nền!  │
│ [Item 7..1000] ◄── CHƯA TỒN TẠI TRONG RAM! KHÔNG TỐN DÙ CHỈ 1 BYTE!    │
└────────────────────────────────────────────────────────────────────────┘
```

#### Phân tích cơ chế:
1. **Subcomposition:** `LazyColumn` không compose toàn bộ danh sách cùng một lúc. Nó trì hoãn việc thực thi lambda của item con cho đến khi item đó thực sự chạm vào ranh giới đo đạc (**Measure Phase**).
2. **Prefetching:** Trong khi người dùng đang lướt ngón tay, Compose Engine sẽ tận dụng các khoảng thời gian rảnh rỗi giữa các khung hình (Idle frame time) để **đo đạc trước 1-2 items tiếp theo**. Khi ngón tay cuộn tới, item đó đã sẵn sàng để vẽ ngay tức khắc mà không bị khựng (Zero jank)!

---

### 2.2 So sánh cơ chế tái sử dụng: `RecyclerView.ViewHolder` vs Compose `LazyColumn`

Nhiều lập trình viên lầm tưởng `LazyColumn` tái sử dụng View giống như `RecyclerView`:

| Tiêu chí | `RecyclerView` (View System) | `LazyColumn` (Jetpack Compose) |
|---|---|---|
| **Cơ chế cốt lõi** | **View Recycling:** Giữ các đối tượng `View` trong một `RecycledViewPool`. Khi cuộn, gọi `onBindViewHolder()` để gán lại dữ liệu mới vào View cũ. | **Slot Re-use / Recomposition:** Tái sử dụng các vùng bộ nhớ trong **Slot Table**. Không có View object nào được tái sử dụng; chỉ có cấu trúc dữ liệu phẳng được hoán đổi. |
| **Độ phức tạp** | Phải viết `Adapter`, `ViewHolder`, `DiffUtil.ItemCallback`, XML layout. | Viết trực tiếp dưới dạng Composable declarative function. |
| **Đóng góp của `contentType`** | Tương đương với `getItemViewType()` trong Adapter. | Giúp Compose biết hai item có cùng cấu trúc Slot hay không để tái sử dụng Slot Table mà không cần tái cấu trúc nhóm (Group). |

---

### 2.3 Bản chất của Baseline Profiles: Xóa bỏ hoàn toàn JIT Overhead

Tại sao ứng dụng Jetpack Compose đôi khi bị giật lag nhẹ ở lần đầu tiên mở app, nhưng sau đó lại mượt mà?

```
CƠ CHẾ BIÊN DỊCH ỨNG DỤNG ANDROID (AOT VS JIT):

1. Mặc định (Just-In-Time - JIT):
   Tải file APK từ Play Store ──► Chứa mã Bytecode trung gian
          │
          ▼
   Người dùng mở app lần đầu ──► CPU vừa chạy vừa phải "thông dịch" (JIT Interpretation)
          │
          ▼ GÂY GIẬT LAG KHỞI ĐỘNG (COLD START) VÀ DROP FRAME KHI CUỘN LẦN ĐẦU!


2. Có Baseline Profiles (Ahead-Of-Time - AOT):
   Google Play Store tải app + Baseline Profile file
          │
          ▼
   Hệ điều hành Android (ART) BIÊN DỊCH SẴN TOÀN BỘ MÃ COMPOSE THÀNH MÃ MÁY NATIVE!
          │
          ▼
   Người dùng mở app ──► CPU CHẠY TRỰC TIẾP MÃ MÁY VỚI TỐC ĐỘ TỐI ĐA!
   => TĂNG TỐC 40% KHỞI ĐỘNG ỨNG DỤNG!
   => TRIỆT TIÊU 90% JANK FRAME KHI CUỘN LAZYCOLUMN!
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi đau khi thiếu Stable Keys trong danh sách

Khi bạn không cung cấp `key` cho `LazyColumn`:
```kotlin
// SAI LẦM: Không cung cấp key
LazyColumn {
    items(items) { item ->
        MovieCard(item)
    }
}
```

#### Hậu quả nghiêm trọng:
1. Mặc định Compose sẽ sử dụng **chỉ số vị trí (Index 0, 1, 2...)** làm định danh duy nhất cho item.
2. Khi người dùng xóa item ở vị trí số 0 hoặc chèn một item mới vào đầu danh sách, toàn bộ các item phía sau đều bị đổi index $\rightarrow$ **TẤT CẢ CÁC ITEM CÒN LẠI ĐỀU BỊ RECOMPOSE LẠI TỪ ĐẦU!**
3. Nếu bên trong item có chứa State nội bộ (như checkbox được chọn, text trong TextField), state đó sẽ **bị gán nhầm sang cho item khác**!
4. **Giải pháp:** Cung cấp định danh duy nhất `key = { it.id }`. Khi danh sách thay đổi vị trí, Compose chỉ việc di chuyển node tương ứng trong cây UI mà không cần vẽ lại nội dung!

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Quy tắc kiến trúc vàng cho Lazy Layouts

#### 1. Luôn cung cấp `key` và `contentType` cho danh sách phức tạp
```kotlin
LazyColumn {
    items(
        items = feedList,
        key = { feed -> feed.id },            // Stable Identity
        contentType = { feed -> feed.itemType } // Layout Classification
    ) { feed ->
        when (feed) {
            is FeedItem.Header -> HeaderView(feed)
            is FeedItem.Post -> PostView(feed)
            is FeedItem.Ad -> SponsoredAdView(feed)
        }
    }
}
```

#### 2. Tuyệt đối không lồng `LazyColumn` trong `Column` có `verticalScroll`
```kotlin
// CRASH RUNTIME HOẶC MẤT TÍNH NĂNG LAZY:
Column(modifier = Modifier.verticalScroll(rememberScrollState())) {
    LazyColumn { ... } // LỖI: Chiều cao tối đa bị đo là vô hạn (Infinity Constraints)!
}

// ĐÚNG: Sử dụng trực tiếp các hàm item {} bên trong 1 LazyColumn duy nhất!
LazyColumn {
    item { HeaderSection() }
    items(products) { ProductRow(it) }
    item { FooterSection() }
}
```

#### 3. Tối ưu hóa Scroll State với `derivedStateOf`
Khi cần ẩn/hiện nút "Back to Top" hoặc ẩn thanh BottomBar khi người dùng cuộn danh sách, **luôn bọc điều kiện trong `derivedStateOf`** để tránh recompose 120 lần/giây theo từng pixel cuộn!

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng **Mạng xã hội tin tức tốc độ cao (High-Performance Social Feed)**:
- Hỗ trợ đa dạng loại nội dung bằng `contentType`.
- Sử dụng Sticky Header hiển thị theo ngày.
- Ẩn/hiện Floating Action Button thông minh với `derivedStateOf`.
- Hướng dẫn thiết lập **Baseline Profile Generator Module** trong Gradle.

### Bước 1: Khai báo Data Models & Sealed Interface Phân Loại

```kotlin
sealed interface FeedItem {
    val id: String

    data class TextPost(
        override val id: String,
        val author: String,
        val text: String,
        val timestamp: String
    ) : FeedItem

    data class ImagePost(
        override val id: String,
        val author: String,
        val imageUrl: String,
        val caption: String
    ) : FeedItem

    data class SponsoredAd(
        override val id: String,
        val advertiser: String,
        val callToActionUrl: String
    ) : FeedItem
}
```

---

### Bước 2: Triển khai `HighPerformanceFeedScreen` với Sticky Header

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun HighPerformanceFeedScreen(
    feedItems: List<FeedItem>,
    modifier: Modifier = Modifier
) {
    val listState = rememberLazyListState()

    // TỐI ƯU HÓA: Chỉ Recompose nút FAB khi vượt ngưỡng hiển thị, không Recompose theo từng pixel cuộn!
    val showScrollToTopButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 3 }
    }

    val coroutineScope = rememberCoroutineScope()

    Scaffold(
        floatingActionButton = {
            AnimatedVisibility(
                visible = showScrollToTopButton,
                enter = fadeIn() + scaleIn(),
                exit = fadeOut() + scaleOut()
            ) {
                FloatingActionButton(
                    onClick = {
                        coroutineScope.launch {
                            listState.animateScrollToItem(0)
                        }
                    }
                ) {
                    Icon(Icons.Default.KeyboardArrowUp, contentDescription = "Cuộn lên đầu")
                }
            }
        }
    ) { padding ->
        LazyColumn(
            state = listState,
            modifier = modifier
                .fillMaxSize()
                .padding(padding),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            // 1. Sticky Header hiển thị cố định ở đầu trang khi cuộn
            stickyHeader {
                Surface(
                    color = MaterialTheme.colorScheme.surfaceVariant,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text(
                        text = "Bản tin hôm nay",
                        style = MaterialTheme.typography.titleSmall,
                        fontWeight = FontWeight.Bold,
                        modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp)
                    )
                }
            }

            // 2. Danh sách đa hình thức (Polymorphic List)
            items(
                items = feedItems,
                key = { item -> item.id }, // BẮT BUỘC: Stable Unique Key
                contentType = { item ->     // BẮT BUỘC: Phân loại để tối ưu tái sử dụng Slot Table
                    when (item) {
                        is FeedItem.TextPost -> "TEXT_POST"
                        is FeedItem.ImagePost -> "IMAGE_POST"
                        is FeedItem.SponsoredAd -> "SPONSORED_AD"
                    }
                }
            ) { item ->
                when (item) {
                    is FeedItem.TextPost -> TextPostCard(item)
                    is FeedItem.ImagePost -> ImagePostCard(item)
                    is FeedItem.SponsoredAd -> SponsoredAdCard(item)
                }
            }
        }
    }
}
```

---

### Bước 3: Triển khai các Composable Cards tối ưu

```kotlin
@Composable
fun TextPostCard(post: FeedItem.TextPost) {
    Card(modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = post.author, fontWeight = FontWeight.Bold)
            Spacer(modifier = Modifier.height(4.dp))
            Text(text = post.text)
            Spacer(modifier = Modifier.height(6.dp))
            Text(text = post.timestamp, style = MaterialTheme.typography.labelSmall)
        }
    }
}

@Composable
fun ImagePostCard(post: FeedItem.ImagePost) {
    Card(modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = post.author, fontWeight = FontWeight.Bold)
            Spacer(modifier = Modifier.height(8.dp))
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp)
                    .background(Color.LightGray, RoundedCornerShape(8.dp)),
                contentAlignment = Alignment.Center
            ) {
                Text("Ảnh: ${post.imageUrl}")
            }
            Spacer(modifier = Modifier.height(6.dp))
            Text(text = post.caption)
        }
    }
}

@Composable
fun SponsoredAdCard(ad: FeedItem.SponsoredAd) {
    Card(
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.tertiaryContainer),
        modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp).fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column {
                Text(text = "Được tài trợ", style = MaterialTheme.typography.labelSmall)
                Text(text = ad.advertiser, fontWeight = FontWeight.Bold)
            }
            Button(onClick = {}) { Text("Tìm hiểu thêm") }
        }
    }
}
```

---

### Bước 4: Hướng dẫn cấu hình Baseline Profile Generator (Macrobenchmark)

Tạo một Module mới trong Android Studio: **`benchmark`**:

```kotlin
// benchmark/src/main/java/com/example/benchmark/BaselineProfileGenerator.kt
@RunWith(AndroidJUnit4::class)
@LargeTest
class BaselineProfileGenerator {

    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() {
        rule.collect(
            packageName = "com.example.myapp",
            profileBlock = {
                // 1. Kịch bản khởi động app
                startActivityAndWait()

                // 2. Kịch bản cuộn danh sách mạng xã hội
                val feedList = device.findObject(By.scrollable(true))
                feedList?.let {
                    it.setGestureMargin(device.displayWidth / 5)
                    it.fling(Direction.DOWN)
                    device.waitForIdle()
                    it.fling(Direction.UP)
                }
            }
        )
    }
}
```
Chạy lệnh `./gradlew :app:generateBaselineProfile` trên thiết bị vật lý. Android Gradle Plugin sẽ tự động đóng gói file `baseline-prof.txt` vào bộ cài APK Release của bạn!

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Tại sao `contentType` lại tạo ra sự khác biệt lớn về hiệu năng trong `LazyColumn` có nhiều loại item?
**Trả lời chuẩn bản chất:**
Mỗi loại Composable (ví dụ: TextCard có 3 Text nodes vs VideoCard có 1 Surface + VideoPlayer controls) có cấu trúc **Slot Table hoàn toàn khác nhau**. 
- Nếu không cung cấp `contentType`, khi cuộn từ TextCard sang VideoCard, Compose sẽ cố gắng tái sử dụng slot của TextCard. Do cấu trúc không tương thích, Compose buộc phải **hủy bỏ nhóm cũ và khởi tạo lại toàn bộ Slot Table mới (Heavy Re-initialization)**.
- Khi cung cấp `contentType`, Compose phân tách các nhóm cấu trúc tương đồng vào các pool riêng biệt, giúp tái sử dụng nguyên vẹn cấu trúc Slot Table có sẵn và chỉ việc thay thế nội dung, tiết kiệm đáng kể chu kỳ CPU!

#### Q2: Sự khác biệt giữa `firstVisibleItemIndex` và `firstVisibleItemScrollOffset` trong `LazyListState`?
**Trả lời chuẩn bản chất:**
- **`firstVisibleItemIndex`**: Là chỉ số vị trí (Position) của phần tử đầu tiên đang nhìn thấy trong viewport (0, 1, 2...). Giá trị này chỉ thay đổi khi một phần tử hoàn toàn bị cuộn ra khỏi mép trên màn hình.
- **`firstVisibleItemScrollOffset`**: Là khoảng cách cuộn tính bằng điểm ảnh vật lý (Pixels) của phần tử đầu tiên đó. Giá trị này biến động liên tục ở **từng pixel cuộn** (1px, 2px, 3px...). Nếu bạn chỉ cần kiểm tra xem người dùng đã cuộn qua trang đầu hay chưa, luôn đọc `firstVisibleItemIndex` qua `derivedStateOf` để tránh recompose liên tục theo từng pixel!

#### Q3: Có nên dùng `LazyColumn` cho danh sách cố định chỉ có 5-10 phần tử không?
**Trả lời chuẩn bản chất:**
**Không nên.** `LazyColumn` đi kèm với chi phí quản lý Subcomposition, Prefetcher, và theo dõi ranh giới Viewport. Đối với các danh sách ngắn, cố định (dưới 15-20 items không biến động), việc sử dụng một `Column` thông thường kết hợp vòng lặp `forEach` đơn giản sẽ có tốc độ thực thi nhanh hơn và tiêu hao ít bộ nhớ hơn rất nhiều!

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Crash: `Vertically scrollable component was measured with an infinity maximum height constraints`** | Đặt `LazyColumn` bên trong một `Column(modifier = Modifier.verticalScroll())`. | Xóa `verticalScroll` ở Column cha, hoặc chuyển toàn bộ nội dung thành các `item {}` bên trong `LazyColumn`. |
| **Trạng thái Checkbox bị nhảy sang item khác khi cuộn** | Thiếu tham số `key` trong hàm `items()`. | Bắt buộc khai báo `key = { it.id }` duy nhất cho từng phần tử. |
| **Giao diện bị giật cục mỗi khi cuộn nhanh** | Tạo mới lambda hoặc khởi tạo format ngày tháng (`SimpleDateFormat`) bên trong thân hàm của từng item. | Đưa logic format ra ngoài ViewModel hoặc bọc bằng `remember`. |

---

*Bài trước: [15 — Compose Animation: animate*AsState, AnimatedVisibility](15-animation.md)*  
*Bài tiếp theo: [17 — Compose Testing: ComposeTestRule, Semantics, UI Tests](17-compose-testing.md)*

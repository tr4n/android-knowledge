# Bài 08 — Repository Pattern với Flow: Offline-First Architecture

> **Module:** 2 — Architecture & ViewModel  
> **Prerequisite:** [Bài 07 — Domain Layer & Use Cases với Flow](07-domain-layer-usecases.md)  
> **Official Docs:**
> - [Data Layer — Android Developers](https://developer.android.com/topic/architecture/data-layer)
> - [Offline-first architecture guide](https://developer.android.com/topic/architecture/data-layer/offline-first)
> - [Room with Kotlin Flow](https://developer.android.com/training/data-storage/room/async-queries#kotlin-flow)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong kiến trúc ứng dụng Android hiện đại, **Data Layer (Tầng dữ liệu)** chịu trách nhiệm quản lý việc lưu trữ, truy xuất và đồng bộ hóa thông tin từ nhiều nguồn khác nhau. Trọng tâm của tầng này là **Repository Pattern**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER ARCHITECTURE                         │
│                                                                        │
│                      ┌────────────────────────┐                        │
│                      │  UI / DOMAIN LAYER     │                        │
│                      └───────────┬────────────┘                        │
│                                  │ Gọi suspend fun / Thu thập Flow     │
│                                  ▼                                     │
│                      ┌────────────────────────┐                        │
│                      │   REPOSITORY (SSOT)    │                        │
│                      │ (Single Source of Truth)│                       │
│                      └─────┬────────────┬─────┘                        │
│                            │            │                              │
│       Đọc/Ghi Dữ liệu      │            │ Đồng bộ dữ liệu              │
│                            ▼            ▼                              │
│             ┌──────────────────┐    ┌──────────────────┐               │
│             │ Local DataSource │    │ Remote DataSource│               │
│             │ (Room / SQLite)  │    │ (Retrofit / Ktor)│               │
│             └──────────────────┘    └──────────────────┘               │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Khái niệm Single Source of Truth (SSOT - Nguồn chân lý duy nhất)
- **SSOT** là nguyên lý thiết kế quy định rằng: **Đối với bất kỳ tập dữ liệu nào trong ứng dụng, chỉ có duy nhất MỘT nguồn lưu trữ được coi là chân lý chính xác nhất**.
- Trong kiến trúc **Offline-First**, **Cơ sở dữ liệu cục bộ (Local Database - Room)** được chỉ định làm SSOT. Mọi thành phần UI chỉ được phép quan sát và đọc dữ liệu trực tiếp từ Local DB, không bao giờ đọc dữ liệu sống phát thẳng từ API!

---

### 1.2 Kiến trúc Offline-First (Local-First) là gì?
- **Offline-First** là triết lý xây dựng ứng dụng sao cho **toàn bộ tính năng cốt lõi có thể hoạt động mượt mà ngay cả khi không có kết nối Internet**.
- Ứng dụng sẽ hiển thị dữ liệu đã lưu trong máy ngay lập tức (Zero-latency startup), sau đó tiến hành đồng bộ dữ liệu mới từ Server ở chế độ nền (Background Sync) và âm thầm cập nhật UI khi có mạng.

---

### 1.3 Phân tách Data Mappers (Entity $\leftrightarrow$ DTO $\leftrightarrow$ Domain Model)

Để đảm bảo tính độc lập giữa các tầng, không bao giờ để lọt đối tượng tầng Data lên UI:

| Đối tượng | Tầng thuộc về | Trách nhiệm |
|---|---|---|
| **DTO (Data Transfer Object)** | Remote DataSource | Ánh xạ trực tiếp với cấu trúc JSON trả về từ API Backend (ví dụ: `NewsArticleDto`). |
| **Entity** | Local DataSource | Ánh xạ trực tiếp với bảng trong SQLite Room Database (ví dụ: `NewsArticleEntity`). |
| **Domain Model** | Domain & UI Layer | Đối tượng thuần Kotlin phục vụ logic nghiệp vụ và hiển thị UI (ví dụ: `NewsArticle`). |

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Reactive Sync Pipeline: Cơ chế Invalidation Tracker của Room

Làm thế nào mà Room Database có thể phát ra dữ liệu mới qua `Flow` mỗi khi có thay đổi trong cơ sở dữ liệu?

```
                 CƠ CHẾ REACTIVE SYNC PIPELINE DƯỚI TẦNG THẤP

1. UI/ViewModel bắt đầu collect Flow:
   NewsRepository.getArticles() ──► Room DAO.getArticles(): Flow<List<NewsArticleEntity>>
                                               │
2. Room trả về dữ liệu đang có trong DB       │
   và đăng ký InvalidationTracker trên bảng    │
                                               ▼
                                  ┌─────────────────────────┐
                                  │ UI hiển thị tức thì     │
                                  │ (Dữ liệu Offline trong DB│
                                  └─────────────────────────┘
3. Network Sync chạy ngầm:
   NewsRepository.refreshArticles()
        │
        ├──► Gọi Retrofit API lấy dữ liệu mới nhất
        │
        └──► Ghi đè vào Room: newsDao.upsertArticles(newItems)
                                               │
4. Kích hoạt Room InvalidationTracker:         │
   - SQLite phát tín hiệu table_changed! ◄─────┘
   - Room tự động truy vấn lại: "SELECT * FROM news_articles"
   - Room EMIT BỘ DỮ LIỆU MỚI QUA FLOW!
                                               │
5. UI tự động cập nhật mượt mà! ◄──────────────┘
```

#### Phân tích cơ chế:
1. Bạn **không cần phải viết code thủ công** để cập nhật UI sau khi gọi API xong.
2. Bạn chỉ cần làm đúng một việc: **Ghi dữ liệu mới vào Room DB**.
3. Hệ thống `InvalidationTracker` của Room sử dụng cơ chế SQLite Triggers để giám sát bảng dữ liệu. Bất kỳ thao tác `INSERT`, `UPDATE` hoặc `DELETE` nào diễn ra, Room sẽ tự động đánh thức coroutine đang collect và phát ra danh sách mới nhất!

---

### 2.2 Chiến lược Xử lý Xung đột Dữ liệu (Conflict Resolution)

Khi ứng dụng hoạt động offline và người dùng thực hiện sửa đổi cục bộ, sau đó kết nối mạng trở lại, việc xung đột dữ liệu giữa Client và Server là không thể tránh khỏi.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      CONFLICT RESOLUTION STRATEGIES                    │
│                                                                        │
│   1. Server-Wins (Mặc định cho News / Feed / Social Media):            │
│      - Dữ liệu từ Server luôn đè lên dữ liệu cục bộ.                   │
│      - Sử dụng OnConflictStrategy.REPLACE trong Room DAO.              │
│                                                                        │
│   2. Client-Wins (Offline Task / Note Taking Apps):                    │
│      - Các thay đổi chưa đồng bộ của người dùng được ưu tiên giữ lại.  │
│                                                                        │
│   3. Last-Write-Wins (Dựa trên Timestamp):                            │
│      - So sánh updatedAt giữa Client và Server:                        │
│        if (clientItem.updatedAt > serverItem.updatedAt) keepClient()   │
│        else updateFromServer()                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 2.3 Chiến lược Cập nhật Lạc quan (Optimistic Updates)

Đối với các ứng dụng có tính tương tác cao (như Like bài viết, Bookmark, Đổi tên Profile), việc chờ đợi mạng phản hồi (300-1000ms) sẽ khiến giao diện có cảm giác ì ạch và chậm chạp.

```
                      MÔ HÌNH OPTIMISTIC UPDATES
                      
User bấm nút "Like"
        │
        ├──► 1. Cập nhật ngay lập tức vào Local DB: isLiked = true!
        │       => UI đổi icon sang màu đỏ NGAY TỨC KHẮC (0ms delay)!
        │
        └──► 2. Bắn Request mạng lên Server ở chế độ nền...
                │
                ├── THÀNH CÔNG: Server xác nhận -> Xong.
                │
                └── THẤT BẠI (Mất mạng / 500):
                    => ROLLBACK LOCAL DB: isLiked = false!
                    => UI tự động đổi lại màu cũ!
                    => Bắn UiEffect thông báo: "Không thể thực hiện, vui lòng thử lại".
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi ám ảnh của ứng dụng "Online-Only" (Không có Offline-First)

| Tình huống thực tế | Ứng dụng Online-Only | Ứng dụng Offline-First (SSOT) |
|---|---|---|
| **Người dùng ở trong thang máy / mất sóng** | Màn hình trắng xóa, xoay vòng loading vô tận hoặc văng hộp thoại "Network Error". | Mở app lên thấy ngay tin tức/dữ liệu cũ đã lưu, trải nghiệm đọc liền mạch. |
| **Tốc độ khởi động ứng dụng** | Phải chờ gọi API (1-2 giây) mới vẽ được giao diện. | Đọc từ SQLite DB nội bộ trong **10ms**, vẽ giao diện tức thì. |
| **Đồng bộ giữa nhiều màn hình** | Sửa thông tin ở màn hình Chi tiết, bấm Back về màn hình Danh sách thì dữ liệu vẫn là giá trị cũ! | Sửa 1 lần vào DB $\rightarrow$ Toàn bộ màn hình đang mở tự động cập nhật ngay lập tức qua Flow. |
| **Tiết kiệm dữ liệu mạng & Pin** | Mỗi lần mở lại màn hình là gọi lại toàn bộ API từ đầu. | Sử dụng Cache Invalidation (ETag / Last-Modified), chỉ tải những gì thực sự mới. |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Quy tắc kiến trúc Data Layer từ Google

1. **Repository KHÔNG BAO GIỜ để lộ DataSource:**
   - ViewModel không được biết `NewsDao` hay `NewsApiService` là gì.
   - ViewModel chỉ tương tác với interface `NewsRepository`.

2. **Hàm đọc trả về `Flow<T>`, hàm ghi là `suspend fun`:**
   ```kotlin
   interface NewsRepository {
       // Đọc dữ liệu: Luôn là Flow để phản ánh thay đổi liên tục
       fun getArticlesStream(): Flow<List<NewsArticle>>
       
       // Ghi / Đồng bộ: Luôn là suspend fun một lần
       suspend fun refreshArticles(): Result<Unit>
       suspend fun toggleBookmark(articleId: String): Result<Unit>
   }
   ```

3. **Chỉ định `Dispatchers.IO` bên trong Repository:**
   - Dù Room và Retrofit đều đã hỗ trợ Main-Safe bên dưới, Repository vẫn nên đảm bảo toàn bộ quá trình biến đổi dữ liệu (Mapping DTO $\rightarrow$ Entity) được thực thi an toàn trên luồng nền.

---

### 4.2 Cạm bẫy phổ biến (Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Nuốt chửng lỗi mạng trong Repository
Nhiều lập trình viên bắt lỗi mạng và trả về `emptyList()`:
```kotlin
// ANTI-PATTERN:
suspend fun refreshNews() {
    try {
        val data = api.getNews()
        dao.insert(data)
    } catch (e: Exception) {
        // Nuốt lỗi im lặng! UI không biết là có lỗi để hiển thị thông báo cho user!
    }
}

// ĐÚNG: Trả về Result<Unit> hoặc ném Exception để ViewModel bắt bằng Result / Catch
suspend fun refreshNews(): Result<Unit> = runCatching {
    val networkData = api.getNews()
    dao.upsertAll(networkData.map { it.toEntity() })
}
```

#### Cạm bẫy 2: Vòng lặp vô tận do Trigger lẫn nhau
Nếu bạn viết một hàm mà khi `Flow` phát ra dữ liệu mới $\rightarrow$ ViewModel lại tự động gọi `refresh()` $\rightarrow$ `refresh()` ghi vào DB $\rightarrow$ DB phát ra Flow $\rightarrow$ Lại gọi `refresh()` $\rightarrow$ **Vòng lặp vô tận làm nóng máy và tốn pin!**
> **Quy tắc:** Chỉ gọi `refresh()` khi có sự kiện rõ ràng (Mở màn hình lần đầu, người dùng kéo Pull-to-Refresh, hoặc WorkManager chạy định kỳ).

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng tính năng **Bảng tin Tin tức Offline-First (News Feed)** hoàn chỉnh:
- Lưu trữ cục bộ với **Room Database**.
- Đồng bộ dữ liệu mới từ **Retrofit API**.
- Hỗ trợ **Pull-to-Refresh** và hiển thị **Banner Offline** trên **Jetpack Compose**.

### Bước 1: Khai báo Room Entities & DAO

```kotlin
// 1. Room Entity
@Entity(tableName = "articles")
data class NewsArticleEntity(
    @PrimaryKey val id: String,
    val title: String,
    val summary: String,
    val author: String,
    val publishedAt: Long,
    val isBookmarked: Boolean = false
)

// 2. Room DAO
@Dao
interface NewsArticleDao {

    // QUAN TRỌNG: Trả về Flow để tự động phát dữ liệu khi bảng thay đổi
    @Query("SELECT * FROM articles ORDER BY publishedAt DESC")
    fun getArticlesStream(): Flow<List<NewsArticleEntity>>

    @Upsert
    suspend fun upsertArticles(articles: List<NewsArticleEntity>)

    @Query("UPDATE articles SET isBookmarked = :isBookmarked WHERE id = :id")
    suspend fun updateBookmark(id: String, isBookmarked: Boolean)
}
```

---

### Bước 2: Remote DataSource DTO & Retrofit Service

```kotlin
// 1. DTO nhận từ Backend API
data class NewsArticleDto(
    @SerializedName("article_id") val articleId: String,
    @SerializedName("headline") val headline: String,
    @SerializedName("description") val description: String,
    @SerializedName("author_name") val authorName: String,
    @SerializedName("timestamp") val timestamp: Long
)

// 2. Retrofit Interface
interface NewsApiService {
    @GET("v1/news/latest")
    suspend fun getLatestNews(): List<NewsArticleDto>
}

// 3. Domain Model
data class NewsArticle(
    val id: String,
    val title: String,
    val summary: String,
    val author: String,
    val isBookmarked: Boolean
)

// Data Mappers
fun NewsArticleDto.toEntity() = NewsArticleEntity(
    id = articleId,
    title = headline,
    summary = description,
    author = authorName,
    publishedAt = timestamp
)

fun NewsArticleEntity.toDomain() = NewsArticle(
    id = id,
    title = title,
    summary = summary,
    author = author,
    isBookmarked = isBookmarked
)
```

---

### Bước 3: Triển khai `NewsRepository` (Offline-First SSOT)

```kotlin
interface NewsRepository {
    fun getArticles(): Flow<List<NewsArticle>>
    suspend fun refreshArticles(): Result<Unit>
    suspend fun toggleBookmark(articleId: String, currentBookmarked: Boolean): Result<Unit>
}

class NewsRepositoryImpl(
    private val newsDao: NewsArticleDao,
    private val newsApi: NewsApiService,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : NewsRepository {

    // CHÂN LÝ DUY NHẤT: Chỉ đọc từ Room DB, map sang Domain Model
    override fun getArticles(): Flow<List<NewsArticle>> {
        return newsDao.getArticlesStream()
            .map { entities -> entities.map { it.toDomain() } }
            .flowOn(ioDispatcher)
    }

    // ĐỒNG BỘ: Gọi API -> Lưu vào DB (Không trả dữ liệu trực tiếp cho UI!)
    override suspend fun refreshArticles(): Result<Unit> = withContext(ioDispatcher) {
        runCatching {
            val remoteDtos = newsApi.getLatestNews()
            val entities = remoteDtos.map { it.toEntity() }
            newsDao.upsertArticles(entities)
        }
    }

    // OPTIMISTIC UPDATE: Cập nhật DB trước
    override suspend fun toggleBookmark(articleId: String, currentBookmarked: Boolean): Result<Unit> = withContext(ioDispatcher) {
        runCatching {
            val newStatus = !currentBookmarked
            // Cập nhật DB ngay tức khắc
            newsDao.updateBookmark(articleId, newStatus)
            // Giả lập gọi API đồng bộ lên server (nếu lỗi có thể rollback)
        }
    }
}
```

---

### Bước 4: ViewModel Layer (Quản lý Sync State & Pull-to-Refresh)

```kotlin
data class NewsFeedUiState(
    val articles: List<NewsArticle> = emptyList(),
    val isRefreshing: Boolean = false,
    val errorMessage: String? = null
)

class NewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    private val _isRefreshing = MutableStateFlow(false)
    private val _errorMessage = MutableStateFlow<String?>(null)

    // Gộp dữ liệu từ Local DB và trạng thái Refresh
    val uiState: StateFlow<NewsFeedUiState> = combine(
        newsRepository.getArticles(),
        _isRefreshing,
        _errorMessage
    ) { articles, isRefreshing, error ->
        NewsFeedUiState(
            articles = articles,
            isRefreshing = isRefreshing,
            errorMessage = error
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = NewsFeedUiState()
    )

    init {
        // Tự động làm mới khi khởi tạo
        refreshNews()
    }

    fun refreshNews() {
        viewModelScope.launch {
            _isRefreshing.value = true
            _errorMessage.value = null

            val result = newsRepository.refreshArticles()
            result.onFailure { error ->
                _errorMessage.value = error.localizedMessage ?: "Không có kết nối mạng"
            }
            _isRefreshing.value = false
        }
    }

    fun onBookmarkClicked(article: NewsArticle) {
        viewModelScope.launch {
            newsRepository.toggleBookmark(article.id, article.isBookmarked)
        }
    }
}
```

---

### Bước 5: UI Layer (Jetpack Compose với Pull-to-Refresh)

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun NewsFeedScreen(
    viewModel: NewsViewModel,
    modifier: Modifier = Modifier
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val pullRefreshState = rememberPullToRefreshState()

    Scaffold(
        topBar = { TopAppBar(title = { Text("Tin tức hàng ngày (Offline-First)") }) }
    ) { padding ->
        PullToRefreshBox(
            isRefreshing = uiState.isRefreshing,
            onRefresh = { viewModel.refreshNews() },
            modifier = modifier.fillMaxSize().padding(padding)
        ) {
            Column(modifier = Modifier.fillMaxSize()) {
                // Banner thông báo Offline nếu refresh gặp lỗi mạng
                if (uiState.errorMessage != null) {
                    Surface(
                        color = MaterialTheme.colorScheme.errorContainer,
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text(
                            text = "Đang xem dữ liệu offline: ${uiState.errorMessage}",
                            modifier = Modifier.padding(12.dp),
                            color = MaterialTheme.colorScheme.onErrorContainer,
                            style = MaterialTheme.typography.bodySmall
                        )
                    }
                }

                // Danh sách bài viết đọc từ Local DB
                LazyColumn(
                    modifier = Modifier.fillMaxSize().padding(16.dp),
                    verticalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    items(uiState.articles, key = { it.id }) { article ->
                        ArticleCard(
                            article = article,
                            onBookmarkToggle = { viewModel.onBookmarkClicked(article) }
                        )
                    }
                }
            }
        }
    }
}

@Composable
fun ArticleCard(article: NewsArticle, onBookmarkToggle: () -> Unit) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier = Modifier.padding(16.dp).fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(text = article.title, style = MaterialTheme.typography.titleMedium)
                Spacer(modifier = Modifier.height(4.dp))
                Text(text = article.summary, style = MaterialTheme.typography.bodyMedium)
                Spacer(modifier = Modifier.height(6.dp))
                Text(text = "Tác giả: ${article.author}", style = MaterialTheme.typography.labelSmall)
            }
            IconButton(onClick = onBookmarkToggle) {
                Icon(
                    imageVector = if (article.isBookmarked) Icons.Default.Favorite else Icons.Default.FavoriteBorder,
                    contentDescription = "Bookmark",
                    tint = if (article.isBookmarked) Color.Red else Color.Gray
                )
            }
        }
    }
}
```

---

### Bước 6: Viết Unit Test kiểm thử Reactive Sync Pipeline với `Turbine`

```kotlin
class NewsRepositoryTest {

    private val testDispatcher = StandardTestDispatcher()
    private val localDbEmitter = MutableStateFlow<List<NewsArticleEntity>>(emptyList())
    private lateinit var fakeDao: NewsArticleDao
    private lateinit var fakeApi: NewsApiService
    private lateinit var repository: NewsRepository

    @Before
    fun setUp() {
        fakeDao = object : NewsArticleDao {
            override fun getArticlesStream() = localDbEmitter
            override suspend fun upsertArticles(articles: List<NewsArticleEntity>) {
                localDbEmitter.value = articles // Giả lập Room phát data mới khi có upsert
            }
            override suspend fun updateBookmark(id: String, isBookmarked: Boolean) {
                localDbEmitter.value = localDbEmitter.value.map {
                    if (it.id == id) it.copy(isBookmarked = isBookmarked) else it
                }
            }
        }

        fakeApi = object : NewsApiService {
            override fun getLatestNews() = listOf(
                NewsArticleDto("1", "Kotlin 2.0 Released", "Mô tả", "JetBrains", 1000L)
            )
        }

        repository = NewsRepositoryImpl(fakeDao, fakeApi, testDispatcher)
    }

    @Test
    fun getArticles_emitsFromDb_andUpdatesAutomaticallyWhenApiRefreshes() = runTest(testDispatcher) {
        repository.getArticles().test {
            // Ban đầu DB rỗng
            assertEquals(emptyList<NewsArticle>(), awaitItem())

            // Khi gọi refresh, API lấy dữ liệu và lưu vào DB
            val syncResult = repository.refreshArticles()
            assertTrue(syncResult.isSuccess)

            // Room tự động phát ra dữ liệu mới cho collector đang chờ!
            val updatedList = awaitItem()
            assertEquals(1, updatedList.size)
            assertEquals("Kotlin 2.0 Released", updatedList.first().title)

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Làm thế nào để giải quyết vấn đề Cache Invalidation khi một bài viết bị xóa trên Server nhưng trong Local DB vẫn còn?
**Trả lời chuẩn bản chất:**
Có 3 chiến lược chính của Senior Architect:
1. **Sync Full Snapshot & Replace:** Khi đồng bộ, API trả về toàn bộ ID hiện có. Trong 1 transaction của Room, gọi `deleteNotIn(serverIds)` rồi `upsert(newArticles)`.
2. **Soft Deletes (Cờ `is_deleted`):** Server không xóa cứng mà đánh dấu `is_deleted = true` kèm `deleted_at`. Client tải cờ này về và xóa khỏi Room DB tương ứng.
3. **Cache Expiry TTL (Time-To-Live):** Lưu cột `last_synced_at` vào từng bản ghi. Nếu bản ghi không được server cập nhật sau 7 ngày, một background job sẽ tự động dọn dẹp (prune) khỏi SQLite.

#### Q2: Tại sao hàm truy vấn Room DAO trả về `Flow<List<T>>` nhưng hàm Insert/Update lại là `suspend fun`?
**Trả lời chuẩn bản chất:**
- **Hàm Đọc (`Flow`)** đại diện cho một **luồng quan sát dữ liệu phản ứng liên tục (Observable Stream)**. Bạn muốn UI tự động nhận dữ liệu mới bất cứ khi nào database biến động trong tương lai, do đó nó phải là `Flow`.
- **Hàm Ghi (`suspend fun`)** đại diện cho một **thao tác tác vụ đơn lẻ (One-shot Write Operation)**. Bạn chỉ muốn thực hiện việc chèn/cập nhật dữ liệu vào đĩa IO một lần duy nhất và nhận về kết quả (hoặc hoàn tất) mà không cần duy trì stream.

#### Q3: Điều gì xảy ra nếu mạng quá chậm và người dùng gọi Pull-to-Refresh liên tục 5 lần?
**Trả lời chuẩn bản chất:**
Nếu không xử lý, 5 coroutines mạng sẽ chạy song song, gây lãng phí băng thông và có thể xảy ra race condition ghi đè DB. Để giải quyết:
- Trong ViewModel, sử dụng một `Job?` tham chiếu. Nếu job cũ đang chạy (`job?.isActive == true`), ta bỏ qua (Drop) các yêu cầu refresh tiếp theo.
- Hoặc sử dụng cấu trúc `conflate()` / `Channel` để đảm bảo chỉ có tối đa 1 tiến trình đồng bộ diễn ra tại một thời điểm.

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Vòng lặp vô tận (Infinite Sync Loop)** | Lắng nghe Flow DB trong ViewModel và mỗi khi có emission lại kích hoạt hàm `refreshArticles()`. | Tách biệt hoàn toàn luồng Đọc và luồng Ghi. Chỉ gọi `refreshArticles()` khi có tác động từ người dùng hoặc timer. |
| **Crash: `SQLiteConstraintException: FOREIGN KEY constraint failed`** | Lưu bảng con khi bảng cha chưa được insert vào DB. | Sử dụng Transaction (`@Transaction`) trong Room DAO để đảm bảo tính toàn vẹn dữ liệu cha-con. |
| **Giao diện bị giật lag khi tải lượng lớn dữ liệu từ DB** | Truy vấn toàn bộ 10,000 bản ghi vào một `List` trong `Flow`. | Sử dụng thư viện **Paging 3** kết hợp Room để phân trang dữ liệu theo từng block 20-50 phần tử. |

---

*Bài trước: [07 — Domain Layer & Use Cases với Flow](07-domain-layer-usecases.md)*  
*Bài tiếp theo: [09 — Paging 3 + Flow](09-paging3-flow.md)*

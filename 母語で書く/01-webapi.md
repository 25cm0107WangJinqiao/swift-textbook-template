## 我要做一个什么样的app？

做一个搜索音乐的app，用户输入歌手或者歌曲名点击搜索后，界面显示搜索的信息，点击搜索结果中的某首歌，还能进入详情页，看到专辑封面、歌名、歌手、专辑名、价格。

---

## 第一步：搞清楚这个app要做什么

用户输入关键词，从iTunes API拿数据，展示列表，可以点进详情。

---

## 第二步：识别数据（数据建模）

这个app里"流动"的数据是从API返回的JSON里，每首歌有：歌名、歌手、专辑、封面图片URL、价格。

所以需要一个`Song`结构体来承载这些字段。API返回的是一个列表，所以还需要一个`SearchResponse`来包装它。

---

## 第三步：识别状态（状态管理）

界面会因为什么而变化？

三种状态驱动界面：
- 正在加载中 → 显示转圈
- 加载失败 → 显示错误
- 加载成功 → 显示列表

这三个状态放在ViewModel里统一管理。

---

## 第四步：拆分界面（UI分层）

界面由哪几块独立的部分组成？
- 搜索栏（输入框+按钮）
- 内容区（三种状态对应三种显示）
- 列表行（每首歌的缩略信息）
- 详情页（点进去看完整信息）
- 错误横幅（出错时显示）

每块独立成一个View，互不干扰。

---

## 第五步：串联逻辑（数据流）

用户点搜索 → ViewModel发网络请求 → 更新状态 → UI响应状态自动刷新。

> 这就是MVVM的本质：View只管显示，ViewModel只管数据和逻辑，两者通过状态绑定自动同步。

---

## 第一步：因为从API返回的JSON数据是一个列表，所以我要写：

1. 一个struct SearchResponse按类区分里面的数据来保存;
2. 里面需要包含两个字段：resultCount（歌曲数量）和results（每首歌的详细信息：歌名、歌手、专辑、价格等）;
3. 每首歌的唯一身份ID;

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }
```

4. 每首歌价格的处理逻辑（价格和货币单位都有值就显示，任意一个没有就显示"価格不明")

```swift
    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}
```

---

## 第二步：谁来具体干活呢？

View只负责显示，不负责发送网络请求，所以需要一个View Model来处理：
1. 处理网络数据
2. 发送网络请求
3. 处理错误

首先需要4个变量：搜索结果，用户输入的关键词，搜索状态变化，报错的错误信息

```swift
@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?
}
```

然后写出4种可能出错的错误类型：URL构建失败，无网络，JSON解析失败，无搜索结果

```swift
enum SearchError: LocalizedError {
    case invalidURL
    case networkError(Error)
    case decodingError(Error)
    case noResults

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "検索URLの作成に失敗しました"
        case .networkError(let error):
            return "通信エラー: \(error.localizedDescription)"
        case .decodingError:
            return "データの読み取りに失敗しました"
        case .noResults:
            return "検索結果が見つかりませんでした"
        }
    }
}
```

最后加载搜索逻辑：输入框是否为空 → 把输入编码成URL安全的字符串 → 构建URL → 发请求，拿数据，解码成SearchResponse → 根据结果更新状态

```swift
func searchMusic() async {
    guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

    guard let encodedText = searchText.addingPercentEncoding(
        withAllowedCharacters: .urlQueryAllowed
    ) else {
        errorMessage = SearchError.invalidURL.errorDescription
        return
    }

    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

    guard let url = URL(string: urlString) else {
        errorMessage = SearchError.invalidURL.errorDescription
        return
    }

    isLoading = true
    errorMessage = nil

    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)

        if response.results.isEmpty {
            errorMessage = SearchError.noResults.errorDescription
            songs = []
        } else {
            songs = response.results
        }
    } catch let error as DecodingError {
        errorMessage = SearchError.decodingError(error).errorDescription
        songs = []
    } catch {
        errorMessage = SearchError.networkError(error).errorDescription
        songs = []
    }

    isLoading = false
}
```

---

## 第三步：构建主界面ContentView

ContentView需要显示什么？
1. 搜索框
2. 内容区
3. 报错显示区

首先需要把ViewModel注入到View里，View通过它读取状态、调用方法。

```swift
struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }
}
```

搜索栏：
1. 按回车键和点按钮也能触发搜索按钮
2. 输入为空或正在加载时，按钮不可用
3. 点击搜索按钮，到结果加载显示出来，中间等待时间时，app还能操作

```swift
    private var searchBar: some View {
        HStack {
            TextField("アーティスト名を入力", text: $viewModel.searchText)
                .textFieldStyle(.roundedBorder)
                .onSubmit {
                    Task { await viewModel.searchMusic() }
                }

            Button("検索") {
                Task { await viewModel.searchMusic() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.searchText.isEmpty || viewModel.isLoading)
        }
        .padding()
    }
```

内容区：
1. 正在加载搜索结果时，显示转圈标志（提升用户体验）
2. 如果歌曲列表是空的，显示一个提示页面
3. 显示歌曲列表

```swift
    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
```

---

## 第四步：用户点击搜索结果后的更详细的信息：

SongRow：用户点击搜索后列表里每一行的样式定义，都包含：
1. 封面缩略图
2. 歌名
3. 歌手名
4. 价格

```swift
struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 8)
                    .fill(.gray.opacity(0.2))
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)
                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(song.priceText)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 4)
    }
}
```

---

## 第五步：用户点击SongRow后显示的界面

需要比SongRow封面更大，信息更完整，而且因为有些歌曲没有专辑名，需要用if let解包

```swift
struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    // .bold() 移除，保持代码原样，如果需要可以保留在代码块里，不受外部星号影响
                    // 由于原文里有 .bold()，代码块中正常展示
                    .bold()

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}
```

---

## 第六步：报错的处理逻辑

把报错的处理逻辑封装在`struct ErrorBanner: View {}`里，ContentView直接调用ErrorBanner就可以了

```swift
struct ErrorBanner: View {
    let message: String

    var body: some View {
        HStack {
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.yellow)
            Text(message)
                .font(.caption)
        }
        .padding(10)
        .frame(maxWidth: .infinity)
        .background(.red.opacity(0.1))
    }
}
```

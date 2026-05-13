# 第1章：WebAPIの基本

> 執筆者：王金橋
> 最終更新：2026-05-13

## この章で学ぶこと

この章では、ユーザーが入力したもう文字列をURLに変換してAPIからデータを取得し、そのデータをリスクとして表示します。

## どんなアプリを作るのか？

音楽を検索するアプリを作る。ユーザーがアーティスト名または曲名を入力して検索ボタンを押すと、検索結果が画面に表示される。検索結果の中から曲をタップすると詳細ページに遷移し、アルバムアートワーク・曲名・アーティスト名・アルバム名・価格を確認できる。

---

## ステップ1：このアプリが何をするのかを整理する

ユーザーがキーワードを入力し、iTunes APIからデータを取得し、リストに表示し、詳細ページへ遷移できる。

---

## ステップ2：データを特定する（データモデリング）

このアプリで「流れる」データは、APIが返すJSONに含まれる。各楽曲には、曲名・アーティスト名・アルバム名・アートワーク画像のURL・価格がある。

そのため、これらのフィールドを保持する`Song`構造体が必要になる。APIはリスト形式で返すため、それを包む`SearchResponse`も必要になる。

---

## ステップ3：状態を特定する（状態管理）

画面はどのような条件で変化するのか？

画面を動かす3つの状態：
- 読み込み中 → ローディングスピナーを表示
- 読み込み失敗 → エラーを表示
- 読み込み成功 → リストを表示

これら3つの状態はViewModelで一元管理する。

---

## ステップ4：画面を分割する（UIの階層化）

画面はどの独立したパーツで構成されているか？
- 検索バー（テキストフィールド＋ボタン）
- コンテンツエリア（3つの状態に対応した3種類の表示）
- リスト行（各楽曲の概要情報）
- 詳細ページ（タップして詳細情報を表示）
- エラーバナー（エラー発生時に表示）

それぞれを独立したViewにし、互いに干渉しないようにする。

---

## ステップ5：ロジックをつなぐ（データフロー）

ユーザーが検索ボタンをタップ → ViewModelがネットワークリクエストを送信 → 状態を更新 → UIが状態に応じて自動更新される。

> これがMVVMの本質：ViewはUIの表示のみ担当し、ViewModelはデータとロジックのみ担当する。両者は状態バインディングを通じて自動的に同期される。

---

## 実装ステップ1：APIから返ってくるJSONはリスト形式なので、以下を書く：

1. `struct SearchResponse`でデータを種類別に整理して保存する
2. 2つのフィールドを含める：`resultCount`（楽曲数）と`results`（各楽曲の詳細情報：曲名・アーティスト名・アルバム名・価格など）
3. 各楽曲の一意なID

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

4. 各楽曲の価格処理ロジック（価格と通貨単位の両方が存在する場合は表示し、どちらかが欠けている場合は「価格不明」と表示する）

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

## 実装ステップ2：誰が実際に処理を行うのか？

Viewは表示のみを担当し、ネットワークリクエストは送信しない。そのため、以下を処理するViewModelが必要になる：
1. ネットワークデータの処理
2. ネットワークリクエストの送信
3. エラーのハンドリング

まず4つの変数が必要：検索結果、ユーザーが入力したキーワード、検索状態の変化、エラーメッセージ

```swift
@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?
}
```

次に、発生しうる4種類のエラーを定義する：URL構築の失敗、ネットワークエラー、JSONのデコード失敗、検索結果なし

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

最後に検索ロジックを追加する：入力欄が空かどうかの確認 → 入力をURLセーフな文字列にエンコード → URLを構築 → リクエストを送信し、データを取得し、`SearchResponse`にデコード → 結果に応じて状態を更新

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

## 実装ステップ3：メイン画面 ContentView の構築

ContentViewに表示するもの：
1. 検索バー
2. コンテンツエリア
3. エラー表示エリア

まず、ViewModelをViewに注入する。ViewはViewModelを通じて状態を読み取り、メソッドを呼び出す。

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

検索バー：
1. Returnキーを押しても、ボタンをタップしても検索が実行される
2. 入力欄が空のとき、または読み込み中はボタンが無効になる
3. 検索ボタンをタップしてから結果が表示されるまでの待機中も、アプリは引き続き操作できる

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

コンテンツエリア：
1. 検索結果の読み込み中はスピナーを表示する（ユーザー体験の向上）
2. 楽曲リストが空の場合はプレースホルダー画面を表示する
3. 楽曲リストを表示する

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

## 実装ステップ4：検索結果をタップした後の詳細情報

SongRow：ユーザーが検索した後のリスト内、各行のスタイル定義。以下を含む：
1. アルバムアートのサムネイル
2. 曲名
3. アーティスト名
4. 価格

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

## 実装ステップ5：SongRowをタップした後に表示される画面

SongRowよりも大きなアートワークを表示し、情報もより詳細にする。また、アルバム名がない楽曲もあるため、`if let`でアンラップする必要がある。

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

## 実装ステップ6：エラー処理のロジック

エラー処理ロジックを`struct ErrorBanner: View {}`にカプセル化し、ContentViewから直接呼び出せるようにする。

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


## この章のまとめ
プログラマーの仕事の本質は常に人と向き合うことであり、人の考えを実現し、人の為に価値を提供することにある。コードはその目的をより効率的に達成する為の手段に過ぎない。


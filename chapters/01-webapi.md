# 第1章：WebAPIの基本

> 執筆者：王金橋
> 最終更新：2026-04-18

## この章で学ぶこと

この章では、ユーザーが入力したもう文字列をURLに変換してAPIからデータを取得し、そのデータをリスクとして表示します。

## 模範コードの全体像
mport SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
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
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

**このアプリは何をするものか：**

音楽、歌手を検索できます

## コードの詳細解説

### データモデル（Codable構造体）

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

**何をしているか：**
APIから取得したJSONをSwiftUIが読めるSongに変換します。

**なぜこう書くのか：**
APIから返されたJSONの構造に対応して、一つ一つデコードします。

**もしこう書かなかったら：**
API取得したJSONをデコードできない、エラーになります。


---

### API通信の処理

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

**何をしているか：**
1. ユーザーが入力された文字列をURLエンコードします
2. URLエンコードでitunesのAPIからデータを取得します
3. 取得したデータをstruct SearchResponseでデコードしてSwiftの構造体にします


**なぜこう書くのか：**
1. ユーザーが入力した文字列をAPI認識できるURLに変換します
2. APIから返ってきたJSONをSwiftで使える構造体に変換します


**もしこう書かなかったら：**
話が通じない

---

### ビューの構成
struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
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
        }
        .padding(.vertical, 4)
    }
}

**何をしているか：**
Songのフィールドを取り出して、それぞれ対応するUIの位置に配置します

**なぜこう書くのか：**
AsyncImageは非同期で画像を読み込むため、UIをブロックしない。
HStackとVStackの組み合わせはSwiftUIの標準的なレイアウト方法で、構造が分かりやすく直感的である。

**もしこう書かなかったら：**
AsyncImageを使わない：URLから画像を読み込めない
placeholderを使わない：読み込み中に画面が空白になり、体験が悪い
HStack/VStackを使わない：横に画像＋縦にテキストのレイアウトができない


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
|　Codable　| Swiftの構造体はJSONと自動的に相互変換できます | struct Song: Codable { ... }|
| async/await | ネットワークリクエスト中でもUIをブロックせず、ユーザーは画面をスクロールしたりボタンを押したりでき、カクつかない | `let data = try await　URLSession.shared.data(from: url)|

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：limit=25　→ limit=5改変して試した
- 結果：検索結果が3件に減ります
- わかったこと：リクエストで内容で取得データ量を制御できます

**実験2：**
- やったこと：.clipShape(RoundedRectangle(cornerRadius: 8)) → .clipShape(Circle())
- 結果：アイコンが正円になります
- わかったこと：SwiftUIはモディファイアの変更だけで直感的にUIを調整できる

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   実際の業務では、プログラマーは考え方とその実装の順序、そしてエラーが出ないようにすることを理解していれば十分で、各アイデアに対応する具体的なコードを全て覚えているわけではない、という理解であっていますか？

   **得られた理解：**
   プログラマーの仕事で最も重要なのはコードを暗記することではなく、そもそもそれは現実的でもない。重要なのはロジック構造と業務フローであり、頭の中に明確なフローチャートを持ち、コードの各ステップで何をすべきかを理解し、防御的プログラミングやエラーハンドリングの方法を把握することだ。エネルギーはアーキテクチャ思考や問題解決の考え方を鍛えることに使うべきである。
   

2. **質問：**
   すべてのプログラマーの思考ロジックはこのような流れになっているのでしょうか？
つまり、「何が欲しいか → どうやって取得するか → 取得した後どう処理するか → 処理した結果をどう表示するか」という理解でいいのでしょうか？

   **得られた理解：**
   本質的には同じで、まず頭の中で人間の思考と言語を使ってアイデアや目的、達成したい目標を明確にし、その上で大枠のフレームワークを構築し、さらに最小実行単位まで分解していく。最後に、Googleやフォーラム、AI、社内ドキュメントなどを活用して具体的なコードを書いていく。

3. **質問：**
   さっきの「何が欲しいか → どうやって取得するか → どう処理するか → どう表示するか」はデータ処理系の具体的なパターンに過ぎない。他の場面、例えばアルゴリズムやシステム設計では別の思考フレームワークになるが、「問題を分解する」という本質は同じ。では、アルゴリズムやシステム設計ではどのように考えるのか？

   **得られた理解：**
   アルゴリズムの思考：
入力は何か → 出力は何か → 途中でどう変換するか → 時間計算量・空間計算量は許容範囲か
　　システム設計の思考：
ユーザー数はどれくらいか → データはどう保存するか → 各モジュールはどう役割分担するか → ボトルネックはどこか → どうスケールさせるか
しかし本質は同じで、大きな問題を小さな問題に分解し、一つずつ解決していくこと。


## この章のまとめ
プログラマーの仕事の本質は常に人と向き合うことであり、人の考えを実現し、人の為に価値を提供することにある。コードはその目的をより効率的に達成する為の手段に過ぎない。


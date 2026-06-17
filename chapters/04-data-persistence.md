# 第4章：データの永続化

> 執筆者：王金橋
> 最終更新：2026-06-16

## この章で学ぶこと

この章では、アプリを閉じたり端末を再起動したりしてもデータが消えないようにする「データの永続化」について学びます。
具体的には、大量のデータを構造化して保存できる最新のフレームワーク SwiftData と、ユーザーの選択や名前などのちょっとした設定を簡単に保存できる AppStorage の2つの方法を使い、実際のメモアプリを実装しながらその違いと使い分けをマスターします。

## 模範コードの全体像

// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

**このアプリは何をするものか：**

ユーザーが追加したメモ（タイトル、内容、日付、お気に入りフラグ）を端末のデータベースにしっかり保存するアプリです。
右上の「＋」からメモを保存すると、リストに新着順で追加されます。左上の設定（ギアのアイコン）から自分の名前を入力するとナビゲーションタイトルが「（名前）のメモ帳」に変わり、「お気に入りを上に表示」をONにすると、星マークがついた重要なメモが一番上に集まるようにソートされます。アプリを完全に落としても、メモの内容やこれらの設定はすべて維持されます。

## コードの詳細解説

### SwiftDataモデル（@Model）

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

**何をしているか：**
ただの普通のクラス（class Memo）の頭に @Model マクロをつけて、このデータ構造をそのまま端末のデータベース（SQLite）のテーブルに自動変換するための定義をしています。

**なぜこう書くのか：**
iOS 17からの SwiftData では、これをつけるだけで、後ろで動く面倒なスキーマ管理やSQL文の作成をOSが全部自動でやってくれるからです。プログラマーは純粋なSwiftオブジェクトを扱うのと同じ感覚でデータベースモデルを定義できます。

**もしこう書かなかったら：**
@Model をつけ忘れると、SwiftDataのシステムがこのクラスを保存可能なデータとして認識できません。後述する @Query で取得しようとしたり、modelContext に入れようとしたりした段階で、コンパイルエラーになるか、アプリが強制終了（クラッシュ）します。

---

### データの追加・削除（modelContext）

// 追加（MemoAddView）
@Environment(\.modelContext) private var modelContext
// ...
let memo = Memo(title: title, content: content)
modelContext.insert(memo)

// 削除（ContentView）
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}

**何をしているか：**
アプリの環境変数から、データベースを操作するための「作業スペース（コンテキスト）」を取り出しています。そして、新しく作ったメモオブジェクトをそこに挿入（insert）したり、選んだメモを削除（delete）したりしています。
**なぜこう書くのか：**
SwiftDataは安全のために、直接ファイルを書き換えるのではなく、一度この modelContext という「仮の作業机」の上でデータの追加や変更を行います。机の上で起きた変更は、SwiftUIの画面の変化に合わせて、OSがバックグラウンドで自動的に実際のディスクファイルへセーブしてくれる仕組みになっているからです。
**もしこう書かなかったら：**
modelContext.insert() や .delete() を呼ばないと、画面の見た目の状態（@State など）が変わるだけで、アプリを再起動したときにデータが元に戻ってしまいます。また、環境変数（\.modelContext）を正しく取得しないと、データをどこに反映していいか分からずエラーになります。
---

### @Queryによるデータ取得

@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]

**何をしているか：**
データベースに保存されているすべての Memo データを、作成日時（createdAt）の新しい順（.reverse：降順）で自動的に取得して、配列（[Memo]）の中にリアルタイムでロードしています。
**なぜこう書くのか：**
@Query を使うと、データベースの中身が書き換わった瞬間（データの追加や削除が起きたとき）を自動的に検知して、SwiftUIの画面を自動で再描画してくれるからです。自分で毎回「データを再読み込みする関数」を呼ぶ必要がなくなります。
**もしこう書かなかったら：**
これを使わずにデータを取ろうとすると、手動でFetchリクエストを毎回組み立てて実行するコードを何行も書く必要があります。また、新しいメモを追加した後にリストを更新するコードも自分で書かないといけないため、バグが起きやすくなり、コードがとても複雑になります。
---

### @AppStorageによる設定保存

@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""

**何をしているか：**
iOSの軽量な保存場所である UserDefaults と連動するプロパティラッパーです。「お気に入りを上にするか（Bool）」や「ユーザーの名前（String）」という、アプリの環境設定レベルの小さなデータを保存しています。
**なぜこう書くのか：**
SwiftData（データベース）を使うまでもない、単一のON/OFFスイッチや短いテキストの設定値は、@AppStorage を使うのが一番コードが短く、動作も軽いからです。キー名（"userName" など）を指定するだけで、変数への代入と永続化保存が同時に行われます。
**もしこう書かなかったら：**
@State だけで管理すると、アプリを閉じるたびに設定した名前やスイッチの状態がリセットされて、初期値（空やfalse）に戻ってしまいます。昔の書き方のように UserDefaults.standard.set() を毎回手動で呼ぶ方法もありますが、コードが長くなってしまいます。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|@Query|データベースから条件を指定してデータを動的に取得するプロパティラッパー。|@Query(sort: \.date) var memos: [Memo]|
|@AppStorage|UserDefaultsと連動し、小さな設定値を簡単に永続化する機能。|@AppStorage("key") var value = false|
|@Bindable|iOS 17以降で、@Model クラスのプロパティをUIと直接双方向バインドするためのラッパー。|@Bindable var memo: Memo|

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
Appのエントリポイントで .modelContainer を書き忘れてみた
•	やったこと：
プレビュー部分やメインのAppファイルにある .modelContainer(for: Memo.self) のコードをわざとコメントアウトして実行してみた。
•	結果：
アプリのビルドは通るが、シミュレータが起動した瞬間に画面が真っ白になり、コンソールに Fatal error: failed to find a currently active container というエラーが出てクラッシュした。
•	わかったこと：
modelContext や @Query を使う前に、必ずアプリのルート（根っこ）の部分で「このデータを使うよ」というコンテナの登録を済ませておかないと、アプリの心臓部が動かないことが本当によく分かりました。

**実験2：**
@Query のソート条件を複数に変えてみた
•	やったこと：
@Query(sort: [SortDescriptor(\Memo.isFavorite, order: .reverse), SortDescriptor(\Memo.createdAt, order: .reverse)]) のように、配列を使ってソート条件を2つ指定してみた。
•	結果：
ContentView の中に書いた手動のソート計算プロパティ（displayedMemos）を使わなくても、データベースから取ってきた時点で「お気に入りが上で、さらにその中で新しい順」に綺麗に並んだ。
•	わかったこと：
SwiftDataの @Query はとても賢くて、簡単な並び替えロジックならSwiftUI側でわざわざ配列を再計算しなくても、取得する段階で一発で綺麗に整理できることが分かりました。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   SwiftDataの @Model と .modelContainer の関係がよく分からないです。
   **得られた理解：**
   @Model は「このデータは保存する対象だよ」という設計図の目印で、.modelContainer は実際に端末の中に保存用のファイル（実物）を作る役割、という役割分担がハッキリ分かりました。

2. **質問：**
   データを追加した後に save() みたいなコマンドを呼ばなくていいのはなぜですか？
   **得られた理解：**
    SwiftUIの画面が更新されるタイミングや、アプリがバックグラウンドに回るタイミングをOSが見張っていて、modelContext の中の変更を自動で保存してくれるからだと知りました。自分で書かなくていいので楽です。

3. **質問：**
   @State と @AppStorage は何が違いますか？
   **得られた理解：**
   どちらも値が変わると画面が変わる「センサー」ですが、@State はメモリの中だけ（アプリを落としたら消える）、@AppStorage は端末のフォルダの中に自動で書き込まれる（アプリを落としても消えない）という保存期間の違いが分かりました。

## この章のまとめ
この章を終えて、モダンなiOSアプリにおける「データの守り方」の王道が分かりました。
一番の大きな気付きは、データの種類によって 「重いデータベース（SwiftData）」 と 「軽い設定ファイル（AppStorage）」 を正しくハサミのように使い分ける重要性です。日記の本文や写真のように、どんどん増える複雑なものは @Model でテーブルを作り、UIのモード切り替えやユーザー名などのデータは @AppStorage でスマートに処理するのが美しい設計だと確信しました。
また、iOS 17以降の @Bindable のおかげで、データベースのデータを直接 TextField に繋いで、書き換えたら自動で保存される流れはとても直感的です。もし将来、データが保存されなくて困ったときは、まずはエントリポイントの .modelContainer があるか、そしてコンテキストにちゃんと insert されているかを確認するようにします。

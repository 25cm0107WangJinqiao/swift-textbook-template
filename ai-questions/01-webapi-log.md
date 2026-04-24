# AI質問ログ：第1章 WebAPIの基本

## 使用した生成AIツール

Gemini

## 質問と回答の記録

### Q1

**質問：**
Swiftのguard let url= URL（string....）という書き方は、どんな意味ですか？
if let とどう違いますか？初心者向けは教えてください

**AIの回答の要点：**
「文字列をURLに変換し、成功した場合はそのまま使い続ける」という意味です。もし無効な文字列で変換に失敗した場合は、エラーを防ぐためにその場で直ちに処理を中断（退出）します。
guard let はダメなら即終了してコードをスッキリ保つ仕組み、if let は成功した時だけ { } の中で処理する仕組み」

**自分の理解：**
StringはURLに変換します
guard letは、その条件が絶対に必要な時に使います。条件を満たさないと、そもそも先に進めないので、すぐ処理を止めます。if letは、条件を満たさなくても別の道があるときに使います。

### Q2

**質問：**
guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        )って、どんな意味ですか？

**AIの回答の要点：**
このコードの役割は、「検索文字（日本語やスペースなど）を、URLとして使える形式（パーセントエンコーディング）に変換すること」です。

**自分の理解：**
English以外のStringはURLに変換できる

### Q3

**質問：**
データフローのライフサイクルを一通り案内してください

**AIの回答の要点：**
データフローのライフサイクル：

1. **ユーザー入力 (searchText)**
   `searchText` に文字が入力される。
2. **URL構築 (Percent Encoding)**
   文字列をパーセントエンコーディングし、`URL` オブジェクトを生成する。
3. **URLSession 非同期リクエスト (isLoading = true)**
   `isLoading` を `true` にし、`await` でネットワーク通信を開始する。
4. **Data取得 → JSONDecoder → [Song]**
   受信したバイナリデータを `JSONDecoder` で `Song` 配列にデコードする。
5. **songs = response.results (isLoading = false)**
   結果を `songs` に格納し、`isLoading` を `false` に戻す。
6. **@State 変化 → SwiftUI 自動再描画 (List)**
   状態の変化を検知し、SwiftUI が `List` を自動的に再レンダリングする。
7. **各行の AsyncImage 並行・遅延読み込み**
   各行の `AsyncImage` がアルバムアートをバックグラウンドで個別に取得・表示する。

**自分の理解：**

① String → URL
② URL → APIにリクエスト → JSONデータを取得
③ JSONデータ → JSONDecoder（変換） → [Song] → SwiftUIでリストを描画


## 今日の質問を振り返って

（どんな質問が良い質問だったか。生成AIの回答で間違いや不正確な部分はあったか。次回はどんな質問をしてみたいか。）

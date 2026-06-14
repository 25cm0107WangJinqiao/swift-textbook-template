# 第3章：カメラの利用

> 執筆者：王金橋
> 最終更新：2026-6-14

## この章で学ぶこと
この章では、PhotosPicker を利用してフォトライブラリから写真を選択し、CoreImage を使って写真にフィルターを適用する方法を学んだ。
最初は画像を表示するだけのアプリだと思った。しかし、実際には写真の読み込み、画像加工、フィルターの切り替え、フォトライブラリへの保存など複数の処理が組み合わされていることが分かった。

## 模範コードの全体像

import SwiftUI
import PhotosUI
import Photos
import CoreImage
import CoreImage.CIFilterBuiltins

// MARK: - フィルター定義

enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }

    func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
        switch self {
        case .original:
            return inputImage
        case .sepia:
            let filter = CIFilter.sepiaTone()
            filter.inputImage = inputImage
            filter.intensity = 0.8
            return filter.outputImage
        case .mono:
            let filter = CIFilter.photoEffectMono()
            filter.inputImage = inputImage
            return filter.outputImage
        case .chrome:
            let filter = CIFilter.photoEffectChrome()
            filter.inputImage = inputImage
            return filter.outputImage
        case .fade:
            let filter = CIFilter.photoEffectFade()
            filter.inputImage = inputImage
            return filter.outputImage
        case .bloom:
            let filter = CIFilter.bloom()
            filter.inputImage = inputImage
            filter.radius = 10
            filter.intensity = 0.8
            return filter.outputImage
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var originalUIImage: UIImage?
    @State private var displayImage: Image?
    @State private var currentFilter: PhotoFilter = .original
    @State private var isSaving = false
    @State private var showSaveAlert = false
    @State private var saveMessage = ""

    private let context = CIContext()

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                // 画像表示
                if let image = displayImage {
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .frame(maxHeight: 350)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                        .padding(.horizontal)
                } else {
                    placeholderView
                }

                // フィルター選択
                if originalUIImage != nil {
                    filterSelector
                }

                // ボタン群
                HStack(spacing: 16) {
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選ぶ", systemImage: "photo")
                    }
                    .buttonStyle(.bordered)

                    if displayImage != nil {
                        Button {
                            saveFilteredImage()
                        } label: {
                            Label("保存", systemImage: "square.and.arrow.down")
                        }
                        .buttonStyle(.borderedProminent)
                        .disabled(isSaving)
                    }
                }
                .padding()

                Spacer()
            }
            .navigationTitle("フォトフィルター")
            .onChange(of: selectedItem) { _, newItem in
                Task { await loadOriginalImage(from: newItem) }
            }
            .onChange(of: currentFilter) { _, _ in
                applyFilter()
            }
            .alert("保存結果", isPresented: $showSaveAlert) {
                Button("OK") {}
            } message: {
                Text(saveMessage)
            }
        }
    }

    // MARK: - プレースホルダー

    private var placeholderView: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(.gray.opacity(0.1))
            .frame(height: 300)
            .overlay {
                VStack(spacing: 8) {
                    Image(systemName: "camera.filters")
                        .font(.system(size: 48))
                        .foregroundStyle(.gray)
                    Text("写真を選んでフィルターを試そう")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .padding(.horizontal)
    }

    // MARK: - フィルター選択UI

    private var filterSelector: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 12) {
                ForEach(PhotoFilter.allCases) { filter in
                    VStack(spacing: 4) {
                        // フィルタープレビュー（サムネイル）
                        if let thumbnail = createThumbnail(filter: filter) {
                            Image(uiImage: thumbnail)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 60, height: 60)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                                .overlay(
                                    RoundedRectangle(cornerRadius: 8)
                                        .stroke(
                                            currentFilter == filter ? Color.blue : Color.clear,
                                            lineWidth: 3
                                        )
                                )
                        }

                        Text(filter.rawValue)
                            .font(.caption2)
                            .foregroundStyle(
                                currentFilter == filter ? .blue : .secondary
                            )
                    }
                    .onTapGesture {
                        currentFilter = filter
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 画像処理

    func loadOriginalImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                originalUIImage = uiImage
                currentFilter = .original
                displayImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像読み込みエラー: \(error)")
        }
    }

    func applyFilter() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return }

        guard let outputImage = currentFilter.apply(to: ciImage, context: context) else { return }

        if let cgImage = context.createCGImage(outputImage, from: ciImage.extent) {
            // 元画像の scale と imageOrientation を引き継がないと
            // EXIF回転情報が落ちて画像が上下反転して表示される
            displayImage = Image(uiImage: UIImage(cgImage: cgImage,
                                                  scale: uiImage.scale,
                                                  orientation: uiImage.imageOrientation))
        }
    }

    func createThumbnail(filter: PhotoFilter) -> UIImage? {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return nil }

        guard let output = filter.apply(to: ciImage, context: context) else { return nil }

        if let cgImage = context.createCGImage(output, from: ciImage.extent) {
            return UIImage(cgImage: cgImage,
                           scale: uiImage.scale,
                           orientation: uiImage.imageOrientation)
        }
        return nil
    }

    func saveFilteredImage() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage),
              let output = currentFilter.apply(to: ciImage, context: context),
              let cgImage = context.createCGImage(output, from: ciImage.extent) else { return }

        // PHAssetChangeRequest は imageOrientation を尊重して保存するため、
        // ここで orientation を渡しておかないと保存ファイルも上下反転する
        let finalImage = UIImage(cgImage: cgImage,
                                 scale: uiImage.scale,
                                 orientation: uiImage.imageOrientation)
        isSaving = true

        // PHPhotoLibrary を使うと、保存の成否をコールバックで受け取れる
        PHPhotoLibrary.shared().performChanges {
            PHAssetChangeRequest.creationRequestForAsset(from: finalImage)
        } completionHandler: { success, error in
            DispatchQueue.main.async {
                isSaving = false
                if success {
                    saveMessage = "写真を保存しました"
                } else {
                    saveMessage = "保存に失敗しました\n\(error?.localizedDescription ?? "原因不明のエラー")"
                }
                showSaveAlert = true
            }
        }
    }
}

#Preview {
    ContentView()
}


**このアプリは何をするものか：**
このアプリは、iPhone のフォトライブラリから写真を選択し、セピアやモノクロなどのフィルターをかけて保存するアプリである。
写真を選択すると画像データを読み込み、画面に表示する。その後、ユーザーがフィルターを選択すると CoreImage によって画像が加工され、加工後の結果が画面に反映される。
また、完成した画像は PHPhotoLibrary を利用してフォトライブラリへ保存できる。

## コードの詳細解説

### PhotosPickerによる写真選択

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("写真を選ぶ", systemImage: "photo")
}

**何をしているか：**
この部分では、ユーザーがフォトライブラリから写真を選択するためのボタンを作成している。
写真が選択されると、その情報が selectedItem に保存される。

**なぜこう書くのか：**
最初は、ボタンを押したらすぐに画像が取得できると思っていた。
しかし、実際には PhotosPicker が写真選択画面を表示し、選択された結果を PhotosPickerItem として受け取る仕組みになっていることが分かった。
また、matching: .images を設定することで、動画ではなく画像だけを選択できる。

**もしこう書かなかったら：**
PhotosPicker がなければ、ユーザーはフォトライブラリから写真を選択できない。
また、matching の設定を変更すると、画像以外のデータも選択できるようになってしまう。

---

### 画像の非同期読み込み
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadOriginalImage(from: newItem)
    }
}

**何をしているか：**
この部分では、選択された写真が変わったことを監視している。
新しい写真が選択されると、loadOriginalImage() を呼び出して画像データの読み込みを開始する。

**なぜこう書くのか：**
最初は、写真を選択した時点で自動的に画面へ表示されると思っていた。
しかし、実際には画像データを読み込む処理が必要であり、写真選択と画像読み込みは別の処理になっていることが分かった。
また、Task と async/await を使うことで、画像の読み込み中でも画面の操作を止めずに処理できる。

**もしこう書かなかったら：**
selectedItem が変更されても画像を読み込む処理が実行されないため、写真を選択しても画面には表示されない。
また、時間のかかる処理を同期的に実行すると、画面の動作が重くなる可能性がある。

---

### フィルターの管理（PhotoFilter）
enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }
}

**何をしているか：**
この部分では、アプリで使用するフィルターの種類をまとめて管理している。
それぞれの case が一つのフィルターを表している。また、CaseIterable に準拠することで、すべてのフィルターを一覧として取得できる。

**なぜこう書くのか：**
最初は、セピアやモノクロなどを別々の変数やボタンで管理しても良いと思った。
しかし、enum を使ってまとめることで、フィルターの種類を一つの場所で管理できることが分かった。
また、CaseIterable を利用することで ForEach と組み合わせてフィルター一覧を自動生成できるため、後から新しいフィルターを追加するときも修正箇所が少なくなる。
**もしこう書かなかったら：**
フィルターごとに個別の処理やボタンを作る必要があり、フィルターの数が増えるほどコードが複雑になる。
また、新しいフィルターを追加するときに複数の場所を修正する必要があり、管理が難しくなる。

---


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|PhotosPicker|フォトライブラリから写真を選択するためのSwiftUIコンポーネント|PhotosPicker(selection: $selectedItem, matching: .images)|
|PhotosPickerItem|選択された写真の情報を保持するオブジェクト|@State private var selectedItem: PhotosPickerItem?|
|CIImage|CoreImageで画像加工を行うための画像データ形式|CIImage(image: uiImage)|

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：PhotoFilter に新しくフィルターを追加して、画面に表示されるか確認した。
- 結果：enum に新しい case を追加するだけで、フィルター一覧に自動的に表示された。
- わかったこと：CaseIterable と ForEach を組み合わせることで、ボタンを一つずつ追加しなくても自動的にUIを生成できることが分かった。

**実験2：**
- やったこと：CIFilter.sepiaTone() の intensity の値を 0.8 から 0.3 に変更して、画像の変化を確認した。
- 結果：セピア色の効果が弱くなり、元の写真に近い見た目になった。
- わかったこと：フィルターにはパラメータがあり、数値を変更することで加工の強さを調整できることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
経験豊富なエンジニアは、この写真フィルターアプリをどのような流れで設計しますか。
   **得られた理解：**
最初からコードを書くのではなく、写真選択、画像加工、表示、保存という機能ごとに役割を分けて設計することが重要だと理解した。
また、ユーザーがどのような順番でアプリを操作するかを考えてから実装することが大切だと分かった。
2. **質問：**
なぜ PhotoFilter を enum で管理しているのですか。 
   **得られた理解：**
最初はフィルターごとに別々の処理を書いても良いと思った。
しかし、enum を利用することでフィルターの種類と処理を一箇所で管理でき、後から新しいフィルターを追加しやすい設計になっていることが分かった。

3. **質問：**
applyFilter() では、画像がどのような流れで加工されて画面に表示されますか。
   
   **得られた理解：**
UIImage をそのまま加工するのではなく、CIImage に変換して CoreImage の処理を行い、その後 CGImage、UIImage へ変換して表示していることを理解した。
また、@State の値が変更されると SwiftUI が自動的に画面を更新する仕組みについても理解できた。

## この章のまとめ
今回の章では、写真を選択するだけではなく、選択した画像を加工し、保存するまでの一連の流れを学んだ。
特に印象に残ったのは、画像処理では複数の画像形式を変換しながら処理を行っている点である。
また、PhotoFilter を enum で管理する設計や、@State による画面の自動更新など、SwiftUIらしいデータ中心の考え方について理解が深まった。
最初は画像処理の流れが複雑に感じたが、「写真を取得する」「フィルターを適用する」「結果を表示する」「保存する」という順番で考えることで、アプリ全体の動きを理解できるようになった。

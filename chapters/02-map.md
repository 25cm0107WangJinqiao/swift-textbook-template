# 第2章：地図アプリの基本

> 執筆者：王金橋
> 最終更新：2026-6-14

## この章で学ぶこと

この章では、MapKit を使って現在地を地図に表示する方法と、現在地の近くにある施設を検索する方法を学んだ。

最初は「地図を表示して検索するだけのアプリ」だと思っていた。しかし、コードを確認すると、位置情報の取得、周辺検索、地図への表示という複数の機能が組み合わされていることが分かった。

## 模範コードの全体像

import SwiftUI
import MapKit

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        userLocation = locations.last?.coordinate
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        switch authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            startUpdating()
        default:
            break
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var searchResults: [MKMapItem] = []
    @State private var selectedCategory: String = "コンビニ"
    @State private var hasInitialSearched = false

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    // CLLocationCoordinate2D は Equatable に準拠していないため、
    // onChange で監視できる文字列キーを派生させる
    private var userLocationKey: String? {
        locationManager.userLocation.map { "\($0.latitude),\($0.longitude)" }
    }

    var body: some View {
        ZStack(alignment: .top) {
            Map(position: $cameraPosition) {
                // 現在地のマーカー
                UserAnnotation()

                // 検索結果のマーカー
                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                            .tint(.orange)
                    }
                }
            }
            .mapControls {
                MapUserLocationButton()
                MapCompass()
                MapScaleView()
            }

            // 検索カテゴリボタン
            VStack {
                categoryButtons
                    .padding(.top, 8)
                Spacer()
            }
        }
        .onAppear {
            locationManager.requestPermission()
        }
        .onChange(of: userLocationKey) { _, _ in
            guard let location = locationManager.userLocation else { return }
            cameraPosition = .region(
                MKCoordinateRegion(
                    center: location,
                    span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
                )
            )
            // 初回の位置取得時に、選択中カテゴリで自動的に周辺検索を行う
            if !hasInitialSearched {
                hasInitialSearched = true
                Task { await searchNearby(query: selectedCategory) }
            }
        }
    }

    // MARK: - カテゴリボタン

    private var categoryButtons: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(searchCategories, id: \.self) { category in
                    Button {
                        selectedCategory = category
                        Task { await searchNearby(query: category) }
                    } label: {
                        Text(category)
                            .font(.subheadline)
                            .padding(.horizontal, 14)
                            .padding(.vertical, 8)
                            .background(
                                selectedCategory == category
                                    ? Color.blue
                                    : Color(.systemBackground)
                            )
                            .foregroundStyle(
                                selectedCategory == category
                                    ? .white
                                    : .primary
                            )
                            .clipShape(Capsule())
                            .shadow(color: .black.opacity(0.1), radius: 2)
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 周辺検索

    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else { return }

        let request = MKLocalSearch.Request()
        request.naturalLanguageQuery = query
        request.region = MKCoordinateRegion(
            center: userLocation,
            span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
        )

        do {
            let search = MKLocalSearch(request: request)
            let response = try await search.start()
            searchResults = response.mapItems
        } catch {
            print("検索エラー: \(error.localizedDescription)")
            searchResults = []
        }
    }
}

#Preview {
    ContentView()
}

**このアプリは何をするものか：**

このアプリは、iPhone の位置情報を取得し、現在地を地図上に表示するアプリである。

また、コンビニ、カフェ、レストラン、駅などのカテゴリボタンを押すことで、現在地の周辺にある施設を検索できる。

最初の設計についてAIに質問した際、アプリ全体は「現在地の取得」「周辺施設の検索」「地図への表示」の3つの大きな役割に分けて考えることが重要だと分かった。

データの流れとしては、現在地が取得されると地図の位置が更新され、その後検索が実行される。検索結果が変わると、SwiftUI の仕組みによって地図上のマーカーも自動的に更新される。

## コードの詳細解説

### データモデル（ランドマーク構造体）

状態管理（State）
@State private var locationManager = LocationManager()
@State private var cameraPosition: MapCameraPosition = .automatic
@State private var searchResults: [MKMapItem] = []
@State private var selectedCategory: String = "コンビニ"

**何をしているか：**
この部分では、アプリ内で変化する情報を保存している。
locationManager は現在地の情報を管理する。
cameraPosition は地図の表示位置を管理する。
searchResults は検索した施設の一覧を保存する。
selectedCategory は現在選択しているカテゴリを保存する。

**なぜこう書くのか：**
最初は、ただ変数を作るだけなら普通の変数でも良いと思った。
しかし、SwiftUI では画面の表示に関係する値が変化したとき、自動的に画面を更新する仕組みがある。
そのため、現在地や検索結果のように画面の内容に影響する情報は State として管理する必要がある。

**もしこう書かなかったら：**
例えば、検索結果のデータだけが変化しても、地図のマーカーは更新されない。
現在地が変わっても地図が移動しないなど、画面とデータの状態が一致しない問題が発生する。

---

### 地図の表示とカメラ制御
Map(position: $cameraPosition) {
    UserAnnotation()

    ForEach(searchResults, id: \.self) { item in
        if let name = item.name {
            Marker(name, coordinate: item.placemark.coordinate)
                .tint(.orange)
        }
    }
}

**何をしているか：**
この部分では、地図を画面に表示している。
UserAnnotation は現在地のマークを表示している。
また、searchResults に保存された検索結果を繰り返し処理して、それぞれの施設を Marker として地図上に表示している。

**なぜこう書くのか：**
最初は、検索結果がどのように地図に表示されるのか分からなかった。
コードを確認すると、検索した施設の情報が searchResults に保存されていて、ForEach を使って一つずつ Marker に変換していることが分かった。
また、Map の position に cameraPosition を設定することで、現在地が更新されたときに地図の中心位置も変更できる。

**もしこう書かなかったら：**
UserAnnotation を書かなければ、自分の現在地を確認できない。
また、ForEach で検索結果を表示しなければ、検索自体は成功していても、ユーザーは地図上で施設の場所を確認できない。
cameraPosition を管理しなければ、現在地を取得しても地図が自動的に移動しない。

---

### マーカーの表示
ForEach(searchResults, id: \.self) { item in

    if let name = item.name {

        Marker(name, coordinate: item.placemark.coordinate)

            .tint(.orange)

    }

}

**何をしているか：**
この部分では、周辺検索で取得した施設を地図上にマーカーとして表示している。
searchResults には検索結果が複数入っているため、ForEach を使って一つずつ取り出し、それぞれの位置に Marker を配置している。
また、Marker の色をオレンジに変更することで、検索結果の施設を分かりやすくしている。

**なぜこう書くのか：**
最初は、検索しただけでなぜ地図上に複数の施設が表示されるのか分からなかった。
コードを確認すると、検索結果のリストを ForEach で順番に処理して、施設ごとに Marker を作成していることが分かった。
この方法を使うことで、検索結果の数が増えたり減ったりしても、自動的に必要な数だけマーカーを表示できる。

**もしこう書かなかったら：**
ForEach を使わなければ、検索結果が複数ある場合に一つずつ手動で Marker を作成する必要があり、実際のアプリでは管理が難しくなる。
また、Marker を表示しなければ検索自体は成功していても、ユーザーは施設がどこにあるのか地図上で確認できない。

---

### フィルター機能
ForEach(searchCategories, id: \.self) { category in

    Button {

        selectedCategory = category

        Task {

            await searchNearby(query: category)

        }

    } label: {

        Text(category)

    }

}

**何をしているか：**
この部分では、コンビニ、カフェ、レストラン、駅などのカテゴリをボタンとして表示している。
ユーザーがボタンを押すと、選択されたカテゴリを保存し、そのカテゴリを検索キーワードとして周辺検索を実行する。

**なぜこう書くのか：**
最初は、カテゴリを切り替えたときにどうやって検索内容が変わるのか疑問だった。
コードを追ってみると、ボタンを押したタイミングで searchNearby() が呼ばれ、選択したカテゴリの文字列が検索条件として渡されていることが分かった。
カテゴリを分けることで、ユーザーは目的の施設を簡単に探すことができる。

**もしこう書かなかったら：**
カテゴリを変更する仕組みがなければ、毎回同じ種類の施設しか検索できない。
また、検索したい施設を自由に切り替えることができず、アプリの使いやすさが下がる。
さらに、ボタンごとに別々の検索処理を書く方法もあるが、ForEach を使ってカテゴリ一覧から自動的にボタンを作ることで、後からカテゴリを追加するときも簡単に変更できる。
---


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|CLLocationManager|iPhoneの位置情報を取得・管理するためのクラス|let manager = CLLocationManager()|
|MKLocalSearch|Apple Mapの検索機能を利用して周辺施設を検索するためのクラス|Apple Mapの検索機能を利用して周辺施設を検索するためのクラス|
|async / await|時間がかかる処理を、画面を止めずに実行するための仕組み|let response = try await search.start()|


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：request.region の latitudeDelta と longitudeDelta の値を 0.02 から 0.1 に変更して検索範囲を広げた。
- 結果：表示される施設の数が増え、現在地から離れた場所の施設も表示された。
- わかったこと：検索範囲の大きさは span の値によって変化し、数値を大きくするとより広い範囲を検索できることが分かった。

**実験2：**
- やったこと：検索カテゴリの「コンビニ」を「スーパー」に変更して、周辺検索の結果が変わるか確認した。
- 結果：地図上に表示される施設がコンビニではなく、近くのスーパーに変わった。
- わかったこと：searchNearby() は固定された施設を表示しているのではなく、渡されたキーワードを使って動的に検索していることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
もし経験豊富なエンジニアがこのアプリをゼロから設計する場合、どのような流れで考えるのか。
   **得られた理解：**
最初からコードを書くのではなく、まず必要な機能やユーザーの操作の流れを考えることが重要だと分かった。
このアプリでは「現在地の取得」「周辺検索」「地図表示」という3つの役割に分けて設計されていることを理解した。

2. **質問：**
LocationManager はなぜ必要なのか。
   **得られた理解：**
最初は ContentView の中に位置情報の処理を書いても良いと思った。
しかし、位置情報の許可、GPSからの取得、位置情報の更新など多くの処理があるため、LocationManager にまとめることでコードが整理されることが分かった。

3. **質問：**
searchNearby() はどのような流れで周辺検索を行っているのか。
   **得られた理解：**
現在地を取得した後、検索キーワードと検索範囲を設定し、Apple Map に検索を依頼していることを理解した。
また、検索結果が更新されると SwiftUI の仕組みによって地図の表示も自動的に変わることが分かった。

## この章のまとめ
今回の章では、地図アプリは単に地図を表示するだけではなく、位置情報の取得、周辺検索、検索結果の表示という複数の機能が連携して動いていることを学んだ。
特に印象に残ったのは、画面の表示を直接変更するのではなく、データが変化すると SwiftUI が自動的に画面を更新する仕組みである。
また、LocationManager のように役割ごとに処理を分けることで、コードが読みやすくなり、後から機能を追加しやすくなることを理解した。
最初は地図アプリのコード量が多くて複雑に感じたが、現在地、検索、表示という順番で分けて考えることで、全体の流れを理解できるようになった。

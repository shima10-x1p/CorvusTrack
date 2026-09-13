# CorvusTrack 技術選定

調査・選定日: 2026-09-13
状態: 技術選定完了。依存関係の解決・ビルド・実機検証は未実施。
対象: Android 17以上の個人用アプリ。実装開始前の選定であり、本書ではアプリを作成していない。
要件: [基本設計](basic-design.md)、[画面設計](screen-design.md)

## 1. 結論

Android専用のKotlinアプリとして作る。画面はJetpack ComposeとMaterial 3 Expressive、地図はMapbox、位置記録は位置情報用Foreground Serviceで動かす。記録データはRoom、設定はDataStoreに分けて保存する。

今回最も重視するのは、画面を消したりレイアウトを変更したりしても、記録処理が独立して継続すること。クロスプラットフォーム化より、Androidの権限・サービス・通知と素直に接続できる構成を選ぶ。

| 担当 | 採用 | わかりやすく言うと | 今回選ぶ理由 |
|---|---|---|---|
| 言語 | Kotlin | アプリを書く言語 | Composeや非同期処理と一貫した書き方ができる |
| 画面 | Jetpack Compose | 状態に応じて画面を描く仕組み | 同じ情報部品を縦・横・Foldで並べ直せる |
| デザイン | Compose Material 3のExpressive系列 | ボタン・カード・色・動きの共通部品 | 合意済みのM3Eを公式部品で実現する |
| 画面遷移 | Navigation 3 | 地図・履歴・設定の移動と戻る履歴の管理 | Compose専用の新規アプリに適している |
| 画面領域 | Material 3 Adaptive＋WindowManager | 幅・高さ・折り目を知る仕組み | 機種名ではなく利用可能領域でバー／レールとカードを切り替える |
| 地図 | Mapbox Maps SDK v11＋公式Compose拡張 | 地図と軌跡を描く仕組み | 採用済みの方針。地図表示をComposeへ組み込める |
| 測位 | Google Play services Fused Location Provider | GPS等を利用して位置を受け取る仕組み | 対象のPixelに適し、位置取得間隔・精度要求を指定できる |
| 継続記録 | Androidのlocation型Foreground Service | 通知を伴って画面外でも動く記録処理 | 開始から停止まで連続するユーザー主導の作業に合う |
| 記録保存 | Room 2系＋KSP | SQLite DBをKotlinから扱う仕組み | セッションと大量の位置点を随時保存・検索・再集計できる |
| 設定保存 | 型付きDataStore＋Kotlin SerializationのJSON | 小さな設定をまとめて安全に更新する仕組み | 表示順や将来の配置設定を構造として持てる |
| 状態管理 | ViewModel＋Coroutines＋StateFlow | 画面状態と非同期のデータ更新をつなぐ仕組み | 記録の最新状態を複数画面へ届けられる |
| 構成 | 単一appモジュール、手動の依存注入 | 処理を分け、必要な部品をコンストラクタで渡す | 個人用4画面で初期のビルド設定を増やしすぎない |

UI・データ層の分離、Compose、Navigation 3、FlowはAndroid公式の推奨と一致する。手動の依存注入も小規模アプリで認められた選択肢。本アプリではDB・位置取得・時計を交換可能にするために利用する。[公式アーキテクチャ推奨](https://developer.android.com/topic/architecture/recommendations)

## 2. アプリ識別とビルド環境

- 表示名: `CorvusTrack`
- applicationId / namespace: `net.shima10x1p.corvustrack`
- `minSdk = 37`、`targetSdk = 37`、`compileSdk = 37`を選定する。
- Android 17はAPI 37。古いAndroid 4.2の「API 17」とは異なる。
- ビルド定義はGradle Kotlin DSL、ライブラリの版はVersion Catalogに集約する。
- Gradle WrapperをGit管理し、PCごとの差を減らす。動的な`+`指定は使わない。

Android 17の公式セットアップはcompile/target 37を案内している。一方、同ページにはPreviewという表記や古いAGP最低要件も残るため、対応可否はAGP側のAPI対応表と突き合わせた。[Android 17 SDK](https://developer.android.com/about/versions/17/setup-sdk)

### 導入時に固定する基準版

以下は公式の公開情報で選んだ開始点であり、組み合わせ全体のビルド成功を意味しない。初回構築でMavenの実体・推移的依存関係・Kotlinメタデータを確認し、最終ロックする。

| 部品 | 選定した基準 | 根拠・扱い |
|---|---|---|
| Android SDK | API 37 | 合意した最低OSと一致。QPRプレビューのAPIは使わない |
| Android Gradle Plugin | 9.2.1 | API 37.0対応が明記された9.2系の修正版。最新版を理由なく追わない |
| Gradle | 9.4.1 | AGP 9.2の互換性表に合わせる |
| JDK / JVM target | 17 | AGP 9.2の要件に合わせ、ビルド用JDKを明示固定 |
| Kotlin | AGPのbuilt-in Kotlin | Kotlin Androidプラグインを二重適用しない。具体的なコンパイラ版はAGP解決結果に合わせて固定 |
| Compose compiler / Serialization plugin | 上記Kotlinと同じ版 | 古いcomposeOptionsのサンプルをコピーしない。初回依存解決時の確定対象 |
| Compose UI / Foundation / Runtime | 1.12.1を安定版の基準 | M3Eがより新しい依存を要求した場合は必要なUI範囲だけ整合させる |
| Material 3 | 1.5.0-alpha28 | M3E採用のための意図的な開発版採用 |
| Material 3 Adaptive | 1.3.0 | レイアウト情報に使用。M3のナビゲーション部品とは役割を区別 |
| Navigation 3 | 1.1.7 | 安定版。1.2 RCを今は選ばない |
| Room | 2.8.5 | Android単独の記録DBに採用 |
| DataStore | 1.2.1 | 型付きJSON設定を採用 |
| Activity / Lifecycle / Window | 1.13.0 / 2.11.0 / 1.5.1 | AndroidXの安定版を基準とする |
| Mapbox | 11.30.0 | Maps本体・Compose拡張の版とNDK系列を揃える |
| Play services location | 21.4.0 | Googleのセットアップ表で確認 |
| KSP・Coroutines・Serialization runtime | 選定済みの方式に合う安定版 | 細かな版の固定はKotlinとRoomの依存解決時に行う |

AGP 9.2の表ではGradle 9.4.1・JDK 17・最大API 37.0が確認できる。Build ToolsはAGP既定を基準とし、SDK 37ガイドが案内する37系も含めて初回ビルド時に整合を確認する。アプリ自身はC++を書かないので、Mapboxのビルド済みライブラリを使うためだけに独自NDKビルドは作らない。[AGP 9.2](https://developer.android.com/build/releases/agp-9-2-0-release-notes)

AGP 9ではKotlinが組み込まれ、従来のkotlin-androidを重複適用できない。Roomのコード生成にはKSPを選び、kaptは新規導入しない。[Built-in Kotlin](https://developer.android.com/build/migrate-to-built-in-kotlin)

ライブラリ版の確認元: [AndroidX一覧](https://developer.android.com/jetpack/androidx/versions)、[Google Play servicesセットアップ](https://developers.google.com/android/guides/setup)。Android StudioはこのAGPに対応する安定版を使用し、ローカルのインストール状況は実装開始時に確認する。

## 3. M3Eと画面構成を選ぶ理由

Composeなら「速度の部品」「配置」「記録の状態」を分離できる。画面回転やFoldの開閉で配置だけを変える今回の設計に合う。XML/View主体は採用しない。Mapbox拡張で不足する低水準操作だけ、地図の境界内で補う。

調査日時点でMaterial 3の安定版は1.4.0、開発版は1.5.0-alpha28。1.4系列からExpressiveの実験APIが取り除かれ、1.5系列を使う案内がある。したがって「安定版だけでM3Eがすべて使える」とは扱わず、1.5系列を採用する。alphaは変更があり得る段階という意味で、更新時には画面側の修正が必要になる可能性がある。[Material 3リリースノート](https://developer.android.com/jetpack/androidx/releases/compose-material3)

対処としてM3Eの利用を`ui/designsystem`とナビゲーションの外枠に寄せ、記録・保存へUI APIを渡さない。必要な箇所だけ実験APIをopt-inし、版は固定する。これはプロセス分離ではなくコードの責務分離であり、UIのクラッシュから記録が完全に隔離されるという意味ではない。

Navigation 3は画面移動の履歴を管理する。Navigation Bar/Railはその移動を指示する見た目の部品であり、別の役割。現在地・記録状態は画面の戻る履歴へ保存せず、記録側で所有する。履歴詳細へ渡すのはセッションIDとし、全位置点を画面遷移の引数へ詰め込まない。

横・Foldで左レールにする切り替え条件は明示する。SDKの既定値が常に希望通りの向きを選ぶとは限らない。またAndroid 17では大画面の向き・サイズ制限に関する従来のopt-outがなくなるため、回転を禁止する設計にはしない。[Android 17の大画面変更](https://developer.android.com/about/versions/17/behavior-changes-17)

## 4. 地図と位置取得を分ける理由

### Mapbox

`com.mapbox.maps:android-ndk27:11.30.0`と`com.mapbox.extension:maps-compose-ndk27:11.30.0`を採用する。16KBページサイズ対応の系列を揃える。標準版とndk27版を混ぜない。[公式導入手順](https://docs.mapbox.com/android/maps/guides/install/)

軌跡はGeoJSONの線を地図レイヤーに渡す方式とし、区切られた区間を別LineStringとして扱う。点ごとに大量のUIマーカーを作らない。長時間記録では更新中の区間と確定区間を分け、全履歴の作り直しを避ける。表示用の簡略化は生データに適用しない。

位置の正本は記録側とする。記録中のマーカー・追従カメラも同じ位置を使い、Mapbox独自の位置購読を別に動かして記録と表示が食い違うことを避ける。待機中は画面表示中だけ現在地更新を行い、権限未許可ならマーカーなしで地図を表示する。

料金表のMaps SDKs for Mobileは月25,000 MAUまで無料と確認できた。今回の個人利用はこの枠内を見込める。Navigation SDK・Search・有料の別APIは導入しない。[Mapbox料金](https://www.mapbox.com/pricing)

Mapboxのpublic access tokenをビルド時にローカル設定から注入する。public tokenはAPKから取り出せるので、秘密鍵と同じ秘匿性は期待しない。secret tokenや署名鍵はAPKにもGitにも入れない。現行導入手順のMaven設定には旧来のダウンロード用secret token認証が記載されておらず、古い手順を前提に不要な秘密鍵を要求しない。

帰属表示・ロゴ・Telemetryのopt-out導線を残す。端末内にGPSログを保存する方針と、地図SDKがネットワーク通信・Telemetryを行うことは区別する。「全情報が端末から一切出ないアプリ」とは説明しない。[Mapbox AttributionControl](https://docs.mapbox.com/android/maps/api/latest/mapbox-maps-android/com.mapbox.maps.extension.compose.ornaments.attribution/-map-attribution-scope/-attribution-control.html)

### Fused Location Provider

PixelのGoogle Play servicesを前提に採用する。端末のLocationManagerを直接使う案もあるが、今回Googleサービス非搭載端末への対応は不要なので、プロバイダ選択を自作する利点は小さい。Google Maps SDKやGCP課金プロジェクトを導入する選択とは別である。

高精度要求を基本とし、設定した1〜60秒をLocationRequestへ反映する。間隔はOSへの要求であり、必ずその秒数に正確に届く保証ではない。過去のキャッシュ点を開始直後の移動として加算せず、受信した点の取得時刻・単調時計・精度を検証する。間隔変更では登録を更新し、同一セッションを維持する。[LocationRequest.Builder](https://developers.google.com/android/reference/com/google/android/gms/location/LocationRequest.Builder)

同一取得時刻の重複や順序逆転に備え、未保存点を保存しつつ集計は時系列で行う。保存用のストリームは間引き・上書きしない。画面への状態通知だけ最新値へまとめる。

## 5. 消灯中も記録する方式

Foreground Serviceは、ユーザーが認識できる継続作業を画面の外でも実行するAndroidの仕組み。今回の開始→移動→停止に一致する。WorkManagerは後で実行する仕事や再試行向けで、1秒程度の連続測位の主役にはしない。ActivityやViewModelの寿命にも記録を結び付けない。

開始画面が表示されているうちに権限・端末設定を確認し、location型Serviceを開始する。Serviceは速やかにForeground化し、その後記録を実行する。`FOREGROUND_SERVICE`と`FOREGROUND_SERVICE_LOCATION`を宣言し、開始コマンドの重複でセッションを二重作成しない。[Service types](https://developer.android.com/develop/background-work/services/fgs/service-types)

### 権限

- `ACCESS_COARSE_LOCATION`と`ACCESS_FINE_LOCATION`を扱う。ユーザーが概算位置だけを許可した状態も識別する。
- 画面から開始したlocation型FGSの継続には、常時許可の`ACCESS_BACKGROUND_LOCATION`を初版では要求しない。バックグラウンドから新しく開始する機能・自動再開機能も作らない。[位置情報権限](https://developer.android.com/develop/sensors-and-location/location/permissions)
- `POST_NOTIFICATIONS`は記録通知のために要求する。ただし通知拒否だけでFGSが開始不能になるわけではない。拒否時は通知欄に記録通知が出ない場合があるため、「必ず通知が見える」とは保証しない。[通知権限](https://developer.android.com/develop/ui/compose/notifications/notification-permission)
- 位置情報設定OFFは取得不能として時間計測を継続。権限取り消しによるプロセス終了は別で、中断記録として回復する。

概算位置のみの場合に記録開始を止めるか、精度低下を表示して開始するか、および通知拒否時の案内文は未合意。技術選定は上記で完了できるが、実装時にはこの2つの利用時の挙動を決める必要がある。

### 終了・中断

`START_NOT_STICKY`を使い、プロセスが終了しても自動で記録を再開しない。再起動受信や予約ジョブによる復活も行わない。[Service API](https://developer.android.com/reference/android/app/Service)

OSから停止されたとき、終了コールバックが来るとは限らない。そのため`onDestroy`だけで保存する設計にしない。DBに残った記録中セッションを、新プロセスの回復処理で最終稼働時刻までの中断記録に変える。画面回転・画面の再生成をプロセス終了と混同しない。[ユーザーによるFGS停止](https://developer.android.com/develop/background-work/services/fgs/handle-user-stopping)

稼働時刻はまず5秒ごとに保存する調整案とする。測位間隔が60秒でも、測位とは別に実行する。5秒は厳密な遅延上限ではなく、OSの休止・I/O遅延で間隔が伸びる可能性がある。経過時間は単調時計で計算し、端末時計を変更しても距離・平均速度が壊れないようにする。通知の経過時間はシステムのchronometer利用を基本とし、毎秒通知を再投稿しない。

## 6. データ保存と内部構造

### Roomを記録に使う

セッション1件に位置点が多数ぶら下がり、位置点を追記し、履歴を日時順に読み、集計を更新する。これはDBが得意な処理。JSON/CSVを毎回書き換える方式やSharedPreferencesでは管理しにくい。RoomはSQLiteを扱いやすくし、クエリの検査とDB移行を支援する。

Room 3も安定版があるがKotlin Multiplatformを中心とする更新であり、Android専用の今回には2.8系を選ぶ。将来3へ移行する場合もRepositoryを境界にして画面への影響を抑える。[Room 3の位置付け](https://developer.android.com/jetpack/androidx/releases/room3)

- 生の位置点は削除せず、判定結果・理由・判定ロジックの版を別のフィールドで持つ。
- 品質判定前の点はpendingとして保存できるようにし、途中終了後にも判定を再実行できるようにする。
- 位置点にはセッションID、順序、取得日時、単調時計、精度、速度等を保存。存在しない速度・高度を0と決め付けない。
- セッションには状態、開始・終了、稼働時刻、集計値を保存。名前の追加に備える。
- segmentIdまたは不連続イベントを保存し、取得不能の前後を結ばない。
- セッション内の位置点を順に読む索引を作る。書き込みと集計の更新は整合するトランザクションで行う。
- DBスキーマをGit管理し、更新時は移行をテストする。破壊的なDB再作成で既存の記録を消さない。

### DataStoreを設定に使う

取得間隔・テーマ・消灯防止と、表示部品のID・順序・表示状態を型付き設定として保存する。将来のサイズや配置も同じ設定モデルへ追加できる。JSON形式のDataStoreを選び、Protocol Buffersのコード生成は初版では導入しない。大きな位置点の集合はここに入れない。[DataStore公式](https://developer.android.com/topic/libraries/architecture/datastore)

端末内保存はアプリ専用領域とする。将来のエクスポート形式はDBスキーマから分離し、版を持った外部形式として後で設計する。Androidの自動バックアップはアプリ専用領域も対象になり得るので、位置DB・設定のクラウドバックアップと端末転送の除外ルールを明示する案を採用する。機種変更機能を意図せず追加しない。[Auto Backup](https://developer.android.com/identity/data/autobackup)

### 構造

```text
app（単一Gradleモジュール）
  ui / designsystem / navigation  画面、カード、M3E
  tracking                       Service、開始停止、測位ストリーム
  domain                         品質判定、区間、距離・速度、時計の抽象
  data                           Room、DataStore、Repository
  di                             部品を組み立てるAppContainer
```

単一プロセスで開始する。Serviceと画面はRepositoryを共有し、Service側が記録の寿命を所有する。UIは状態を観測し、画面の購読がなくなっても記録の位置購読は止めない。記録エンジンの障害は握りつぶして「記録中」を出し続けず、保存可能な範囲を確定して失敗を示す。

Hiltは有力な代替だが、今回は手動のAppContainerとコンストラクタ注入で十分と判断する。必要以上のinterfaceや1関数だけのUseCaseは量産しない。時計・位置取得・保存など、テストで交換する境界は明確にする。

## 7. 採用しないもの

| 候補 | 今回採用しない理由 |
|---|---|
| Flutter / React Native / KMP | Androidだけの個人利用。継続記録・権限・M3Eをネイティブに実装する利点が大きい |
| Google Maps / MapLibre | Mapboxはユーザー選定済み。再選定を必要とする問題は今回見つかっていない |
| Mapbox Navigation SDK | 道案内は不要。地図表示と自前の位置記録だけで構成する |
| WorkManagerによる位置記録 | 連続する高頻度測位ではなく、延期可能な作業向け |
| 常時位置情報権限と自動起動 | 今回の操作フローに不要で、自動再開しない要件にも合わない |
| 全GPSログのJSON/CSV直接保存 | 随時追記・関連付け・検索・再集計はRoomが適する。外部形式は後で設計 |
| 多数のGradleモジュール | 初版規模ではビルド構成の管理負担が増える。責務はパッケージで分ける |
| クラウド同期・分析SDK・クラッシュ送信 | 現時点の要件にない。まずローカルログと実機検証で進める |

## 8. 次の工程で確認すること

今回確認したのは、公式情報に基づく機能・要件の適合と、候補版の公開状況。インストール、認証、依存解決、ビルド、実機テストは行っていない。したがって技術選定は完了だが、動作保証や実装完了ではない。

| 順序 | 確認 | 合格条件 |
|---|---|---|
| 1 | SDK・AGP・Kotlin・KSP・Composeの依存解決 | 選定版の実体を取得し、ビルドとRoomコード生成が通る。必要な調整版を記録 |
| 2 | Mapbox接続とM3Eの最小画面 | API 37で地図・帰属・バー／レール・カード・スナックバーが表示できる |
| 3 | 位置記録の縦断検証 | 開始→随時保存→消灯／他アプリ→停止が同一セッションで成立 |
| 4 | 中断 | OS停止・プロセス終了・再起動後に保存済み点が残り、自動再開せず中断として復元 |
| 5 | 品質と時間 | 1/5/60秒、途中の間隔変更、0点、位置飛び、取得不能、時計変更をテスト |
| 6 | 端末適合 | Pixel実機で回転・Fold開閉・文字拡大・電池消費・長時間記録を確認 |
| 7 | 日常利用版の更新 | 同じアプリID・署名鍵で更新し、既存DBが維持される |

日常利用版は専用の署名鍵で署名したAPKとし、開発中はADBでインストールする。テスト用debug版には`.debug`を付けて日常の記録と分ける。鍵・パスワードはGit外に保管する。鍵の作成や端末へのインストールは今回行わない。

なお、概算位置のみの許可・通知拒否の案内はユーザー動作として未合意。GPSの閾値、取得不能の検出、最終稼働保存周期は実測調整事項として残す。これらを未検証のまま「解決済み」とは扱わない。

# Dumb Watch Androidアプリ実装仕様書

## 0. この文書の目的

この文書は、Dumb WatchのAndroid側アプリを、別の開発者または将来のセッションから実装できるようにするための引き継ぎ仕様書である。

この文書だけで、以下を実装できる状態を目標とする。

- BLEスキャンと時計の発見
- 時計のアドレス保存と再接続
- 128x64画像の描画
- 1024バイトのSSD1306 page形式への変換
- BLEチャンク送信
- Foreground Serviceによるバックグラウンド動作
- 接続、送信、タイムアウト、再接続の状態管理

## 1. 現在の実装状態

### 1.1 リポジトリ

プロジェクト: `xiao_test2`

主なファイル:

- `spec.md`: 元のハードウェア・通信・ファームウェア仕様
- `src/main.cpp`: 現在のXIAO nRF52840 Sense側実装
- `platformio.ini`: PlatformIO設定
- `plan-dumbWatch.prompt.md`: ファームウェア実装計画

この文書の正本は `docs/android-app-spec.md` とする。チャット履歴が失われても、まずこの文書と `src/main.cpp` を確認すること。

### 1.2 ファームウェアとの現時点の契約

現在のファームウェアは、sleepを使わず、起動後にBLE広告を開始する。

実装済みの主な動作:

1. 起動時にBLE広告を開始
2. AndroidがService UUIDを検出
3. Androidが接続
4. RX Characteristicへ画像を複数回Write
5. ファームウェアが合計1024バイト受信した時だけOLEDを更新
6. 画像表示は約8秒
7. 表示終了後にBLE広告を再開
8. 接続後3秒以内に画像が完成しなければ、時計に以下を表示

```text
BLE TIMEOUT
NO IMAGE RECEIVED
```

注意: 現在は「ダブルタップした時だけ広告開始」ではない。ダブルタップとSystem OFFをAndroid側の前提にしてはならない。

## 2. BLE通信仕様

### 2.1 ロール

- 時計: BLE Peripheral / GATT Server
- Android: BLE Central / GATT Client
- OS標準のBluetooth Bondingは使用しない

### 2.2 UUID

Service UUID:

```text
7d5a0001-8f2a-4c7e-9b41-2c6b6b7a1001
```

RX Characteristic UUID:

```text
7d5a0002-8f2a-4c7e-9b41-2c6b6b7a1001
```

Android側ではUUIDを一箇所に定義し、画面やServiceから直接参照しない。

### 2.3 ServiceとCharacteristic

対象Service:

- UUID: 上記Service UUID
- 広告パケットにService UUIDが含まれる

対象Characteristic:

- UUID: 上記RX Characteristic UUID
- Writeを使用可能
- Write Without Responseを使用可能
- Android側は、Characteristicのpropertyを検査して利用可能な方式を選択する

### 2.4 画像の送信単位

画像は必ず1024バイトである。

- 1フレーム = 1024バイト
- ヘッダなし
- フレーム番号なし
- CRCなし
- ACKなし
- 再送要求なし

送信側は1024バイトをチャンクに分割する。各チャンクの境界に意味はない。ファームウェアは受信順に単純連結する。

送信例:

```text
payload[0..243]       -> 244 bytes
payload[244..487]     -> 244 bytes
payload[488..731]     -> 244 bytes
payload[732..975]     -> 244 bytes
payload[976..1023]    -> 48 bytes
```

チャンク長は固定値にせず、実効MTUに基づいて計算する。

### 2.5 MTU

Androidは接続直後にMTU 512を要求する。

```kotlin
gatt.requestMtu(512)
```

ただし、現在のXIAO側フレームワークはATT MTU最大247としてビルドされている。そのため、実効的な最大ATT payloadは通常244バイトである。

Android側の扱い:

- `requestMtu(512)`を試行する
- `onMtuChanged`で実効値を保存する
- 失敗または247未満の場合は、実効MTUからチャンクサイズを計算する
- アプリが512バイトを前提に固定送信してはならない
- `payloadSize = max(20, mtu - 3)`を基本値とする

### 2.6 Write方式

初回実装では、以下の優先順位を推奨する。

1. `WRITE_TYPE_NO_RESPONSE`
2. 使用できない場合は`WRITE_TYPE_DEFAULT`

Write Without Responseは高速だが、アプリ側で全Write完了を確認しにくい。したがって、送信完了直後に切断せず、短い安定化待ちを置く。

初回実装の送信完了条件:

- 全チャンクの`writeCharacteristic`投入が完了
- 最終チャンク送信後、少なくとも100ms待機
- 必要に応じて接続を維持したまま時計の表示更新を待つ
- タイムアウト時は接続を閉じてエラー扱い

将来、欠落が確認された場合は、フレームヘッダ・CRC・ACKを追加する。この場合は本仕様書を改訂する。

## 3. Androidの状態機械

アプリのBLE処理は、UIの状態ではなく、次の通信状態として管理する。

```text
IDLE
  -> SCANNING
  -> CONNECTING
  -> DISCOVERING_SERVICES
  -> REQUESTING_MTU
  -> READY
  -> SENDING
  -> WAITING_DISPLAY
  -> READY
```

異常時の遷移:

```text
SCANNING              -> IDLE       (scan timeout)
CONNECTING            -> READY      (retry可能)
DISCOVERING_SERVICES  -> READY      (retry可能)
REQUESTING_MTU        -> READY      (MTU失敗時も実効値で継続)
SENDING               -> READY      (切断・送信timeout)
WAITING_DISPLAY       -> READY      (表示待ちtimeout)
すべての状態          -> SCANNING   (時計アドレス未保存または再探索要求)
```

UIに公開する最小状態:

- 時計を探しています
- 接続中
- 画像を送信中
- 表示完了
- 時計が見つかりません
- Bluetooth権限が必要です
- Bluetoothが無効です
- 送信に失敗しました

## 4. スキャンと時計の紐付け

### 4.1 初回登録

1. ユーザーがアプリの時計登録画面を開く
2. AndroidのBluetoothが有効であることを確認
3. 必要なBluetooth権限を確認
4. Service UUIDフィルタ付きスキャンを開始
5. 広告に対象Service UUIDを含む端末だけを候補にする
6. 候補を表示し、ユーザーが時計を選択
7. Bluetooth device addressを永続化
8. スキャンを停止
9. 接続してService/Characteristicを確認

現在のファームウェアは起動時から広告するため、登録時にダブルタップを要求しない。

### 4.2 保存データ

DataStore Preferencesを推奨する。

保存キー:

```text
watch_address: String
watch_name: String? 
last_success_at: Long?
```

MACアドレスはAndroidのBluetooth APIから取得した値を保存する。Service UUIDは固定値なので保存不要。

### 4.3 再探索

次の場合は保存アドレスだけに固執せず、Service UUIDスキャンへ戻る。

- `connectGatt`に失敗
- `GATT_ERROR`が連続する
- Service UUIDが見つからない
- RX Characteristicが見つからない
- ユーザーが「時計を再登録」を選択

MACアドレスのランダム化やOSバージョン差を考慮し、再登録時は保存アドレスを上書きする。

## 5. Android権限とバックグラウンド動作

### 5.1 権限

対象Androidバージョンに応じてManifestとruntime permissionを実装する。

Android 12以降で最低限検討する権限:

- `BLUETOOTH_SCAN`
- `BLUETOOTH_CONNECT`

位置情報権限が必要な対象APIレベルでは、端末仕様に合わせて追加する。権限拒否時はクラッシュせず、設定画面への導線を表示する。

### 5.2 Foreground Service

アプリが画面外でも時計を監視するため、BLE管理はForeground Serviceへ置く。

Service要件:

- 常駐通知を表示
- 通知チャンネルを作成
- `startForeground()`をサービス開始直後に呼ぶ
- BLEスキャン・接続・再接続・送信をServiceが管理
- ActivityはServiceの状態を購読するだけにする
- Service停止時にスキャンとGATT接続を閉じる

### 5.3 スキャン設定

- `ScanSettings.SCAN_MODE_LOW_LATENCY`を登録時に使用
- 継続監視では電池消費を考慮してスキャン時間を制限
- Service UUIDの`ScanFilter`を使用
- 保存アドレスがある場合は、アドレスフィルタを優先してもよい
- 同じ端末からの結果を重複排除する

## 6. 画像レンダリング

### 6.1 描画キャンバス

Android側で、時計の表示内容を128x64ピクセルのオフスクリーンBitmapへ描画する。

- Bitmapサイズ: `128 x 64`
- 色形式: `ARGB_8888`または同等の8bit輝度を取得できる形式
- 背景: 黒
- 点灯画素: 白
- グレースケールの中間値は最終変換時に二値化
- ディザリングはしない

描画処理はUIスレッドから分離し、画像生成中も画面操作を止めない。

### 6.2 Pixel Shift

焼き付き防止のため、描画ごとにオフセットを変える。

- Xオフセット: -2, -1, 0, 1, 2のいずれか
- Yオフセット: -2, -1, 0, 1, 2のいずれか
- 連続送信で同じ組み合わせを避けることが望ましい
- 移動後の画素は128x64の範囲外を破棄
- 時計の外周近くの重要情報が欠けないよう、コンテンツ側に余白を確保する

### 6.3 二値化

各画素のRGB平均を使用する。

```text
luminance = (R + G + B) / 3
luminance >= 128 -> 1
luminance <  128 -> 0
```

アルファ値は、描画前に黒背景へ合成する。アルファ値を無視して白として扱わない。

### 6.4 SSD1306 page形式への変換

出力は1024バイトのByteArrayとする。

配列インデックス:

```text
index = page * 128 + x
page = y / 8
bit = y % 8
```

白画素の場合:

```kotlin
output[page * 128 + x] =
    (output[page * 128 + x].toInt() or (1 shl (y % 8))).toByte()
```

仕様:

- Page 0はY=0..7
- Page 1はY=8..15
- Page 7はY=56..63
- 各Page内はX=0..127
- 上側の画素はLSB
- 下側の画素はMSB
- 出力長は常に1024バイト

重要なテストパターン:

| 点灯画素 | 期待される出力 |
|---|---|
| `(0, 0)` | `output[0] = 0x01` |
| `(0, 7)` | `output[0] = 0x80` |
| `(0, 8)` | `output[128] = 0x01` |
| `(127, 63)` | `output[1023] = 0x80` |
| 全消灯 | 1024バイトすべて`0x00` |
| 全点灯 | 1024バイトすべて`0xFF` |

## 7. 送信処理

### 7.1 送信前チェック

送信前に次を確認する。

- Bluetoothが有効
- 必要な権限がある
- GATT接続済み
- Serviceが見つかっている
- RX Characteristicが見つかっている
- payload長が1024
- payloadが空でない

### 7.2 送信キュー

BLE APIのWrite完了コールバックを待ってから次のチャンクを送る。複数のWriteを同時に発行しない。

疑似コード:

```kotlin
suspend fun sendFrame(frame: ByteArray) {
    require(frame.size == 1024)

    val mtu = effectiveMtu
    val chunkSize = maxOf(20, mtu - 3)
    var offset = 0

    while (offset < frame.size) {
        val end = minOf(offset + chunkSize, frame.size)
        val chunk = frame.copyOfRange(offset, end)
        writeChunkAndAwaitResult(chunk)
        offset = end
    }

    delay(100)
}
```

実際のBLE API呼び出しは、使用するライブラリの仕様に合わせる。Android Nativeでは`BluetoothGatt.writeCharacteristic`とコールバック、または対応する非同期ラッパーを使用する。

### 7.3 送信失敗

次の場合は送信失敗として扱う。

- チャンクWriteが失敗
- GATT切断
- 送信全体が3秒を超過
- RX Characteristicが無効
- Bluetoothが無効化された

失敗時:

1. Writeキューを破棄
2. GATTを安全に閉じる
3. Service状態を無効化
4. 最大2回まで再接続して再送
5. それでも失敗したらUIへエラー通知

再送はフレーム全体を先頭から行う。途中チャンクから再開してはならない。

## 8. 時計のタイムアウトとの対応

時計側のタイムアウトは約3秒である。したがって、Android側は次の時間制限を守る。

- 接続完了からMTU/Service discoveryを速やかに完了
- 画像分割とWriteの開始を接続後すぐに行う
- 1024バイト全体の送信を3秒以内に完了させる
- 送信前にネットワーク通信や重い描画処理を挟まない
- 天気やカレンダーの取得は、BLE接続前に完了させるか、前回値を使用する

時計側が`BLE TIMEOUT`を表示した場合、Android側は同じ接続を再利用せず、一度切断して再接続する。

## 9. バックグラウンド更新方針

初回実装では、以下の二つを分離する。

### 時計をタップした時の即時表示

現在のファームウェアは常時広告のため、Android Serviceが保存アドレスを監視し、接続可能になったら画像を送る。

### 定期更新

定期更新はAndroidのWorkManagerまたはForeground Serviceのタイマーで実装する。ただし、CR2032電池とBLE接続時間を考慮し、次を守る。

- 不要なスキャンを常時実行しない
- 内容が変わらない画像を送らない
- 天気API取得中に時計接続を占有しない
- 画像更新周期を設定値にする

## 10. 推奨モジュール構成

Kotlin Android Nativeを推奨する。

```text
app/
  ble/
    WatchBleService.kt
    WatchGattClient.kt
    WatchScanner.kt
    BleModels.kt
  rendering/
    WatchCanvasRenderer.kt
    Ssd1306PageEncoder.kt
  storage/
    WatchPreferences.kt
  ui/
    PairingScreen.kt
    WatchStatusScreen.kt
  notification/
    WatchNotification.kt
```

責務:

- `WatchScanner`: スキャン、フィルタ、候補の通知
- `WatchGattClient`: 接続、Service discovery、MTU、Characteristic Write
- `WatchBleService`: Foreground Serviceと状態機械
- `WatchCanvasRenderer`: Canvas描画、Pixel Shift
- `Ssd1306PageEncoder`: 128x64 Bitmapから1024バイトへの変換
- `WatchPreferences`: アドレスと設定の保存
- UI: BLE処理を直接実行せず、状態を表示する

## 11. 最低限のテスト仕様

### 11.1 Encoder単体テスト

- 空Bitmapが1024バイトのゼロになる
- 全点灯Bitmapが1024バイトの`0xFF`になる
- `(0,0)`, `(0,7)`, `(0,8)`, `(127,63)`の境界を検証
- 出力長が常に1024
- 縦方向のbit順が逆になっていない

### 11.2 BLE単体・結合テスト

- Service UUIDで時計を発見できる
- 保存アドレスで再接続できる
- MTU 247で244バイトチャンクを送れる
- MTUが小さい場合も送れる
- 1024バイトを最後まで送れる
- 途中チャンク失敗時に全体を再送する
- 送信後に時計の表示が更新される
- 3秒以内に送信できない場合にエラーとなる
- 時計が`BLE TIMEOUT`を表示した後、再接続できる
- Bluetooth OFF/ONから復帰できる
- 権限拒否時にクラッシュしない
- Service停止後にスキャンとGATTが残らない

### 11.3 手動試験

1. 時計を起動し、Service UUIDで検出
2. Androidから全消灯画像を送信
3. 時計が真っ黒になることを確認
4. 全点灯画像を送信
5. 時計が全面点灯することを確認
6. テストパターンを送信し、上下左右の向きを確認
7. 時計を圏外にして送信失敗を確認
8. 1024バイト送信途中でBluetoothをOFFにする
9. 再接続後にフレーム先頭から再送する
10. Foreground Serviceが画面OFF後も動作することを確認

## 12. 未解決事項と将来拡張

以下は現時点で初回実装に含めない。

- 前回画像のFlash保存
- CRC
- フレーム番号
- ACK/NACK
- 部分再送
- 暗号化・認証
- 時計側のSystem OFF
- ダブルタップ時だけ広告する動作
- Android以外のCentral

画像だけを送る現在のプロトコルでは、BLEリンク層より上で欠落を検知できない。実機試験で送信欠落が確認された場合、次の仕様改訂で少なくとも以下を追加する。

```text
version | frame_id | chunk_index | chunk_count | payload_length | payload | crc
```

## 13. セッション再開用チェックリスト

別セッションで作業を再開する場合は、以下の順に確認する。

1. `docs/android-app-spec.md`を読む
2. `src/main.cpp`でUUIDと現在の広告・受信・タイムアウト動作を確認する
3. UUIDが以下と一致することを確認する
   - Service: `7d5a0001-8f2a-4c7e-9b41-2c6b6b7a1001`
   - RX: `7d5a0002-8f2a-4c7e-9b41-2c6b6b7a1001`
4. ファームウェアが1024バイトの単純連結を期待していることを確認する
5. Android側はMTU 512を要求しつつ、実効MTU 247以下にも対応する
6. Encoderの境界値テストを先に実装する
7. BLE接続より先にCanvas描画とEncoderを単体テストする
8. `pio run`でファームウェアの状態を確認する
9. Androidの結合試験では、まず244バイト単位の5チャンクを送る
10. 仕様を変更したら、この文書と`spec.md`を同時に更新する

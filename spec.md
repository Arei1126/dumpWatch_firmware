# Dumb Watch ソフトウェア開発技術仕様書

## 1. プロジェクト概要 (Philosophy)
本デバイスは「極限まで機能を削ぎ落としたリモートディスプレイ（Dumb Terminal）」である。
時計側のマイコンは一切のロジック（時計・フォント・状態遷移）を持たず、Android端末がレンダリングした128x64pxの1bit白黒ビットマップをBLE経由で受信し、表示するのみの役割を担う。

## 2. ハードウェア構成 (Target Hardware)
*       MCU: Seeed Studio XIAO nRF52840 Sense (LSM6DS3 6軸IMU内蔵)
*       Display: 0.96インチ OLED SSD1306 (I2C接続 / 4ピン / 白色)
*       Power: CR2032 (3.0V) コイン電池 + 220µF バイパスコンデンサ
*       Enclosure: 3Dプリント一体型（完全密閉、V字ノッチ破壊型）

---

## 3. 通信プロトコル仕様 (BLE Data Protocol)

### 3.1 接続ロール
*       Peripheral (GATT Server): Dumb Watch (XIAO)
*       Central (GATT Client): Android App

### 3.2 UUID定義
*       Service UUID: `[開発時に一意のUUIDを発行]`
*       RX Characteristic UUID: `[開発時に一意のUUIDを発行]` (Property: Write / Write Without Response)

### 3.3 MTU（Maximum Transmission Unit）
*       Android側から接続直後に **MTU 512 (最低247)** を要求（`requestMtu()`）。
*       1024バイトのデータをペイロード最大長（例: 244バイト）ごとにチャンク分割し、連続でWriteを実行する。

---

## 4. Androidアプリ仕様 (Central / 脳みそ)
**推奨環境**: Kotlin (Android Native) または Flutter

### 4.1 バックグラウンドBLEスキャン (Dozeモード対策)
Androidの省電力機能（Dozeモード）によるBLE切断を防ぎ、キミがタップした瞬間に即座に反応するための必須要件。
*       **Foreground Service**: アプリはフォアグラウンドサービスとして稼働し、システムに殺されないよう小さな常駐通知を表示する。
*       **ScanSettings**: `SCAN_MODE_LOW_LATENCY` を設定。
*       **フィルタリング**: 対象の `Service UUID` またはペアリング済みの `MACアドレス` のみを含むAdvertiseを対象とする。

### 4.2 ペアリング（紐付け）フロー
OS標準のBonding（設定画面からのBluetoothペアリング）は使用しない。
1.      アプリのScan画面を開く。
2.      時計をダブルタップしてAdvertiseを開始させる。
3.      アプリが対象Service UUIDを発見後、そのMACアドレスをローカルストレージ（SharedPreferences / DataStore等）に永続化。以降はこのMACアドレスのみを監視する。

### 4.3 盤面レンダリングパイプライン (Canvas -> 1KB Binary)
時計の描画内容はすべてAndroid内で完結させる。
1.      **Canvas描画**: `128x64` ピクセルのオフスクリーンCanvasに、時刻・天気・カレンダー等を白黒で描画。
2.      **焼き付き防止 (Pixel Shift)**: 毎回の描画時、キャンバスのオフセットをXY方向に `±1〜2ピクセル` の範囲でランダムにずらす。
3.      **二値化 (Binarization)**: ディザリングは行わず、輝度（RGB平均）が128以上を `1` (点灯)、未満を `0` (消灯) とする。
4.      **SSD1306 Page Addressing 変換**:
        *       SSD1306のVRAM構造に合わせ、縦8ピクセルを1バイト（8ビット）にパッキング。
        *       **エンディアン**: 上側のピクセルを LSB (ビット0)、下側のピクセルを MSB (ビット7) とする。
        *       **配列順序**: Page 0 (Y:0-7) の X:0〜127 を格納後、Page 1 (Y:8-15) の X:0〜127 を格納。これをPage 7まで繰り返し、**計1024バイトの一次元配列(`ByteArray`)**を生成する。

---

## 5. ファームウェア仕様 (Peripheral / 手足)
**推奨環境**: PlatformIO (Arduino framework / C++)

### 5.1 状態遷移 (State Machine)
複雑な状態を持たず、以下の一方通行のライフサイクルを繰り返す。
1.      `System OFF` (ディープスリープ / µAオーダーの待機)
2.      IMU (LSM6DS3) の INT1ピン 割り込み検知で起床
3.      BLE Advertise 開始 (タイムアウト設定: 3.0秒)
4.      Androidと接続確立 -> MTU拡張 -> データ(1024バイト) 受信
5.      OLED (SSD1306) へI2Cでデータ一括転送
6.      **8.0秒間** の画面点灯待機 (点灯中に再タップされた場合はタイマーをリセット)
7.      OLED表示オフコマンド送信 -> BLE切断 -> `System OFF` へ移行

### 5.2 IMU割り込み (ダブルタップ検知)設定
歩行ノイズ等による誤検知（チャタリング）を防ぐため、LSM6DS3のレジスタを厳格に設定する。
*       `TAP_CFG` / `TAP_THS` / `INT_DUR2`: ダブルタップ専用設定。
*       **デバウンス処理**: 画面点灯状態（8.0秒間）に入った直後、一時的にIMUの割り込みを無視するかINTピンの検知をデタッチし、消灯直前に再アタッチする。

### 5.3 通信エラー・フェイルセーフ処理
パケット欠落やAndroid端末圏外時のUXを損なわないための安全装置。
*       **バッファ整合性検証**: 受信したデータサイズが `1024バイト` に満たない状態でBLEが切断された場合、そのバッファは破棄しOLEDに転送しない（砂嵐防止）。
*       **タイムアウト時の表示**: 起床後3.0秒以内にAndroidから完全なデータが届かなかった場合、MCUのRAM/Flashに保持されている「前回受信した1KBのキャッシュ」をOLEDに表示し、安全にスリープへ移行する。

### 5.4 I2C通信要件
*       描画ライブラリ（U8g2等）の利用は任意だが、究極の軽量化を狙う場合は、I2C API (`Wire.h`) を用いてSSD1306へ初期化コマンドと1024バイトのデータペイロードを直接流し込むベアメタル実装を推奨する。
*       I2Cクロック周波数: `400kHz` (Fast Mode) を設定し、転送遅延を最小化する。
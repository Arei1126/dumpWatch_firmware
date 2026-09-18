## Plan: Dumb Watch Firmware

仕様書を実装可能な形に補足し、XIAO nRF52840 Sense向けのBLE受信、SSD1306表示、LSM6DS3ダブルタップ起床、8秒表示、System OFFを段階的に実装・検証する。既存のmain.cppはSSD1306サンプルを置き換え、PlatformIO/Arduinoの既存ライブラリを優先する。

## 実装の指針
- 人間の実装者が理解できるような形にするため、最適化よりも可読性や素朴さを重要視する
- 必要に応じて、実装例やコメントを記載する
- pioコマンドは~/.platformio/penv/bin以下に存在するので、こちらを利用すること(パスは通っていない)

**Steps**

### Phase 0: 仕様・配線の確定
1. 開発用UUIDを仕様書へ固定する。候補: Service `7d5a0001-8f2a-4c7e-9b41-2c6b6b7a1001`、RX `7d5a0002-8f2a-4c7e-9b41-2c6b6b7a1001`。Android側も同じ値を使用する。
2. XIAOの標準I2CをSDA=D4、SCL=D5として記載し、OLEDアドレスは実機スキャンで`0x3C`/`0x3D`を確定する。LSM6DS3 INT1は基板回路図と実機導通でGPIO番号を確定し、`IMU_INT1_PIN`として一箇所に集約する。電池直結でOLEDとXIAOが許容する電圧、220uFの実装可否も確認する。
3. 画像ペイロードは当面1024バイトのみとする。受信は接続単位で`rxBuffer`を0から開始し、各Writeのバイト列を順番に累積する。累積長が1024になった時だけ完成扱いにし、それを超えるWrite/長さ不整合/切断時の未完成データは破棄する。フレームヘッダ、連番、CRC、再送は初回スコープ外だが、受信処理を将来追加できる関数境界にする。
4. 「前回キャッシュ」は初回実装しない。RAM上の受信完了画像だけを表示対象とし、未受信時は表示更新せず消灯・切断・スリープへ進む。Flash/Preferencesの保存は別フェーズとする。
5. BLEの`requestMtu(512)`はAndroid側の要求であり、Peripheral側は実効Write長を受け入れる。MTUが247以上でなくても受信累積できるよう、BLEコールバックはチャンク長に依存しない。連続Write Without Responseの完了保証が必要になった場合は、将来のACK/CRC拡張で対応する。

### Phase 1: 最小ビルドと表示経路
6. `src/main.cpp`からサンプルアニメーションを除去し、定数、BLE/IMU/OLED初期化、状態管理、コールバックを分離する。依存は既存のAdafruit SSD1306/GFXを優先し、BLEはボードコアで提供されるAPIを確認して採用する。不足する場合だけPlatformIO依存を追加する。
7. Wireを400 kHzで初期化し、SSD1306を初期化する。1024バイトのpage形式をそのままAdafruit SSD1306のGFXバッファへコピーするか、直接I2C転送するかを小さな表示ドライバ関数に隠蔽する。仕様のビット順はpage内でbit0が上位Y、bit7が下位Yであることをテストパターンで検証する。
8. 画像受信なしでも、固定パターンを表示してOLEDのアドレス、向き、page形式、表示オン/オフを確認する。

### Phase 2: BLE Peripheralと受信
9. 固定UUIDのService/RX Characteristicを作り、WriteとWrite Without Responseを有効にする。起床後だけadvertiseし、3秒のadvertise/受信待ちタイムアウトを状態機械で管理する。
10. 接続、切断、Writeコールバックからメインループへフラグ/受信長を渡す。BLEコールバック内ではOLED転送、長い待機、System OFFを行わない。新規接続/再接続で受信バッファと完了フラグを確実にリセットする。
11. 1024バイト完成時のみ表示処理へ渡す。切断または3秒経過時に未完成なら破棄する。受信完了後に接続を維持する時間、切断タイミングを定数化する。
12. Android側の送信検証用に、固定1024バイトパターンを分割Writeする簡易テスト手順またはテストスクリプトを用意し、247/512 MTU、異なるチャンク長、途中切断、1024バイト超過を確認する。

### Phase 3: IMU起床と低消費電力
13. LSM6DS3をI2Cで初期化し、`TAP_CFG`、`TAP_THS`、`INT_DUR2`、必要な軸/割り込みルーティングをデータシートと実機で設定する。ダブルタップの閾値は機械筐体・装着条件に依存するため、初期値を定数化しシリアルログで調整できるようにする。
14. INT1 GPIOの割り込みを短いISRでラッチし、起床後にイベントを消費する。表示中はIMU割り込みを無視または割り込み入力を無効化し、8秒終了時に再アームする。再タップで8秒タイマーをリセットする仕様を採用する。
15. Nordic nRF52840のSystem OFF移行APIとGPIO/IMU割り込みによるwake設定を確認し、すべてのペリフェラル停止、OLED display off、BLE停止、GPIO状態を整えてからスリープする。System OFFからの復帰がリセット起動になるかを前提に、起動理由を確認して通常の待受フローへ入る。
16. 消費電流をUSB給電時とCR2032実機で測定し、System OFF時がµAオーダー、起床中の最大電流とOLED/BLE同時動作が電池・コンデンサの条件内であることを確認する。

### Phase 4: 統合・堅牢化
17. 状態を`SLEEP`, `ADVERTISING`, `CONNECTED_RECEIVING`, `DISPLAYING`, `SHUTDOWN`等に限定し、各遷移条件とタイムアウトをコード上の単一箇所に集約する。
18. シリアルログをデバッグビルド限定にし、受信長、接続、タイムアウト、画像更新、起床理由だけを記録する。通常運用で常時ログ待ちしない。
19. Android仕様書にも、UUID、広告開始条件、スキャンフィルタ、MTUは最大512/最低247のベストエフォート、実際のチャンク長、連続Write完了待ち、切断時の再接続方針、1KB送信完了の判定を追記する。Foreground Service/Doze対策はAndroid側の実装範囲として分離する。

**Relevant files**
- `/home/yuuri/Documents/PlatformIO/Projects/xiao_test2/spec.md` — UUID、ピン、OLEDアドレス、初回キャッシュ非対応、画像のみプロトコル、タイムアウトと状態遷移を追記する。
- `/home/yuuri/Documents/PlatformIO/Projects/xiao_test2/src/main.cpp` — 既存サンプルを置き換え、BLE受信、状態機械、OLED表示、IMU割り込み、低電力移行を実装する。
- `/home/yuuri/Documents/PlatformIO/Projects/xiao_test2/platformio.ini` — BLE/IMU依存がボードコアで不足する場合のみ依存ライブラリ、テスト用build flagを追加する。
- `/home/yuuri/Documents/PlatformIO/Projects/xiao_test2/test/` — 1024バイト累積、境界値、pageパッキングのホスト側または実機テストを追加する。PlatformIOの通常テストでハードウェア依存部分を分離できない場合は、純粋な受信/変換関数をテスト対象にする。

**Verification**
1. `pio run`でビルドし、警告・未解決API・RAM使用量を確認する。
2. OLED固定パターンで、`0x3C`/`0x3D`、128x64、上下ビット順、全1024バイト表示、display offを実機確認する。
3. AndroidまたはBLEテストツールから1024バイトを複数チャンクで送信し、全量時のみ表示が更新されること、途中切断/超過/3秒タイムアウトで更新されないことを確認する。
4. ダブルタップでadvertiseが始まり、未接続なら約3秒で終了、接続・完全受信なら表示が8秒点灯、表示中の再タップでタイマーが延長されることを確認する。
5. BLE切断後と表示終了後にSystem OFFへ移行し、ダブルタップで復帰することを確認する。
6. 電流計でSystem OFF、advertise、BLE受信、OLED表示の各状態を測定する。
7. Android側でForeground Service、低遅延スキャン、Service UUID/MACフィルタ、DataStore等へのアドレス保存、MTU要求、チャンク送信完了待ちを確認する。

**Decisions**
- 初回はファームウェアとAndroid通信仕様を対象にする。Androidアプリ本体は別実装だが、相互運用に必要な送信手順と検証項目は仕様書に含める。
- UUIDは上記の開発用固定値を採用する。
- 初回データは1024バイトのみで、ヘッダ/CRC/連番/ACK/再送は追加しない。したがってBLEリンク層の信頼性に依存し、未完成フレームは破棄する。
- 前回画像のFlash/RAMキャッシュは初回対象外。未完成・未受信時に古い画像を表示しない。
- OLEDは標準I2C D4/D5を仮定するが、INT1 GPIOとOLEDアドレスは実機確認を必須とする。
- 既存Adafruit SSD1306/GFXを優先し、直接I2C実装は測定で必要な場合に限定する。

**Further Considerations**
1. LSM6DS3 INT1の正確なGPIO番号は、XIAO Senseの基板リビジョンと回路図を確認して実装前に確定する。
2. 「8秒間の点灯待機中に再タップ」の扱いは、表示中に新規BLE通信も受けるのか、タイマー延長だけにするのかを実装前に決める。推奨は初回では再タップをタイマー延長だけとし、BLE再接続/再送は次段階に分離する。
3. 画像のみプロトコルは欠落検知ができないため、実機でWrite Without Responseの連続送信信頼性を確認し、失敗があればCRC/フレーム番号/ACKを次の仕様改訂で追加する。

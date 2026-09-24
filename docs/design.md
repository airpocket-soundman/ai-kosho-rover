---
layout: page
title: 詳細検討
permalink: /design/
---

企画書の構想を、決定済みの制約(Geekservo DC モーター、LEGO フレーム、台の下に潜って持ち上げる搬送方式、ESP32-CAM ストリーミング、外部カメラの利用、自作の下半身検出モデル)に沿って具体化した結果です。
数値は公開されている仕様や計算値で、実測で更新します。未確認の項目は「要確認」と明記しています。

## 1. 全体構成

```text
[外部固定カメラ ESP32-CAM ×1〜2]   [ローバー前方カメラ ESP32-CAM ×1]
   玄関・廊下を俯瞰                    床上 8〜10 cm の低い視点
        │ Wi-Fi MJPEG                        │ Wi-Fi MJPEG
        ▼                                     ▼
┌──────────────────────────────────────────────────────┐
│ Arduino UNO Q  Linux 側 (Cortex-A53 ×4, Debian, Python)  │
│  ・AprilTag 検出 → ローバー位置 (x, y, θ)                  │
│  ・人物検出・人物識別・扉の開閉 (外部カメラ)                 │
│  ・下半身検出モデル (ローバーカメラ)                          │
│  ・状態系列の構築 → LAYA で高レベル行動を選択                 │
│  ・経路計画・ウェイポイント追従・ドッキング制御                │
└──────────────┬───────────────────────────────────────┘
               │ Arduino Bridge (RPC)  目標速度 vx, vy, ω / リフト指令
┌──────────────▼───────────────────────────────────────┐
│ Arduino UNO Q  MCU 側 (STM32U585)                        │
│  ・メカナム運動学 → 4 輪 PWM                               │
│  ・ToF / バンパー監視 → 非常停止 (Linux 側と独立)            │
│  ・指令が途切れたら停止するウォッチドッグ                     │
│  ・リフト用サーボ、電池電圧監視                              │
└──────────────┬───────────────────────────────────────┘
               │
   モータードライバ → Geekservo DC ×4 → メカナムホイール ×4
   サーボ → リフト
```

役割分担の考え方は次の通りです。

- 「何をするか」は Linux 側の Decision Model(LAYA)が決める。
- 「どう動くか」は Linux 側の古典制御(ウェイポイント追従・ドッキング)が決める。
- 「止まるべきか」は MCU 側が単独で決め、Linux 側が固まっても止まれる。

## 2. 機体(フレーム・駆動・車輪)

### 2.1 駆動モーター: Geekservo DC モーター(赤、2 線)

| 項目 | 値 |
| --- | --- |
| 動作電圧 | 3.3〜6 V(定格 4.8 V) |
| 最高回転数 | 70 rpm |
| 最大トルク | 500 g·cm |
| 定格電流 / 拘束電流 | 200 mA / 700 mA |
| 出力軸 | LEGO Technic 十字軸 |
| 備考 | ギア保護用クラッチ内蔵。極性反転で逆転 |

兄弟品の Geekservo 2 kg モーター(90 rpm、500 g·cm、両軸)はトルクが同じで速度が高いだけなので、まずは小型の 9 g 赤モーターを採用します。

### 2.2 車輪: 60 mm LEGO 互換メカナムホイール

Nexus Robot 14144 系の 60 mm LEGO 互換メカナム(左右 2 個ずつ、Technic ハブ付き)を第一候補にします。Geekservo の十字軸に直結できます。予備案として Technic 軸対応の 3D プリント 70 mm メカナム(Thingiverse / Cults3D に公開データあり)を用意します。

### 2.3 計算上の性能(要実測)

| 項目 | 計算値 | 前提 |
| --- | --- | --- |
| 最高速度 | 約 0.22 m/s | 70 rpm × π × 60 mm |
| 実用速度 | 0.10〜0.15 m/s | 負荷時の回転低下を見込む |
| 1 輪あたり推進力 | 約 170 gf | 500 g·cm ÷ 3 cm |
| 横移動時の合計推進力 | 約 330 gf | メカナム横移動の効率を 50 % と仮定 |
| 総重量の上限目安 | 約 1.5〜2.0 kg | フローリングの転がり抵抗を重量の 10 % 以下と仮定 |

フローリングでは成立する見込みですが、カーペットや段差では不足する可能性があります。不足した場合は LEGO ギア(12T:20T など)で減速してトルクを 1.5 倍程度に上げます。速度は約 45 rpm に落ちますが、廊下 5 m を 40 秒程度で移動でき、用途上は問題ありません。

### 2.4 フレーム

- LEGO Technic のビームとフレームで、平面に近い低い車台を作る。
- 目標寸法: 幅 180 mm × 長さ 200 mm 以内、車高(リフト下降時)90 mm 以下。台の下に潜れる高さで決める。
- 4 輪はモーターを直付けし、軸受けを Technic フレームで両持ちにして車輪のガタを抑える。
- UNO Q、電池、モータードライバはフレーム中央に載せて重心を低く保つ。
- ローバー上面中央に AprilTag(36h11、1 辺 60〜80 mm)を貼る。
- 前面下部に ESP32-CAM を床上 8〜10 cm の高さで前向きに固定する。

### 2.5 車輪エンコーダーを使わない判断

Geekservo DC モーターにはエンコーダーがありません。企画書ではエンコーダー + IMU の Dead Reckoning を想定していましたが、外部カメラで位置を推定してよいと決めたため、車輪エンコーダーは使いません。代わりに次の構成にします。

- 位置: 外部固定カメラで見たローバー上面の AprilTag から (x, y, θ) を約 10 Hz で推定。
- 短時間の補間: IMU(任意)のヨー角で回転を補う。
- ドッキング時の精密位置: ローバー前方カメラで台の脚に貼った小型 AprilTag を見て相対位置を出す。

## 3. 搬送: 台の下に潜って持ち上げる

倉庫ロボット(Kiva 方式)と同じ考え方で、荷物を載せた「台」の下にローバーが潜り込み、リフトで台ごと持ち上げて運びます。

### 3.1 台の設計

```text
       ┌───────────────────────┐  ← 天板: 財布・鍵・時計を置く
       │  父用 / 母用 / 子用    │
       └┬─────────────────┬───┘
        │                   │      ← 脚: 4 本。内側にドッキング用 AprilTag
        │   ローバーが潜る   │
        │   空間 100 mm 以上  │
   ─────┴───────────────────┴─────  床
```

- 台は LEGO で製作。脚の内寸はローバー幅 + 左右 20 mm 以上の余裕を取る。
- 台の下面中央に「受け皿」(凹み)を作り、ローバーのリフト先端の凸部が入ることで持ち上げ時に自動で芯出しされるようにする。
- 台の質量目標: LEGO 部分 200〜300 g、荷物 300 g 程度、合計 500〜600 g。
- 各台には人物ごとの識別用 AprilTag(小型)を脚に貼り、どの台かをカメラで確認できるようにする。

### 3.2 リフト機構

- 駆動: Geekservo 2 kg 270° サーボ(LEGO 互換、位置制御)×1。9 g 270° サーボを使う場合はてこ比で負荷を下げる。
- 方式: サーボでクランクを回し、平行リンクで天板を 10〜15 mm 上下させる。ストロークは台の脚が床から離れるだけで十分。
- 持ち上げ時の目標: 台 + 荷物 600 g をリフトし、走行中に傾かないこと。サーボの保持トルクは要実測。

### 3.3 ドッキング手順

1. 外部カメラの位置推定で台の正面 30 cm まで移動する。
2. 前方カメラで台の脚の AprilTag を検出し、左右ずれと角度を出す。
3. メカナムの横移動で中心を合わせ、低速で前進して潜り込む。
4. 前方 ToF で台の奥側脚との距離を見て停止する。
5. リフトを上げ、外部カメラで台の AprilTag が上昇(見かけの大きさが変化)したことを確認する。
6. 目的地へ搬送し、逆手順で降ろす。

## 4. 電装

### 4.1 コンピュータ: Arduino UNO Q

| 項目 | 値 |
| --- | --- |
| MPU | Qualcomm Dragonwing QRB2210、Cortex-A53 ×4 @ 2.0 GHz、Adreno GPU |
| MCU | STM32U585、Cortex-M33 @ 160 MHz、2 MB Flash、786 kB SRAM |
| RAM / ストレージ | 2 GB または 4 GB LPDDR4 / 16 GB eMMC |
| 無線 | Wi-Fi 5(2.4 / 5 GHz)、Bluetooth 5.1 |
| OS / 開発 | Debian Linux + Arduino App Lab(Sketch + Python + Bricks) |
| 拡張 | UNO ヘッダー、Qwiic(I2C)、MIPI-CSI カメラコネクタ |

LAYA を搭載する計画のため、**4 GB 版を推奨**します(第 6 節参照)。USB-C ポートは 1 つで給電に使うため、USB カメラは使わず、Wi-Fi のストリーミングカメラにします。

### 4.2 モータードライバ

- 第一候補: Adafruit Motor Shield V2 互換品(TB6612 ×2、DC モーター 4 個、I2C 制御、サーボ用ピン 2 本)。UNO ヘッダーに重ねるだけで配線が少なく、MCU 側の I2C だけで制御できる。
- 予備案: DRV8833 モジュール ×2 を MCU の PWM ピンで直接駆動する(PWM 4 本 + 方向 4 本)。
- モーター電源は 5 V(4.8 V 定格に近い)とし、UNO Q の電源とは分けて GND のみ共通にする。

### 4.3 電源

| 系統 | 供給 | 消費の目安 |
| --- | --- | --- |
| UNO Q | USB-C 5 V(3 A 出力のモバイルバッテリー、または 2S Li-ion + 5 V 3 A 降圧) | 推論時 1.5〜2 A(要実測) |
| モーター・サーボ | 2S Li-ion + 5 V 3 A 降圧、または同じモバイルバッテリーの 2 口目 | 4 モーター定格 0.8 A、拘束時最大 2.8 A + サーボ |

保護回路付きのモバイルバッテリー(USB PD 対応、300 g 以下)を第一候補にします。重量が課題になれば 2S Li-ion に切り替えます。MCU 側で電池電圧を監視し、低電圧でホームポジションへ戻す判断材料にします。

### 4.4 センサー

| センサー | 用途 | 接続 |
| --- | --- | --- |
| ToF VL53L1X ×4(前後左右) | 近接停止、壁との距離 | Qwiic / I2C(アドレス変更または I2C マルチプレクサ) |
| マイクロスイッチ バンパー ×4 | 接触時の非常停止 | MCU デジタル入力 |
| IMU(BNO055 または MPU-6050、任意) | ヨー角の補間 | I2C |
| 電圧分圧 | 電池監視 | MCU アナログ入力 |
| 前方 ESP32-CAM | 下半身検出、ドッキング用 AprilTag | Wi-Fi |

Multi-zone ToF(VL53L5CX)は企画書にありましたが、外部カメラで位置がわかる前提なので、第 1 段階では単点 ToF で十分と判断します。

## 5. カメラとストリーミング

### 5.1 カメラの配置

| カメラ | 位置 | 視野 | 役割 |
| --- | --- | --- | --- |
| 外部カメラ A | 廊下の天井付近または壁の高い位置、玄関方向を俯瞰 | 廊下 + 玄関全体 | ローバー位置推定、人物検出・識別、玄関到着、扉の開閉 |
| 外部カメラ B(任意) | 収納スペース側 | 収納スペース + 台 | 台の有無、ホームポジション |
| ローバー前方カメラ | 床上 8〜10 cm | 前方の下半身・足元・台の脚 | 靴の着脱、しゃがみ、荷物、ドッキング |

### 5.2 ESP32-CAM の運用

- AI-Thinker ESP32-CAM(OV2640)を標準品とし、MJPEG を HTTP で配信する。フレームレートが不足すれば XIAO ESP32S3 Sense に置き換える。
- 解像度は 640×480 で 10〜15 fps を目標にする。AprilTag 検出は 640×480、下半身検出は 320×320 に縮小して処理する。
- UNO Q 側は OpenCV の VideoCapture で URL を開き、フレーム取得→処理→結果を共有メモリ(または ZeroMQ)に書く独立プロセスにする。
- 遅延は 100〜300 ms を見込む。位置制御はこの遅延を前提に、速度を低めに(0.1 m/s 程度)して安定させる。
- 2.4 GHz Wi-Fi の混雑が懸念される場合は、ローバー専用の SSID を用意する。

### 5.3 外部カメラによる自己位置推定

1. 廊下の床に 4 点以上の基準 AprilTag を一時的に置き、画像座標と床座標のホモグラフィを求める(初回のみ)。
2. ローバー上面の AprilTag の中心をホモグラフィで床座標に変換し、タグの向きから θ を得る。
3. 高さ補正: タグは床から車高分だけ高いため、カメラ位置を基に補正する(校正時にローバー自身を使えば自動で吸収できる)。
4. 見えないとき(死角・遮蔽)は最後の位置を保持し、1 秒以上見えなければ停止する。

期待精度は ±3 cm、±3°(カメラ高さ 2.2 m、640×480 の場合の概算、要実測)。ドッキングの最終段は前方カメラの相対位置で行うので、外部カメラの精度はウェイポイント到達に足りれば十分です。

### 5.4 玄関イベントの検出

外部カメラ A で次を検出します。

- 人物検出(全身が見えるので既存の軽量モデルを使う。UNO Q の Video Object Detection Brick、または YOLO の小型モデル)。
- 人物識別: 家族 3〜4 人の識別を、全身画像の分類モデルで行う(自宅データで学習)。顔認識は使わない。
- 扉の開閉: 扉領域の画像変化 + リードスイッチ(ESP32 側の GPIO で検出し、同じ Wi-Fi で通知)。物理センサーを併用して誤検出を防ぐ。
- 位置の時系列: 人物の足元位置を床座標に変換し、「廊下→玄関へ接近」「玄関で停止」などを判定する。

## 6. 判断モデル: LAYA の位置づけ

### 6.1 LAYA の実態(調査結果)

| 項目 | 内容 |
| --- | --- |
| 種別 | 非自己回帰の System 1 Decision Model。テキストの「状態」と型付き質問を与え、1 回の順伝播で答える |
| 出力の型 | choice(選択肢から 1 つ)、score(段階)、noul(yes/no の確率)。各答えに確率と確信度が付く |
| チェックポイント | laya(ModernBERT-large、421M、512 トークン)、laya-multilingual(mmBERT-base、322M、1,024 トークン)、laya-typed-decisions(421M) |
| ライセンス | Apache 2.0(重み公開、ローカル実行可) |
| 実行環境 | PyTorch、ONNX Runtime。GPU T4 で 1 問 33〜40 ms、x86 CPU で 1 問 193〜464 ms |
| Jev との関係 | Jev は TypeSafe AI のクローズドなホスト API。LAYA はその互換のオープン実装 |

### 6.2 企画書からの変更点

企画書では LAYA に vx, vy, ω まで出力させることを目標にしていましたが、LAYA の出力は離散の選択・段階・yes/no であり、**連続値の速度指令は出せません**。したがって次のように分担します。

- LAYA: 高レベル行動の選択(WAIT / GET_FATHER_STAND / OBSERVE_LEFT / RETURN_HOME など)、外出・帰宅の判定(noul)、緊急度(score)。
- 古典制御: 選ばれた行動をウェイポイント列に展開し、外部カメラの位置推定を使って P 制御で追従。ドッキングは前方カメラの相対位置で制御。
- Active Perception: 「OBSERVE_LEFT / OBSERVE_RIGHT / OBSERVE_ENTRANCE」を行動の選択肢に含めることで、判断に迷うときに観測位置を変える動きを LAYA の選択として表現する。

### 6.3 状態テキストの例

```text
time: 07:42 weekday
weather: rain, 18C
door: closed (last opened 14 min ago)
person: father (0.83) hallway -> near_shoes -> bending -> standing near door
lower_body: shoes_on 0.91, bag 0.72
father_stand: stored at home slot
mother_stand: stored
rover: home, battery 78%, last_action WAIT x6

questions:
  next_action: choice [WAIT, GET_FATHER_STAND, GET_MOTHER_STAND, GET_CHILD_STAND,
                       BRING_UMBRELLA, RECEIVE_ITEMS, STORE_STAND, RETURN_HOME,
                       OBSERVE_ENTRANCE, OBSERVE_LEFT, OBSERVE_RIGHT]
  going_out: noul
  urgency: score 0-2
```

### 6.4 UNO Q での実行可能性(要検証)

| 項目 | 見込み |
| --- | --- |
| メモリ | 421M パラメータは fp32 で約 1.7 GB、int8 量子化で約 0.45 GB。2 GB 版では OS と画像処理を含めると厳しく、4 GB 版を推奨 |
| 遅延 | x86 CPU で 0.2〜0.5 秒なので、Cortex-A53 では 1〜3 秒程度と推定。1 Hz 未満の判断周期になる |
| 対策 | int8 ONNX 化、multilingual(322M)版の利用、状態テキストを 200 トークン以下に圧縮、判断周期を 2 秒に設定 |
| 代替 | 検証で成立しない場合は、自宅 PC(NVIDIA GPU)で LAYA を動かし LAN 経由で判断を返す。「家庭内ローカル」は保ちつつ、機体側の負荷を減らす |

さらに比較用に、同じ状態テキストから同じ行動を決めるルールベース版を用意し、LAYA の判断と一致率・誤介入率を比較します。

## 7. 認識モデル: ローバー視点の下半身検出

ローバーの前方カメラは床上 8〜10 cm にあるため、人の上半身は映りません。この視点の画像を自分たちで収集・ラベリングして検出モデルを学習します。詳細は[データセット計画](../dataset/)にまとめました。要点は次の通りです。

- クラス案: `feet_socks`、`feet_shoes`、`feet_bare`、`shoe_on_floor`(履かれていない靴)、`hand_low`(しゃがんで手が低い位置)、`bag`、`umbrella`、`stand_leg`(台の脚)、`door_gap`(扉の隙間)。
- モデル: YOLO 系の最小モデル(YOLOv8n / YOLO11n 相当)を自宅 PC の GPU で学習し、ONNX に変換して UNO Q の CPU で実行。目標 5 fps 以上。
- 代替: Edge Impulse で FOMO / YOLO を学習し、App Lab の Video Object Detection Brick として配備。
- 人物識別はこの視点では行わず、外部カメラの全身画像に任せる。

## 8. 安全設計

- MCU 側で ToF が 12 cm 未満、またはバンパー押下を検出したら、Linux 側の指令に関係なくモーターを停止する。
- Linux 側からの速度指令が 300 ms 途切れたら停止する(ウォッチドッグ)。
- 速度の上限を MCU 側で 0.15 m/s に固定し、Linux 側の不具合で暴走しないようにする。
- 物理的な電源スイッチをモーター電源側に付け、手で止められるようにする。
- リフトを上げたまま速度を出さない(上昇中は 0.08 m/s に制限)。
- 人の足元 30 cm 以内では自動的に減速する。
- 家庭内の映像・人物データはリポジトリに入れない(`.gitignore` で `data/`、`captures/` 等を除外済み)。

## 9. 未解決の課題

| 課題 | 現時点の見通し | 決める時期 |
| --- | --- | --- |
| Geekservo のトルク不足 | ギア減速で対応可能。カーペット走行は保証しない | フェーズ 1 の走行試験 |
| メカナムホイールの Technic 軸との嵌合 | 市販品はきつい場合がある。現物で確認 | 部品到着後 |
| ESP32-CAM の fps と遅延 | 640×480 で 10 fps 前後を期待。不足なら ESP32-S3 系 | フェーズ 2 |
| LAYA の UNO Q 上での遅延・メモリ | 4 GB 版 + int8 で 2 秒周期を目標。不成立なら PC 実行 | フェーズ 6 |
| 人物識別の精度 | 服装依存で日ごとに変わる。1 日単位で再学習または服装に依らない特徴(身長・歩幅)を併用 | フェーズ 5 |
| 自動充電 | 第 1 段階では手動充電。ドッキング充電は後回し | フェーズ 7 以降 |

## 参考

- [Geekservo Motor(RobotShop)](https://www.robotshop.com/products/geekservo-motor-compatible-with-lego)
- [Geekservo Motor 2kg(RobotShop)](https://www.robotshop.com/products/geekservo-motor-2kg-compatible-w-lego)
- [60mm LEGO Compatible Mecanum Wheels 14144(Nexus Robot)](https://www.nexusrobot.com/product/a-set-of-60mm-lego-compatible-mecanum-wheels-4-piecesbearing-rollers-14144.html)
- [70mm Mecanum wheel for LEGO Technic axle(Thingiverse)](https://www.thingiverse.com/thing:5245484/files)
- [Arduino UNO Q ドキュメント](https://docs.arduino.cc/hardware/uno-q)
- [Arduino UNO Q 製品ページ](https://www.arduino.cc/product-uno-q)
- [App Bricks: video-generic-object-detection](https://github.com/arduino/app-bricks-examples/blob/main/examples/video-generic-object-detection/README.md)
- [Edge Impulse: Run Arduino App Lab](https://docs.edgeimpulse.com/hardware/deployments/run-arduino-app-lab)
- [LAYA(GitHub)](https://github.com/NandhaKishorM/laya)
- [What is Laya?](https://jevmodel.org/what-is-laya/)

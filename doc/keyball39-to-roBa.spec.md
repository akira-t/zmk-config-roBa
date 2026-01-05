# Keyball39 → roBa 移植メモ / 仕様（ドラフト）

作成日: 2026-01-03  
更新日: 2026-01-04  
目的: `doc/keymap_cheatsheet_keyball39 (44) (1).pdf` の Keyball39 キーマップを、ZMK の roBa で「だいたい再現」しつつ、roBa 側の構成も整理する。

---

## 参照資料（このリポジトリ内）

- Keyball39 Remap cheatsheet（レイヤ0〜3）  
  - `doc/keymap_cheatsheet_keyball39 (44) (1).pdf`
  - 生成日時: 2025-10-03（`pdfinfo`より）
- roBa 側の現行 ZMK キーマップ  
  - `config/roBa.keymap`（GitHub Actions の keymap-drawer もここを参照）
- roBa 側のレイヤ案（Keyball39 っぽい雰囲気）  
  - `doc/roBa.keymap.draft`
- QMK 側（Keyball39 ファーム、ジョイスティック加速/修飾の実装がある）  
  - `qmk/keyboards/keyball/keyball39/keymaps/via/keymap.c`
  - ※`qmk/` はこの repo ではシンボリックリンク

---

## 現状整理（roBa / ZMK）

### ハード/Shield 構成の要点

- roBa shield 定義: `boards/shields/roBa/roBa.dtsi` / `boards/shields/roBa/roBa_*.overlay`
- 右手側に PMW3610 トラックボール
  - `boards/shields/roBa/roBa_R.overlay` で `pixart,pmw3610` を有効化
  - `boards/shields/roBa/roBa_R.conf` で `CONFIG_PMW3610_*` を設定
- 左エンコーダ（EC11）は有効: `boards/shields/roBa/roBa_L.overlay`

### ポインティング設定（ZMK）

`config/roBa.keymap` 内:

- `&trackball { automouse-layer = <4>; scroll-layers = <5>; };`
  - layer 4: MOUSE（自動マウスレイヤ）
  - layer 5: SCROLL（スクロール用レイヤ）
- `&trackball_listener` の layer 7(BOOST) override で、押している間だけポインタ速度を約3倍にする（`zip_xy_scaler 3/1`）

右手側 `boards/shields/roBa/roBa_R.conf` の要点:

- `CONFIG_PMW3610_CPI=400`
- `CONFIG_PMW3610_SCROLL_TICK=16`
- `CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=700`
- `CONFIG_PMW3610_SNIPE_*` はコメントアウト（現状未使用）

### 現行レイヤ構成（`config/roBa.keymap`）

- 0: `default_layer`
- 1: `FUNCTION`
- 2: `NUM`
- 3: `ARROW`
- 4: `MOUSE`（`automouse-layer`）
- 5: `SCROLL`（`scroll-layers`）
- 6: `layer_6`（BT/bootloader等）
- 7: `BOOST`（ポインタ加速用の透過レイヤ）

※コンボ/マクロも定義済み（TAB/Shift+TAB/無変換など）。

---

## 目標（Keyball39 cheatsheet の再現）

### レイヤ0（ベース）

PDF（Layer0）の読み取り結果（概略）:

- `E` が **Alt モッドタップ**
- `A` が **Ctrl モッドタップ**
- `Z` が **Shift モッドタップ**
- `,` が **Alt モッドタップ**、`.` が **Shift モッドタップ**
- `P` が **Layer(1) タップ/ホールド**（LT）
- `Enter` が **Layer(2) タップ/ホールド**（LT）
- `-` が **Layer(3) タップ/ホールド**（LT）
- `英数` / `かな`（IME切替）を親指付近に配置

### レイヤ1（メディア＋マウスボタン）

PDF（Layer1）の読み取り結果（概略）:

- `F15`, `Mute`, `Vol-`, `Vol+`, `F14`
- `Mouse Btn1/2/3`
- `TO(0)`（ベースへ戻る）

### レイヤ2（数字＋ナビ＋マウスボタン4/5）

PDF（Layer2）の読み取り結果（概略）:

- 数字（7/8/9, 4/5/6, 1/2/3, 0 系）
- `Page Up/Down`, `BS`
- 矢印（←↓↑→）
- `Mouse Btn3/4/5`
- `TO(0)`

### レイヤ3（記号＋Keyball固有キー＋F12）

PDF（Layer3）の読み取り結果（概略）:

- 記号系（`(*S+2)` 等の表記は「そのキーを Shift したもの」を意味）
- `F12`
- `RGB Toggle`
- `Kb 10`, `Kb 1`
- `Any 0xCE90`（要調査：後述）

---

## Keyball の “Kb XX” は何か

Keyball の特殊キーコード（Remap 上の表記）対応は `qmk/keyboards/keyball/lib/keyball/keycodes.md` に定義されている。

本件で PDF に出てくるもの:

- `Kb 1` = `KBC_SAVE`（Keyball 設定を EEPROM に保存）
- `Kb 10` = `AML_TO`（自動マウスレイヤーのトグル）

参考（関連しそうなもの）:

- `Kb 6` = `SCRL_TO`（スクロールモード切替）
- `Kb 7` = `SCRL_MO`（押している間スクロール）
- `Kb 2..5` = CPI 増減
- `Kb 11/12` = 自動マウスレイヤーのタイムアウト増減

---

## QMK 側の追加挙動（ジョイスティック）

`qmk/keyboards/keyball/keyball39/keymaps/via/keymap.c` に以下の実装がある（Remap 編集前の素朴版とのこと）:

### 1) ジョイスティック “Hat” を修飾キーとして使う

ジョイスティックの方向で **修飾キーを押しっぱなし**にする:

- 上: Shift
- 右: Gui（Win/Cmd）
- 左: Ctrl
- 下: Alt
- 斜め: 上記の組み合わせ（例: 右下=Gui+Alt）

### 2) ジョイスティックYでトラックボール速度を連続調整

`current_speed_multiplier` を計算して `pointing_device_task_kb()` 内でポインタ移動に乗算:

- 上方向（Yが負）: **加速**（倍率 > 1）
- 下方向（Yが正）: **減速**（倍率 < 1）

---

## ZMK/roBa での実装（2026-01-04 時点）

### 変更ファイル

- `config/roBa.keymap`
- `keymap-drawer/roBa.yaml`

### レイヤ番号（`config/roBa.keymap` の「記述順」がそのまま layer 番号）

- 0: `default_layer`（Keyball L0 近似）
- 1: `FUNCTION`（Keyball L1 近似: メディア）
- 2: `NUM`（Keyball L2 近似: 数字/ナビ）
- 3: `ARROW`（暫定: 記号。Keyball L3 近似だが要整理）
- 4: `MOUSE`（自動マウスレイヤ: `automouse-layer=<4>`）
- 5: `SCROLL`（スクロール用レイヤ: `scroll-layers=<5>`）
- 6: `layer_6`（BT/bootloader）
- 7: `BOOST`（ポインタ加速用の透過レイヤ）

### ポインティング（トラックボール）

- Automouse:
  - トラックボールが動いた後に layer 4 が自動で有効化される（timeout は `CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=700`）。
- スクロール:
  - layer 5 を押している間は、トラックボール移動がスクロールになる（`scroll-layers=<5>`）。
  - ベース(layer0)の中央キーに `SCROLL = &mo 5` を割当（押している間だけ）。
- 加速（Boost）:
  - ベース(layer0)の中央キーに `BOOST = &mo 7` を割当（押している間だけ）。
  - layer 7 が有効な間だけ、`&trackball_listener` の override で XY を 3倍スケール（`zip_xy_scaler 3/1`）。
- スナイプ:
  - 要求により使わない（`CONFIG_PMW3610_SNIPE_*` は無効のまま）。

### マウスボタンの方針（★反映済み）

- マウスボタンは **自動マウスレイヤ（layer4=MOUSE）にのみ**置く。
- 配置:
  - `,` = `MB1`（左クリック）
  - `.` = `MB2`（右クリック）
  - `M` = `MB3`（中クリック）
- 上記は MOUSE レイヤでだけ有効（それ以外のレイヤには `&mkp` を置かない）。

### Keyball 固有キー（Kb 1 / Kb 10）→ ★やめる

- `Kb 10 (AML_TO)`:
  - Keyball(QMK)では自動マウスレイヤのトグルだが、roBa(ZMK)では **実装しない**。
  - automouse は `automouse-layer=<4>` で固定。
- `Kb 1 (KBC_SAVE)`:
  - Keyball の設定保存 (EEPROM) 概念が roBa(ZMK)には無いので **実装しない**。

### ジョイスティック由来の機能 → ★やめる（再現しない）

- Hat mods（方向でCtrl/Alt/Shift/Gui）: 再現しない。
- Y軸連続加減速: 再現しない。
  - 代替として「押している間だけ3倍」の BOOST を用意。

### レイヤの重なり / `&trans` の動き（FAQ）

- layer は同時に複数アクティブになれる（`&lt`, `&mo`, `&tg` など）。  
- 解決順は **番号が大きいlayerが優先**。キーごとに上から探して、binding が `&trans` なら次の下位layerにフォールバックする。
- 例:
  - automouseで layer4(MOUSE) が有効でも、中央キーは layer4 側が `&trans` なので、layer0 の `SCROLL(&mo 5)` / `BOOST(&mo 7)` がそのまま動く。
  - layer7(BOOST) は全 `&trans` なので、キー出力は変わらない。一方で input-listener の override だけが有効になるので「押している間だけ加速」が成立する。

---

## 未解決（要確認）

- `Any 0xCE90` は何のキーか → ★不要
  - 移植対象外として扱う。
- 記号レイヤ（layer3: `ARROW`）の整理（下の「追加で検討したい」参照）
- エンコーダ押し込みでミュート
  - 現状の roBa 物理レイアウト/DT では encoder push が keymap に存在しないため、実現するならハード配線＋`boards/shields/roBa/roBa.dtsi` へのキー追加が必要。
- macOS / Windows どちらを主用途にするか（IMEキー/修飾の扱いが変わる）

## 追加で検討したい（方針案）

### 1) 記号レイヤ（@ と ` の関係 / ショートカット向けの集約）

- JIS配列前提だと `@` と `` ` `` は同じ物理キー（Shift有無）で、押し心地の近さが期待できる。  
  - QMK参考: `qmk/quantum/keymap_extras/keymap_japanese.h`
  - ZMK側のキーコード近似（OSがJIS配列である前提）:
    - `@` = `&kp LBKT` / `` ` `` = `&kp LS(LBKT)`
    - `^` = `&kp EQUAL` / `~` = `&kp LS(EQUAL)`
    - `[` = `&kp RBKT` / `{` = `&kp LS(RBKT)`
    - `;` = `&kp SEMICOLON` / `+` = `&kp LS(SEMICOLON)`
    - `:` = `&kp SQT` / `*` = `&kp LS(SQT)`
    - `,` = `&kp COMMA` / `<` = `&kp LS(COMMA)`
    - `.` = `&kp DOT` / `>` = `&kp LS(DOT)`
    - `/` = `&kp SLASH` / `?` = `&kp LS(SLASH)`
    - `_` =（ZMKのキーコード名が要確認: QMKでは Shift+INT1）
- 配置方針（案）:
  - layer3 の右手ホーム列（`H J K L` 周辺）に `@ ^ [ ] ; :` を集約
  - 下段に `, . / _` を集約
  - “Keyball L3の記号群（$%&...）”は周辺に残し、まず「ショートカット頻出記号」を最優先にする
- 実装手順:
  - `config/roBa.keymap` の `ARROW` を “SYM” として再設計し、必要なら `keymap-drawer/roBa.yaml` も追従

### 2) エンコーダ（回転）の役割

- 実装済み（2026-01-04）:
  - デフォルト(layer0): 音量 `Vol- / Vol+`（`sensor-bindings`）
  - MOUSE(layer4): ホイールスクロール（`sensor-bindings = <&encoder_msc_down_up>;`）
- 追加案:
  - FUNCTION(layer1): 画面輝度 `C_BRI_DN / C_BRI_UP`
  - NUM(layer2) or 記号(layer3): “早送り/巻き戻し” or “次/前トラック” の割当
  - 押し込みでミュートは、encoder push のキーをDTに追加できる場合のみ対応

---

## 次の作業（提案）

1. layer3（記号）の「集約したい記号セット」と「配置（どの指に置くか）」を決める
2. 必要なら encoder の追加モード（輝度/早送り）を layer1/2/3 に割り当てる
3. 実機で tapping-term と mod-tap の誤爆を微調整する

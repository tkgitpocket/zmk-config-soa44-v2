# 左半分フリーズ問題 調査メモ

**症状**: 左半分が急に入力を受け付けなくなる。リセットボタンまたは電源OFF/ONで復旧。  
**調査日**: 2026-06-27

---

## 症状の整理

```
正常時                     問題発生時
┌──────────┐              ┌──────────┐
│  左半分  │──BLE──┐     │  左半分  │ ←キーを押しても
│(ペリフェラ)│      │     │(応答なし)│   反応しない
└──────────┘      ↓     └──────────┘
              ┌──────────┐          ↓
              │  右半分  │──BLE──PC  リセットボタン
              │(セントラル)│          で復旧
              └──────────┘
```

- ハンダ不良ではない（毎回リセットで復旧するため）
- 他の無線分割キーボードでは再現しない
- 突発的・散発的に発生

---

## 他キーボードとの比較で分かったこと

調査対象として以下の動作実績ありキーボードと比較した。

| キーボード | 非LiPoモジュール | BLE_EXPERIMENTAL_CONN | WS2812_WIDGET | 問題発生 |
|---|:---:|:---:|:---:|:---:|
| SOA44（このキーボード） | ✅ 使用 | ✅ 有効 | ✅ 使用 | **あり** |
| dya-dash (cormoran) | ❌ | ❌ | ❌ | なし |
| zonkey (kureyakey) | ❌ | ❌ | ❌ | なし |
| roBa (kumamuk-git) | ❌ | ❌ | ❌ | なし |
| torabo-tsuki-lp (sekigon-gonnoc) | ✅ 使用 | ❌ | ❌ | 不明 |

**動作OKの3台が持っていなくて、SOA44 だけが持っているもの**：
1. `zmk-feature-non-lipo-battery-management`（sekigon-gonnoc製）
2. `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y`
3. `CONFIG_WS2812_WIDGET=y`（gohanda11製カスタムモジュール）

---

## 根本原因の特定

### 有力候補: `zmk-feature-non-lipo-battery-management` の広告タイムアウト

このモジュールには「BLE接続が切れてから一定時間再接続できなかった場合に電源を落とす」機能がある。

```c
// モジュールのソースコード（sekigon-gonnoc/zmk-feature-non-lipo-battery-management）
static void disconnected_cb(...) {
    advertising_start_time = k_uptime_get();   // 切断時刻を記録
    k_work_schedule(&adv_timeout_work, K_MSEC(10000));  // 10秒後から監視開始
}

// タイムアウトしたら...
zmk_pm_suspend_devices();
sys_poweroff();    // ← 通常のスリープではなくハード電源断！
```

SOA44 の設定：
```
CONFIG_ZMK_NON_LIPO_ADV_SLEEP_TIMEOUT=60000   ← 60秒でsys_poweroff()
```

### 通常のZMKスリープとの違い

```
通常のZMKスリープ (PM_STATE_SOFT_OFF)      このモジュールの電源断 (sys_poweroff)
┌─────────────────────┐                  ┌─────────────────────┐
│ nRF52840             │                  │ nRF52840             │
│                      │                  │                      │
│  低消費電力スリープ   │                  │  System OFF モード   │
│                      │                  │                      │
│  ↓ 復旧方法          │                  │  ↓ 復旧方法          │
│  キーを押す でOK     │                  │  リセットボタン必須  │
└─────────────────────┘                  └─────────────────────┘
         ↑ 他のキーボードはこちら                  ↑ SOA44の左半分はこちら
```

---

## 再現シナリオ（具体的な時刻付き）

### パターンA: 右側しか使っていない間に左が落ちる

```
時刻       イベント
─────────────────────────────────────────────────────────
14:00:00  左右のBLE接続が切断（電波干渉・ノイズなど）
          ユーザーは気づいていない
          左半分: アドバタイジング開始（右半分を探している）

14:00:00  たまたま右手側キーのみ使用中
│         （右→PCのBLEは別経路なので右キーは正常動作）
│
14:01:00  ── 60秒経過 ──
│         左半分: sys_poweroff() 実行 → System OFF へ
│
14:01:30  左キーを押す → 反応なし（すでに電源断）
          リセットボタン → 復旧
```

### パターンB: 「少し待てば戻るかな」と思っているうちにタイムアウト

```
時刻       イベント
─────────────────────────────────────────────────────────
14:00:00  左右のBLE接続が切断
          すぐ左キーを押す → 動かない（切断中なので届かない）

14:00:01  「すぐ直るかな」と右キーだけ使いながら様子見
│
14:01:00  ── 60秒経過 ──
│         左半分: sys_poweroff() 実行
│
14:01:01  もう一度左キー → 反応なし
          リセットボタン → 復旧
```

### 「左を連続して打っていても発生するか？」

**発生する。** BLE切断はキー入力とは無関係に起きる。  
ただし、左キーを頻繁に打っている場合は「切断した瞬間から」左が効かないので 60秒待たずに気づく。  
60秒経過後は sys_poweroff() になるので、その後はキーを押しても意味がない。

---

## 適用した修正

### 修正1: 左半分の広告タイムアウトを無効化（`soa44_L.conf`）

```diff
- CONFIG_ZMK_NON_LIPO_ADV_SLEEP_TIMEOUT=60000
+ CONFIG_ZMK_NON_LIPO_ADV_SLEEP_TIMEOUT=0
```

`0` に設定するとこの機能が無効になる（Kconfig ドキュメントに明記）。  
電池節約は `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000`（15分無操作でスリープ）で代替。

**理由**: 左半分はPCではなく右半分と接続する。接続が切れても「電源断する必要」はなく、  
再接続できれば正常に戻るべき。sys_poweroff() は過剰な対応。

### 修正2: 実験的BLE機能を無効化（`soa44_L.conf` / `soa44_R.conf`）

```diff
- CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
+ # CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y  # 無効化: 実験的機能、切断頻度が増える可能性
```

動作実績のある他キーボードは全て有効化していない。  
BLE切断の頻度を下げることで問題発生確率を下げる効果を期待。

### 修正3: 右半分の広告タイムアウトを延長（`soa44_R.conf`）

```diff
- CONFIG_ZMK_NON_LIPO_ADV_SLEEP_TIMEOUT=60000
+ CONFIG_ZMK_NON_LIPO_ADV_SLEEP_TIMEOUT=300000
```

右半分はPCと接続するため、PC電源オフ時のスリープとして機能を残す。  
ただし60秒はBTプロファイル切替中などに誤動作する可能性があるため5分に延長。

---

## torabo-tsuki-lp との比較（追加調査）

同じ sekigon-gonnoc が作った分割トラックボールキーボード。  
同じ非LiPoモジュールを使っているが設定の差が参考になる。

| 設定 | SOA44 | torabo-tsuki-lp |
|---|---|---|
| `ADV_SLEEP_TIMEOUT` | 60000（明示設定） | 未設定（デフォルト60000） |
| `BLE_EXPERIMENTAL_CONN` | 有効 | **未設定（無効）** |
| LEDモジュール | `WS2812_WIDGET`（カスタム） | `STATUS_LED`（公式） |
| `IDLE_SLEEP_TIMEOUT` | 900000（15分） | 9000000（150分） |

torabo-tsuki-lp は `BLE_EXPERIMENTAL_CONN` を使わず、`WS2812_WIDGET` も使っていない。

---

## 残る懸念事項

修正1〜3を適用後も症状が続く場合、`WS2812_WIDGET` モジュールが原因の可能性がある。

- gohanda11 製のカスタムモジュール
- SPI制御・外部電源制御・複数タイマーを組み合わせた独自実装
- タイマー競合やSPIハングでファームウェア全体がフリーズする可能性がある
- **torabo-tsuki-lp はこのモジュールを使っておらず**、代わりに公式の `STATUS_LED` を使用

その場合の対処: `WS2812_WIDGET` を一時的に無効化して再現しなくなるか確認する。

---

## ファイル変更一覧

| ファイル | 変更内容 |
|---|---|
| `boards/shields/soa44/soa44_L.conf` | `ADV_SLEEP_TIMEOUT=0`、`BLE_EXPERIMENTAL_CONN` 無効化 |
| `boards/shields/soa44/soa44_R.conf` | `ADV_SLEEP_TIMEOUT=300000`、`BLE_EXPERIMENTAL_CONN` 無効化 |

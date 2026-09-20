# macOS キーリマップツール 要件・仕様書

> ステータス: Draft v0.2
> 最終更新: 2026-08-16

---

## 1. 背景

現在 Karabiner-Elements を使って以下のリマップを行っている。

| 用途 | 内容 |
| --- | --- |
| 日本語入力切り替え | 左⌘ 単押し → 英数 / 右⌘ 単押し → かな |
| 記号入れ替え | `-` ⇄ `'` |
| 記号入れ替え | `;` ⇄ `return` |

> **M0 での判明事項 (2026-08-16):** 当初 Karabiner の管轄と思っていた Caps Lock ⇄ Control は、実際には macOS システム設定の修飾キー機能で実現されていた（Caps→Control に加え、左 Control→Caps Lock の逆方向も意図的に使用中）。既に Karabiner 非依存かつアップデート耐性があるため、本ツールの守備範囲から除外した（非ゴール N6）。

### 課題

1. **macOS のアップデートのたびに壊れる / 再設定が必要になる**
   Karabiner-Elements は DriverKit の system extension として仮想 HID キーボードを実装し、root 権限のデーモンで実デバイスを grab して仮想デバイスにイベントを再注入している。
   実質的に「キーボードそのものを差し替える」アーキテクチャであり、この層に居る限り、system extension の再承認・SIP / 署名まわりの仕様変更・DriverKit API の変更に毎回追従する必要がある。これは実装品質の問題ではなく、そのレイヤーに居ることの必然的コスト。

2. **設定画面が過剰**
   Karabiner は任意のキーの複雑な条件分岐リマップを提供する汎用ツールであり、上記4項目しか使わない用途に対して UI・設定項目・概念が多すぎる。

### 中心となる気づき

**実際にやっていることのうち、Karabiner と同じ深さのレイヤーを必要とするものは一つもない。**

- 記号入れ替えは **単純な 1:1 キーコード置換** → `hidutil`（macOS 標準）で完結。Caps ⇄ Control はそもそもシステム設定側で実現済みでツール不要
- 英数/かなだけが **単押し判定（tap vs hold）という状態機械** を必要とする → ユーザ空間の `CGEventTap` で十分

したがって、Karabiner を作り直すのではなく、**必要な機能だけを 2 つの安定した層に分解して再実装する**。

---

## 2. ゴール / 非ゴール

### ゴール

- G1. macOS のメジャーアップデートをまたいでも、再承認・再設定なしに動き続ける
- G2. 設定が単一のテキストファイルで完結し、GUI 設定画面を必須としない
- G3. 自分の 4 項目のリマップを Karabiner なしで完全に再現する
- G4. 障害時の切り分けが容易であること（どの層が死んだかが即座に分かる）
- G5. 常駐プロセスが軽量であること（メモリ・CPU ともに無視できる水準）

### 非ゴール（明確にやらない）

- N1. **Karabiner の完全な代替を目指さない。** 複雑な条件分岐、アプリ別プロファイル、同時押し（simultaneous）、複数段レイヤーは実装しない
- N2. **ログイン画面・FileVault プリブートでの動作は諦める**
- N3. **Secure Input 中の動作は諦める**（パスワードフィールド等。英数/かな切り替えが必要な文脈ではない）
- N4. **Windows / Linux 対応はしない**（macOS 専用）
- N5. マウス・トラックパッドのリマップはしない
- N6. **修飾キーの入れ替え（Caps ⇄ Control 等）はやらない**（macOS システム設定の修飾キー機能で実現済み。hidutil で重ねると二重適用で壊れるため本ツールでは触らない）

> **重要な設計判断:** N1〜N3 のいずれかが必要になった時点で、それは「このツールの守備範囲外であり Karabiner を使うべき領域」と判断する。この線引きを曖昧にしないことが、本プロジェクトが Karabiner の二の舞にならないための最大の防御線。

---

## 3. アーキテクチャ

### 2 層構成

```
┌─────────────────────────────────────────────────┐
│ 物理キーボード (内蔵 / Corne)                     │
└────────────────────┬────────────────────────────┘
                     │ HID レポート
                     ▼
┌─────────────────────────────────────────────────┐
│ Layer 1: hidutil (IOKit HID / macOS 標準)        │
│  - 1:1 キーコード置換                             │
│  - `-`⇄`'`, `;`⇄return                          │
│  - 権限不要 / Secure Input 中も有効               │
└────────────────────┬────────────────────────────┘
                     │ 置換済みイベント
                     ▼
┌─────────────────────────────────────────────────┐
│ Layer 2: 自作アプリ (CGEventTap / ユーザ空間)      │
│  - 左右⌘の単押し検出のみ                          │
│  - 英数(0x66) / かな(0x68) を post                │
│  - アクセシビリティ権限のみ                        │
└────────────────────┬────────────────────────────┘
                     ▼
                  アプリケーション
```

### 層の分離が生む利点

- **障害の切り分けが自明。** 記号が壊れた → Layer 1。英数/かなが効かない → Layer 2
- **フェイルセーフ。** Layer 2 が死んでも ⌘Space に一時退避すれば作業は止まらない。Layer 1 は権限を持たないため、そもそも OS 側の都合で無効化される経路がほぼない
- **競合しない。** hidutil は event tap より上流で確定的に処理されるため、順序が固定される

### なぜ Layer 1 を自作アプリに統合しないか

hidutil は macOS 10.12 から存在する標準コマンドで、**自分でコードを書かないぶん、壊れる余地がない**。CGEventTap でも同じ置換は実装できるが、権限依存のコードパスを増やす意味がない。「書かなくていいコードは書かない」を優先する。

なお「統合しない」のは**置換ロジックを CGEventTap で再実装しない**という意味であり、hidutil コマンドの適用・管理は常駐アプリが担う（プロセスは 1 つ。FR-3 参照）。

---

## 4. 機能要件

### FR-1: 1:1 キーリマップ（Layer 1）

以下の置換を行う。

| Src | Dst | Src usage | Dst usage |
| --- | --- | --- | --- |
| `-` hyphen | `'` quote | `0x70000002D` | `0x700000034` |
| `'` quote | `-` hyphen | `0x700000034` | `0x70000002D` |
| return | `;` semicolon | `0x700000028` | `0x700000033` |
| `;` semicolon | return | `0x700000033` | `0x700000028` |

- FR-1.1: 設定はログイン時に自動適用される
- FR-1.2: 適用対象デバイスを限定できる（後述の FR-4 参照）
- FR-1.3: ワンコマンドで全解除できる
- FR-1.4: 置換は HID レベルの無条件置換であり、**修飾キー併用時も入れ替わる**（例: `⌘Return` は `⌘;` に、`Shift+-` は `"` になる）。これは意図した挙動である

> Caps ⇄ Control はシステム設定（キーボード > 修飾キー）で実現済みのため本ツールでは扱わない（N6）。実環境では Caps→Control に加えて左 Control→Caps Lock の逆方向も設定されており、hidutil で caps を触ると「hidutil で Caps→Ctrl → システム設定で Ctrl→Caps」と一周して打ち消される。

### FR-2: ⌘ 単押しによる入力ソース切り替え（Layer 2）

- FR-2.1: **左⌘を単独で押して離した**時、英数キー（virtual keycode `0x66`）を送出する
- FR-2.2: **右⌘を単独で押して離した**時、かなキー（virtual keycode `0x68`）を送出する
- FR-2.3: ⌘ を修飾キーとして使った場合（`⌘C` 等）は送出しない
- FR-2.4: ⌘ イベント自体は consume せず、必ず下流に流す

### FR-3: 常駐と自動起動

- FR-3.1: 常駐プロセスはメニューバーアプリ **1 つ**。ログイン項目として自動起動する
- FR-3.2: Layer 1（hidutil）の適用はこの常駐アプリが行う。適用タイミングは「起動時 / スリープ復帰時 / 設定再読み込み時」
- FR-3.3: Dock アイコンを表示しない（`LSUIElement = true`）

> **LaunchAgent を別に立てない理由:** 適用主体を 2 系統（LaunchAgent + アプリ）に分けると、設定リロード・一時停止・スリープ復帰時の再適用のたびに両者の整合を取る必要が生じる。hidutil の設定は一度適用すれば再起動まで残るため、「アプリが死んでいる間 Layer 1 も未適用」になるのは再起動直後〜アプリ起動までの一瞬だけで、分離の利点は実質ない。**層の分離は「どの API に依存するか」の話であり、プロセスを分ける理由にはしない。**

### FR-4: デバイス指定（Corne との共存）

自作キーボード（Corne / ZMK）を使用する場合、リマップは firmware 側で完結しており、ホスト側の hidutil が二重に適用されると壊れる。

- FR-4.1: hidutil の適用対象を VendorID / ProductID でフィルタできること
- FR-4.2: デフォルトでは内蔵キーボード（Apple Internal Keyboard）のみを対象とする。内蔵キーボードの VendorID / ProductID は機種で異なるため、matching 条件は実機の `hidutil list` で確定する
- FR-4.3: **Layer 2（⌘単押し）は全キーボードに効く。** CGEventTap に届くイベントからは発生元デバイスを public API で特定できないため、デバイス別の有効/無効は提供しない（制約として受容する）

> ZMK 側では英数/かなも HID の LANG キーで直接送出できるため、Layer 2 すら不要になる。
> ```
> &kp LANG1   // かな (HID usage 0x90)
> &kp LANG2   // 英数 (HID usage 0x91)
> ```

### FR-5: 設定ファイル

- FR-5.1: 単一の JSON ファイルで全設定を表現する（Foundation の Codable だけで読める。依存ゼロ）
- FR-5.2: 設定ファイルを編集して再読み込みするだけで反映される
- FR-5.3: GUI 設定画面は提供しない（v1 時点）。メニューバーからは「設定ファイルを開く」「再読み込み」「一時停止」のみ
- FR-5.4: 「一時停止」は**両層を無効化**する（hidutil の UserKeyMapping を空にし、event tap を止める）。再開で両方を再適用する

---

## 5. 非機能要件

| ID | 項目 | 要件 |
| --- | --- | --- |
| NFR-1 | アップデート耐性 | macOS のメジャーバージョンアップ後、再承認・再設定なしに動作すること |
| NFR-2 | 権限 | アクセシビリティ権限のみ。system extension / kext / root 権限を要求しない |
| NFR-3 | レイテンシ | event tap のコールバックは 1ms 以内に返す。ブロッキング I/O を含めない |
| NFR-4 | メモリ | 常駐時 30MB 未満 |
| NFR-5 | 復旧性 | event tap が OS に無効化された場合、自動的に再有効化すること |
| NFR-6 | 署名 | Developer ID 署名 + notarization 済み。Bundle ID と証明書は固定する |

> **NFR-6 の理由:** アクセシビリティ権限（TCC）は Bundle ID と署名のペアに紐づく。署名が変わると権限がリセットされ、ユーザから見ると「アップデートしたら壊れた」に見える。ここを固定できていれば、OS アップデートで権限が飛ぶことはまずない。

---

## 6. 技術仕様

### 6.1 Layer 1: hidutil

HID Usage Page 0x07（Keyboard/Keypad）を使用。値は `0x700000000 + usage`。

**主要な usage code:**

| キー | usage | キー | usage |
| --- | --- | --- | --- |
| return | 0x28 | Caps Lock | 0x39 |
| escape | 0x29 | LANG1 (かな) | 0x90 |
| delete | 0x2A | LANG2 (英数) | 0x91 |
| tab | 0x2B | Left Control | 0xE0 |
| space | 0x2C | Left Shift | 0xE1 |
| `-` hyphen | 0x2D | Left Option | 0xE2 |
| `=` equal | 0x2E | Left Command | 0xE3 |
| `[` | 0x2F | Right Control | 0xE4 |
| `]` | 0x30 | Right Shift | 0xE5 |
| `\` | 0x31 | Right Option | 0xE6 |
| `;` semicolon | 0x33 | Right Command | 0xE7 |
| `'` quote | 0x34 | | |
| `` ` `` grave | 0x35 | | |
| `,` comma | 0x36 | | |
| `.` period | 0x37 | | |
| `/` slash | 0x38 | | |

**適用コマンド:**

```bash
hidutil property --set '{"UserKeyMapping":[
  {"HIDKeyboardModifierMappingSrc":0x70000002D,"HIDKeyboardModifierMappingDst":0x700000034},
  {"HIDKeyboardModifierMappingSrc":0x700000034,"HIDKeyboardModifierMappingDst":0x70000002D},
  {"HIDKeyboardModifierMappingSrc":0x700000028,"HIDKeyboardModifierMappingDst":0x700000033},
  {"HIDKeyboardModifierMappingSrc":0x700000033,"HIDKeyboardModifierMappingDst":0x700000028}
]}'
```

**デバイス限定:**

```bash
hidutil property --matching '{"VendorID":<vid>,"ProductID":<pid>}' --set '{"UserKeyMapping":[...]}'
```

デバイス一覧の取得は `hidutil list`。

> **M0 実測 (Apple Silicon MacBook):** 内蔵キーボードのサービスは VendorID `0x0` / ProductID `0x35C`（vendor usage page 側）で、keyboard service（UsagePage 1 / Usage 6）は VID/PID とも `0x0`。VID/PID による matching が内蔵キーボードに効くかは未検証。**確定するまでは matching なしの全デバイス適用で運用**し、Corne を接続するタイミングで実測して FR-4 の実装方法を決める。

**解除:**

```bash
hidutil property --set '{"UserKeyMapping":[]}'
```

**既知の挙動:**

- 設定は再起動で消える → 常駐アプリが起動時に適用する（FR-3.2）
- OS バージョンによってはスリープ復帰後に外れる報告がある → `NSWorkspace.didWakeNotification` で再適用する（Layer 2 の tap 検証と同じフックで行える）
- 外付けキーボードを後から接続した場合、そのデバイスには適用されないことがある → 再実行が必要。IOKit の device matching notification を監視して自動再適用するのが理想（v1.1 以降）
- `--matching` 付きで適用した設定は、解除時も同じ `--matching` を付けないと消えない
- **「クリア → 即時再適用」（数秒以内）を行うと、per-service プロパティ上は正しく見えても実効しない状態になることがある**（M2 で実測。機序未特定）。再適用は既存値の上書きのみで行い、クリアを挟む場合は数秒空ける。**実効性の確認は `--get` でなく実際の打鍵で行う**（`--get` はプロパティを見ているだけで、ドライバの実効状態を保証しない）

### 6.2 Layer 2: CGEventTap による単押し検出

**扱う定数:**

```swift
// virtual keycode (kVK_)
let kVK_Command      : CGKeyCode = 55   // 0x37 左⌘
let kVK_RightCommand : CGKeyCode = 54   // 0x36 右⌘
let kVK_JIS_Eisu     : CGKeyCode = 102  // 0x66
let kVK_JIS_Kana     : CGKeyCode = 104  // 0x68

// device-dependent modifier mask (NX_)
let NX_DEVICELCMDKEYMASK : UInt64 = 0x00000008
let NX_DEVICERCMDKEYMASK : UInt64 = 0x00000010
```

修飾キーは `keyDown` ではなく `flagsChanged` として飛んでくる。左右の区別は `CGEventFlags.rawValue` の device-dependent ビットで行う（`.maskCommand` では左右が区別できない）。

**状態機械:**

```
        ┌──────┐
        │ IDLE │◄──────────────────────────────┐
        └───┬──┘                               │
            │ 左/右⌘ down                       │
            ▼                                  │
      ┌──────────┐  他のキー/マウス/他の修飾キー  │
      │ CANDIDATE├───────────────────────►┌─────┴────┐
      └───┬──────┘                        │ ABORTED  │
          │ 同じ⌘ up                       └─────┬────┘
          │ (かつ閾値時間内)                      │ 同じ⌘ up
          ▼                                     │
   ┌─────────────┐                              │
   │ FIRE        │──────────────────────────────┘
   │ 英数/かな post│
   └─────────────┘
```

**アボート条件:**

| 条件 | 理由 |
| --- | --- |
| `keyDown` / `keyUp` が発生 | `⌘C` 等の修飾キー用途 |
| マウスクリックが発生 | `⌘クリック` 用途 |
| スクロールが発生 | `⌘スクロール`（ズーム等）用途 |
| 別の修飾キーの `flagsChanged` | `⌘⇧` 等 |
| 反対側の⌘が押される | 左右同時押し |
| 押下から離すまで N ms 超過 | 長押しホールドとみなす（推奨: 300〜500ms、設定可能） |

**イベントタップの設定:**

```swift
let mask = (1 << CGEventType.flagsChanged.rawValue)
         | (1 << CGEventType.keyDown.rawValue)
         | (1 << CGEventType.keyUp.rawValue)
         | (1 << CGEventType.leftMouseDown.rawValue)
         | (1 << CGEventType.rightMouseDown.rawValue)
         | (1 << CGEventType.otherMouseDown.rawValue)
         | (1 << CGEventType.scrollWheel.rawValue)

CGEvent.tapCreate(
    tap: .cgSessionEventTap,
    place: .headInsertEventTap,
    options: .defaultTap,      // listenOnly ではなく defaultTap（下記参照）
    eventsOfInterest: CGEventMask(mask),
    callback: callback,
    userInfo: nil
)
```

> **defaultTap を選ぶ理由:** FR-2.4 の通りイベントは一切改変しないため、機能的には `.listenOnly` で足りる。しかし listen-only のキーボード tap には Input Monitoring 権限が必要で、イベント送出（`CGEventPost`）用の Accessibility と合わせて権限が 2 つになる。権限を Accessibility 1 つに保つため（NFR-2）defaultTap とする。代償として、コールバックが遅いと全キー入力に遅延が波及するため、NFR-3 は「守らないとシステム全体のタイピングが遅くなる」制約であることを意識する。

**イベント送出:**

```swift
func postKey(_ key: CGKeyCode) {
    let src = CGEventSource(stateID: .hidSystemState)
    let down = CGEvent(keyboardEventSource: src, virtualKey: key, keyDown: true)
    let up   = CGEvent(keyboardEventSource: src, virtualKey: key, keyDown: false)
    down?.flags = []        // ⌘フラグの残留を明示的にクリア
    up?.flags   = []
    down?.post(tap: .cgSessionEventTap)
    up?.post(tap: .cgSessionEventTap)
}
```

**必須のエラーハンドリング:**

```swift
case .tapDisabledByTimeout, .tapDisabledByUserInput:
    CGEvent.tapEnable(tap: eventTap, enable: true)
```

コールバックが遅いと OS が tap を切る。これを再有効化しないと「たまに効かなくなる」という再現性の低いバグになる。**最も踏みやすい落とし穴。**

**その他のエッジケース:**

| ケース | 対応 |
| --- | --- |
| スリープ復帰 | `NSWorkspace.didWakeNotification` で tap の有効性を検証・再作成 |
| ディスプレイ切り替え / ロック解除 | 同上 |
| 自分が post したイベントの再入 | `CGEventSource` の `userData` にマーカーを設定して自己判別、または送出中フラグで抑止 |
| キーリピート | `flagsChanged` にリピートはないため影響なし。ただし ABORTED 判定用に `keyDown` は拾う |
| 他アプリの合成イベント | `.eventSourceUserData` で判別可能だが v1 では区別しない |
| Secure Input 中 | イベントが来ないため自然に無効。エラーとしない。メニューバーに状態表示があると親切（`IsSecureEventInputEnabled()`） |

### 6.3 設定ファイル

`~/.config/key-remapper/config.json`

```json
{
  "remaps": [
    { "from": "hyphen",          "to": "quote" },
    { "from": "quote",           "to": "hyphen" },
    { "from": "return_or_enter", "to": "semicolon" },
    { "from": "semicolon",       "to": "return_or_enter" }
  ],
  "tap_actions": [
    { "key": "left_command",  "action": "eisu" },
    { "key": "right_command", "action": "kana" }
  ],
  "tap_threshold_ms": 400,
  "devices": {
    "include_builtin_only": false
  }
}
```

`remaps` は hidutil のコマンド文字列に変換して適用、`tap_actions` は event tap 側で解釈する。**キー名は Karabiner の命名に寄せる**（`return_or_enter` 等）と移行時の認知コストが下がる。`devices.include_builtin_only` は matching 手段が確定するまで `false`（全デバイス適用）で運用する（§6.1 の M0 実測を参照）。

> **M1 での判明事項 (TCC):** launchd 起動のプロセスは、設定ファイルが `~/Library/CloudStorage/`（Dropbox 等）配下への symlink だと `Operation not permitted` で読めない（CloudStorage は TCC 保護対象で、責任プロセスが Full Disk Access を持たないとアクセス不可）。暫定 LaunchAgent はインストール時に設定を解決して hidutil の引数を plist に焼き込むことで回避した。
>
> **M2 での検証結果:** LaunchServices 経由で起動した .app（ログイン項目相当）からは、CloudStorage への symlink 経由・直パスとも**問題なく読めた**（最小プローブアプリで実測）。制約は launchd 直下のプロセスのみ。したがって常駐アプリは設定を dotfiles 管理のまま普通に読んでよい。

---

## 7. 配布・運用

| 項目 | 方針 |
| --- | --- |
| 言語 | Swift（AppKit / Core Graphics） |
| Bundle ID | `io.github.nyshk97.keyrc`（dev は `.dev`。配布開始済み。TCC 永続化のため以後変更しない） |
| 最低対応 OS | macOS 13 Ventura 以降 |
| アーキテクチャ | arm64（Apple Silicon 専用で可） |
| 署名 | Developer ID Application、notarization 必須 |
| 自動更新 | Sparkle（polepole の署名・公証・配信パイプラインを流用） |
| 配布 | GitHub Releases + Homebrew Cask |
| ライセンス | MIT |

> polepole.dev で構築済みの署名・公証・Sparkle 配信パイプラインがそのまま再利用できる。ここが初期コストの大半を占める部分なので、実質的な新規実装は Layer 2 のアプリ本体（数百行）と設定ローダのみ。

---

## 8. マイルストーン

### M0: 検証（半日）

- 現在の Karabiner を無効化し、hidutil コマンドを手打ちして FR-1 の 5 項目が再現できることを確認
- `hidutil list` で内蔵キーボードの matching 条件（VendorID / ProductID）を実機確認する
- ⌘英かな（`dominion525/cmd-eikana`）を入れて FR-2 が満たせることを確認
- **この状態で 1 週間生活し、他に無意識に使っている Karabiner 機能がないかを洗い出す**

> M0 は省略しないこと。「実は使っていた」機能が後から出てくると設計が崩れる。逆にここで困らなければ、非ゴール N1〜N3 の線引きが正しいことが実証される。

> **進捗 (2026-08-16):** セットアップ完了・生活検証開始。Karabiner は空プロファイル「M0 (no remap)」へ切り替え（元設定は「Default profile」に温存）、hidutil 4 件を手動適用、cmd-eikana v2.4.2 導入・アクセシビリティ権限付与済み。記号打鍵はこれまで通りであることを確認。Caps がシステム設定管轄と判明（→ N6）。旧 Karabiner の単押しタイムアウトは実質 1 秒だった（`to_if_alone` デフォルト）ため、`tap_threshold_ms` のデフォルト 400ms は生活検証の体感で見直す可能性あり。

### M1: Layer 1 の固定化（半日）

- 設定ファイル → hidutil コマンド変換スクリプト
- 暫定の LaunchAgent で自動適用（M3 でアプリに統合し、撤去する）
- 再起動をまたいで永続することを確認

> **進捗 (2026-08-16):** `scripts/key-remapper-apply`（変換・適用）と `scripts/install-launchagent.sh` / `uninstall-launchagent.sh` を作成、LaunchAgent 登録済み。設定は `~/.config/key-remapper/config.json`（dotfiles 管理・Dropbox symlink）。クリア → RunAtLoad で 4 件再適用を確認済み。実際の再起動またぎは未確認（次回再起動時に確認）。

### M2: Layer 2 の実装（2〜3日）

- CGEventTap のセットアップとアクセシビリティ権限リクエスト
- 単押し状態機械の実装
- tap 無効化からの自動復旧
- スリープ復帰ハンドリング（tap の検証・再作成 + hidutil 再適用）

> **進捗 (2026-08-16):** 実装・検証完了。fire（左→英数/右→かな）、keyDown・Shift・スクロールによるアボート、長押しタイムアウトの全ケースをログで確認。cmd-eikana は停止し、以後は本アプリでドッグフーディング（Debug ビルドを暫定ログイン項目に登録。M3 で正式な自動起動に置換）。
>
> **方針変更:** wake 時の hidutil 再適用はいったん M3 送りとしたが、「クリア→即時再適用で hidutil が実効しなくなる」障害（§6.1 既知の挙動）を踏んだのを機に、`HidutilApplier.swift`（remaps の読み込み・変換・適用）を M2 中に前倒し実装した。アプリが起動時と wake 時に Layer 1 を適用する（FR-3.2 の形が既に成立）。閾値 400ms と tap_actions の設定読み込みは引き続き M3。

### M3: 統合とパッケージング（1〜2日）

- メニューバー常駐 UI（状態表示 / 一時停止 / 設定を開く / 再読み込み）
- hidutil 適用のアプリへの統合（起動時・wake 時・再読み込み時）と暫定 LaunchAgent の撤去
- 設定ファイルのローダとバリデーション
- 署名・notarization・Sparkle 組み込み

> **進捗 (2026-08-16):** Sparkle・notarization・Homebrew Cask を除き完了。実装内容:
> - dev/本番の Bundle ID 分離（Debug = `io.github.nyshk97.key-remapper.dev`「KeyRemapper Dev」/ Apple Development 署名、Release = `io.github.nyshk97.key-remapper` / Developer ID + Hardened Runtime + timestamp）。TCC が独立し併存可能
> - メニューバー UI（状態表示・一時停止/再開・再読み込み・設定を開く・ログイン時に起動・終了）。一時停止は FR-5.4 通り両層を無効化し、再開時は「クリア後 3 秒」の安全マージンを挟んで hidutil を再適用
> - 設定ローダ（remaps / tap_actions / tap_threshold_ms、キー名バリデーション付き）。読み込み失敗時は直前の設定を維持し hidutil に触らない
> - ログイン項目は SMAppService（本番のみ初回起動時に自動登録。未登録時の status は `.notRegistered` でなく `.notFound` が返ることがある点に注意）。手製ログイン項目と暫定 LaunchAgent は撤去済み
> - Release ビルドを `/Applications` に配置し常用へ切り替え。残タスク: Sparkle 組み込み、notarization、Homebrew Cask（次フェーズ）

### M4: ドッグフーディング（2週間）

- Karabiner を完全にアンインストールして常用
- 不具合の記録と修正

> **進捗 (2026-08-16):** Karabiner のアンインストールを前倒しで完了（公式 uninstall.sh。dmg インストールだったため Brewfile 変更は不要）。仮想キーボードが HID から消え、イベント経路は「内蔵キーボード → hidutil → CGEventTap」の素の構成になった。root 所有の残存プロセスと system extension は次回再起動で消える。setup-dotfiles.sh から karabiner の symlink 定義を除去（`~/.config/karabiner/` と Dropbox 内の設定実体は履歴として残置）。cmd-eikana は応急フォールバックとして M4 完了まで /Applications に温存。

### M5: 公開（任意）

- README、Homebrew Cask、リリース

---

## 8.5 残作業チェックリスト（2026-08-16 時点の現在地）

M0〜M2 完了、M3 はアプリ実装まで完了・配布まわりが未了、M4 進行中。

### M3 残り（配布まわり）

- [x] アプリ名の決定とリネーム（2026-08-16: **keyrc** に決定。.zshrc 系譜の「キーボードの rc ファイル」で、差別化の核＝テキスト設定1枚をそのまま名前にした。target・Bundle ID `io.github.nyshk97.keyrc(.dev)`・設定 `~/.config/keyrc/config.json`・ログ `/tmp/keyrc(-dev).log`・`scripts/keyrc-apply` まで一貫リネーム。dotfiles 側も `config/keyrc/` に改名済み。Bundle ID 変更に伴いアクセシビリティ権限は再付与）
- [x] Sparkle 組み込み（2026-08-16: SwiftPM で Sparkle 2 追加・本番のみ有効。EdDSA 鍵は keychain の既存デフォルト鍵を再利用（Sparkle 公式が複数アプリ1鍵を推奨）。SUFeedURL は `releases/latest/download/appcast.xml`。XcodeGen デフォルトの CFBundleVersion 1.0 が入り版数比較が壊れる問題を発見し MARKETING_VERSION 連動に修正）
- [x] notarization（2026-08-16: `mise run release-zip` で配布適格化 → ユーザー Terminal で submit、status: Accepted・staple 済み。ハマり所: ① Sparkle 内部 XPC が adhoc 署名のまま残る → inside-out 再署名で解決 ② get-task-allow → `CODE_SIGN_INJECT_BASE_ENTITLEMENTS: NO` で根本停止 ③ ネットワークの IPv6 経路が connection refused で notarytool が失敗 → `networksetup -setv6off Wi-Fi` で回避 ④ アプリ用パスワードは新規発行が必要だった）
- [x] GitHub Release v0.1.0（2026-08-16: `mise run publish-release` で appcast.xml 生成込みで公開。https://github.com/nyshk97/keyrc/releases/tag/v0.1.0 ）
- [x] Homebrew Cask（2026-08-16: 既存の `nyshk97/homebrew-tap` に `Casks/keyrc.rb` を追加。auto_updates true。`depends_on macos:` の文字列比較書式は deprecated → symbol 形式）
- [x] Brewfile に cask を追記（2026-08-16: `cask 'nyshk97/tap/keyrc'`。手動配置していた /Applications/keyrc.app は削除し cask 管理に移行）

### 次回以降のリリース手順（v0.1.2〜）

1. `docs/CHANGELOG.md` の `[Unreleased]` を埋めて commit → push（書き方は CHANGELOG 冒頭。
   `git log <前回タグ>..HEAD` を読んで Claude Code のセッションが書く。空だと preflight で止まる）
2. `mise run release [patch|minor|major|x.y.z]`（既定 patch）。Claude Code のセッションから叩いてよい。
   preflight → `project.yml` の bump と CHANGELOG の切り出しを commit → 配布適格に再署名して zip →
   notarize + staple → appcast.xml（EdDSA 署名・CHANGELOG を `<description>` に）→ push → GitHub Release →
   homebrew-tap の `Casks/keyrc.rb` 更新まで通す。push 前に失敗したら bump commit は trap で巻き戻る。
   IPv6 が腐っている回線では先に `sudo networksetup -setv6off Wi-Fi`
3. 既存インストール環境は Sparkle が自動更新する（更新ダイアログに CHANGELOG の内容が出る）

途中で失敗したときは個別タスクで再開できる: `release:preflight` / `release:zip` / `release:notarize`
（公開だけの再実行は無い。bump commit が巻き戻るので `mise run release` を叩き直す）。notarize の
資格情報は keychain プロファイル `nyshk97-notary`（自作 Mac アプリ共通。中身は App Store Connect の
API キーで、`.p8` は Dropbox の `secrets/`）。画面ロック中は読めないので preflight で止まる。
`NOTARY_PROFILE` 環境変数で上書きできる。

リポジトリのディレクトリを移動・改名した後の初回ビルドは `build/` を消してから行う（derivedData 内の SwiftPM
キャッシュが旧絶対パスを指し、`There is no XCFramework found` で失敗する。`~/key-remapper` → `~/keyrc` で実例）。

### M4 残り（ドッグフーディング、〜2026-08-30 目安）

- [x] 再起動時のブート永続チェーン検証（2026-08-16 実施。boot 15:27:06 → KeyRemapper 自動起動 15:27:36 → `hidutil: applied 4 mappings (exit 0)` 15:27:38 → 直後から fire ログあり。全チェーン OK。ただし Karabiner の system extension（VirtualHIDDevice dext）は「次回再起動で消える」想定に反し再起動後も activated enabled で残存 → `sudo systemextensionsctl gc`（孤児 extension の回収）で除去完了。`uninstall` サブコマンドは SIP 有効時は使えない）
- [ ] `tap_threshold_ms: 400` の体感確認（旧 Karabiner は実質 1 秒だった）
  - 経過 (2026-08-16): この Mac での使用感は問題なしとのユーザー報告。400ms への短縮も違和感なし。期間満了（8/30 目安）まで継続観察
- [ ] 誤発火・「実は使っていた Karabiner 機能」の洗い出し
  - 経過 (2026-08-16): ここまで誤発火・機能不足の報告なし。継続観察
- [x] 問題なければ cmd-eikana（応急フォールバック）を /Applications から削除（2026-08-16 前倒しで削除。M2 以降未使用・本アプリで問題が出ていないため。アクセシビリティ許可も `tccutil reset` で掃除済み。必要になれば dominion525/cmd-eikana の配布バイナリを再導入すればよい。教訓: ゴミ箱移動だけだと BTM 登録のログインヘルパーがゴミ箱内バンドルから起動し「URL スキームのハンドラなし」ダイアログが出る → ゴミ箱からの完全削除と設定 plist 削除まで実施して解消）

### 積み残し（優先度低・時期未定）

- [ ] FR-4 デバイス matching の実測（内蔵キーボードの keyboard service は VID/PID とも 0x0 で matching 手段未確定。Corne を接続するタイミングで検証）
- [ ] `devices.include_builtin_only` の実装（↑の結果待ち。現状は全デバイス適用）
- [ ] 外付けキーボード接続時の自動再適用（IOKit device matching notification。v1.1）
- [ ] M5 公開判断（README 整備・周知。M4 通過後に判断）

---

## 9. リスクと制約

| リスク | 影響 | 対策 |
| --- | --- | --- |
| CGEventTap の仕様変更 | 中 | 10.4 以降ほぼ不変で可能性は低い。発生時は ⌘Space に退避しつつ対応 |
| アクセシビリティ権限のリセット | 中 | Bundle ID と署名証明書を固定。変更が必要な場合はリリースノートで明示 |
| Secure Input による無効化 | 低 | 仕様として受容。メニューバーに状態表示 |
| 外付けキーボード接続時に hidutil が未適用 | 中 | 手動再実行で回避可能。v1.1 で IOKit 通知による自動再適用 |
| hidutil がスリープ復帰後に外れる（OS バージョン依存の報告あり） | 中 | `didWakeNotification` で再適用（§6.1） |
| Corne との二重適用 | 中 | `--matching` によるデバイス限定をデフォルトに |
| Corne 側の素の ⌘ 単押しにも Layer 2 が効く | 低 | 仕様として受容（FR-4.3）。ZMK 側は mod-tap で LANG1/2 を直接送るため通常は顕在化しない |
| OS バージョンによっては Input Monitoring 権限も追加要求される報告 | 低 | 発生時はユーザに許可を案内。NFR-2 の例外として記録 |
| CloudStorage 配下へ symlink された設定ファイルが TCC で読めない | 低 | launchd 直下のみの制約と実測確認済み（§6.3）。LaunchAgent は plist 焼き込みで回避、常駐アプリは通常読み取りで問題なし |
| 他のリマップツールとの競合 | 中 | Karabiner / ⌘英かな の併用を明示的に非推奨とし、起動時に検出して警告 |

---

## 10. 公開する場合の位置付け（任意）

既存の選択肢との差別化。

| ツール | レイヤー | 守備範囲 | アップデート耐性 |
| --- | --- | --- | --- |
| Karabiner-Elements | DriverKit（仮想 HID） | 全部。条件分岐・レイヤー・同時押し | 低（system extension） |
| ⌘英かな (cmd-eikana) | CGEventTap | ⌘単押し + 簡易リマップ。GUI 設定 | 高 |
| **本ツール** | hidutil + CGEventTap | 日本語入力切り替え + 1:1 リマップに限定 | 高 |

**ポジション:** 「日本語入力ユーザが Karabiner に求めている 8 割を、Karabiner の 1 割の複雑さで提供する」。

差別化の核は機能数ではなく **設定がテキストファイル 1 枚で完結し、dotfiles 管理下に置ける** こと。⌘英かなは GUI 設定 + plist で、リポジトリ管理しづらい。開発者向けにはここが刺さる。

ただし M0 の検証結果次第では「⌘英かな + hidutil スクリプトで十分」という結論も十分あり得る。その場合は自作せず、hidutil 部分だけを dotfiles に組み込んで終わりにするのが合理的。**M5 は M4 を通過した場合のみ実行する。**

---

## 付録: 参考実装

- `iMasanari/cmd-eikana` — CGEventTap による単押し検出のオリジナル実装（Swift）
- `dominion525/cmd-eikana` — Apple Silicon 向けフォーク。署名・公証済みバイナリを配布。macOS Sequoia (15) / Tahoe (26) 対応
- `pqrs-org/Karabiner-Elements` — DriverKit 層の実装。**何をやらないかを確認する目的**で読む
- Apple: Quartz Event Services リファレンス（CGEventTap 系 API）

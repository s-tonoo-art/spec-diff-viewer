# 諸元差分ビューア — Claude Code 引継ぎ

## プロジェクト概要

ソフトバンク携帯基地局の諸元JSON（OPEN-UIから取得）を2ファイル比較するシングルファイルHTMLビューア。
A＝現行諸元、B＝今回PJ反映後として差分を可視化する。

- **ファイル**: `spec_diff_viewer_v3.html`（現在 v3.6）
- **構成**: HTML/CSS/JS のシングルファイル。ビルド不要、ブラウザで直接開く。
- **Git**: `main` ブランチ運用。コミット後 `git push` でリモートへ。

---

## カラー規約（絶対に変えるな）

| 意味 | 色 | クラス | 16進 |
|------|----|--------|------|
| 撤去（削除・A側旧値） | 青 | `val-del` / `badge-del` / `device-chip del` | `#58a6ff` |
| 新設（追加・B側新値） | 赤 | `val-new` / `badge-new` / `device-chip add` | `#f85149` |
| 変更（A→Bで値変化） | 紫 | `val-chg` / `badge-chg` | `#d2a8ff` |
| 既設（変化なし）     | 白 | `val-same` / `badge-same` | `#e6edf3` |

---

## JSONデータ構造（主要パス）

```
responseBody
├── head
│   ├── specId / specNm       # 諸元ID・諸元名
│   ├── siteId                # サイトID
│   └── specSts               # 諸元状態
├── shared
│   ├── site.area.name        # エリア名
│   ├── site.address.detailAddressNm
│   ├── bases[].baseNameKanji / baseNumber   # 局名・局番号
│   ├── bases[].powerSupply.powerIncomingMethod / battCls / powerSupplyBehiclesFlg
│   ├── radioGroups[]         # MU（無線機グループ）
│   │   ├── radioGroupId / radioGroupIdName
│   │   ├── ranKindCd         # 1=LTE 2=5G NR 3=3G
│   │   ├── bbuAccomGcBbt.name/code
│   │   ├── alarmPatternL/W/Fiveg
│   │   ├── caConfigs[].frequencyBandCd
│   │   └── devices[]
│   │       ├── deviceName         # ★装置名
│   │       └── equipment.equipModelName  # ★機種名
│   ├── radioChilds[]         # RU（無線機子機）
│   │   └── deviceIdName
│   ├── antennas[]
│   │   ├── deviceIdName
│   │   ├── devAzimuth / devHeight / devMTilt
│   │   └── equipment.equipModelName
│   ├── ancillaries[]
│   │   ├── deviceIdName
│   │   └── equipment.equipModelName
│   └── orders[]
│       └── orderId / project.name
└── networks[]
    ├── nwIdName / frequencyBand.name
    ├── lteServiceBand / fivegServiceBand / wcdmaServiceBand
    │   └── radioGroupId      # MUとの紐付け
    ├── sectorPorts[]
    │   ├── sectorIdName / portNo
    │   ├── deviceIdName / equipModelName  # ★RU機種名の正パス
    │   └── transReceiveClsName
    └── sectorAntennas[]
        ├── sectorIdName / deviceIdName
        ├── electricalTilt
        └── sectorAntennaDesign.equipModelName  # ★アンテナ機種名（B側優先）
```

---

## 主要関数

| 関数 | 役割 |
|------|------|
| `runDiff()` | 2ファイル読み込み後のエントリポイント。全build関数を呼ぶ |
| `buildSum()` | 変化サマリーカード＋コピー用テキスト生成 |
| `buildMU()` | 無線機グループ（MU）A/B比較パネル |
| `buildRU()` | RU一覧テーブル。機種変更検知→A=青/B=赤 |
| `buildAnt()` | アンテナ一覧＋セクター紐付けテーブル |
| `buildPow()` | 電源情報テーブル |
| `buildAnc()` | 付属装置（Ancillary）テーブル |
| `buildPort()` | 無線機ポート情報テーブル |
| `getRuMod()` | `networks[].sectorPorts[]` からRU名→機種名マップを構築 |

---

## 実装済みルール（変更時は要注意）

### MU装置チップ
- `deviceId` でA/B間を照合し、装置単位で色を決定（v3.6で修正。以前はside固定で誤表示していた）
- 両側にdeviceIdが存在し機種名も一致 → 無色（既設）
- 両側にdeviceIdが存在するが機種名が異なる → 紫（`device-chip chg`、変更）
- 片側にのみ存在 → A側は青（del/撤去）、B側は赤（add/新設）

### RU機種変更
- A・B両方に存在するがモデル名が異なる場合 → `modelChanged=true`
- A列機種名: `val-del`（青）、B列機種名: `val-new`（赤）
- サマリーテキストの「削除カウント」に旧機種、「追加カウント」に新機種を `[機変:機種名]` 形式で追記

### サマリーテキスト（コピー用）
- 諸元ID・オーダーは **B側の値のみ** 表示（A/B比較行なし）
- アンテナ機種名は `networks[].sectorAntennas[].sectorAntennaDesign.equipModelName` から取得（B側）

---

## バージョン履歴

| Ver | 内容 |
|-----|------|
| v3.6 | ①サマリーのアラームパターン表示削除 ②印刷ページ区切りを動的制御 ③MU装置チップをdeviceId単位の色判定に修正（既設誤表示バグ修正） |
| v3.5 | 印刷ボタン追加（A4横）／印刷時ダークテーマ→白背景自動変換／不要UI非表示・ページ区切り最適化 |
| v3.4 | ①サマリー諸元ID/オーダーをB値のみ表示 ②MU装置A=青/B=赤固定 ③アンテナB側機種名(sectorAntennaDesign) ④RU機種変更色付け+カウント |
| v3.3 | ブラウザタブに局名・局番号表示 |
| v3.2 | 電源情報項目名修正 / 4桁コード対応 |
| v3.1 | 凡例変更 / Ancillary機種ベース / ポート表示修正 |
| v3.0 | 初回リリース |

---

## Claude Code ↔ Cowork の行き来について

**結論：問題ない。** ファイルとGitが共通基盤なので、どちらで作業してもデータは同じ。

### 共有されるもの（行き来しても消えない）
- ファイル変更（同じフォルダを触るだけ）
- Git履歴（どちらからcommitしても同じrepo）
- `CLAUDE.md`（両ツールから参照できる）

### 共有されないもの（ツールごとに独立）
- 会話履歴（セッションをまたぐと消える。両ツール共通）
- Claude Codeのコンテキスト（ただし `CLAUDE.md` を自動読み込みするので実害は少ない）
- Coworkのメモリ（Cowork独自の記憶ファイルはClaude Codeには見えない）

### 実運用のコツ
- Claude Codeで作業 → `CLAUDE.md` を更新してcommit → Coworkで続き、が一番スムーズ
- Coworkで大きな仕様変更 → 変更内容をこのファイルに追記すればClaude Codeに引き継がれる
- **同時に両方で同じファイルを編集しない**（それだけ気をつければOK）

---

## 作業メモ

- コードマップ（`PM` / `BC` / `RK` / `BD`）は先頭部分のグローバル定数で管理
- `numSort()` でRU名・アンテナ名を数値順ソート
- `nets(d)` / `sh(d)` / `hd(d)` はデータアクセスの糖衣関数
- Windowsで `git add` すると LF→CRLF の warning が出るが無害

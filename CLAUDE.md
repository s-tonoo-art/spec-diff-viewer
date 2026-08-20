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
| v3.7 | ヘッダーのタイトル横に緯度経度（度分秒・DMS）と標高を表示。`shared.site.degree.latitudeDeg/longitudeDeg`をDMS変換、標高は`shared.site.billding.groundLevel`。A側優先、無ければB側にフォールバック |
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

---

## 関連ツール（同ディレクトリ）

### spec_input_check.html — 諸元入力チェッカー v1.2

施工会社担当50項目の入力状況をA/B比較で確認するツール。

| セクション | 内容 | 主なJSONパス |
|-----------|------|-------------|
| ① サイト情報 | 座標・住所・物件種別・塩害/多雷/積雪・併設 | `shared.site.*` |
| ② 電源情報 | 受電方法・電源車・バッテリー（基地局リスト） | `shared.bases[].powerSupply.*` |
| ③ アラームパターン | **MU構成/C-RAN構成を自動判別。** MU構成→alarmPatternL/W/Fiveg、C-RAN構成→rruExtAlarm（最低周波数側セクタ）。詳細は「アラームパターン ビジネスルール」節参照 | MU構成: `shared.radioGroups[].alarmPattern*`、C-RAN: `networks[].sectors[].rruExtAlarm` |
| ④ 付帯デバイス | Ancillary機種名 | `shared.ancillaries[].equipment.equipModelName` |
| ⑤ アンテナ付属機器 | RETシリアル・TMA・キャンセラ・Bias-T・RET設置パターン・制御インプット | `networks[].sectorAntennas[].sectorAntennaAncillary.*` |
| ⑥ 親RRU_Cascade | カスケード接続時の接続元セクタ（NOKIA C-RAN / ERI RP6339対象） | `networks[].sectors[].parentRruCascade/Name` |

**バージョン履歴**

| Ver | 内容 |
|-----|------|
| v1.2 | ①バリデータ追加（`badge-err`/`v-err`＝赤：値異常）②サマリーバー③MUアラームをバンド有無で期待値自動判定④C-RAN構成（devices=[]）を自動検出し`rruExtAlarm`を低周波数側セクタで検証 |
| v1.1 | ⑥親RRU_Cascade追加（BBU直収 or セクタ管理番号形式バリデーション） |
| v1.0 | 初回リリース（6セクション構成） |

**カラー規約（diff_viewerとは別系統）**
- 赤（`v-err` / `badge-err`）: 値異常（形式・範囲・期待値違反）← v1.2追加
- 紫（`v-chg`）: B入力済・A→B変更あり
- 白（`v-same`）: B入力済・変更なし
- 黄（`v-empty`）: 未入力 ※ケースバイケースで空でOKな項目あり

**C-RAN自動判別ロジック（③セクション）**
```javascript
// devices[] が空 → C-RAN。devicesに要素あり → MU構成
const cranMuNamesB = new Set(
  shB.radioGroups.filter(r => !r.devices.length).map(r => r.radioGroupIdName)
);
// C-RAN担当ネットワークを周波数順ソート → 最低周波数をアラーム取得側とする
const FREQ_ORDER = {'700M':1,'900M':2,'1.5G':3,'1.7G':4,'2.1G':5,'2.5G':6,...};
```
- 最低周波数側セクタ: rruExtAlarm 入力必須（未入力→🟡未入力、"-"入力→🔴値異常）
- その他セクタ: 空が正解（値があれば🔴値異常）
- HTML要素: `#mu-table`（MU構成）/ `#cran-table`（C-RAN構成）を構成に応じて表示切替

**親RRU_Cascade の入力ルール**（`_ref/【DUKe】親RRU_Cascade 入力方法について.pdf` 参照）
- 記載対象: ERI RP6339 (2G/900/1.5G) / NOKIA C-RAN構成 FD_BAND
- `"BBU直収"` = BBUに直接接続のセクタ
- `"KX-XXXXX_00001"` = カスケード接続時の接続元セクタ管理番号
- スター接続（FY21以降共同構築等）は全セクタ `"BBU直収"`
- Hardsplit構成はセクタの若番のセクタ管理番号
- 入力タイミング: **詳細図提出まで**

---

### spec_check.html — 図面チェックリスト v2.1

B側JSON + 図面PDF（省略可）をD&Dして図面照合するツール。
PDF.jsによる自動テキスト照合機能付き。

| セクション | 確認内容 | PDF自動照合対象 |
|-----------|---------|----------------|
| アンテナ | 方位・高さ・M-tilt・機種名 | 機種名 + 方位角 + 高さ |
| セクター×アンテナ | 電気チルト（eTilt）・機種名 | eTilt値 + 機種名 |
| MU装置 | 装置名・機種名・RAN種別 | 装置機種名 |
| RU×ポート | ポート番号・機種名・送受 | RU機種名 |
| 電源 | 受電方法・バッテリー | 受電方法名称 |

**自動照合の仕組み**
- PDF.js（CDN: v3.11.174）で全ページテキスト抽出 → JSON値と文字列照合
- 🟢 全値検出 / 🟡 一部検出 / 🔴 未検出
- 1桁数値は `3°` `3.0` 形式で検索（誤検出抑制）
- CAD出力PDF（テキスト選択可）で有効。スキャン画像PDFは不可

**レイアウト**
- PDF読み込みあり → 左48%にPDF表示 / 右にチェックリスト（トグル可）
- PDF読み込みなし → チェックリストのみ全幅

**Claude Codeで追加照合する場合**
1. `_ref/` に図面PDFをコピー
2. Claude Codeに依頼: `_ref/xxx.pdf と _ref/B_actual_spec.json を照合して不一致を指摘して`

---

---

## アラームパターン ビジネスルール（重要）

### 構成の判定
`shared.radioGroups[].devices[]` の有無で構成を判定する。

| 構成 | 判定条件 | アラーム取得箇所 | 入力フィールド |
|------|---------|----------------|---------------|
| **MU構成**（D-RAN）| `devices[]` に装置が存在 | MU（無線機グループ）でアラーム取得 | `radioGroups[].alarmPatternL/W/Fiveg` |
| **C-RAN構成** | `devices[]` が空 | RRU側でアラーム取得 | `networks[].sectors[].rruExtAlarm` |

### MU構成のアラームパターン入力ルール
各MUのサービスバンド（`networks[].lteServiceBand/fivegServiceBand/wcdmaServiceBand`）で判定：

| バンドの有無 | alarmPatternL / alarmPatternW | alarmPatternFiveg |
|-------------|------------------------------|-------------------|
| 対象バンドあり | パターン値を入力（`-` は不可） | パターン値を入力（`-`・`No` は不可） |
| 対象バンドなし（L/W） | `-` を入力 | — |
| 5Gなし | — | `No` を入力 |

### C-RAN構成のアラームパターン入力ルール
- フィールド: `networks[].sectors[].rruExtAlarm`
- **低い周波数側のRRUがアラームを取得する**
  - 例: 1.7G + 700M の工事 → 700M側のセクタに入力
  - 例: 1.7G + 2.5G の工事 → 1.7G側のセクタに入力
- 高い周波数側のセクタは入力不要（または `-`）

### 周波数の大小関係（参考）
低 ← 700M < 900M < 1.5G < 1.7G < 2.1G < 2.5G < 3.4G < 3.5G < 3.9G < 4.7G → 高

### サイト例: S280329129
- 1.7G: LTE + 5G、700M: 5G、2.5G: LTE
- C-RAN構成 → 700M側（低い周波数）のセクタ `rruExtAlarm` に入力

---

## _ref/ フォルダ（.gitignore対象）

| ファイル | 内容 |
|---------|------|
| `A_actual_spec.json` | 実際のA側諸元JSON（89KB）|
| `B_actual_spec.json` | 実際のB側諸元JSON（220KB・MU×3）|
| `B_actual_spec_S280329129.json` | C-RANサイト実データ（MU_1/2/3全てdevices=0、700Mセクタ1/2にrruExtAlarm入力）|
| `json_fields.csv` | JSONフィールド一覧（60項目）|
| `【関西】DUKe諸元管理更新担当項目.xlsx` | 施工会社担当50項目の元ソース（J列フィルタ）|
| `【関西】 基地局建設業務 フロー変更説明会（補足）.pdf` | DUKe入力SOW変更内容（アラームパターン・親RRU_Cascade施工会社移管）|
| `【DUKe】親RRU_Cascade 入力方法について.pdf` | 親RRU_Cascade入力ルール・入力例（カスケード/スター/Hardsplit）|

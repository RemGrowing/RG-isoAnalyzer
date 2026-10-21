# ISO Analyzer

**実行環境:** Microsoft Edge / Windows 11 / ローカルHTML  
**基本方針:** 外部CDN・有料APIなし、ブラウザ内ローカル処理、大容量ISOを全量RAM展開しない

---

## 1. 概要

ISO Explorer Edge Pro は、ブラウザ上でISO/IMG/BIN系ディスクイメージを解析し、ファイルシステムをツリー表示するとともに、選択項目またはISO全体をZIPとして取り出すための単一HTMLアプリケーションである。

v8では、従来のISO9660中心の解析から、**ISO/IEC 13346（ECMA-167）を基礎とするUDFファイルシステムの実解析**へ大幅に拡張した。

UDFの公式仕様はEcma TR/112で、UDF 1.02 / 1.50 / 2.00 / 2.01 / 2.50 / 2.60が収録されている。ECMA-167はISO/IEC 13346として、ボリューム・ファイル・レコード構造を規定する基盤規格である。

---

# 2. バージョン履歴

## 2.1 `ISO_Explorer_Edge_Final_Reviewed.html`

### 位置付け
初期の「実用版」に相当する基礎実装。

### 主な機能
- ISO9660解析
- PVD解析
- ボリューム情報取得
- ルートディレクトリ解析
- ファイル/ディレクトリツリー表示
- Joliet検出
- Rock Ridge検出・基本情報取得
- El Torito / Boot Record解析
- Path Table解析
- セクタマップ作成
- UDF構造の基本検出
- ZIP STORE方式による抽出
- 解析データのZIP化

### 主な問題
初期版ではISO全体をメモリへ保持する処理が残っており、10GB級ISOを扱う用途には不適切だった。

---

# 3. `ISO_Explorer_Edge_Pro_v2_10GB.html`

## 目的
10GB級ISOを想定した大容量化。

## 主な変更
- `Blob.slice()`ベースの部分読み出し
- ISO全体のRAM展開を回避
- 2048 / 2352 / 2336 / 2448系のセクタ配置を検出
- ZIP64を導入
- Edge File System Access APIを利用した直接保存方式を導入
- 大容量ファイルのチャンク処理
- ISO9660 / Joliet / Rock Ridge / El Torito / Path Table / UDF構造解析を維持

## 効果
10GB級ISOを「全体をUint8Arrayにする」方式から脱却し、必要な範囲だけ読み出せる構造になった。

---

# 4. `ISO_Explorer_Edge_Pro_v3_Fixed.html`

## 目的
v2の大容量ZIP生成に存在したZIP64周辺の問題とUDF検査の遅さを修正。

## 主な変更
- ZIP64 Local HeaderのExtra Fieldを修正
- ZIP64サイズ情報を正しい構造に整理
- UDFサンプル読み込みを多数の個別非同期読み出しから連続読み出しへ変更
- 4GB超ファイルを扱うための64bitサイズ表現を強化
- ZIP生成時の余計なメモリ保持を削減

## 重要
v3以降は「10GBを扱える」だけでなく、ZIPコンテナ自体が4GB境界を超えることを前提に設計されている。

---

# 5. `ISO_Explorer_Edge_Pro_v4_Fast.html`

## 目的
10GB級ISOで「解析中」のまま長時間待たされる問題を改善。

## 主な変更
- ISO9660 Volume Descriptor領域を個別セクタごとに読む方式から、最大256論理ブロックを一括取得する方式へ変更
- Descriptor解析をRAM上で実行
- UDF Anchor検出を標準位置中心に整理
- 不要な大量の非同期Blob読み出しを削減

## 効果
Windows/Edge環境で問題になりやすい大量の`slice().arrayBuffer()`境界を減らし、解析開始時の待ち時間を短縮。

---

# 6. `ISO_Explorer_Edge_Pro_v5_RuntimeFixed.html`

## 目的
実際のSliTaz ISOで発生したランタイムエラーを修正し、ISO9660解析を実用状態へ。

## 主な修正
`parseDescriptors()`の戻り値が

`{ descriptors, terminated }`

であるのに対し、El Torito解析へ誤った形式を渡していた問題を修正。

修正後はDescriptor集合を正しく渡す構造となった。

## 実ISO確認
SliTaz ISOで以下を確認済み。

- PVD検出
- Volume ID: `SLITAZ_BASE_X86_64`
- ISO9660 logical block size: 2048
- Root entries:
  - BOOT
  - BOOT.CAT;1
  - EFI
  - INDEX.HTM;1
  - README.;1
- 再帰エントリ数: 15
- El Torito Boot Record検出
- 解析時間が極端に長くなる問題を解消

この段階で「ISOが空に見える」問題から脱却した。

---

# 7. `ISO_Explorer_Edge_Pro_v6_10GB_ZIP64_Fixed.html`

## 目的
10GB ISOから生成したZIPがWindows 11で「無効」と表示される問題を修正。

## 主な変更

### ZIP64
- 4GB超ファイルをZIP64として扱う
- ZIP64 Local Header
- ZIP64 Data Descriptor
- ZIP64 Central Directory
- ZIP64 End of Central Directory
- ZIP64 End of Central Directory Locator
- 64bitオフセット

### データディスクリプタ
ZIP64ではCRC32に続くサイズ値を64bitで記録する。

従来の32bitサイズを誤って使用するとWindowsの圧縮フォルダーで「無効です」となるため、24バイト形式へ修正。

### ストリーミング
Edge File System Access APIへ直接書き込む方式を採用。

### CRC32
slicing-by-8方式を採用。

### 進捗
- 現在のファイル番号
- 処理済みサイズ
- 総サイズ
- 進捗率
- CRC32

を表示する。

---

# 8. `ISO_Explorer_Edge_Pro_v7_ISO13346_AutoSwitch.html`

## 目的
ISO9660だけでなく、ISO/IEC 13346 / ECMA-167ベースのUDFファイルシステムを実際にファイルツリーまで解析する。

## 自動判定

```text
ISO
 ├─ ISO9660検出
 │   ├─ UDFも検出 → UDFを優先
 │   └─ UDFなし → ISO9660 / Joliet
 │
 └─ ISO9660なし
     └─ UDF Anchor検出 → UDF
```

## UDF解析

- Anchor Volume Descriptor Pointer
- Volume Descriptor Sequence
- Primary Volume Descriptor
- Partition Descriptor
- Logical Volume Descriptor
- Partition Map
- File Set Descriptor
- Root Directory ICB
- File Entry
- Extended File Entry
- File Identifier Descriptor
- Short Allocation Descriptor
- Long Allocation Descriptor
- Embedded Data
- CS0 Unicode
- UDFディレクトリ再帰解析
- UDFファイルの実データ取得

## ISO9660との共存

ISO9660 + UDF Bridge形式では、利用可能なUDFファイルシステムツリーを優先する設計。

---

# 9. `ISO_Explorer_Edge_Pro_v8_UDF_1.02-2.60_Advanced.html`

## 位置付け

v8は現在の高度版。

UDFの通常ファイルツリーだけでなく、UDFのPartition Mapを解析し、特殊パーティション方式の情報を取得する。

## UDF対象リビジョン

- UDF 1.02
- UDF 1.50
- UDF 2.00
- UDF 2.01
- UDF 2.50
- UDF 2.60

ただし、これは「全仕様の書き込みまで完全実装」という意味ではなく、**読取・解析を主目的とした実装**である。

---

# 10. v8の内部解析フロー

```text
ISO / IMG / BIN
      │
      ▼
セクタ配置検出
      │
      ├─ 2048
      ├─ 2352
      ├─ 2336
      └─ 2448
      │
      ▼
ISO9660 / UDF Anchor判定
      │
      ├─────────────┐
      ▼             ▼
 ISO9660            UDF
      │             │
      │             ├─ Anchor
      │             ├─ Descriptor Sequence
      │             ├─ Partition Descriptor
      │             ├─ Logical Volume Descriptor
      │             ├─ Partition Map
      │             ├─ File Set Descriptor
      │             └─ Root ICB
      │                       │
      │                       ▼
      │                  File Entry
      │                       │
      │                       ▼
      │                  File Identifier
      │                       │
      │                       ▼
      │                  Directory Tree
      │
      ▼
  ファイルツリー
      │
      ▼
  ZIP / ZIP64
```

---

# 11. 対応しているもの

| 分類 | 対応項目 | 状態 | 詳細 |
|---|---|---|---|
| ISO9660 | Primary Volume Descriptor | 実装済み | Volume ID、容量、Logical Block Size、Path Table、Root Directory等 |
| ISO9660 | Volume Descriptor Sequence | 実装済み | Terminatorまで解析 |
| ISO9660 | Path Table | 実装済み | Little-endian Path Tableを解析 |
| ISO9660 | Directory Record | 実装済み | 名前、属性、LBA、サイズ、日付 |
| ISO9660 | Multi-extent | 実装済み | 複数Extentを統合 |
| ISO9660 | Joliet検出 | 実装済み | Supplementary Volume Descriptorを検出 |
| ISO9660 | Rock Ridge | 基本対応 | NM/PX等を解析 |
| ISO9660 | El Torito | 実装済み | Boot Record、Boot Catalog、Entry情報 |
| ISO9660 | Raw sector layout | 対応 | 2048/2352/2336/2448系を検出 |
| UDF | ISO/IEC 13346 / ECMA-167 | 対応 | UDFの基盤構造を解析 |
| UDF | Anchor Volume Descriptor Pointer | 実装済み | 256、末尾付近等の候補を検査 |
| UDF | Main Volume Descriptor Sequence | 実装済み | 連続領域を取得して解析 |
| UDF | Primary Volume Descriptor | 実装済み | Volume ID等 |
| UDF | Partition Descriptor | 実装済み | Partition Number、Start、Length等 |
| UDF | Logical Volume Descriptor | 実装済み | Logical Block Size、Contents Use、Partition Map等 |
| UDF | Type 1 Partition Map | 実装済み | Partition Numberを解決 |
| UDF | Type 2 Partition Map | 実装済み | Identifierを解析 |
| UDF | File Set Descriptor | 実装済み | Root ICBを取得 |
| UDF | Root Directory ICB | 実装済み | File Entryへ解決 |
| UDF | File Entry | 実装済み | サイズ、File Type、Allocation Descriptor |
| UDF | Extended File Entry | 実装済み | Tag 266を解析 |
| UDF | File Identifier Descriptor | 実装済み | 名前、属性、ICB |
| UDF | Short Allocation Descriptor | 実装済み | 物理LBAへ変換 |
| UDF | Long Allocation Descriptor | 実装済み | Partition Referenceを使用 |
| UDF | Embedded Data | 実装済み | File Entry内部データを直接取得 |
| UDF | CS0 UTF-8 | 実装済み | Character Set ID 8 |
| UDF | CS0 UTF-16BE | 実装済み | Character Set ID 16 |
| UDF | Virtual Partition Map | 部分対応 | VAT情報を読み取り、論理ブロックを物理位置へマップ |
| UDF | VAT | 部分対応 | `*UDF Virtual Alloc Tbl`を検出しVAT Entryを取得 |
| UDF | Previous VAT ICB | 解析対応 | VATの前世代ICB位置を取得 |
| UDF | Sparable Partition Map | 部分対応 | Packet/Sparing Table位置等を解析 |
| UDF | Sparing Table | 部分対応 | Original→Mappedの再配置情報を取得 |
| UDF | Metadata Partition Map | 部分対応 | Metadata/Mirror/Bitmap位置、Allocation Unit等を取得 |
| UDF | Metadata File | 構造情報対応 | Metadata File ICB位置を取得 |
| UDF | Metadata Mirror | 構造情報対応 | Mirror位置を取得 |
| UDF | Metadata Bitmap | 構造情報対応 | Bitmap位置を取得 |
| UDF | Logical Block → Physical LBA | 実装済み | Partition Mapを通して解決 |
| UDF | Directory recursion | 実装済み | 深度上限64、循環参照対策 |
| UDF | Entry上限 | 実装済み | 最大200,000エントリ |
| 大容量 | 部分読み込み | 実装済み | Blob.slice()で必要範囲のみ取得 |
| 大容量 | 10GB級ISO | 対応設計 | ISO全体をRAMへ展開しない |
| ZIP | STORE | 実装済み | 圧縮なしでデータをそのまま格納 |
| ZIP | ZIP64 | 実装済み | 4GB超ファイル/アーカイブを対象 |
| ZIP | ZIP64 Data Descriptor | 実装済み | 64bitサイズ |
| ZIP | ZIP64 Central Directory | 実装済み | 64bitサイズ/オフセット |
| ZIP | ZIP64 EOCD | 実装済み | ZIP64 End Record |
| ZIP | ストリーム保存 | 実装済み | Edge File System Access API |
| ZIP | CRC32 | 実装済み | slicing-by-8 |
| UI | ドラッグ&ドロップ | 実装済み | ISO/IMG/BIN |
| UI | ツリー表示 | 実装済み | ファイル/ディレクトリ |
| UI | 詳細表 | 実装済み | 名前、種類、サイズ、LBA、セクタ数 |
| UI | 解析ログ | 実装済み | エラー・進捗を表示 |
| 解析 | Analysis ZIP | 実装済み | JSON/RAW構造データを出力 |

---

# 12. 解析対象となるUDF構造

## Descriptor Tag

v8では以下の主要Tagを扱う。

- 1: Primary Volume Descriptor
- 2: Anchor Volume Descriptor Pointer
- 3: Volume Descriptor Pointer
- 4: Implementation Use Volume Descriptor
- 5: Partition Descriptor
- 6: Logical Volume Descriptor
- 7: Unallocated Space Descriptor
- 8: Terminating Descriptor
- 9: Logical Volume Integrity Descriptor
- 256: File Set Descriptor
- 257: File Identifier Descriptor
- 258: Allocation Extent Descriptor
- 259: Indirect Entry
- 260: Terminal Entry
- 261: File Entry
- 266: Extended File Entry

---

# 13. UDF Partition Map

## Type 1

通常のPartition Map。

```text
Logical Block
      ↓
Partition Number
      ↓
Partition Descriptor
      ↓
Partition Start
      ↓
Physical LBA
```

## Virtual Partition

```text
Logical Block
      ↓
VAT Entry
      ↓
Physical Block
      ↓
Partition Start
      ↓
Physical LBA
```

v8は`*UDF Virtual Alloc Tbl`を検索し、VAT Entryを読み取って論理ブロックを物理位置へ変換する。

## Sparable Partition

```text
Logical Block
      ↓
Sparing Table
      ↓
Original Block → Mapped Block
      ↓
Partition Start
```

Sparing Tableの再配置情報を取得する。

## Metadata Partition

Metadata File、Metadata Mirror、Metadata Bitmapの位置と、Allocation Unit Size / Alignment Unit Size / Flagsを解析する。

ただし、Metadata Partitionの完全なAllocation処理は「高度な構造解析・部分対応」の位置付けであり、全UDFケースで完全なファイルアクセスを保証するものではない。

---

# 14. 大容量ISO設計

## メモリ

10GB ISOを次のように処理する。

```text
10GB ISO
  ↓
必要な範囲だけ Blob.slice()
  ↓
チャンク処理
  ↓
CRC32
  ↓
ZIPへ直接write()
```

ISO全体を以下のような単一配列へ変換する設計ではない。

```text
new Uint8Array(10GB)
```

## ZIP出力

Edge File System Access APIが利用可能な場合、

```text
ISO
 ↓
ZIP Local Header
 ↓
ISOデータ
 ↓
Data Descriptor
 ↓
次のファイル
 ↓
Central Directory
 ↓
ZIP64 EOCD
```

の順に直接書き込む。

---

# 15. ZIP64仕様

ZIP64を使用する条件の主なもの。

- ファイルサイズ >= 4GiB境界
- ZIP内オフセット >= 4GiB
- Central Directoryサイズ >= 4GiB
- エントリ数 >= 65,535

ZIP64では以下を使用する。

- 64bit Uncompressed Size
- 64bit Compressed Size
- 64bit Local Header Offset
- ZIP64 End of Central Directory
- ZIP64 Locator

圧縮方式はSTOREであり、ISOファイルを再圧縮しない。

これはISOイメージからの抽出用途では処理時間とCPU負荷を抑えるための設計である。

---

# 16. 解析データZIP

v7/v8系では、解析結果をJSON/RAWとして保存する。

代表例:

```text
json/
  analysis_summary.json
  iso9660_descriptors.json
  iso9660_path_table.json
  eltorito.json
  joliet.json
  udf_structure.json
  udf_descriptors.json
  iso9660_sector_map.json
  rock_ridge.json

raw/
  sector_16_pvd.bin
  volume_descriptor_set_first_32_sectors.bin
  udf_anchor_sector.bin
```

解析ZIP自体は小容量のメタデータ中心であるため、10GBのISOデータ本体をそのZIPへコピーする方式ではない。

---

# 17. 自動切替仕様

## 優先順位

1. UDFが検出され、利用可能なUDFファイルツリーが構築できる
   - UDFを使用
2. UDFが利用できずISO9660が利用可能
   - ISO9660を使用
3. Jolietが存在する場合
   - ISO9660側のUnicode名称情報として扱う
4. Rock Ridgeが存在する場合
   - Rock Ridgeの名称等を優先して表示
5. どの形式も認識できない
   - 解析エラー

## UDF Bridge

DVD等のUDF BridgeではISO9660とUDFが共存する。

v8ではUDFが有効なファイルシステムとして構築できた場合、UDF側を優先する。

---

# 18. 安全性・破損ISOへの対応

以下を実装している。

- ディレクトリ深度制限
- UDF循環参照対策
- UDFエントリ数上限
- ファイルデータ不足チェック
- 解析エラー表示
- 不正なDescriptor長の検査
- 範囲外Blob読み込みの抑制
- 物理LBA変換失敗の明示
- ZIP書き込み失敗時のabort

---

# 19. 現時点での「完全対応ではない」項目

「UDF 1.02～2.60対応」は、**各仕様書に存在するすべての書込み機能・すべての特殊ケースを完全実装した**という意味ではない。

特に以下は追加検証・強化対象。

- VATの複数世代チェーンを完全追跡
- 巨大VATの複数ブロック処理
- Metadata Partitionの完全なICB/Allocation処理
- Metadata Mirrorへの自動フォールバック
- Metadata Bitmapによる全状態解決
- Sparing Tableの複数テーブル完全処理
- 複雑なMulti-session
- 複雑なVirtual Partition Map
- Indirect Entryの完全処理
- Terminal Entryの完全処理
- Extended Attribute
- Named Stream
- Hard Link
- UDF ACL
- Secure UDF
- 全UDF破損リカバリ
- 全DVD/Blu-ray実機イメージでの互換性検証

したがって仕様書上の表では「実装済み」「部分対応」「構造情報対応」を分けている。

---

# 20. 推奨テストISO

最低限、次のカテゴリで検証する。

### ISO9660
- SliTaz
- Debian
- Ubuntu
- CentOS

### UDF Bridge
- DVD-Video
- Windows系インストールメディア
- UDF + ISO9660共存イメージ

### UDF
- UDF 1.02
- UDF 1.50
- UDF 2.00
- UDF 2.01
- UDF 2.50
- UDF 2.60

### 大容量
- 4GB未満
- 4GB境界付近
- 8GB
- 10GB
- 20GB以上

### ZIP互換性
生成したZIPを以下で検証する。

- Windows 11 エクスプローラー
- 7-Zip
- PowerShell Expand-Archive
- Python zipfile

特に10GB級では、生成後のZIPサイズ、Central Directory、ZIP64 EOCD、各EntryのCRC32を検証する。

---

# 21. 現在のアーキテクチャ評価

| 項目 | 評価 |
|---|---|
| ISO9660基礎解析 | 高 |
| ISO9660大容量対応 | 高 |
| Joliet | 中～高 |
| Rock Ridge | 中 |
| El Torito | 中 |
| UDF基本解析 | 高 |
| UDF通常ファイルツリー | 高 |
| UDF特殊Partition | 中 |
| VAT | 中 |
| Sparing | 中 |
| Metadata Partition | 中～低 |
| 10GB級読み出し | 高 |
| ZIP64構造 | 高 |
| Windows ZIP互換性 | v6以降で大幅改善 |
| 全UDF 1.02～2.60完全互換 | 未達 |
| 破損ISO耐性 | 中 |

---

# 22. 公式仕様との関係

ECMA-167はISO/IEC 13346として、非シーケンシャル記録媒体のボリューム・ファイル・レコード構造を規定する。

UDFはECMA-167のサブセットとして定義され、Ecma TR/112にはUDF 1.02、1.50、2.00、2.01、2.50、2.60が収録されている。

UDF BridgeはECMA-119（ISO9660）とUDF 1.02を組み合わせた形式として定義されている。

---

# 23. 結論

ISO Explorer Edge Proは、

**v1系**
→ ISO9660を解析するブラウザISO Explorer

から、

**v2～v6**
→ 10GB級ISOとZIP64を扱える大容量ISO Explorer

へ進化し、

**v7**
→ ISO/IEC 13346 / ECMA-167 / UDFを実際のファイルシステムとして自動判定

へ拡張され、

**v8**
→ UDF Partition Map、VAT、Sparing、Metadata Partitionなどの高度な構造解析まで対応した。

現行v8の最大の特徴は、

> **ISOの中身を「単なるバイナリ」としてではなく、ISO9660またはUDFのファイルシステムとして解析し、必要なファイルデータだけを読み出してZIP64へストリーム出力する**

という構成である。

ただし、UDF 1.02～2.60の「全仕様完全互換」を名乗るには、特殊Partitionの実ISO検証と、VAT/Metadata/Sparing/Multi-session等の追加実装が必要である。

---

## 参考仕様

- ECMA-167: Volume and file structure for write-once and rewritable media using non-sequential recording for information interchange
- ISO/IEC 13346: ECMA-167に対応する国際規格番号
- ECMA TR/112: Universal Disk Format (UDF) specification
- ECMA TR/71: DVD read-only disk / UDF Bridge
- ECMA-119: CD-ROM volume and file structure

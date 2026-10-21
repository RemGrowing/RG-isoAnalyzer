# ISO Analyzer 仕様書

## 1. 対応規格・ファイルシステム一覧

ソースコード内で判別・パース処理が実装されている規格および構造の一覧です。

### 1.1. ISO 9660 規格および拡張規格
* **ISO 9660 (Base Level)**
  * **ボリューム記述子 (Volume Descriptors):**
    * Primary Volume Descriptor (PVD, Type 1) のパース（System ID, Volume ID, Volume Space, Logical Block Size, Path Table 位置, Root Directory Record, 各種日付フィールドなど）。
    * Boot Record Descriptor (Type 0) の読み込み。
    * Supplementary Volume Descriptor / Volume Descriptor Set Terminator (Type 255) の認識。
  * **セクタサイズおよびオフセット自動検出:**
    * セクタサイズ 2048, 2352, 2336, 2448 バイト、データオフセット 0, 8, 16, 24, 32, 36 バイトの組み合わせ（CD001 マジックナンバー検出）に対応。
  * **パステーブル (Path Table):**
    * LPathTable (リトルエンディアン) および MPathTable (ビッグエンディアン) の構造解析。
  * **マルチエクステント (Multi-extent):**
    * ファイルフラグ `0x80` を検出した場合、同一ファイルの複数エクステント（LBAおよびサイズ）を単一ノードとして結合・管理。
* **Joliet 拡張**
  * SVD (Type 2) かつ エスケープシーケンス `% /` (0x25, 0x2f) を判定して識別。UCS-2 / UTF-16 ロングファイルネームに対応。
* **Rock Ridge 拡張**
  * Directory Record 内の System Use (SU) フィールドを解析。
  * `NM` シグネチャ: Rock Ridge ロングファイルネームのデコード。
  * `PX` シグネチャ: POSIX 属性（ファイルモード、UID、GID）の取得。
* **El Torito ブート規格**
  * Boot Record (Type 0) の `bootCatalogLba` より Boot Catalog セクタを読み込み。
  * Platform ID, Bootable フラグ (`0x88`), Load Segment, Sector Count, Image LBA 等のブートエントリ構造を解析。

### 1.2. UDF (Universal Disk Format) / ISO-13346 (ECMA-167) 規格
* **アンカー位置検出 (Anchor Volume Descriptor Pointer - AVDP):**
  * LBA 256、またはイメージ末尾（`total-1`, `total-256`, `total-257` 付近）からの検出、および 4MiB 境界フォールバック境界スキャンを実装。
* **対応 Structural Tags (記述子タグ):**
  * Tag 1: Primary Volume Descriptor (PVD)
  * Tag 2: Anchor Volume Descriptor Pointer (AVDP)
  * Tag 5: Partition Descriptor (PD)
  * Tag 6: Logical Volume Descriptor (LVD)
  * Tag 8: Terminating Descriptor (TD)
  * Tag 256: File Set Descriptor (FSD)
  * Tag 257: File Identifier Descriptor (FID)
  * Tag 261: File Entry (FE)
  * Tag 266: Extended File Entry (EFE)
* **対応パーティションマップ (Partition Maps):**
  * **Type 1 (Physical Partition):** 物理LBAへの直線マッピング。
  * **Type 2 (Advanced Partition Maps):**
    * **Virtual Partition (VAT / Virtual Allocation Table):** `*UDF Virtual Alloc Tbl` 文字列検出による VAT 読み取り (UDF 1.50/2.50+)。
    * **Sparable Partition:** `*UDF Sparing Table` 文字列検出による欠陥管理・代替セクタの非物理マッピング補正。
    * **Metadata Partition:** メタデータファイル ICB 経由の空間アドレス解決。
* **アロケーション記述子 (Allocation Descriptors - AD):**
  * Mode 0: Short Allocation Descriptors
  * Mode 1: Long Allocation Descriptors
  * Mode 2: Extended Allocation Descriptors
  * Mode 3: Embedded Data (File Entry 内に直接埋め込まれたデータ)
* **文字コード (CS0 Character Set):**
  * CS0 形式の 8-bit (UTF-8 互換 ascii/latin1) および 16-bit (UTF-16BE) デコード処理。

### 1.3. 除外・非対応ファイルの自動判定
* **AppleDouble Sidecar ファイル拒否:**
  * ファイル名が `._` で始まり、かつ先頭 4 バイトのマジックナンバーが `0x00051607` または `0x00051600` の場合、ISO ファイルではないとして即座にエラーを返却。

---

## 2. 技術的アーキテクチャと主要アルゴリズム

### 2.1. メモリ効率とオンデマンド読み込み
* **Blob.slice() による部分読み取り:**
  * 数 GB〜10 GB 超の ISO ファイルであっても、ファイル全体を RAM にロードせず、必要なセクタ/バイト範囲のみを Blob / File API 経由でオンデマンド読み込み。
* **Electron デスクトップ領域ブリッジ:**
  * Electron 環境においては `window.isoExplorerDesktop.readFileRange(path, offset, length)` を使用し、ネイティブファイル I/O による高速部分読み込みを実行。

### 2.2. アロケーション境界をまたぐデータ読み取り (`readNodeRangeV17`)
* 巨大ファイルや複数エクステントに分散したファイル読み出しにおいて、16 MiB などの一定チャンクサイズ要求に対してアロケーション境界（Extent 境界）で途切れないよう、要求範囲 `[pos, pos+take)` に該当する全 Extent を走査・結合して連結して返却するロジックを搭載。

### 2.3. マルチスレッド並列一括処理 (Parallel Batch Isolation)
* **Iframe ベースのサンドボックス並列化:**
  * JSの単一スレッドおよびモジュールレベル状態（`iso`, `fsInfo` 等のグローバル変数）による競合を防ぐため、1 ISO あたり 1 つの隠し `iframe`（Web Worker 類似の独立 scope）を生成。
  * `PARALLEL_BATCH_LIMIT` (最大 4 スレッド、または CPU コア数-1) による制御を実施。
  * メインウィンドウと iframe 間は `window.postMessage` および非同期 RPC 方式で通信し、Electron ネイティブバインディングの呼び出しを中継。

### 2.4. GPU アクセラレーション & ハッシュ算定機構
リカバリメタデータ（16 MiB 単位のチャンク指紋）生成処理において、3 段階のアクセラレーション階層を持つ。

1. **Electron Native C++ Accelerator (最優先):**
   * `window.isoExplorerDesktop.nativeFingerprint` による C++ ネイティブ処理。
2. **WebGPU Compute Shader (第 2 優先):**
   * WGSL (WebGPU Shading Language) の Compute Shader により、16 MiB チャンク並列で CRC32 および 128-bit FNV 派生ハッシュ (`h0, h1, h2, h3`) を計算。
   * ワークグループサイズ 64 の Compute Pipeline を使用。
3. **CPU Fallback / Slicing-by-8 (最下層):**
   * Slicing-by-8 アルゴリズムを用いたルックアップテーブル (`CRC8[8][256]`) による高速 CPU CRC32 計算および `cpuChunkFingerprint` (JavaScript 実装) へのフォールバック。

### 2.5. ストリーミング ZIP64 生成機構
* **メモリ非依存のディスク直接ストリーミング:**
  * `FileSystemWritableFileStream` (File System Access API `showSaveFilePicker`) または Electron の Native Zip Writer を利用し、ZIP データを直接ストレージに書き込み（RAM 上への大容量 Blob アセンブリを回避）。
* **ZIP64 構造の厳密な出力:**
  * 4GB 超のファイルまたはオフセットに対応するため、Zip64 Extra Field (Tag `0x0001`), Zip64 End of Central Directory Record (`0x06064b50`), Zip64 End of Central Directory Locator (`0x07064b50`), 24 バイト Data Descriptor 形式を正しく構築。

---

## 3. リカバリ・解析データ生成機能 (Manifest / Data Export)

* **Analysis ZIP 生成:**
  * 検出した descriptors (PVD, SVD, UDF Tags)、Path Table, El Torito, Rock Ridge, Sector Map, および先頭 32 セクタ / PVD / UDF Anchor セクタの生バイナリ (`.bin`) を JSON/BIN 形式で一括 ZIP 出力する機能。
* **ISO Recovery Chunks & Manifest:**
  * ISO 元データ全体を 16 MiB 毎のチャンク分割形式 (`__ISO_RECOVERY__/chunks/chunk_XXXXXX.bin`) で保存。
  * 128-bit 指紋、CRC32、重複チャンクインデックス (`duplicateOf`)、ソースレイアウト情報を記述した `manifest.json` を動的に生成し同梱。

---

## 4. 結論

本コードは、純粋な JavaScript/HTML5 環境および Electron デスクトップ環境の両方で動作し、**ISO 9660 (Joliet, Rock Ridge, El Torito 含む)** および **UDF/ISO-13346 (各種パーティションマップ, VAT, Sparing Table 含む)** の高度な自動判定とパースを行うグラフィカル・エクスプローラーです。WebGPU と Stream ZIP64 処理により、大容量イメージであってもメモリ枯渇を引き起こさずに安全かつ高速に処理できるよう設計されています。

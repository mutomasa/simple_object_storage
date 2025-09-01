# オブジェクトストレージ自作設計メモ（Go + Raft）

## 全体構成
- **Control plane（メタデータ層）**  
  Bucket/オブジェクト名 → 場所、バージョン、パーツ、ポリシー等の**唯一の真実**。  
  Raftリーダが書き込みを直列化。

- **Data plane（データ層）**  
  実体（オブジェクト/パーツ）を複数ノードに**レプリカ**または**イレージャーコーディング(EC)**で配置。  
  HTTP Range GETや並列PUTを高速化。

- **Gateway/API 層**  
  S3互換REST (SigV4)、認証・認可、マルチパート、一覧、ポリシーなど。

---

## MVP

### 1) メタデータと合意（Raft）
- **ライブラリ**：`etcd/raft`（低レベルで柔軟）または `hashicorp/raft`（実装しやすい）
- **K/Vスキーマ例**
  - `bucket/<name>`：バケット属性
  - `obj/<bucket>/<key>`：ヘッダ（サイズ、ETag、バージョン、パーツ一覧、placement）
  - `mpu/<bucket>/<uploadId>`：マルチパート進行中
- **永続化**：Raftログ + スナップショット（Badger/Pebble/Boltなどの埋め込みKV）  
  → 起動高速化とログ肥大防止のスナップショット/ログ圧縮は必須

### 2) データ保存（Data node）
- **レイアウト**：`/data/<volume>/<shard>/<objectID>/<partN>` の **append-only**（追記）+ **tombstone**  
  → クラッシュ後のリプレイが楽。断片化は後述GCで回収
- **配置**：コンシステントハッシュ（仮想ノード）でN台選出（MVPはN=3レプリカ）
- **整合性**：  
  PUTはデータノードN台へ書き込み → **クォーラムACK**（例：2/3） → 最後にメタデータをRaftでコミット  
  途中失敗分は**オーファンGC**で後始末

### 3) S3 API最小セット
- **Bucket**：Create / Delete / List
- **Object**：PUT / GET / HEAD / DELETE、Range GET、ETag(MD5またはcontent-hash)
- **Multipart Upload**：Initiate / UploadPart / Complete / Abort（並列化の要）
- **Auth**：AWS SigV4（HMAC）
- **一覧 (LIST)**：プレフィックス+マーカー（v2のContinuationTokenでもOK）

### 4) 基本の可用性・監視
- **機能**：ヘルスチェック、リバランス（ノード増減時に再配置）、修復 (healing)（欠損レプリカの自動再生成）
- **メトリクス**：Prometheus（レイテンシ、失敗率、Raftラグ、ディスク使用）+ OpenTelemetry Traces
- **TLS**：最初はサーバTLS、後でmTLSも

---

## 本番向けに足すべき要素（ロードマップ）

### A) スケールするメタデータ
- 単一Raftグループの限界：QPS/レイテンシの頭打ち
- 水平分割：メタデータシャーディング（複数Raftグループ）＋ Placement Driver（TiKV風）  
  → バケットやキー範囲でシャード分け、ホットシャードのスプリット/マージ
- ストア：Pebble or Badger（LSM）+ Bloom FilterでLIST/HEAD高速化

### B) データ耐久性と効率
- **イレージャーコーディング**：`k+m`（例：4+2や8+3）。Goなら `klauspost/reedsolomon` が実績大
  - 耐久性：m台まで喪失OK
  - Writeパス：パリティ生成 → m+1以上のACKで可（可用性/耐久性のトレードオフ）
- **障害ドメイン**：ラック/AZアフィニティで配置（同一筐体偏りを回避）
- **リペア (healing)**：バックグラウンドで不足フラグメントを再生成

### C) 一貫性モデル
- **読み取り整合性**
  - 同一キーのRead-after-Write強整合（メタデータをリーダ経由/lease-read）
  - Listは最初はEventually、要件次第で整合List（高コスト：インデックスやVersion Mapが必要）
- **バージョニング**：`obj`に`versionId`付与、DeleteMarker対応、オブジェクトロック/WORM（準拠要件向け）

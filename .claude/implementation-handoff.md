# 実装フェーズ 引き継ぎメモ — SystemResourceMCP

> セッションのコンテキストを跨ぐための引き継ぎ資料。設計は `docs/` に確定済み。本書は**実装者が最短で正しく着手するための地図と、レビューで繰り返し指摘された落とし穴（再発防止）**を集約したもの。詳細は必ず `docs/` を参照すること（本書は要約でありソースではない）。

## 0. 現在地

- フェーズ体系: 要求 → 要件定義 → 基本仕様 → 詳細設計 ✅（全て `main` にマージ済み）→ **実装（←ここから）**
- 成果物（`main`、これが唯一の正）:
  - `docs/hearing.md` … 要求記録
  - `docs/reqire.md` … 要件定義（FR-1〜9, FR-7.1, NFR-1〜5）
  - `docs/basic-spec.md` … 基本仕様（**rev.3**。規範値の正）
  - `docs/detail-design.md` … 詳細設計（Go 型・関数シグネチャ・アルゴリズム）
- まだコードは無い（`cmd/`・`internal/` 未作成）。README/LICENSE/.gitignore(Go) のみ。

## 1. これは何か（1行）

ローカルで実験を回す AI エージェント（Claude 等）が計算資源の空きを判断できるよう、システムリソースを **STDIO トランスポートの MCP サーバー**として公開する Go 製単一バイナリ。公開ツールは `get_current_resources`（現状スナップショット）と `get_resource_trend`（直近ウィンドウの傾向を**英語・決定的・LLM非依存**で生成）。任意で将来 `search_processes`（MVP外）。

## 2. 文書の優先順位と改訂規律（最重要の作法）

- 優先順位: **要求/要件定義 > 基本仕様 > 詳細設計 > 実装**。
- **規範値（normative）は `basic-spec.md` が正**（傾向判定定数 §8.3、NFR-1 数値目標 §12.1、MCPスキーマ §6、設定規則 §5.3 等）。
- **実装や詳細設計で規範値を勝手に変えない**。実測で不適切と判明したら、握りつぶさず **`basic-spec.md` を改訂**してから追従する（`basic-spec.md` 冒頭「本書の確定と改訂の原則」）。
- **レビュー対応のために新しい要求・非目標・規範値を勝手に足さない**（依頼者が繰り返し強調）。実装時に一意かつ合理的に決まる詳細は、上位契約を変えない限り**実装判断でよい**（過剰に詳細設計へ書き戻さない）。

## 3. ブランチ / PR ワークフロー

- designated ブランチ: `claude/requirements-definition-hearing-6tjors`。
- 過去 PR（#1 要件, #2 基本仕様, #3 詳細設計）は**すべてマージ済み**。新規作業は **`main` から同名ブランチを作り直す**：
  `git fetch origin main && git checkout -B claude/requirements-definition-hearing-6tjors origin/main`
- **`main` へ直接 push しない**。作業はブランチ → **draft PR** → レビュー → マージ。PR 作成後は購読して CI/レビューに追従。
- コミット末尾に `Co-Authored-By: Claude ...` と `Claude-Session:` を付ける運用。

## 4. 実装時に絶対に外してはいけない確定事項（落とし穴・再発防止）

レビューで実際に指摘され、確定した内容。**実装で再発させないこと。**

### 設定（`basic-spec §5.3`）
- キーは TOML の `sampling_interval_seconds` / `window_seconds`。既定 10 / 60。
- 各項目を独立に検証（不正は該当項目のみ既定へ）。**実効ウィンドウ = `max(window, interval)`**（`window<interval` の矛盾回避、最低1サンプル保証）。引き上げたら stderr に warn。
- 探索順: CLI `--config` → env `SYSTEMRESOURCEMCP_CONFIG` → `os.UserConfigDir()/systemresourcemcp/config.toml`。無ければ既定。

### CPU（`basic-spec §7.2` / `detail-design §5.3`）
- 使用率は gopsutil（`cpu.Percent`）。**マルチソケットの論理→ソケット対応は OS 別トポロジ API**（Win: `GetLogicalProcessorInformationEx`、Linux: `/proc/cpuinfo` physical id、macOS: sysctl）。`PhysicalID` 単純対応は Win/mac で不成立なので使わない。
- **トポロジ取得不能時 `CPUPerSocket = []`（空配列）**。`[overall]` にしない（2ソケット機の誤表示になる）。単一ソケットと判明した場合のみ 1 要素 = 全体値。
- gopsutil の初回 `cpu.Percent` は基準未確立で **error を返しうる** → その場合 `CPU.OverallPercent=nil` の**部分サンプルを捨てずに Store へ入れる**（起動直後の即時応答を空にしない）。

### GPU（`basic-spec §7.3/§7.4` / `detail-design §5.4-5.7`）— 条件付きソース選択
- **1つのGPUは常に単一ソースから揃える。異種ソース間の StableID 突合はしない。**
- NVIDIA: `nvidia-smi` があれば `nvidia-smi -q -x` で完結（この時 DXGI 列挙は NVIDIA VendorId `0x10DE` を除外）。
- NVIDIA（Windows で `nvidia-smi` 不在）: **DXGI+PDH で列挙・使用率・VRAM を取得、CUDA のみ N/A**（この時 DXGI は NVIDIA を除外しない）。→ FR-3 全列挙を満たす。
- NVIDIA（Linux で不在）: sysfs 列挙のみ、使用率/VRAM/CUDA は N/A。
- 非NVIDIA（Windows AMD/Intel）: DXGI 列挙 + PDH メトリクス。
- **Windows GPU 使用率（PDH `GPU Engine`）**: 物理エンジンのキーは **`(LUID, phys, eng)`**（`engtype` は識別子でない）。同一エンジンをプロセス横断で合算し、**同一 LUID 内はエンジン間で最大値**。**全エンジン合算や 100 clamp はしない**。
- **Windows VRAM 使用量**: PDH `GPU Adapter Memory` の `Dedicated Usage`（システム全体）。**`QueryVideoMemoryInfo().CurrentUsage` は自プロセス使用量なので使わない**。VRAM 総量は DXGI `DedicatedVideoMemory`。
- DXGI は **PCIBusID を返さない**（LUID のみ）。`StableID` 導出: `uuid:`→`pci:`→`luid:`→最終手段 `idx:<i>:<name>`（warn）。**StableID は「同一プロセス実行中で安定」**（LUID は再起動をまたぐ一意性を保証しない。永続IDは非要求）。
- PDH は **query ハンドルと counter ハンドルが別**。`PdhAddEnglishCounter` は counter ハンドルを返し `PdhGetFormattedCounterArray` も counter ハンドルを取る。`PdhCollectQueryData` は query ハンドル。**rate系は2サンプル必要**：probe で query 生成＋counter 追加＋baseline collect、以後 tick ごとに collect→各 counter を format。

### N/A の契約（`reqire.md FR-7 / FR-7.1`）
- 取得不能・欠損は **JSON `null`**（内部は**ポインタ型** `*float64`/`*uint64`/`*string`）。
- **取得手段（ツール/API/コマンド）が環境に無い項目は N/A 許容・欠陥ではない**。ただし**手段が一つでもあれば取れる範囲は取る**。採取失敗でプロセスを落とさない（`Collect()` は常に利用可能な Sample を返す。errは情報用）。

### 公開スキーマ（`basic-spec §6.2`）
- `get_current_resources` の出力は**ネスト**：`{timestamp, cpu:{overall_percent, per_socket_percent}, memory:{used_bytes,total_bytes,used_percent}, gpus:[...]}`。フラットにしない。`memory.used_percent` を含む。`gpus[]` は識別子層（stable_id/pci_bus_id/uuid/os_adapter_id/index）＋name/vendor/utilization_percent/vram_used_bytes/vram_total_bytes/cuda_version。
- 内部 `sample.Sample` をこの凍結スキーマに一致させてある（`detail-design §5.1`）。
- **VRAM の GB はスキーマに持たない**（bytes のみ）。GB 変換は**傾向レポート生成時のみ**。

### 傾向分析・自然言語（`basic-spec §8/§9` / `detail-design §7/§8`）
- 判定定数は `basic-spec §8.3` の**規範値をそのまま写す**（`trend/thresholds.go` に集約、勝手に変えない）。
- **回帰は「時刻に対する」最小二乗**。`computeStats([]point{T,V})`（値だけの `[]float64` にしない。N/A除去で実時間間隔が壊れる）。
- **spike = `最大 > 平均＋2σ` かつ最大が高位、かつ `σ=0` なら false**（`≥` にしない。定数系列 `[10,10,…]` の誤検出防止）。
- **pinned と spike は排他・pinned 優先**（pinned 成立時は spike 判定しない）。
- **steady と fluctuating は独立に立ちうる**（fluctuating は range 条件 `最大−最小≥40pt` でも成立。「CV背反で同時成立しない」は誤り）。**向きと変動も独立**（両方の文を出しうる。順序 向き→変動）。新たな優先順位は導入しない。
- 出力は**英語固定・決定的・外部LLM/ネットワーク非依存**。データ不足時は `data_complete=false` と冒頭に不足文（`needed = ceil(実効window/interval)`）。テンプレートは `detail-design §8.2` の網羅表。

### MCP サーバー（`basic-spec §6` / `detail-design §9`）
- SDK: **`github.com/modelcontextprotocol/go-sdk` v1.6.1** にピン留め（v1.7.0系は pre-release で不採用）。実 API は **`mcp.AddTool[In,Out]`**（typed handler）。
- 両ツールとも入力なし（空 struct）。`structuredContent` は Out。
- **`get_resource_trend` の `content` は英語レポート本文を明示指定**（`&mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: out.Report}}}`）。SDK 任せだと structured の JSON が content に入り `§6.3` と矛盾する。`get_current_resources` は JSON 自動生成に任せてよい（§6.2 と一致）。
- `search_processes` は登録しない（MVP外）。

### 横断（`basic-spec §2.2/§10/§12`）
- **STDIO の標準出力は JSON-RPC 専用。ログ・診断は標準エラーのみ（`slog` → `os.Stderr`）**。
- サンプラー（書き手）とツールハンドラ（読み手）は Store 経由のみ。Snapshot は複製してロック外で分析。
- リングバッファ容量 `ceil(window/interval)+2`、**起動時確定・実行中は非再確保**。履歴は**非永続**。
- **`CGO_ENABLED=0` の単一バイナリ**。Windows の DXGI/PDH/トポロジは `golang.org/x/sys/windows`（cgo なし）。NVIDIA は NVML でなく `nvidia-smi` サブプロセス。

## 5. 推奨実装順（`detail-design §12/§14`）

決定性が中心なので、純ロジックからテスト駆動で固める：

1. `internal/sample`（型。公開スキーマ一致）
2. `internal/trend`（`thresholds.go` 規範値 → `analyze.go` 統計・判定 → `report.go` 英語テンプレート）＋**テーブル駆動テスト**（3系統、複数成立、spike定数系列非検出、データ不足、決定性）
3. `internal/config`（TOML・実効ウィンドウ=max・境界テスト）
4. `internal/store`（リングバッファ・容量・Snapshot 複製）
5. `internal/collector`：`cpu.go`/`mem.go`（gopsutil）→ `gpu.go`＋OS別（`gpu_windows.go`/`gpu_linux.go`/`gpu_darwin.go`, ビルドタグ）。**GPU パーサは golden file テスト**（`testdata/nvidia_smi.xml` 等、実機不要）
6. `internal/mcpserver`（`mcp.AddTool`、outputSchema、content 明示）
7. `cmd/systemresourcemcp/main.go`（配線・プライミング・サンプラー goroutine・signal/EOF 終了）
8. CI: `go vet` / `go test` / 各OS `CGO_ENABLED=0` クロスビルド

## 6. 実装フェーズに委ねてよい詳細（過剰に固定しない）

`detail-design §14` の申し送り。上位契約を変えない限り実装判断でよい：
- DXGI LUID と PDH LUID の内部正規化方法、LUID を文字列/構造体どちらで持つか
- Win32 API ラッパーの戻り値表現、COM vtbl 呼び出しの定型化
- CV 計算のゼロ除算防御など通常の防御実装
- PDH インスタンス名 parser の具体実装
- **macOS の logical→socket API は不確実（best-effort）**。確実に作れなければ `perSocket=[]`（FR-7.1）。Apple Silicon は単一パッケージで `[overall]` で足りる
- `nvidia-smi`/`system_profiler`/PDH の実出力を `testdata/` に採取してゴールデン化
- MCP SDK v1.6.1 実 API への細部整合

## 7. 環境メモ

- Go プロジェクト。`go.mod` 未作成（`go mod init` から）。module path は要確認（例: リポジトリURL準拠）。
- リモート実行環境。GitHub 操作は `gh` 不可、GitHub MCP ツール（`mcp__github__*`）を使う。
- 作業後は draft PR を作成し購読、CI/レビューに追従。

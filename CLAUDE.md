# SystemResourceMCP

システムリソースを AI エージェント向けに公開する **STDIO トランスポートの MCP サーバー**（Go・単一バイナリ・`CGO_ENABLED=0`）。公開ツール: `get_current_resources` / `get_resource_trend`（英語・決定的・LLM非依存）。

## 現在のフェーズ

要求 → 要件定義 → 基本仕様 → 詳細設計 ✅（`main` にマージ済み）→ **実装（進行中/これから）**。

## まず読むもの（順に）

1. `.claude/implementation-handoff.md` … **実装の地図＋落とし穴（再発防止）**。最初に読むこと。
2. `docs/reqire.md` … 要件（FR/NFR、FR-7.1）
3. `docs/basic-spec.md` … **基本仕様 rev.3（規範値の正）**
4. `docs/detail-design.md` … 詳細設計（Go 型・シグネチャ・アルゴリズム）
5. `docs/hearing.md` … 要求記録（背景）

## 作法（重要）

- **規範値（傾向判定定数・NFR-1目標・MCPスキーマ・設定規則）は `basic-spec.md` が正**。実装で勝手に変えず、必要なら `basic-spec.md` を改訂してから追従。
- レビュー対応のために新しい要求・非目標・規範値を勝手に足さない。上位契約を変えない実装詳細は実装判断でよい。
- 作業ブランチ: `claude/requirements-definition-hearing-6tjors`（過去PRはマージ済みなので `main` から作り直す）。`main` へ直接 push しない。draft PR → レビュー → マージ。
- STDIO の標準出力は JSON-RPC 専用。ログは標準エラーのみ。

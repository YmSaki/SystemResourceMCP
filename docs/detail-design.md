# 詳細設計書 — SystemResourceMCP

> このドキュメントは、`basic-spec.md`（rev.3）の**基本仕様**を、コードに落とせる粒度の**詳細設計**へ展開したものである。
> 主に `basic-spec.md §15` が実装フェーズ手前で詰めるとした事項を確定させる。
> 各設計判断には、対応する基本仕様の節（`basic-spec.md §x`）を添える。
>
> - 日付: 2026-08-23（rev.3 の基本仕様から再導出）
> - フェーズ: **詳細設計**（前フェーズ＝基本仕様／次フェーズ＝実装）
> - フェーズ体系: 要求 → 要件定義 → 基本仕様 → **詳細設計** → 実装
> - 上位文書: `docs/basic-spec.md`（さらに上位に `reqire.md` / `hearing.md`）

### 本書と上位仕様の関係（改訂の原則）

本書は詳細設計であり、**基本仕様で確定した規範値を変更しない**（`basic-spec.md` 冒頭原則）。
2回のレビューで基本仕様側に矛盾・不成立前提が見つかったため、詳細設計で握りつぶさず**基本仕様を差し戻して再確定**（`basic-spec.md` rev.2／rev.3 差し戻し改訂履歴）し、本書はその改訂後の基本仕様から再導出している。

初版（rev.1）レビューで解消した不備:

| 出所 | 不備 | 解消 |
| --- | --- | --- |
| 基本仕様差し戻し | 設定 `window < interval` の矛盾 | 実効ウィンドウ=`max`（§4） |
| 基本仕様差し戻し | `PhysicalID` 集約が Win/mac で不成立 | OS別トポロジAPI＋フォールバック（§5.3） |
| 基本仕様差し戻し | DXGI が PCIBusID を返さず層間突合が破綻 | ソース単一化（§5.4〜5.7） |
| 詳細設計 | `ResourceSnapshot` が凍結スキーマと別物 | 公開型を `basic-spec §6.2` に一致（§5.1, §9.2） |
| 詳細設計 | `QueryVideoMemoryInfo` は per-process | VRAM使用は `GPU Adapter Memory` カウンタ（§5.5） |
| 詳細設計 | 使用率を全engine合算→100clamp | 最ビジーエンジン値（§5.5） |
| 詳細設計 | gopsutil 初回CPUは error でサンプルを落とす | 初回は CPU=N/A の部分サンプルを保存（§5.3, §10） |
| 詳細設計 | MCP SDK 版を実装へ先送り | v1.6.1 に確定（§9.1） |

rev.2 レビューで解消した不備（本 rev.3）:

| 出所 | 不備 | 解消 |
| --- | --- | --- |
| 新契約（要件） | 取得手段が無い項目の扱いが未契約 | FR-7.1 を要件へ追加し、本書で反映（§5.4, §5.5） |
| 基本仕様差し戻し | `nvidia-smi` 不在時 Windows NVIDIA が列挙すら消える（FR-3衝突） | 条件付きソース選択（在→nvidia-smi／不在→DXGI+PDH・CUDAのみN/A）（§5.4, §5.5） |
| 基本仕様差し戻し | `CPUPerSocket` 取得不能を `[overall]` とし2ソケット機を誤表示 | 取得不能は**空配列**（§5.3） |
| 詳細設計 | PDH を1サンプルで読む（rate系は2サンプル要） | PDH クエリを状態保持し baseline＋周期collect（§5.5） |
| 詳細設計 | engine 集約キーが `engtype`（物理エンジン非一意） | キーを `(LUID, phys, eng)` に（§5.5） |
| 詳細設計 | `mcp.AddTool` が擬似APIで、trend の `content` が JSON になる | 実API `mcp.AddTool` ＋ trend は `content` に英語本文を明示（§9.2） |
| 詳細設計 | 回帰の関数が timestamp を受けない | `computeStats([]point{T,V})` で時刻に対する回帰（§7.2） |
| 詳細設計（要確認） | macOS の logical→socket API が不確実 | best-effort 明記、不能時 `perSocket=[]`（§5.3） |

---

## 1. 本書の範囲と方針

`basic-spec.md §15` の5点に、上記の差し戻し反映を加えて確定する。言語は Go、`CGO_ENABLED=0` の単一バイナリを維持する（`basic-spec.md §12.2 NFR-4`）。
以下のコードはシグネチャ・型・アルゴリズムの確定であり、公開インターフェース（MCPツール名・スキーマ、設定キー、規範値）は基本仕様のとおりで変更しない。

---

## 2. モジュールと依存

`basic-spec.md §3` のパッケージ構成を踏襲する。外部依存は次に限定する。

| 依存 | 用途 | バージョン方針 |
| --- | --- | --- |
| `github.com/shirou/gopsutil/v4` | CPU・メモリ採取 | 最新 v4 系。cgo 不要 |
| `github.com/BurntSushi/toml` | 設定解釈（§4） | 最新安定。cgo 不要 |
| `github.com/modelcontextprotocol/go-sdk` | MCP サーバー（§9） | **v1.6.1 にピン留め**（§9.1） |
| 標準 `log/slog` | stderr への構造化ログ | — |
| `golang.org/x/sys/windows` | Windows の PDH / DXGI / トポロジAPI（§5.3, §5.5） | `//go:build windows` のみ |

`nvidia-smi` / `system_profiler` / `rocm-smi` は外部コマンド（サブプロセス）であり依存ライブラリではない。

---

## 3. エラーハンドリングの共通規約

`basic-spec.md §7.5, §11` の劣化継続をコード規約にする。

- 採取処理は `error` を返すが、**サンプラーはエラーでプロセスを止めない**。ログを stderr に出し、当該フィールドを欠損（nil）にして続行する。
- 「取得不能」と「値が0」を区別するため、**欠損しうる数値はポインタ型**（`*float64`/`*uint64`/`*string`）で表し、`nil`＝N/A＝JSON `null`（`basic-spec.md §4.3`）。
- `Collect()` は**常に利用可能な `Sample` を返す**（部分失敗でも timestamp と取れた項目を含む）。error は情報用で、呼び出し側は Add を止めない（§10、初版の finding #8 対策）。

---

## 4. 設定（`config` パッケージ）

`basic-spec.md §5`（rev.2）を実装に落とす。

### 4.1 型と既定値

```go
package config

import "time"

const (
    DefaultSamplingInterval = 10 * time.Second
    DefaultWindow           = 60 * time.Second
)

// Config は検証済みの実効設定。
type Config struct {
    SamplingInterval time.Duration
    Window           time.Duration // 実効ウィンドウ（= max(configured window, interval)）
}

type fileConfig struct {
    SamplingIntervalSeconds *int `toml:"sampling_interval_seconds"`
    WindowSeconds           *int `toml:"window_seconds"`
}
```

### 4.2 探索

```go
func Load(cliPath string, logger *slog.Logger) (Config, error)
// resolvePath: cliPath > 環境変数 SYSTEMRESOURCEMCP_CONFIG >
//   os.UserConfigDir()/systemresourcemcp/config.toml。無ければ既定 Config。
func resolvePath(cliPath string) (path string, found bool)
```

### 4.3 検証（`basic-spec.md §5.3` rev.2：2段階）

```go
func validate(fc fileConfig, logger *slog.Logger) Config {
    interval := DefaultSamplingInterval
    window := DefaultWindow

    // 段階1: 各項目を独立に検証（不正は該当項目のみ既定へ）
    if fc.SamplingIntervalSeconds != nil {
        if *fc.SamplingIntervalSeconds >= 1 {
            interval = time.Duration(*fc.SamplingIntervalSeconds) * time.Second
        } else {
            logger.Warn("invalid sampling_interval_seconds; using default",
                "value", *fc.SamplingIntervalSeconds, "default_seconds", 10)
        }
    }
    if fc.WindowSeconds != nil {
        if *fc.WindowSeconds >= 1 {
            window = time.Duration(*fc.WindowSeconds) * time.Second
        } else {
            logger.Warn("invalid window_seconds; using default",
                "value", *fc.WindowSeconds, "default_seconds", 60)
        }
    }

    // 段階2: 実効ウィンドウ = max(window, interval)。最低1サンプルを保証。
    if window < interval {
        logger.Warn("window raised to match sampling interval to guarantee >=1 sample",
            "configured_window", window.String(), "effective_window", interval.String())
        window = interval
    }
    return Config{SamplingInterval: interval, Window: window}
}
```

これにより `interval=120, window 未指定/不正` でも実効ウィンドウ 120 となり矛盾しない（`basic-spec.md §5.3` rev.2）。

---

## 5. データモデルと採取（`sample` / `collector`）

### 5.1 データ型（`sample` パッケージ）＝公開スキーマに一致（finding #4 対策）

**内部の `Sample` を、そのまま `basic-spec.md §6.2` の凍結公開スキーマに一致させる**（初版はフラットで別物だった）。これにより公開出力（§9.2）はこの型をそのまま使え、スキーマ違反が起きない。

```go
package sample

import "time"

type CPU struct {
    OverallPercent   *float64  `json:"overall_percent"`
    PerSocketPercent []float64 `json:"per_socket_percent"` // 空可、要素は非nil
}

type Memory struct {
    UsedBytes   *uint64  `json:"used_bytes"`
    TotalBytes  *uint64  `json:"total_bytes"`
    UsedPercent *float64 `json:"used_percent"` // used/total*100（採取時に算出）
}

type Sample struct {
    Timestamp time.Time   `json:"timestamp"`
    CPU       CPU         `json:"cpu"`
    Memory    Memory      `json:"memory"`
    GPUs      []GPUSample `json:"gpus"` // 非搭載時は空配列
}

type Vendor string
const (
    VendorNVIDIA  Vendor = "NVIDIA"
    VendorAMD     Vendor = "AMD"
    VendorIntel   Vendor = "Intel"
    VendorApple   Vendor = "Apple"
    VendorUnknown Vendor = "Unknown"
)

type GPUSample struct {
    StableID     string  `json:"stable_id"` // 同一プロセス実行中で安定（再起動をまたぐ永続性は非保証。§5.7）
    PCIBusID     *string `json:"pci_bus_id"`
    UUID         *string `json:"uuid"`
    OSAdapterID  *string `json:"os_adapter_id"`
    Index        int     `json:"index"`
    Name         *string `json:"name"`
    Vendor       Vendor  `json:"vendor"`
    UtilPercent    *float64 `json:"utilization_percent"`
    VRAMUsedBytes  *uint64  `json:"vram_used_bytes"`
    VRAMTotalBytes *uint64  `json:"vram_total_bytes"`
    CUDAVersion    *string  `json:"cuda_version"`
}
```

このトップレベル（`cpu`/`memory`/`gpus` のネスト、`memory.used_percent` の存在）が `basic-spec.md §6.2` の例と厳密に一致する。`GPUSample` の各キーも同 §6.2 の `gpus[]` と一致する。

### 5.2 Collector インターフェース

```go
package collector

type Collector interface {
    Collect() (sample.Sample, error) // Sample は常に利用可能。error は部分失敗の記録用（§3）
}
func New(logger *slog.Logger) Collector // 内部で起動時の能力プローブ（§5.8）

type collector struct {
    logger *slog.Logger
    cpu    cpuCollector
    mem    memCollector
    gpu    gpuCollector // OS別・ビルドタグ（§5.4〜5.6）
}
```

`Collect()` は CPU→メモリ→GPU の順に測り、個々の失敗は当該フィールドを nil にして継続。GPU 全滅で `GPUs` は空配列（`basic-spec.md §7.5`）。errors.Join で部分失敗を束ねて返すが、`Sample` は常に返す。

### 5.3 CPU・メモリ（`cpu.go` / `mem.go`）— rev.3

`basic-spec.md §7.2` rev.3。使用率は gopsutil、ソケットトポロジは OS 別 API に分離する。

```go
// collect は全体使用率と、ソケット別使用率を返す。
// 初回呼び出しは gopsutil が基準時刻を持たず error を返しうる（cpu.Percent の仕様）。
// その場合 overall=nil を返す（error も返す）が、呼び出し側はサンプルを捨てない（§10）。
func (c *cpuCollector) collect() (overall *float64, perSocket []float64, err error)
```

手順:
1. `overall`: `cpu.Percent(0, false)` の `[0]`。error（初回など）なら `overall=nil`。
2. `perCore`: `cpu.Percent(0, true)`（論理コア別）。
3. `perSocket`: OS別トポロジ `logicalToSocket()`（下記）で論理コアをソケットへ束ね、ソケット番号昇順に平均。**トポロジが取得できない場合は `perSocket = []`（空配列＝ソケット別内訳は取得不能）にフォールバック**（`basic-spec.md §7.2` rev.3, FR-7/FR-7.1）。rev.2 の `[overall]` は 2ソケット機を1ソケットに誤表示するため誤り。トポロジが「1ソケット」と判明した単一ソケット機のみ `perSocket = [overall]`（1要素）になる（これは取得不能ではなく正しい単一ソケット値）。

```go
// logicalToSocket は論理コア index → ソケット番号 の対応を返す。ビルドタグで実体が変わる。
//   windows: GetLogicalProcessorInformationEx(RelationProcessorPackage)（x/sys/windows）
//   linux:   /proc/cpuinfo の physical id
//   darwin:  （下記の注意。確実な logical→package API が乏しく best-effort）
// 取得不能なら (nil, false) → 呼び出し側で perSocket=[]（FR-7.1）。
func logicalToSocket() (mapping []int, ok bool)
```

**macOS の注意（要実装時確認）**: `sysctl` の `hw.packages` / `hw.physicalcpu` 等は**個数・能力情報**であり、論理CPU index → package の対応を直接は返さない。したがって macOS で `logicalToSocket` を確実に構築できるかは実装前に検証を要する。構築できない場合は `(nil, false)` を返し `perSocket=[]` にフォールバックする（FR-7.1 により、取得手段が無ければ N/A で許容）。実運用上、Apple Silicon は単一パッケージであり `perSocket=[overall]`（1要素）で足り、マルチソケットは旧 Intel Mac Pro に限られるため、当面 macOS は「単一パッケージ＝[overall]、判別不能＝[]」で扱う。

```go
// メモリ: mem.VirtualMemory() の Used/Total/UsedPercent を格納。used_percent もここで確定。
func (c memCollector) collect() (sample.Memory, error)
```

初回 CPU が error でも、`Collect()` は `CPU.OverallPercent=nil`（＝N/A）の部分サンプルを返し、`main` はそれを Store に入れる（起動直後の即時応答を空にしない。`basic-spec.md §6.2, FR-9`。初版 finding #8 対策）。2サンプル目以降は基準が揃い値が入る。

### 5.4 GPU：条件付きソース選択と OS 振り分け（rev.3）

`basic-spec.md §7.3` rev.3 の**条件付きソース選択**を実装する。1つのGPUの全項目は常に単一ソースから揃え、異種ソース間の突合はしない。ソースは起動時プローブ（§5.8）の `nvidia-smi` 有無で決める（`basic-spec.md §7.3` rev.3, FR-7.1）。

```go
type gpuCollector interface {
    probe(logger *slog.Logger)            // nvidia-smi 有無等を判定（§5.8）
    collect() ([]sample.GPUSample, error) // 条件付きソースで完結した GPUSample を結合して返す
}
```

| ファイル | ビルドタグ | `nvidia-smi` 在 | `nvidia-smi` 不在 | 非NVIDIA |
| --- | --- | --- | --- | --- |
| `gpu_windows.go` | `windows` | NVIDIA=`nvidia-smi` 完結、**DXGIはNVIDIA除外**（§5.4.1） | NVIDIA=**DXGI＋PDH**（列挙・使用率・VRAM取得、**CUDA=nil**）、DXGIはNVIDIA除外しない | DXGI 列挙＋PDH（§5.5） |
| `gpu_linux.go` | `linux` | NVIDIA=`nvidia-smi` 完結 | NVIDIA=sysfs列挙のみ（使用率/VRAM/CUDA=nil、FR-7.1） | sysfs 列挙＋AMD=`rocm-smi`/sysfs |
| `gpu_darwin.go` | `darwin` | — | — | `system_profiler` 列挙のみ（§5.6） |

`collect()` は次の擬似ロジックで構築する（`basic-spec.md §7.3` rev.3）。

```
nvSmiAvail := probe 結果
if nvSmiAvail:
    nvGPUs  := parseNvidiaSmiXML(...)         // NVIDIA、全項目
    others  := enumerateDXGINonNVIDIA()+PDH   // 非NVIDIA のみ（VendorId 0x10DE 除外）
else (Windows):
    others  := enumerateDXGIAll()+PDH         // NVIDIA も含めて DXGI+PDH（CUDA=nil）
    nvGPUs  := []                             // nvidia-smi 由来は無し
result := append(nvGPUs, others...)           // どのGPUも単一ソース由来。突合なし
```

結合後に表示用 `Index` を 0 から連番で振る（§8.1 の表示にのみ使用。同定は `StableID`）。
`nvidia-smi` 不在時に NVIDIA を DXGI+PDH で扱っても、`nvidia-smi` 由来が空なので二重計上は起きず、識別は DXGI の LUID で一貫する（§5.7）。

#### 5.4.1 `nvidia-smi` XML → `GPUSample`（全項目・全OS共通）

`nvidia-smi -q -x` を解釈し、NVIDIA GPU の**列挙・使用率・VRAM・CUDA・識別子をすべて**得る（`basic-spec.md §7.4` rev.2）。

| XML パス（`nvidia_smi_log/gpu` 配下、CUDA はルート直下） | 例 | 格納先 | 変換 |
| --- | --- | --- | --- |
| `@id` または `pci/pci_bus_id` | `00000000:01:00.0` | `PCIBusID` | 小文字・`0000:01:00.0` 正規化 |
| `uuid` | `GPU-4e1f…` | `UUID` | そのまま |
| `product_name` | `NVIDIA GeForce RTX 5090` | `Name` | そのまま |
| `utilization/gpu_util` | `91 %` | `UtilPercent` | ` %` 除去→float |
| `fb_memory_usage/used` | `18253 MiB` | `VRAMUsedBytes` | MiB→bytes |
| `fb_memory_usage/total` | `32607 MiB` | `VRAMTotalBytes` | MiB→bytes |
| `nvidia_smi_log/cuda_version` | `12.4` | `CUDAVersion` | 全GPU共通 |
| — | — | `Vendor` | 固定 `NVIDIA` |
| — | — | `StableID` | `uuid:` + UUID（無ければ `pci:` + PCIBusID）（§5.7） |

```go
func parseNvidiaSmiXML(xmlBytes []byte) ([]sample.GPUSample, error) // golden XML でテスト（§12）
```

`nvidia-smi` 不在（プローブ）なら **nvidia-smi 由来の** NVIDIA 群は空になり、その場合 NVIDIA は §5.5 の DXGI＋PDH 側で列挙・使用率・VRAM を取得する（CUDA=nil。§5.4 の条件付きソース選択）。「NVIDIA が消える」わけではない。

### 5.5 Windows DXGI 列挙 ＋ PDH メトリクス層 — rev.3

`basic-spec.md §7.4` rev.3。`golang.org/x/sys/windows` から cgo なしで呼ぶ。この層は非NVIDIA を常に、加えて **`nvidia-smi` 不在時は NVIDIA も**扱う（§5.4）。

**列挙・VRAM総量・LUID・ベンダー（DXGI）**
1. `CreateDXGIFactory1` → `IDXGIFactory1`（COM vtbl を `syscall.SyscallN` で呼ぶ）。
2. `EnumAdapters1` を 0 から `DXGI_ERROR_NOT_FOUND` まで。各 `GetDesc1` で `DXGI_ADAPTER_DESC1` を取得。
3. `Description`→`Name`、`DedicatedVideoMemory`→`VRAMTotalBytes`、`AdapterLuid`→`OSAdapterID`（`"%d:%d"` 表記）、`VendorId`→`Vendor`（0x10DE=NVIDIA, 0x1002=AMD, 0x8086=Intel, 0x106B=Apple, その他=Unknown）。
4. **NVIDIA（`VendorId == 0x10DE`）の除外は条件付き**（rev.3）: `nvidia-smi` が在れば除外（`enumerateDXGINonNVIDIA`）、無ければ除外しない（`enumerateDXGIAll`）。§5.4 のソース選択に従う。
5. ソフトウェアアダプタ（`DXGI_ADAPTER_FLAG_SOFTWARE`）は除外。
6. `PCIBusID` は DXGI から**得られない**ため nil のまま（`OSAdapterID`=LUID を識別に使う）。NVIDIA を DXGI 経路で扱う場合も CUDA は nil（取得手段が無い。FR-7.1）。

**PDH のサンプルライフサイクル（finding: PDH は2サンプル必要）**
PDH の rate/timer 系カウンタは、formatted value を得るのに**前回値と現在値の2サンプル**が必要である（初回 `PdhCollectQueryData` は基準値のみ）。そこで **PDH クエリをコレクタの状態として保持**する。

PDH は **query ハンドルと counter ハンドルが別**である。`PdhAddEnglishCounter` は **counter ハンドル（`PDH_HCOUNTER`）を返し**、`PdhGetFormattedCounterArray` が要求するのも **counter ハンドル**（query ハンドルではない）。`PdhCollectQueryData` は query ハンドルを取る。したがって両方を状態として保持する。

```
probe/init 時:
    q := PdhOpenQuery()                                             // query ハンドル
    hEngine := PdhAddEnglishCounter(q, `\GPU Engine(*)\Utilization Percentage`)  // counter ハンドル
    hMemory := PdhAddEnglishCounter(q, `\GPU Adapter Memory(*)\Dedicated Usage`) // counter ハンドル
    PdhCollectQueryData(q)          // baseline（1回目）。値はまだ読まない
各 sampling tick（周期ごと）:
    PdhCollectQueryData(q)                                   // query 単位で収集
    engArr := PdhGetFormattedCounterArray(hEngine, PDH_FMT_DOUBLE)  // counter 単位で取得
    memArr := PdhGetFormattedCounterArray(hMemory, PDH_FMT_DOUBLE)
```

`windowsGPU` は query ハンドルと各 counter ハンドルを保持し、`collect()` ごとに collect→各 counter を format する。初回 tick は差がまだ整わない項目がありうるため N/A になりうる（FR-7/FR-9 と整合）。

**使用率（PDH `GPU Engine`）— 最ビジーエンジン方式（finding #6 ＋ engine key 修正）**
1. 上記クエリの `GPU Engine` インスタンス配列を得る。
2. インスタンス名 `pid_<pid>_luid_0x0_0x<LUID>_phys_<p>_eng_<n>_engtype_<type>` から `LUID`・`phys`・`eng`（参考に `engtype`）を抽出。
3. 集約: **物理エンジンの識別キーは `(LUID, phys, eng)`**。`engtype` は分類ラベルであって識別子ではなく、同一 `engtype` に別 `eng` が複数ありうるため単独では物理エンジンを一意化できない。同一 `(LUID, phys, eng)` をプロセス横断で合算し、**同一 `LUID` 内ではエンジン間で最大値**を採る。すなわち `adapterUtil = max_{(phys,eng)}( sum_pid(util) )`。全エンジン合算や 100 clamp はしない（`basic-spec.md §7.4`）。
4. `LUID`→`UtilPercent` を、DXGI 列挙の `OSAdapterID`(LUID) と突き合わせる（同一ソース内の突合。§5.7）。

**VRAM 使用量（PDH `GPU Adapter Memory`）— システム全体（finding #5 対策）**
- `\GPU Adapter Memory(luid_...)\Dedicated Usage` を LUID ごとに読み、`VRAMUsedBytes` に入れる（システム全体の専用メモリ使用量）。
- **`IDXGIAdapter3::QueryVideoMemoryInfo().CurrentUsage` は呼び出しプロセス自身の使用量**のため VRAM 使用量には使わない（`basic-spec.md §7.4`）。

```go
type windowsGPU struct {
    pdhQuery     pdhQueryHandle    // PdhOpenQuery
    engineCtr    pdhCounterHandle  // \GPU Engine(*)\Utilization Percentage
    memoryCtr    pdhCounterHandle  // \GPU Adapter Memory(*)\Dedicated Usage
    nvSmiAvail   bool
}
func (g *windowsGPU) probe(logger *slog.Logger)          // DXGI factory・PDH query生成＋counter追加＋baseline collect・nvidia-smi 有無
func enumerateDXGINonNVIDIA() ([]sample.GPUSample, error) // NVIDIA 除外（nvidia-smi 在時）
func enumerateDXGIAll() ([]sample.GPUSample, error)       // NVIDIA 含む（nvidia-smi 不在時、CUDA=nil）
func (g *windowsGPU) queryUtilByLUID() (map[string]float64, error) // (LUID,phys,eng) 集約→max-engine
func (g *windowsGPU) queryVRAMUsedByLUID() (map[string]uint64, error)
```

いずれも Win32 API で cgo なし（`basic-spec.md §12.2 NFR-4`）。使えない環境（カウンタ無効等）はプローブで検出し当該項目を N/A（`basic-spec.md §11`）。

### 5.6 macOS 列挙（`system_profiler` JSON）

`basic-spec.md §7.4`。`system_profiler SPDisplaysDataType -json`。

| JSON パス | 格納先 | 備考 |
| --- | --- | --- |
| `SPDisplaysDataType[].sppci_model` | `Name` | 機種名 |
| （名前から推定） | `Vendor` | 既定 Unknown |
| — | `UtilPercent` / `VRAMUsedBytes` / `VRAMTotalBytes` / `CUDAVersion` | 常に nil（`basic-spec.md §7.4`, A-5） |

`StableID` は PCI/UUID/LUID が無いため index+name フォールバックになりうる（§5.7）。macOS は非NVIDIAソースのみ。

### 5.7 `StableID` 導出（ソース内一意）

`basic-spec.md §4.4` rev.3。`StableID` は**同一ソース内で**GPUを一意に指す。

```go
func deriveStableID(g sample.GPUSample) string {
    switch {
    case g.UUID != nil && *g.UUID != "":               return "uuid:" + *g.UUID   // NVIDIA
    case g.PCIBusID != nil && *g.PCIBusID != "":         return "pci:" + *g.PCIBusID
    case g.OSAdapterID != nil && *g.OSAdapterID != "":   return "luid:" + *g.OSAdapterID // Windows 非NVIDIA
    default:
        return fmt.Sprintf("idx:%d:%s", g.Index, deref(g.Name)) // 最終手段（warn）
    }
}
```

**突合はソースをまたがない**（`basic-spec.md §7.3` rev.3）。
- NVIDIA（`nvidia-smi` 在）: `nvidia-smi` の1回の出力で全項目が揃うので突合自体が不要（`StableID` は uuid/pci）。
- NVIDIA（`nvidia-smi` 不在, Windows）: DXGI+PDH 側で扱われ、非NVIDIA と同じく **LUID で突合**（`StableID` は luid、CUDA=nil）。`nvidia-smi` 由来が無いので DXGI 側と競合しない。
- 非NVIDIA（Windows）: DXGI 列挙（LUID付き）と PDH メトリクス（LUID キー）を **LUID で突合**。同一ソースなのでキーが一致する。
これにより、rev.1 の「DXGI(LUID) と nvidia-smi(UUID) が一致せず index+name に落ちて同型GPU誤結合」が構造的に起こらない。1つのGPUは常にどちらか一方のソースからのみ現れる。

### 5.8 能力プローブ

`basic-spec.md §7.1`。起動時に一度: `nvidia-smi`/`rocm-smi` の有無（`exec.LookPath`）、Windows は DXGI factory 生成可否・PDH クエリ可否、GPU 列挙 0 枚判定。以後は可能な手段のみ呼ぶ。

---

## 6. リングバッファ（`store` パッケージ）

`basic-spec.md §10.2, §10.3` と §15「サイズ余裕・再確保」。

```go
package store

type Store struct {
    mu   sync.RWMutex
    buf  []sample.Sample
    head, size, cap int
}

// capacityFor は 実効window/interval に +2 の余裕を足す（境界のずれ対策）。
func capacityFor(window, interval time.Duration) int {
    n := int(math.Ceil(float64(window) / float64(interval)))
    if n < 1 { n = 1 }
    return n + 2
}

func New(window, interval time.Duration) *Store          // cap = capacityFor(...)
func (s *Store) Add(smp sample.Sample)                    // 満杯なら最古を上書き
func (s *Store) Latest() sample.Sample                    // 直近1件（get_current_resources 用）
func (s *Store) Snapshot(window time.Duration) []sample.Sample // 直近window分を複製（時刻昇順）
```

`Snapshot` は複製の間だけロックし、分析・文生成はロック外（`basic-spec.md §10.3`）。
**実行中の再確保はしない**。設定は起動時確定（`basic-spec.md §5, A-3`）で、容量は `New` で一度だけ決める。オンライン変更は本スコープ外（YAGNI）。

---

## 7. 傾向分析（`trend/analyze.go` ＋ `thresholds.go`）

`basic-spec.md §8` の規範値を**そのまま写す**（変更は基本仕様改訂を伴う）。

### 7.1 判定定数（`thresholds.go`）

```go
package trend
// basic-spec §8.3 の規範値の写し。ここで値を変えない。
const (
    steadyCV      = 0.10
    fluctCV       = 0.30
    fluctRangePt  = 40.0
    trendNetPt    = 15.0
    pinnedFrac    = 0.90
    pinnedLevel   = 95.0
    spikeSigma    = 2.0
    highLevel     = 90.0
    trendNetRatio = 0.25 // 量的系列（VRAM等）
)
```

### 7.2 統計量と判定

```go
type seriesKind int
const ( kindPercent seriesKind = iota; kindQuantity )

type point struct{ T time.Time; V float64 } // 時刻付き測定値（basic-spec §8.2「時刻に対する回帰」）
type stats struct{ mean, std, min, max, first, last, slope float64; n int }

// computeStats は非nil値のみを時刻付きで受け、統計量を出す（basic-spec §8.1）。
// slope は「時刻に対する」最小二乗回帰の傾き。n<2 は slope=0。
// 値だけの []float64 にすると、null を除いた際に実時間間隔が失われ回帰が歪む
// （例: t=0,10,20 で t=10 が N/A のとき [v0,v2] を等間隔とみなすのは誤り）。
// そのため時刻を保持したまま回帰する。
func computeStats(points []point) stats

// 各系列は非nil値のみを (Timestamp, value) の point として抽出する。
func series(samples []sample.Sample, pick func(sample.Sample) *float64) []point

type levelVerdict struct{ steady bool; mean float64 }              // steady と fluctuating は CV で背反
// 向き(rising/falling/flat) と 変動(fluctuating) は独立に立つ（basic-spec §8.3.1）
type dirVerdict struct{ dir dirKind; fluctuating bool; start, end, min, max float64 }
// pinned と spike は排他・pinned 優先（basic-spec §8.3.1）。peakKind ∈ {none, pinned, spike}
type peakVerdict struct{ kind peakKind; max float64 }
func judgeLevel(s stats) levelVerdict
func judgeDirection(s stats, k seriesKind) dirVerdict            // dir を1つに定め、fluctuating を独立に立てる
func judgePeak(points []point, s stats, k seriesKind) peakVerdict // pinned 成立時は spike を判定しない
```

判定式は `basic-spec.md §8.3` の表そのまま。使用率系は pt、量的系列は平均比。決定性は入力統計量のみに依存（`basic-spec.md §8.4`）。回帰は各 `point.T`（実時刻）に対して行う。
**複数成立の扱い（`basic-spec.md §8.3.1`）**: `judgeDirection` は向き（rising/falling/flat のいずれか1つ）と `fluctuating`（独立の bool）を両方返し、両立時は両方の文を出す（順序は 向き → 変動）。`judgePeak` は `pinned` を先に判定し、成立すれば `spike` は評価しない（全張り付き `[99,...]` が σ=0 で spike も満たす二重成立を防ぐ）。`steady` と `fluctuating` は変動係数の条件が背反なので同時には立たない。

### 7.3 分析

```go
type MetricTrend struct {
    Metric  string; Kind seriesKind
    Level   levelVerdict; Dir dirVerdict; Peak peakVerdict
    HasData bool
}
// CPU / Memory / GPU[i].Util / GPU[i].VRAM を対象（basic-spec §8.1）。
// GPU 系列は StableID で同定（同型複数枚を取り違えない）。
func Analyze(samples []sample.Sample) []MetricTrend
```

---

## 8. 自然言語生成（`trend/report.go` ＋ `templates.go`）

`basic-spec.md §9`。全判定網羅と `{metric}` 規則。

### 8.1 `{metric}` 文字列規則

| 系列 | `{metric}` |
| --- | --- |
| CPU 全体 | `CPU` |
| メモリ使用率 | `Memory usage` |
| GPU i 使用率 | `GPU {index} ({name})`（name nil なら `GPU {index}`） |
| GPU i VRAM | `GPU {index} VRAM` |

`{index}` は表示用（`basic-spec.md §4.4, §9.2`）。内部同定は `StableID`。

### 8.2 テンプレート全網羅（英語固定）

| 判定 | 系統 | テンプレート |
| --- | --- | --- |
| steady | 水準 | `{metric} has stayed around {mean}% steadily.` |
| level(一般) | 水準 | `{metric} is around {mean}%.` |
| rising | 向き | `{metric} is rising, from {start}% to {end}%.` |
| falling | 向き | `{metric} is falling, from {start}% to {end}%.` |
| flat | 向き | （出さない） |
| fluctuating | 変動 | `{metric} is fluctuating widely (between {min}% and {max}%).` |
| pinned | 極値 | `{metric} is pinned at {max}%.` |
| spike(%) | 極値 | `{metric} spiked up to {max}%.` |
| VRAM level | 水準 | `{metric} is around {mean_gb} GB.` |
| VRAM rising | 向き | `{metric} is rising, from {start_gb} GB to {end_gb} GB.` |
| VRAM falling | 向き | `{metric} is falling, from {start_gb} GB to {end_gb} GB.` |
| VRAM fluctuating | 変動 | `{metric} is fluctuating widely (between {min_gb} GB and {max_gb} GB).` |
| VRAM spike | 極値 | `{metric} spiked up to {max_gb} GB.` |

`%` は整数、GB は小数1桁。

### 8.3 組み立てと出力

```go
func BuildReport(trends []MetricTrend, window time.Duration, sampleCount, needed int) TrendReport
type TrendReport struct {
    Report        string `json:"report"`
    WindowSeconds int    `json:"window_seconds"`
    SampleCount   int    `json:"sample_count"`
    DataComplete  bool   `json:"data_complete"`
}
```

順序は水準→向き/変動→極値（`basic-spec.md §9.1`）。先頭 `Over the last {window}s:`。
`needed` は**実効ウィンドウ**基準 `ceil(window/interval)`（§4）。`sampleCount < needed` で `DataComplete=false`、冒頭に不足文（`basic-spec.md §9.3`）。

---

## 9. MCP サーバー（`mcpserver` パッケージ）

### 9.1 SDK 選定とバージョン確定（finding #9 対策）

公式 **`github.com/modelcontextprotocol/go-sdk`** を採用し、**バージョンは `v1.6.1`（安定版）にピン留めする**。
`v1.7.0` 系は pre-release のため採用しない。`basic-spec.md §15` が「採用版とバージョンまで確定」を求めていたため、実装フェーズへ先送りせず本書で確定する。
選定理由は `basic-spec.md §6.1`（STDIO・`outputSchema`・`structuredContent`、単一バイナリ）に合致すること。

### 9.2 公開型（凍結スキーマに一致）と登録

公開出力は §5.1 の型をそのまま使う（`basic-spec.md §6.2` に一致するため別DTO不要。finding #4 対策）。

```go
type ResourceSnapshot = sample.Sample      // §5.1 で §6.2 スキーマに一致済み
type TrendReport      = trend.TrendReport

func NewServer(st *store.Store, cfg config.Config, logger *slog.Logger) *Server
func (s *Server) Run(ctx context.Context) error // STDIO・ブロッキング。ctx キャンセルで終了
```

登録は **v1.6.1 の実 API `mcp.AddTool[In, Out]`**（typed handler）を使う。ハンドラは `(*mcp.CallToolResult, Out, error)` を返し、`Out` が `structuredContent` になる。

**重要（finding: content の自動生成と仕様の衝突）**: v1.6.1 の `mcp.AddTool` は、ハンドラが `CallToolResult.Content` を指定しないと、`structuredContent` を**JSON 化して `TextContent` に自動挿入**する。
- `get_current_resources`: `content` は「スナップショットの JSON 直列化」でよい（`basic-spec.md §6.2` と一致）ので、自動生成に任せる（`CallToolResult` は nil でよい）。
- `get_resource_trend`: `content` は**英語レポート本文そのもの**でなければならない（`basic-spec.md §6.3`）。自動生成に任せると JSON（`{"report":...}`）が `content` になってしまうため、**明示的に `TextContent{Text: out.Report}` を返す**。

```go
// get_current_resources: content は JSON 自動生成に任せる（§6.2 と一致）
mcp.AddTool(server,
    &mcp.Tool{Name: "get_current_resources", Description: "..."},
    func(ctx context.Context, req *mcp.CallToolRequest, _ struct{}) (*mcp.CallToolResult, ResourceSnapshot, error) {
        return nil, st.Latest(), nil // Content 未指定 → structured を JSON 化して TextContent 自動挿入
    })

// get_resource_trend: content は英語本文を明示（§6.3。JSONにさせない）
mcp.AddTool(server,
    &mcp.Tool{Name: "get_resource_trend", Description: "..."},
    func(ctx context.Context, req *mcp.CallToolRequest, _ struct{}) (*mcp.CallToolResult, TrendReport, error) {
        samples := st.Snapshot(cfg.Window)
        needed := int(math.Ceil(float64(cfg.Window) / float64(cfg.SamplingInterval)))
        out := trend.BuildReport(trend.Analyze(samples), cfg.Window, len(samples), needed)
        res := &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: out.Report}}}
        return res, out, nil // structuredContent=out、content=英語本文
    })
```

入力引数なしは空 struct（`struct{}`）で表す。`outputSchema` は `AddTool` が `Out` 型（`ResourceSnapshot` / `TrendReport`）から生成する。`search_processes` は登録しない（MVP外、`basic-spec.md §6.4`）。
（`mcp.AddTool` / `mcp.CallToolResult` / `mcp.TextContent` は v1.6.1 の実型名。細部シグネチャは採用タグで最終確認する。）

---

## 10. 並行処理とライフサイクル（`cmd/systemresourcemcp/main.go`）

`basic-spec.md §10`。初回CPU error でもサンプルを落とさない（finding #8 対策）。

```go
func main() {
    logger := slog.New(slog.NewTextHandler(os.Stderr, nil)) // stderr のみ（basic-spec §2.2）
    cliPath := flag.String("config", "", "path to config.toml")
    flag.Parse()

    cfg, _ := config.Load(*cliPath, logger)
    coll := collector.New(logger)                 // 能力プローブ（§5.8）
    st := store.New(cfg.Window, cfg.SamplingInterval)

    // プライミング兼初回サンプル: error でも捨てず Add（初回CPUはN/Aになりうる。§5.3, FR-9）
    s0, err := coll.Collect()
    if err != nil { logger.Warn("initial collect partial failure", "err", err) }
    st.Add(s0)

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    go sampler(ctx, coll, st, cfg.SamplingInterval, logger)

    if err := mcpserver.NewServer(st, cfg, logger).Run(ctx); err != nil {
        logger.Error("server exited", "err", err)
    }
}

func sampler(ctx context.Context, coll collector.Collector, st *store.Store, interval time.Duration, logger *slog.Logger) {
    t := time.NewTicker(interval); defer t.Stop()
    for {
        select {
        case <-ctx.Done(): return
        case <-t.C:
            s, err := coll.Collect()
            if err != nil { logger.Warn("collect partial failure", "err", err) }
            st.Add(s) // 部分失敗でも Add（§3）
        }
    }
}
```

終了: STDIO の EOF で `Run` が戻る、または SIGINT/SIGTERM で `ctx` キャンセル。永続状態なしのため後処理不要（`basic-spec.md §10.4`）。

---

## 11. ロギングと異常系

`slog` を `os.Stderr` に固定（`basic-spec.md §2.2, §11`）。レベル: 起動=Info、設定フォールバック/実効ウィンドウ引き上げ/採取部分失敗=Warn、パース詳細=Debug。
`basic-spec.md §11` の事象表を分岐に対応（GPU非搭載=空配列、`nvidia-smi` 不在（Windows）=NVIDIA を DXGI+PDH で扱い CUDA のみ N/A、`nvidia-smi` 不在（Linux）=NVIDIA は sysfs 列挙のみ、PDH/DXGI 不可=使用率/VRAM を N/A、安定ID欠如=index+name＋warn、設定不正=既定へ、等）。

---

## 12. テスト詳細

`basic-spec.md §13`。

- **trend（最優先）**: テーブル駆動で固定サンプル→期待英語文。3系統・VRAM量的系列・データ不足・決定性（2回同一）。
- **thresholds 境界**: CV 0.10/0.30、正味15pt、張り付き90%×95%、スパイク2σ。
- **config（rev.2）**: 欠如/不正、そして **`interval=120, window 未指定/30` で実効ウィンドウ=120** になり最低1サンプルを保証すること。
- **CPU トポロジ**: `logicalToSocket` の golden 入力でソケット集約、**取得不能時 `per_socket=[]`（空配列）フォールバック**、単一ソケットは `[overall]`（1要素）になること、初回CPU error で overall=nil の部分サンプルを保存すること。
- **GPU パーサ/突合（golden file）**: `nvidia_smi.xml`、`system_profiler.json`、Windows PDH インスタンス名配列を固定。**同型NVIDIA2枚が nvidia-smi 単一ソースで正しく2枚に分かれること**、非NVIDIA が DXGI+PDH の LUID 突合で正しく結合すること、そして**条件付きソース選択の両分岐**——`nvidia-smi` 在時は NVIDIA が DXGI 列挙から除外され二重計上しないこと、`nvidia-smi` 不在時は NVIDIA が DXGI 列挙に含まれ CUDA=nil で返ること——を検証。
- **傾向の複数成立（`basic-spec §8.3.1`）**: 乱高下しつつ +15pt 上昇する系列で fluctuating と rising が両方出ること、全張り付き `[99,99,99,99,99,99]` で pinned のみ（spike は出ない）になること、PDH engine `(LUID,phys,eng)` 集約で 3D 60%＋Copy 60% が adapterUtil=60 になること。
- **PDH 使用率集約**: 3D 60%＋Copy 60% の入力で adapterUtil=60（max-engine、合算120や100clampでない）を検証。
- **公開スキーマ**: `get_current_resources` の出力が `basic-spec §6.2`（`cpu`/`memory.used_percent`/`gpus` ネスト）に一致すること（スキーマ回帰テスト）。
- **store**: 容量算出、上書き、`Snapshot` の window 絞り込み・昇順・複製独立性。
- **軽量性（NFR-1, `basic-spec.md §12.1`）**: アイドルで監視プロセスの CPU/RSS を計測し規範値内。

---

## 13. トレーサビリティ（基本仕様 rev.3 → 詳細設計）

| 基本仕様（rev.3） | 本書での確定箇所 |
| --- | --- |
| §4 データモデル／§6.2 公開スキーマ | §5.1（公開スキーマに一致する型） |
| §4.4 GPU 識別子（ソース内一意） | §5.7 |
| §5.3 設定（実効ウィンドウ=max） | §4.3 |
| §6 リングバッファ | §6 |
| §7.2 CPU（gopsutil＋OSトポロジ・取得不能は空配列） | §5.3 |
| §7.3–§7.4 GPU（条件付きソース選択・PDH max-engine・Adapter Memory） | §5.4〜§5.7 |
| FR-7.1（取得手段が無い項目は N/A 許容） | §5.3, §5.4, §5.5 |
| §8 傾向判定（規範値） | §7 |
| §9 自然言語生成 | §8 |
| §6 MCP スキーマ | §9（SDK v1.6.1・登録・structuredContent） |
| §10 並行処理・ライフサイクル | §10 |
| §11 異常系・ログ | §11 |
| §13 テスト | §12 |

---

## 14. 次フェーズ（実装）への申し送り

設計判断ではなく、コード化・実機確認の作業に限る。

- MCP SDK `v1.6.1` の実 API へ §9.2 の擬似コードを整合。
- Windows の DXGI/PDH/トポロジAPI の `x/sys/windows` バインディング実装（COM vtbl 呼び出しの定型化）と実機確認。特に PDH インスタンス名フォーマットの実測確認。
- `nvidia-smi`／`system_profiler`／PDH 出力を `testdata/` に採取しゴールデン化。
- CI（`go vet`／`go test`／各OS `CGO_ENABLED=0` クロスビルド）整備。
- NFR-1 軽量性（`basic-spec.md §12.1`）の実測。超過時は詳細設計で握りつぶさず基本仕様の改訂要否を判断。

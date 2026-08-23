# 詳細設計書 — SystemResourceMCP

> このドキュメントは、`basic-spec.md` の**基本仕様**を、コードに落とせる粒度の**詳細設計**へ展開したものである。
> 主に `basic-spec.md` §15 が実装フェーズ手前で詰めるとした事項を確定させる。
> 各設計判断には、対応する基本仕様の節（`basic-spec.md §x`）を添える。
>
> - 日付: 2026-08-23
> - フェーズ: **詳細設計**（前フェーズ＝基本仕様／次フェーズ＝実装）
> - フェーズ体系: 要求 → 要件定義 → 基本仕様 → **詳細設計** → 実装
> - 上位文書: `docs/basic-spec.md`（さらに上位に `reqire.md` / `hearing.md`）

### 本書と上位仕様の関係（改訂の原則）

本書は詳細設計であり、**基本仕様で確定した規範値（傾向判定定数 `basic-spec.md §8.3`、NFR-1 数値目標 §12.1、ツールスキーマ §6 等）を変更しない**。
実装過程でこれらを変える必要が生じた場合は、詳細設計側で握りつぶさず、**基本仕様（`basic-spec.md`）を改訂してから本書に反映する**（`basic-spec.md` 冒頭「本書の確定と改訂の原則」）。
本書が新たに確定するのは、関数シグネチャ・型・API項目の対応づけ・具体手順といった**実装粒度**に限る。

---

## 1. 本書の範囲と方針

`basic-spec.md §15` が挙げた次の5点を本書で確定する。

1. 各採取器の関数シグネチャと、`nvidia-smi` / `system_profiler` / Windows GPU カウンタ・DXGI 各APIの取得項目と `Sample`/`GPUSample` フィールドの対応づけ（§4, §5）。
2. Windows パフォーマンスカウンタ（PDH）と DXGI/DXCore を `golang.org/x/sys/windows` から呼ぶ具体手順とアダプタ単位への集約（§5.5）。
3. Go 向け MCP SDK の選定確定と、`outputSchema` 宣言・ツール登録の具体コード（§9）。
4. 英語テンプレートの全判定網羅と `{metric}` 具体化の文字列規則（§8）。
5. リングバッファのサイズ余裕分と、周期・ウィンドウ変更時の再確保の扱い（§6）。

言語は Go、`CGO_ENABLED=0` の単一バイナリを維持する（`basic-spec.md §12.2 NFR-4`）。
以下のコードは詳細設計としてのシグネチャ・型・アルゴリズムであり、最終実装で軽微な調整はありうるが、公開インターフェース（MCPツール名・スキーマ、設定キー、規範値）は変更しない。

---

## 2. モジュールと依存

`basic-spec.md §3` のパッケージ構成を踏襲する。外部依存は次に限定する。

| 依存 | 用途 | 備考 |
| --- | --- | --- |
| `github.com/shirou/gopsutil/v4` | CPU・メモリ・プロセス採取 | cgo 不要（NFR-4） |
| `github.com/BurntSushi/toml` | 設定ファイル解釈（§4） | cgo 不要 |
| `github.com/modelcontextprotocol/go-sdk` | MCP サーバー実装（§8） | STDIO・structured output 対応。バージョンは実装時に最新安定へピン留め（§8.1） |
| 標準 `log/slog` | 標準エラーへの構造化ログ | 標準出力は使わない（`basic-spec.md §2.2`） |
| `golang.org/x/sys/windows` | Windows の PDH / DXGI 呼び出し（§5.5） | `//go:build windows` のみ |

`nvidia-smi` / `system_profiler` / `rocm-smi` は外部コマンドであり、依存ライブラリではない（サブプロセス起動）。

---

## 3. エラーハンドリングの共通規約

`basic-spec.md §7.5, §11` の劣化継続を、コードでは次の規約で実現する。

- 採取処理は Go の `error` を返すが、**呼び出し側（サンプラー）はエラーでプロセスを止めない**。ログを stderr に出し、当該フィールドを欠損（後述 nil）にして続行する。
- 「取得できなかった」と「値が0だった」を区別するため、**欠損しうる数値はポインタ型**（`*float64` / `*uint64` / `*string`）で表し、`nil` を N/A ＝ JSON `null` に対応させる（`basic-spec.md §4.3`）。
- パニックは想定しない。外部コマンド実行・パース境界には `recover` を置かず、代わりに全戻り値を検査する。

---

## 4. 設定（`config` パッケージ）

`basic-spec.md §5` を実装レベルに落とす。

### 4.1 型と既定値

```go
package config

import "time"

const (
    DefaultSamplingInterval = 10 * time.Second
    DefaultWindow           = 60 * time.Second
)

// Config は検証済みの実効設定。秒ではなく time.Duration で持つ。
type Config struct {
    SamplingInterval time.Duration
    Window           time.Duration
}

// fileConfig は TOML の生の形。未指定を検出するためポインタで受ける。
type fileConfig struct {
    SamplingIntervalSeconds *int `toml:"sampling_interval_seconds"`
    WindowSeconds           *int `toml:"window_seconds"`
}
```

### 4.2 探索と読み込み

```go
// Load は basic-spec §5.2 の優先順（CLI > 環境変数 > OS標準ディレクトリ）で
// 設定ファイルを探し、無ければ既定値の Config を返す。
// 返す Config は常に検証済みで、error はファイルの物理的読み取り失敗のみを表す
// （不正値は error ではなく既定へフォールバックし、warn ログを出す）。
func Load(cliPath string, logger *slog.Logger) (Config, error)

// resolvePath: cliPath 非空ならそれ。次に os.Getenv("SYSTEMRESOURCEMCP_CONFIG")。
// 次に filepath.Join(os.UserConfigDir(), "systemresourcemcp", "config.toml")。
// いずれも存在しなければ ("", false) を返し、Load は既定 Config を返す。
func resolvePath(cliPath string) (path string, found bool)
```

### 4.3 検証（`basic-spec.md §5.3` の規則）

```go
// validate は fileConfig を Config へ変換する。
// 各項目は独立に検証し、満たさない項目だけを既定へ落とす（起動は失敗させない）。
func validate(fc fileConfig, logger *slog.Logger) Config {
    cfg := Config{SamplingInterval: DefaultSamplingInterval, Window: DefaultWindow}

    if fc.SamplingIntervalSeconds != nil {
        if *fc.SamplingIntervalSeconds >= 1 {
            cfg.SamplingInterval = time.Duration(*fc.SamplingIntervalSeconds) * time.Second
        } else {
            logger.Warn("invalid sampling_interval_seconds; using default",
                "value", *fc.SamplingIntervalSeconds, "default_seconds", 10)
        }
    }
    if fc.WindowSeconds != nil {
        if time.Duration(*fc.WindowSeconds)*time.Second >= cfg.SamplingInterval {
            cfg.Window = time.Duration(*fc.WindowSeconds) * time.Second
        } else {
            logger.Warn("window_seconds must be >= sampling interval; using default",
                "value", *fc.WindowSeconds, "default_seconds", 60)
        }
    }
    return cfg
}
```

`window >= samplingInterval` を課すのは、ウィンドウに最低1サンプルを保証するため（`basic-spec.md §5.3`）。既定同士（60s ≥ 10s）は常に成立する。

---

## 5. データモデルと採取（`sample` / `collector`）

### 5.1 データ型（`sample` パッケージ）

`basic-spec.md §4` を Go 型に落とす。JSON タグは MCP 応答（§8）の `structuredContent` のキーになる。

```go
package sample

import "time"

type Sample struct {
    Timestamp        time.Time  `json:"timestamp"`
    CPUOverall       *float64   `json:"cpu_overall_percent"`
    CPUPerSocket     []float64  `json:"cpu_per_socket_percent"` // 空配列可、要素は非nil
    MemUsedBytes     *uint64    `json:"mem_used_bytes"`
    MemTotalBytes    *uint64    `json:"mem_total_bytes"`
    GPUs             []GPUSample `json:"gpus"`                   // 非搭載時は空配列
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
    // 識別子層（basic-spec §4.4）
    StableID     string  `json:"stable_id"`      // 必ず非空（導出は §5.7）
    PCIBusID     *string `json:"pci_bus_id"`
    UUID         *string `json:"uuid"`
    OSAdapterID  *string `json:"os_adapter_id"`
    Index        int     `json:"index"`          // 表示用の一時番号
    // 記述層
    Name         *string `json:"name"`
    Vendor       Vendor  `json:"vendor"`
    // メトリクス層
    UtilPercent    *float64 `json:"utilization_percent"`
    VRAMUsedBytes  *uint64  `json:"vram_used_bytes"`
    VRAMTotalBytes *uint64  `json:"vram_total_bytes"`
    CUDAVersion    *string  `json:"cuda_version"`
}
```

`Sample` は生成後に書き換えない（不変）。採取器が新しい `Sample` を作り、Store へ渡す。

### 5.2 Collector インターフェース

```go
package collector

// Collector は1時点のリソースを測定して Sample を1つ返す。
// error は致命ではなく、部分的失敗の記録用（呼び出し側は落とさない。§3）。
type Collector interface {
    Collect() (sample.Sample, error)
}

// New は各サブ採取器を束ね、起動時の能力プローブ（§5.8）を一度だけ行う。
func New(logger *slog.Logger) Collector

// 内部は3つのサブ採取器を合成する。
type collector struct {
    logger *slog.Logger
    cpu    cpuCollector
    mem    memCollector
    gpu    gpuCollector // OS別・ビルドタグで実体が変わる（§5.4〜5.6）
}
```

`Collect()` は CPU→メモリ→GPU の順に測り、個々の失敗は当該フィールドを nil にして継続する。GPU 採取が全滅すれば `GPUs` は空配列になる（`basic-spec.md §7.5`）。

### 5.3 CPU・メモリ（`cpu.go` / `mem.go`）

gopsutil を使う（`basic-spec.md §7.2`）。マルチソケット集約が唯一非自明なので手順を固定する。

```go
// CPU 全体使用率: cpu.Percent(0, false) の [0]。直前呼び出しからの差分平均。
//   サンプラーが固定周期で呼ぶため、区間平均になる（basic-spec §7.2）。
// ソケット別: cpu.Percent(0, true) で論理コア別使用率を得て、
//   cpu.Info() の PhysicalID でグルーピングし、ソケットごとに算術平均する。
func (c cpuCollector) collect() (overall *float64, perSocket []float64, err error)
```

`PhysicalID` でまとめ、キーの昇順に `perSocket` を並べる（決定的順序）。物理CPUが1つなら要素1個で全体値と一致する（`basic-spec.md §4.1`）。

```go
// メモリ: mem.VirtualMemory() の Used / Total を *uint64 で返す。
func (c memCollector) collect() (used, total *uint64, err error)
```

gopsutil の CPU 割合は「前回呼び出しからの差分」で算出されるため、**採取器はプロセス内で状態を持つ gopsutil の内部に依存する**。起動直後の初回サンプルは差分基準がなく 0 になりうるので、`main` は起動時に1回捨てサンプル（プライミング）を採ってから本採取に入る（§9.1）。

### 5.4 GPU の三層と OS 振り分け

`basic-spec.md §7.3` の三層（列挙層 → OS共通メトリクス層 → ベンダー拡張層）を、次のインターフェースで表す。

```go
// gpuCollector は OS ごとにビルドタグで実体が入れ替わる。
type gpuCollector interface {
    // probe は起動時に一度呼ばれ、利用可能な手段を判定する（§5.8）。
    probe(logger *slog.Logger)
    // collect は列挙＋メトリクス＋ベンダー拡張を統合した []GPUSample を返す。
    collect() ([]sample.GPUSample, error)
}
```

| ファイル | ビルドタグ | 列挙層 | OS共通メトリクス層 | ベンダー拡張層 |
| --- | --- | --- | --- | --- |
| `gpu_windows.go` | `//go:build windows` | DXGI/DXCore | GPU Engine / Adapter Memory カウンタ（PDH）＋DXGI（§5.5） | NVIDIA=`nvidia-smi`（§5.4.1） |
| `gpu_linux.go` | `//go:build linux` | sysfs `/sys/class/drm` | （限定的） | NVIDIA=`nvidia-smi`、AMD=`rocm-smi`／sysfs |
| `gpu_darwin.go` | `//go:build darwin` | `system_profiler`（§5.6） | なし（使用率・VRAM は N/A） | なし |

各 OS 実装は「列挙で得た `[]GPUSample` の骨（識別子＋名前＋ベンダー）」に、メトリクス層・ベンダー拡張層で得た値を `StableID` で突き合わせて上書きする（§5.7）。

#### 5.4.1 `nvidia-smi` XML → `GPUSample` の対応づけ

NVIDIA 拡張は `nvidia-smi -q -x` の XML を解釈する（`basic-spec.md §7.4`）。解釈に使う XML 要素と格納先を固定する。

| XML パス（`nvidia_smi_log/gpu` 配下） | 例 | 格納先 | 変換 |
| --- | --- | --- | --- |
| `@id`（属性）または `pci/pci_bus_id` | `00000000:01:00.0` | `PCIBusID` | 小文字化し `0000:01:00.0` 形へ正規化 |
| `uuid` | `GPU-4e1f…` | `UUID` | そのまま |
| `product_name` | `NVIDIA GeForce RTX 5090` | `Name` | そのまま |
| `utilization/gpu_util` | `91 %` | `UtilPercent` | ` %` を除き float |
| `fb_memory_usage/used` | `18253 MiB` | `VRAMUsedBytes` | MiB→bytes（×1024²） |
| `fb_memory_usage/total` | `32607 MiB` | `VRAMTotalBytes` | MiB→bytes |
| `nvidia_smi_log/cuda_version` | `12.4` | `CUDAVersion` | ルート直下。全GPU共通 |

パース失敗・要素欠如は当該フィールドを nil にする。`nvidia-smi` 不在（プローブで判定）ならこの層はスキップする。

```go
func parseNvidiaSmiXML(xmlBytes []byte) ([]nvGPU, error) // 単体テスト対象（golden XML, §12）
// nvGPU.pciBusID を鍵に、列挙層 GPUSample へ util/vram/uuid/cuda を上書き。
```

### 5.5 Windows 共通メトリクス層（PDH ＋ DXGI）

`basic-spec.md §7.4` の Windows 共通層を、`golang.org/x/sys/windows` から cgo なしで呼ぶ手順に落とす。ここが本書で最も具体化を要する箇所である。

**列挙・VRAM総量・LUID（DXGI）**
1. `CreateDXGIFactory1` で `IDXGIFactory1` を得る（`syscall.SyscallN` で COM vtbl を叩く／`x/sys/windows` の COM ヘルパを使用）。
2. `EnumAdapters1` を index 0 から `DXGI_ERROR_NOT_FOUND` まで回し、各 `IDXGIAdapter1` の `GetDesc1` で `DXGI_ADAPTER_DESC1` を取得。
3. `Description`（機種名→`Name`）、`DedicatedVideoMemory`（→`VRAMTotalBytes`）、`AdapterLuid`（→`OSAdapterID`、`LUID` を `"%d:%d"` 表記）、`VendorId`（→`Vendor` 判定: 0x10DE=NVIDIA, 0x1002=AMD, 0x8086=Intel, 0x106B=Apple）を得る。
4. ソフトウェアアダプタ（`DXGI_ADAPTER_FLAG_SOFTWARE`）は除外する。

**VRAM使用量（DXGI 3）**
- `IDXGIAdapter3.QueryVideoMemoryInfo(nodeIndex=0, DXGI_MEMORY_SEGMENT_GROUP_LOCAL)` の `CurrentUsage` を `VRAMUsedBytes` にする。`IDXGIAdapter3` が得られない旧環境では nil。

**使用率（PDH パフォーマンスカウンタ）**
1. `PdhOpenQuery` → `PdhAddEnglishCounter` でワイルドカードカウンタ `\GPU Engine(*)\Utilization Percentage` を追加（English 版APIでロケール非依存）。
2. `PdhCollectQueryData` を2回（間に短い待ち。ただし常時サンプラー周期に合わせ、前回値を保持して差分ではなく瞬時値を読む方式でも可）。
3. `PdhGetFormattedCounterArray(PDH_FMT_DOUBLE)` でインスタンスごとの値を得る。インスタンス名は `pid_<pid>_luid_0x0_0x<LUID>_phys_0_eng_<n>_engtype_<type>` 形式。
4. **アダプタ単位への集約**: インスタンス名から `luid_...` を抽出し、同一 LUID のエンジン値を合算（3D/Compute/Copy 等を含む全エンジン合計）。ただし合計が 100 を超える場合は 100 にクランプする。
5. 集約した LUID→使用率を、DXGI 列挙で得た `OSAdapterID`(LUID) と突き合わせて `UtilPercent` に入れる。

同様に `\GPU Adapter Memory(luid_...)\Dedicated Usage` を集約すれば、DXGI 3 が無い環境の VRAM 使用量フォールバックにできる。

いずれの手順も Win32 API 呼び出しであり `x/sys/windows` の `syscall` 経由で cgo を持ち込まない（`basic-spec.md §12.2 NFR-4`）。API が使えない環境（カウンタ無効等）はプローブで検出し、当該項目を N/A にする（`basic-spec.md §11`）。

```go
func (g *windowsGPU) probe(logger *slog.Logger)   // DXGI factory 生成可否、PDH クエリ可否を判定
func enumerateDXGI() ([]sample.GPUSample, error)   // 列挙＋VRAM総量＋LUID＋Vendor
func queryGPUUtilByLUID() (map[string]float64, error) // PDH 集約（LUID→util）
```

### 5.6 macOS 列挙（`system_profiler` JSON）

`basic-spec.md §7.4`。`system_profiler SPDisplaysDataType -json` を解釈する。

| JSON パス | 格納先 | 備考 |
| --- | --- | --- |
| `SPDisplaysDataType[].sppci_model` | `Name` | 機種名 |
| （ベンダー推定） | `Vendor` | 名前に "Apple" 等を含むかで判定、既定 Unknown |
| — | `UtilPercent` | 常に nil（特権要のため N/A、`basic-spec.md §7.4`） |
| — | `VRAMUsedBytes` / `VRAMTotalBytes` | 統合メモリのため nil |
| — | `CUDAVersion` | 常に nil（A-5） |

Apple Silicon は PCI バスIDや UUID が取りにくいため、`StableID` は §5.7 のフォールバック（index+name）になりうる。

### 5.7 `StableID` 導出と層の突合

`basic-spec.md §4.4` のアルゴリズムを確定する。

```go
func deriveStableID(g sample.GPUSample) string {
    switch {
    case g.UUID != nil && *g.UUID != "":       return "uuid:" + *g.UUID
    case g.PCIBusID != nil && *g.PCIBusID != "": return "pci:" + *g.PCIBusID
    case g.OSAdapterID != nil && *g.OSAdapterID != "": return "luid:" + *g.OSAdapterID
    default:
        // 最終手段: 不安定である旨を warn ログ（basic-spec §4.4, §11）
        return fmt.Sprintf("idx:%d:%s", g.Index, deref(g.Name))
    }
}
```

**突合**: 列挙層で作った `[]GPUSample` を `StableID`→*GPUSample の map にし、メトリクス層／ベンダー拡張層の結果を同じ鍵で引いて上書きする。鍵が一致しない（安定IDが取れない）場合のみ、index と名前の一致で対応づけ、warn を出す。これにより同型GPU複数枚でも取り違えない（`basic-spec.md §4.4`）。

### 5.8 能力プローブ

`basic-spec.md §7.1`。起動時に一度、次を判定してフラグ化し、以後のサンプリングでは可能な手段のみ呼ぶ。

- `nvidia-smi` の存在（`exec.LookPath`）。
- （Linux）`rocm-smi` の存在。
- （Windows）DXGI factory 生成可否、PDH クエリ生成可否。
- GPU 列挙自体の成否（0枚なら以後 GPU 層をスキップ）。

---

## 6. リングバッファ（`store` パッケージ）

`basic-spec.md §10.2, §10.3` と、§15 の「サイズ余裕・再確保」を確定する。

### 6.1 型と容量

```go
package store

type Store struct {
    mu   sync.RWMutex
    buf  []sample.Sample // 長さ cap の固定長リング
    head int             // 次に書く位置
    size int             // 現在の有効件数（起動直後は cap 未満）
    cap  int
}

// capacityFor は window/interval に余裕を足したリング容量を返す。
// 余裕分（+2）は、境界のタイミングずれで直近ウィンドウのサンプルを
// 取りこぼさないためのマージン（basic-spec §10.1「若干の余裕」）。
func capacityFor(window, interval time.Duration) int {
    n := int(math.Ceil(float64(window) / float64(interval)))
    if n < 1 { n = 1 }
    return n + 2
}
```

### 6.2 操作

```go
func New(window, interval time.Duration) *Store        // cap = capacityFor(...)
func (s *Store) Add(smp sample.Sample)                  // 書き込み。満杯なら最古を上書き
// Snapshot は直近 window 相当のサンプルを新しいスライスに複製して返す。
// ロックは複製の間だけ保持し、分析・文生成はロック外で行う（basic-spec §10.3）。
func (s *Store) Snapshot(window time.Duration) []sample.Sample
```

`Snapshot` は `Timestamp` が「最新から window 以内」のサンプルだけを、時刻昇順で複製して返す。呼び出し側（trend）はこの複製を触るだけなので、サンプラーの書き込みと競合しない。

### 6.3 周期・ウィンドウ変更時の再確保

設定は起動時に確定し、実行中の動的変更は仕様に無い（`basic-spec.md §5, FR-6` は「設定ファイルで変更」＝再起動反映）。
したがって**実行中の再確保は行わない**。容量は起動時に一度だけ `capacityFor` で決める。
将来オンライン変更を入れる場合は、新 cap のバッファを確保して既存有効分をコピーするが、本フェーズのスコープ外とし、`store` の API は再確保を前提としない設計にとどめる（YAGNI）。この判断は基本仕様の「非永続・起動時確定」（§10.1, A-3）と整合する。

---

## 7. 傾向分析（`trend/analyze.go` ＋ `thresholds.go`）

`basic-spec.md §8` の規範値をコードにする。**定数は基本仕様の値をそのまま写す**（変更は基本仕様の改訂を伴う。本書冒頭）。

### 7.1 判定定数（`thresholds.go`）

```go
package trend

// これらは basic-spec §8.3 の規範値の写し。ここで値を変えてはならない。
const (
    steadyCV        = 0.10 // 変動係数 < 0.10 → 安定
    fluctCV         = 0.30 // 変動係数 >= 0.30 → 乱高下
    fluctRangePt    = 40.0 // または 最大-最小 >= 40pt（使用率系）
    trendNetPt      = 15.0 // 正味変化 >= 15pt で上昇/下降（使用率系）
    pinnedFrac      = 0.90 // サンプルの90%以上が…
    pinnedLevel     = 95.0 // …95%以上 → 張り付き
    spikeSigma      = 2.0  // 最大 >= 平均+2σ → スパイク
    highLevel       = 90.0 // 「高位」= 90%以上
    // 量的系列（VRAM 等）は割合で評価（basic-spec §8.3）
    trendNetRatio   = 0.25 // 正味変化 >= 平均の25%
)
```

### 7.2 系列と統計量

```go
type seriesKind int
const (
    kindPercent seriesKind = iota // 使用率系（絶対pt評価）
    kindQuantity                  // 量的系列（相対%評価。VRAM等）
)

type stats struct{ mean, std, min, max, first, last, slope float64; n int }

// computeStats は null を除いた実測値だけで統計量を出す（basic-spec §8.1）。
// slope は時刻に対する最小二乗回帰の傾き。n<2 は slope=0。
func computeStats(values []float64) stats
```

### 7.3 三系統判定

```go
type levelVerdict struct{ steady bool; mean float64 }
type dirVerdict   struct{ kind dirKind; start, end float64 } // rising/falling/fluctuating/flat
type peakVerdict  struct{ kind peakKind; max float64 }       // pinned/spike/none

func judgeLevel(s stats) levelVerdict
func judgeDirection(s stats, k seriesKind) dirVerdict
func judgePeak(values []float64, s stats, k seriesKind) peakVerdict
```

判定式は `basic-spec.md §8.3` の表をそのまま実装する。使用率系は pt、量的系列は平均比。`pinned` は「値≥`pinnedLevel` のサンプル比率 ≥ `pinnedFrac`」（使用率系のみ）。`spike` は `max ≥ mean + spikeSigma*std` かつ `max ≥ highLevel`（使用率系）／量的系列は `max ≥ mean + spikeSigma*std`。

### 7.4 分析結果

```go
type MetricTrend struct {
    Metric string // §8 の {metric} 展開済み文字列（§8のReporterが使用）
    Kind   seriesKind
    Level  levelVerdict
    Dir    dirVerdict
    Peak   peakVerdict
    HasData bool
}

// Analyze は Snapshot 済みサンプル列から系列ごとの MetricTrend を作る。
// CPU / Memory / GPU[i].Util / GPU[i].VRAM を対象（basic-spec §8.1）。
// GPU 系列は StableID で同定し、同型複数枚を取り違えない。
func Analyze(samples []sample.Sample) []MetricTrend
```

決定性: すべて入力サンプルの統計量のみに依存し、乱数・時刻依存分岐を持たない（`basic-spec.md §8.4`）。

---

## 8. 自然言語生成（`trend/report.go` ＋ `templates.go`）

`basic-spec.md §9` の §15 申し送り「全判定網羅」「`{metric}` 文字列規則」を確定する。

### 8.1 `{metric}` 文字列規則

| 系列 | `{metric}` 文字列 |
| --- | --- |
| CPU 全体 | `CPU` |
| メモリ使用率 | `Memory usage` |
| GPU i 使用率 | `GPU {index} ({name})`（name が nil なら `GPU {index}`） |
| GPU i VRAM | `GPU {index} VRAM` |

`{index}` は表示用番号（`basic-spec.md §4.4, §9.2`）。系列の内部同定は `StableID` で行い、表示番号が入れ替わっても取り違えない。

### 8.2 テンプレート全網羅表（英語固定）

`basic-spec.md §9.2` を全判定について確定する。`%d`/`%.0f` は整数％表示、VRAM は GB 換算（小数1桁）。

| 判定 | 出力系統 | テンプレート |
| --- | --- | --- |
| steady | 水準 | `{metric} has stayed around {mean}% steadily.` |
| level(一般) | 水準 | `{metric} is around {mean}%.` |
| rising | 向き | `{metric} is rising, from {start}% to {end}%.` |
| falling | 向き | `{metric} is falling, from {start}% to {end}%.` |
| flat | 向き | （文を出さない） |
| fluctuating | 変動 | `{metric} is fluctuating widely (between {min}% and {max}%).` |
| pinned | 極値 | `{metric} is pinned at {max}%.` |
| spike(使用率) | 極値 | `{metric} spiked up to {max}%.` |
| spike(VRAM等) | 極値 | `{metric} spiked up to {max_gb} GB.` |
| VRAM level | 水準 | `{metric} is around {mean_gb} GB.` |
| VRAM rising | 向き | `{metric} is rising, from {start_gb} GB to {end_gb} GB.` |
| VRAM falling | 向き | `{metric} is falling, from {start_gb} GB to {end_gb} GB.` |
| VRAM fluctuating | 変動 | `{metric} is fluctuating widely (between {min_gb} GB and {max_gb} GB).` |

### 8.3 組み立て順と出力

```go
// BuildReport は MetricTrend 群から basic-spec §9.1 の順（水準→向き/変動→極値）で
// 文を並べ、TrendReport を返す。成立しない系統の文は出さない。
func BuildReport(trends []MetricTrend, window time.Duration, sampleCount, needed int) TrendReport

type TrendReport struct {
    Report       string `json:"report"`
    WindowSeconds int   `json:"window_seconds"`
    SampleCount   int   `json:"sample_count"`
    DataComplete  bool  `json:"data_complete"` // sampleCount >= needed
}
```

先頭行は `Over the last {window}s:`。各系列は箇条書き（`- `）。
データ不足時（`sampleCount < needed`）は `DataComplete=false` とし、`basic-spec.md §9.3` の書式で不足を明示する冒頭文を置く（`Only N samples collected so far (need M for a full Ws window). Preliminary trend:`）。1サンプル以下で傾向を出せない系列は「limited data」を添えるか省略する。

---

## 9. MCP サーバー（`mcpserver` パッケージ）

### 9.1 SDK 選定（§15 の確定事項）

Go 向け MCP 実装は **`github.com/modelcontextprotocol/go-sdk`（公式 SDK）** を採用する。
選定理由:
- STDIO トランスポートと、ツールの `outputSchema` / 構造化結果（`structuredContent`）を標準で扱える（`basic-spec.md §6.1` の要件に直結）。
- 公式実装のため MCP 仕様追随が見込め、単一バイナリ（cgo 不要）で組み込める（NFR-4）。
- Go の struct から入出力スキーマを生成でき、`ResourceSnapshot`/`TrendReport` の型（§5.1, §8.3）をそのまま公開できる。

> 確認事項: 採用バージョンは実装時点の最新安定タグにピン留めする。SDK の API 名は版により細部が変わりうるため、下記コードはインターフェースの意図を示すものとし、実装時に採用版のシグネチャへ整合させる（これは規範値ではなく実装粒度の調整に当たる）。

### 9.2 型と登録

```go
package mcpserver

// 出力スキーマの型（JSON タグは sample / trend 側と一致）
type ResourceSnapshot = sample.Sample          // 別名で公開
type TrendReport      = trend.TrendReport

func NewServer(st *store.Store, cfg config.Config, coll collector.Collector, logger *slog.Logger) *Server

// Run は STDIO で JSON-RPC を待つ（ブロッキング）。ctx キャンセルで終了。
func (s *Server) Run(ctx context.Context) error
```

ツール登録（意図を示す擬似コード）:

```go
// get_current_resources: 引数なし。最新サンプルを ResourceSnapshot で返す。
//   structuredContent = snapshot、content = その JSON 直列化（後方互換, basic-spec §6.2）。
server.AddTool("get_current_resources", withOutputSchema[ResourceSnapshot](),
    func(ctx, _ struct{}) (ResourceSnapshot, error) {
        s := st.Latest() // 直近1件（起動直後はプライミング済みの1件）
        return s, nil
    })

// get_resource_trend: 引数なし。直近ウィンドウの傾向を TrendReport で返す。
//   structuredContent = report、content = report.Report（英語本文, basic-spec §6.3）。
server.AddTool("get_resource_trend", withOutputSchema[TrendReport](),
    func(ctx, _ struct{}) (TrendReport, error) {
        samples := st.Snapshot(cfg.Window)
        needed := int(math.Ceil(float64(cfg.Window)/float64(cfg.SamplingInterval)))
        return trend.BuildReport(trend.Analyze(samples), cfg.Window, len(samples), needed), nil
    })
```

`search_processes` は MVP 外のため登録しない（`basic-spec.md §6.4`）。将来追加時も同じ構造化出力方式に従う。

`content`（後方互換 TextContent）は、SDK が structured 結果から自動生成しない場合に限り、ハンドラ側で JSON 直列化（snapshot）／本文（trend）を明示的に添える。

---

## 10. 並行処理とライフサイクル（`cmd/systemresourcemcp/main.go`）

`basic-spec.md §10` を配線コードに落とす。

```go
func main() {
    logger := slog.New(slog.NewTextHandler(os.Stderr, nil)) // 標準エラーのみ（basic-spec §2.2）
    cliPath := flag.String("config", "", "path to config.toml")
    flag.Parse()

    cfg, _ := config.Load(*cliPath, logger)
    coll := collector.New(logger)          // 内部で能力プローブ（§5.8）
    st := store.New(cfg.Window, cfg.SamplingInterval)

    if s, err := coll.Collect(); err == nil { st.Add(s) } // プライミング（§5.3, §9.1）

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    go sampler(ctx, coll, st, cfg.SamplingInterval, logger) // 周期採取
    srv := mcpserver.NewServer(st, cfg, coll, logger)
    if err := srv.Run(ctx); err != nil { logger.Error("server exited", "err", err) }
}

func sampler(ctx context.Context, coll collector.Collector, st *store.Store, interval time.Duration, logger *slog.Logger) {
    t := time.NewTicker(interval)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done(): return
        case <-t.C:
            s, err := coll.Collect()
            if err != nil { logger.Warn("collect partial failure", "err", err) }
            st.Add(s) // 部分失敗でも取れた分は入れる（§3）
        }
    }
}
```

終了: STDIO クライアント切断（`Run` が EOF で戻る）または SIGINT/SIGTERM で `ctx` がキャンセルされ、サンプラーも停止する。永続状態が無いため後処理は不要（`basic-spec.md §10.4`）。

---

## 11. ロギングと異常系

- ロガーは `slog` を `os.Stderr` に固定（`basic-spec.md §2.2, §11`）。標準出力は MCP 専用。
- レベル: 起動情報＝Info、設定フォールバックや採取部分失敗＝Warn、パース失敗詳細＝Debug。
- `basic-spec.md §11` の事象表をそのまま実装の分岐に対応させる（GPU非搭載＝空配列、`nvidia-smi` 不在＝Windowsは共通層で継続・他OSは名前のみ、権限不足＝取れた分のみ、等）。

---

## 12. テスト詳細

`basic-spec.md §13` を具体化する。

- **trend（最優先）**: `analyze_test.go` / `report_test.go` をテーブル駆動。固定サンプル列 → 期待英語文を検証。3系統の成立/不成立、VRAM 量的系列、データ不足（§8.3）を網羅。決定性は同一入力2回で同一出力を確認。
- **thresholds 境界**: 変動係数 0.10/0.30、正味 15pt、張り付き 90%×95%、スパイク 2σ の境界値ちょうどを検証。
- **config**: ファイル欠如／不正値／`window < interval` 境界で既定へ落ちること。
- **GPU パーサ（golden file）**: `testdata/nvidia_smi.xml`、`testdata/system_profiler.json`、Windows カウンタ配列のサンプルを固定し、パース結果を検証（実機不要）。
- **StableID / 突合**: 同型GPU2枚（同名・異なる PCI/UUID）を模した入力で正しく結合、安定ID欠如時に index+name フォールバックし warn することを検証。
- **store**: 容量算出、上書き、`Snapshot` の window 絞り込みと昇順・複製独立性。
- **軽量性（NFR-1, `basic-spec.md §12.1`）**: アイドルで監視プロセスの CPU/RSS を既定設定のまま計測し規範値内であることを確認するベンチ／手順を用意。

---

## 13. トレーサビリティ（基本仕様 → 詳細設計）

| 基本仕様 | 本書での確定箇所 |
| --- | --- |
| §4 データモデル | §5.1（Go 型・JSON タグ・nullable） |
| §4.4 GPU 識別子 | §5.7（導出・突合アルゴリズム） |
| §5 設定 | §4（型・探索・検証コード） |
| §6 リングバッファ／容量・再確保 | §6（容量算出・非再確保方針） |
| §7 採取（手段選定） | §5.2〜5.8（CPU/mem/GPU 三層・API マッピング） |
| §7.4 Windows 共通層 | §5.5（PDH/DXGI 具体手順・LUID 集約） |
| §8 傾向判定（規範値） | §7（統計量・判定関数・定数の写し） |
| §9 自然言語生成 | §8（テンプレート全網羅・{metric} 規則） |
| §6 MCP スキーマ | §9（SDK 選定・登録・structuredContent） |
| §10 並行処理・ライフサイクル | §10（main 配線・sampler・終了） |
| §11 異常系・ログ | §11 |
| §13 テスト | §12 |

`basic-spec.md §15` の5申し送りは、それぞれ §5（関数シグネチャ・API対応）、§5.5（PDH/DXGI）、§9.1（SDK確定）、§8（テンプレート網羅）、§6.3（リングバッファ再確保方針）で確定した。

---

## 14. 次フェーズ（実装）への申し送り

本書で実装に必要な設計は固めた。実装フェーズで扱うのは、設計判断ではなく次のコード化・検証作業に限る。

- 採用 MCP SDK バージョンの確定と、§9.2 の擬似コードを採用版 API に整合させる。
- Windows の DXGI/PDH 呼び出しの `x/sys/windows` バインディング実装（COM vtbl 呼び出しの定型化）と実機確認。
- `nvidia-smi` / `system_profiler` の実出力を `testdata/` に採取してゴールデン化。
- CI（`go vet`／`go test`／各OSクロスビルド `CGO_ENABLED=0`）の整備。
- NFR-1 軽量性（`basic-spec.md §12.1`）の実測と、規範値超過時の基本仕様改訂要否の判断。

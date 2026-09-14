# VRSki-GazeGuidance

VRスキー実験におけるコース走行、視線誘導、通過・注視指標、CSV記録を扱うUnityプロジェクトである。

## Sceneと設計資料

Build Settingsで有効なSceneは次の4つだけである。

| Scene | 役割 |
|---|---|
| `Settings` | 実験設定とScene起動 |
| `Test2` | CSV seedで構成したコースをスクロールする本実験 |
| `Test` | Splineに沿ってPlayerが前進する検証コース |
| `Calibration` | flag sequenceを用いた視線誤差記録 |

責務境界、PlayerPrefs、起動route、guidance順、seed・CSV互換仕様は[実験アーキテクチャ](docs/experiment-architecture.md)を参照する。テストとCIの実行・運用手順は[Unityテスト・CI運用ガイド](docs/testing-and-ci.md)を参照する。

## Gaussian Blur

このプロジェクトでは、VRスキー走行中に次の通過対象へ視線を誘導するため、対象方向だけを鮮明に残し、その外側へ一定強度のGaussian Blurを適用する。

| 要点 | 現行仕様 |
|---|---|
| Clear領域の中心 | コース前方にある次の通過対象 `p+1` の方向 |
| Clear領域 | 中心から固定角度以内。`Test2` sceneでは1° |
| Clear領域の表示 | full-resolutionの元画像（Original） |
| Clear領域外の表示 | 位置・距離によらず固定sigmaのBlurred image |
| Gaussian kernel | `radius = min(ceil(3 * sigmaEffective), 64)` |
| 描画 | half-resolution Horizontal → half-resolution Vertical → full-resolution Composite |

Quest/Androidで使われる描画経路は、`Mobile` Quality → `Mobile_RPAsset` → `Mobile_Renderer` → Activeな`RadialGaussianBlurRendererFeature` → `RadialGaussianBlurXR.shader`である。

### Clear領域

`DynamicGaussianBlur`は最も近い未通過フラグを`p`、その次を`p+1`として、`p+1`のworld位置をClear領域の中心にする。固定Clear angle以内ではOriginalを表示し、その外側では完成済みのBlurred imageを表示する。

- depthによってClear angleやsigmaを変更しない。
- inner / outer領域や段階的なsigma補間を使用しない。
- Horizontal / VerticalではClear判定を行わず、CompositeだけがOriginalとBlurred imageを選択する。

### Gaussian kernel

UIで指定する`sigmaFull`はfull-resolution pixel単位である。Blur用Textureは縦横1/2解像度なので、kernel計算には次のeffective sigmaを使う。

$$
\sigma_{\mathrm{effective}}=\frac{\sigma_{\mathrm{full}}}{2}
$$

$$
R=\min\left(\left\lceil3\sigma_{\mathrm{effective}}\right\rceil,64\right)
$$

CPUはsigma変更時だけ、offset $i=0\ldots R$のraw weightを次式で計算する。固定長65要素の配列を再利用し、radius外はゼロにする。

$$
w_i=\exp\left(-\frac{i^2}{2\sigma_{\mathrm{effective}}^2}\right)
$$

Shaderでは事前計算済みweightを参照し、kernel内のweight sumで正規化する。各pixel・各tapで`exp()`を再計算しない。

2次元Gaussianは、

$$
G(x,y)=\frac{1}{2\pi\sigma^2}
\exp\left(-\frac{x^2+y^2}{2\sigma^2}\right)
=G_x(x)G_y(y)
$$

と分離できるため、HorizontalとVerticalの2回の1D convolutionとして計算する。

### Bilinear tap pairing

隣接tap $i$と$i+1$は、次のcombined weightとfractional offsetを使用して1回のlinear samplingへまとめる。

$$
W=w_i+w_{i+1}
$$

$$
d=i+\frac{w_{i+1}}{W}
$$

正方向と負方向をそれぞれ$+d$、$-d$からsampleする。最後のtapがpairにならない場合は正負を個別にsampleする。半径$R$の1D passにおけるsample数は、中心を含めて概ね次となる。

$$
1+2\left\lceil\frac{R}{2}\right\rceil
$$

### 描画フロー

```text
Original（full resolution）
  │
  ├────────────────────────────────────┐
  ▼                                    │
Horizontal Gaussian（1/2 resolution）  │
  - CPU事前計算weight                  │
  - bilinear tap pairing               │
  ▼                                    │
Vertical Gaussian（1/2 resolution）    │
  ▼                                    │
Blurred（1/2 resolution）              │
  │                                    │
  └────────────────┬───────────────────┘
                   ▼
             Composite（full resolution）
                   ├─ Clear領域 → Original
                   └─ その他    → Blurredをlinear samplingで拡大
                   ▼
             Final（full resolution）
```

RenderGraphはHorizontal → Vertical → Compositeの3 passを維持する。`horizontal`と`blurred`だけを縦横1/2解像度、Bilinear、Clampで生成し、OriginalとFinalはfull-resolutionのままにする。`sigmaFull <= 1e-4`ではBlur用RenderGraph処理を追加せずOriginalを維持する。

XR向けの`TEXTURE2D_X`、stereo eye setup、instancing / multiview variant、per-eye view / projection matrixは維持する。

実装: [DynamicGaussianBlur.cs](Assets/Scripts/Method/DynamicGaussianBlur.cs)、[RadialGaussianBlurRendererFeature.cs](Assets/Scripts/Method/Rendering/RadialGaussianBlurRendererFeature.cs)、[RadialGaussianBlurXR.shader](Assets/MyAssets/Shader/RadialGaussianBlurXR.shader)

## Performance optimization

App GPU Timeを主指標として、Meta Quest Pro実機でOVR Metrics ToolのBasic metricsを計測した。同一のコース、走行時間、測定方法を使用し、Clear Angleは1°、Unity 6000.4.5f1 / URP 17.4.0 / Android OpenXR、Render Scaleは1.0とした。

### Blur implementation optimization

| 実装 | 変更内容 |
|---|---|
| Optimization 1 | Gaussian weightをCPUでsigma変更時だけ事前計算 |
| Optimization 2 | 隣接tapをbilinear samplingでpair化 |
| Optimization 4 | Horizontal / Verticalの中間Textureを縦横1/2へdownsampling |
| Integrated | Optimization 1 + 2 + 4を統合 |

各Optimizationの単体値はBaselineから独立して変更・計測した結果であり、累積結果ではない。コードBaselineは[`5371c56`](https://github.com/shihu-14/VRSki-GazeGuidance/commit/5371c569d9c7b92ddfe7c8b8c8103e0ab5a62fa6)である。

| 実装 | 条件 | App GPU Time median | p95 | FPS median | GPU Util. median | CPU Util. median | Stale Frames |
|---|---|---:|---:|---:|---:|---:|---:|
| Baseline | Blur Off | 16.12 ms | 17.32 ms | 53 | 97% | 16% | 1101 |
| Baseline | Weak | 27.61 ms | 30.15 ms | 31 | 97% | 11% | 2111 |
| Baseline | Strong | 29.66 ms | 31.44 ms | 30 | 98% | 10% | 2133 |
| Optimization 1: Weight precompute | Weak | 27.36 ms | 28.98 ms | 32 | 97% | 11% | 1868 |
| Optimization 1: Weight precompute | Strong | 30.55 ms | 32.63 ms | 29 | 98% | 11% | 2030 |
| Optimization 2: Bilinear sampling | Weak | 27.069 ms | 28.240 ms | 32 | 97% | 31% | 1890 |
| Optimization 2: Bilinear sampling | Strong | 28.694 ms | 30.596 ms | 31 | 98% | 33% | 1923 |
| Optimization 4: Downsampling | Weak | 25.236 ms | 26.289 ms | 35 | 97% | 33% | 1730 |
| Optimization 4: Downsampling | Strong | 25.792 ms | 27.432 ms | 34 | 97% | 35% | 1772 |
| Integrated: Opt1 + Opt2 + Opt4 | Weak | 24.229 ms | 25.031 ms | 36 | 98% | 33% | 1608 |
| Integrated: Opt1 + Opt2 + Opt4 | Strong | 24.649 ms | 26.343 ms | 36 | 98% | 33% | 1674 |

Integrated版とBaselineの比較：

| 条件 | Baseline median | Integrated median | 差 | 改善率 |
|---|---:|---:|---:|---:|
| Weak | 27.61 ms | 24.229 ms | -3.381 ms | 12.2% |
| Strong | 29.66 ms | 24.649 ms | -5.011 ms | 16.9% |

Optimization 3（Composite fusion）は、最終統合版には含めていない。

単体計測の補足：

| 実装 | 条件 | 集計区間 | Samples | GPU frequency median | Battery temperature median |
|---|---|---:|---:|---:|---:|
| Optimization 2 | Weak | 8.002〜52.006秒 | 45 | 525 MHz | 28℃ |
| Optimization 2 | Strong | 4.001〜47.004秒 | 44 | 525 MHz | 28℃ |
| Optimization 4 | Weak | 6.001〜50.004秒 | 45 | 525 MHz | 28℃ |
| Optimization 4 | Strong | 4.001〜48.005秒 | 45 | 525 MHz | 28℃ |

画質面では、Weak条件で低解像度化された見え方が目立ち、StrongではBlurに隠れて違和感が比較的目立ちにくかった。これは実機での定性的観察であり、Weakのsigma約0.8はユーザー申告値である。

### Rendering configuration optimization

BlurコードをIntegrated（Opt1 + Opt2 + Opt4）に固定し、Rendering PathとMSAAだけを変更した。基準はForward+ / MSAA 4xである。

| Rendering Path | MSAA | Weak GPU median | Strong GPU median | Weak FPS | Strong FPS |
|---|---:|---:|---:|---:|---:|
| Forward+ | 4x | 24.229 ms | 24.649 ms | 36 | 36 |
| Forward | 4x | 21.865 ms | 22.605 ms | 40 | 39 |
| Forward+ | 2x | 15.680 ms | 16.353 ms | 52 | 50 |
| Forward | 2x | 13.796 ms | 14.323 ms | 62 | 60 |

補助指標：

| Rendering Path | MSAA | 条件 | App GPU Time p95 | GPU Util. median | CPU Util. median | Stale Frames |
|---|---:|---|---:|---:|---:|---:|
| Forward+ | 4x | Weak | 25.031 ms | 98% | 33% | 1608 |
| Forward+ | 4x | Strong | 26.343 ms | 98% | 33% | 1674 |
| Forward | 4x | Weak | 22.529 ms | 97% | 35% | 1495 |
| Forward | 4x | Strong | 23.219 ms | 98% | 35% | 1520 |
| Forward+ | 2x | Weak | 16.724 ms | 97% | 42% | 972 |
| Forward+ | 2x | Strong | 17.056 ms | 97% | 39% | 1017 |
| Forward | 2x | Weak | 14.299 ms | 97% | 43% | 486 |
| Forward | 2x | Strong | 15.113 ms | 97% | 40% | 531 |

Forward+ / MSAA 4xとの差：

| 変更 | Weak差 | Weak改善率 | Strong差 | Strong改善率 |
|---|---:|---:|---:|---:|
| Forward+ → Forwardのみ | -2.364 ms | 9.8% | -2.044 ms | 8.3% |
| MSAA 4x → 2xのみ | -8.549 ms | 35.3% | -8.296 ms | 33.7% |
| Forward + MSAA 2x | -10.433 ms | 43.1% | -10.326 ms | 41.9% |

各変更の効果は相互作用を含むため、単純には加算できない。

計測ログの条件対応：

| Rendering Path | MSAA | 条件 | CSV |
|---|---:|---|---|
| Forward | 4x | Weak | `20260818_221956.csv` |
| Forward | 4x | Strong | `20260818_221845.csv` |
| Forward+ | 2x | Weak | `20260818_223221.csv` |
| Forward+ | 2x | Strong | `20260818_223332.csv` |
| Forward | 2x | Weak | `20260818_215914.csv` |
| Forward | 2x | Strong | `20260818_220027.csv` |

Forward / MSAA 4xは、Weakを13.001〜57.005秒、Strongを4.000〜48.003秒で集計した。各45 samples、GPU frequency medianは525 MHz、Battery temperature medianは26℃であった。

Display refresh rateは72 Hzで、1 frameのbudgetは約13.89 msである。Forward / MSAA 2xのApp GPU Time medianは、Weakが13.796 msでbudget内、Strongが14.323 msでわずかに超過した。

### Summary

- Blurコード統合により、App GPU Time medianはBaseline比でWeak 12.2%、Strong 16.9%改善した。
- Forwardへの変更だけで、Forward+ / MSAA 4x比のApp GPU Time medianが約8〜10%改善した。
- MSAA 2xへの変更だけで約34〜35%、Forward + MSAA 2xでは約42〜43%改善した。
- 72 Hzのframe budgetは約13.89 msである。

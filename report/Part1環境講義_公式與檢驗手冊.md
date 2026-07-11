# Part 1 環境講義 — 公式、出處與檢驗手冊

> 對象：FR3_BeamMgmt 專題（`D:\FR3_BeamMgmt`）Part 1 環境層。
> 每一節固定四段：**①這是什麼（白話）→ ②完整公式（LaTeX）→ ③出處（論文+庫內路徑）→ ④怎麼檢驗**。
> 規格唯一真相來源：`D:\FR3_BeamMgmt\SPEC.md`；決策history：`D:\FR3_BeamMgmt\docs\待討論與決策.md`。
> 最後更新：2026-07-12（Episode 改雙層：穿越制自然終止＋4000 保險絲，去除硬拗公式）

---

## 0. 系統參數與符號表

| 符號 | 意義 | 值 | 程式位置 |
|---|---|---|---|
| $f_c$ | 載波頻率 | 16.95 GHz | `config.m` |
| $B$ | 頻寬 | 100 MHz | `config.m` |
| $\lambda$ | 波長 | $c/f_c \approx 17.7$ mm | 衍生 |
| $N_t$ | 天線數（UPA $16\times 8$） | 128 | `config.m` |
| $P_{tx}$ | 發射功率 | 30 dBm | `config.m` |
| $N_0$ | 雜訊功率 | −109 dBm | `config.m` |
| $\Delta t$ | 1 TTI | 10 ms | `config.m` |
| $d_{cor}$ | 陰影去相關距離 | 10 m | `config.m` |

---

## 0½. 邏輯 ↔ MATLAB 檔案總覽（快速查找）

| 邏輯                        | 程式檔                                    | 講義節        |
| ------------------------- | -------------------------------------- | ---------- |
| 場景地圖                      | `env/sceneGeometry.m`                  | §1         |
| LOS/NLOS 判定               | `env/isLos.m`                          | §2         |
| 路損/陰影/多路徑增益相位             | `env/channelModelV2.m`                 | §3         |
| 散射點生成                     | `env/generateScatterers.m`             | §4         |
| 導向向量 / DFT 碼本             | `env/upaArray.m` / `env/dftCodebook.m` | §5         |
| echo 功率與 SNR              | `env/echoSignalV2.m`                   | §6         |
| CRLB                      | `env/computeCrlb.m`                    | §7         |
| 連線判定                      | `env/linkSuccess.m`                    | §8         |
| 通訊 SNR / 吞吐量 / RLF / 獎勵組裝 | `env/myStep.m`（**總裝線**）                | §8–10      |
| 車輛移動                      | `env/vehicleMobility.m`                | §11        |
| Episode 初始化               | `env/myReset.m`                        | §9–11      |
| 觀測向量（9 維）                 | `env/buildObs.m`                       | Part 2 重設計 |
| 全部參數                      | `config.m`                             | §0         |

> 各節內另有「②½ 程式對應」表標到行號/區塊。行號以 2026-07-04 版為準，日後會漂移——認「檔案+區塊註解」比認行號穩。

---

## 1. 場景幾何（sceneGeometry.m）

### ① 這是什麼
一張 255×165 m 的曼哈頓式地圖：3 條 15 m 寬街道（主街 y=82.5、直街 x=82.5 / 172.5）、6 棟 75×75×20 m 街廓大樓、gNB 固定於 (127.5, 90, 10)（街廓 B 南牆、主街中央）、車高 1.5 m、6 個進入點。**所有幾何的唯一真相來源**——LOS 判定、散射點、車輛移動全部從這裡取值，避免多處定義不同步。

### ② 公式與 Episode 定義（雙層，2026-07-12 修正）
幾何本身無公式；關鍵是 **episode 怎麼結束**。分兩層講清楚各自角色：

**主終止 = 穿越制（真正在用的）**：
$$
\text{episode 結束} \iff \text{車駛離場景 (is\_out)}
$$
變動長度、平均 ~1250 TTI。**出處**：軌跡驅動 episode 為 2023–2025 主流——[[notes/Zhao2024_IBTD_ISAC_V2V]]（episode=車輛運動序列 起點→終點）、Ye&Ge Nature 2023（接近模擬邊界即提早終止，計畫書 ref[7]）、arXiv 2511.02260（2025，軌跡自然結束=駛離範圍）；Lane-change RL「軌跡結束 or 逾時」雙層。

**安全上限 = 4000 TTI（工程保險絲，非物理量）**：防移動狀態機 bug / 未來改碼 / 新速度導致無限繞圈。蒙地卡羅 10000 episodes（30km/h，最慢=最長）確認最長合法 episode 是 **3206 TTI 硬邊界**（99.9th=max=3206、0 次突破；2 路口、轉彎狀態用完必離場 → 路徑集合幾何有界），4000 留餘裕、實質永不觸發。

> ⚠️ **誠實附註**：直線估算 $255\text{m}/(30/3.6)/0.01 \approx 3060$ TTI 只是量級 sanity check，且**低估**（實測 3206>3060，忽略轉彎額外路程+減速 0.6 倍）——正是它**不可**當上限的理由。舊版曾把它硬包裝成「上限推導」，2026-07-12 依 playbook 60「不硬拗」原則修正為雙層。

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| 全部幾何（大樓/道路/gNB/進入點/車高） | `env/sceneGeometry.m` | 全檔（單一真相來源） |
| 主終止 `is_out`（穿越制） | `env/vehicleMobility.m` | 駛離場景旗標 |
| 安全上限 $CAP=4000$（保險絲、非物理） | `config.m` | `EPISODE_CAP_TTI` |
| 實測驗證（各速度時長 + 尾巴有界） | `analyzeEpisodeLength.m` | 1000/10000 episodes 蒙地卡羅 |
![[gnb_side_view 1.png]]
![[Pasted image 20260709051743.png]]
### ③ 出處
- 拓樸概念：WINNER II B1（曼哈頓都會微蜂窩）；gNB 10 m / 車 1.5 m：3GPP TR 38.901 Table 7.2-1
- **公尺數為設計選擇**（查證過 ITU-R M.2135 UMi 實為六角形 ISD 200 m，非 75/15；見 [[../FR3_BeamMgmt/SPEC.md|SPEC 1.0]] 修正記錄）
- **gNB 高度 10m < 樓高 20m 之設計，已查證符合 3GPP 官方規格（2026-07-07 查證）**：3GPP TR 38.901 UMi 街道峽谷情境明定「基地台安裝於周圍建築物屋頂以下（below rooftop levels），典型高度 10 公尺」。這正是「微蜂窩（microcell）」的標準部署方式——區別於基地台高於屋頂、覆蓋範圍達數公里的「宏蜂窩（macrocell，典型高度 25–35m）」。UMi 情境刻意讓 gNB 低於樓高，目的是模擬密集都會短距離部署（適配 FR3 等高頻段），LOS/NLOS 交替遮蔽是此部署方式下的**預期物理現象**，非場景設計缺陷。
- **在地法規佐證（第二層依據）**：台灣 NCC（國家通訊傳播委員會）對微型基地台（Micro Cell）的天線高度規範同樣落在約 3–10 公尺（常掛於外牆或路燈），與本場景 gNB=10m 一致；大型基地台（Macro Cell，天線約 20–45m、架設於建築頂樓）則對應宏蜂窩，非本場景採用的部署型態。此為國際標準（3GPP）之外的獨立在地法規交叉驗證。

### ④ 檢驗
```matlab
runtests('tests/testSceneGeometry.m')   % 5 測試：尺寸/gNB/大樓/進入點/樓不壓路
s = sceneGeometry(); % 畫圖目檢：大樓不壓路、gNB 在主街北牆、進入點在路口
```
**物理問句**：大樓有沒有蓋到馬路上？gNB 位置合理嗎？

---

## 2. LOS/NLOS 幾何遮蔽（isLos.m）

### ① 這是什麼
判斷「車看不看得到 gNB」==判斷有沒有被大樓擋住==：把 gNB→車的 2D 連線當一根線段，只要穿過任何一棟大樓的 footprint 矩形就是 NLOS。取代舊的距離門檻 `d<80`（距離門檻完全模擬不出轉角遮蔽）。

### ② 公式
Bai–Vaze–Heath 遮蔽模型：連線 $\ell(\mathbf{x}_{gNB}, \mathbf{x}_{veh})$ 與建築集合 $\mathcal{B}$：

$$
\text{LOS} = \mathbb{1}\!\left\{ \ell(\mathbf{x}_{gNB}, \mathbf{x}_{veh}) \cap b = \varnothing,\ \forall b \in \mathcal{B} \right\}
$$

實作用 Liang–Barsky 線段-矩形裁剪：線段參數化 $\mathbf{p}(t) = \mathbf{p}_0 + t(\mathbf{p}_1-\mathbf{p}_0),\ t\in[0,1]$，對矩形四邊解出 $[t_{in}, t_{out}]$，若 $t_{in} < t_{out}$（內部交集長度 > 0）即被擋。

2D 簡化的合法性：gNB 高 10 m、車高 1.5 m 皆 **低於樓高 20 m** → 2D footprint 穿越 ⟺ 3D 穿越樓體。

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| LOS 判定主函式 | `env/isLos.m` | 主體（逐棟大樓檢查） |
| Liang–Barsky 線段-矩形裁剪 | `env/isLos.m` | 內部函式 `segIntersectsRect` |
| 被誰呼叫 | `env/channelModelV2.m`（每步判 LOS）、檢視腳本 | — |
![[gnb_3d_view.png|688]]

### ③ 出處
- **Bai, Vaze, Heath, "Analysis of Blockage Effects on Urban Cellular Networks," IEEE TWC 2014**（arXiv 1309.4141，遮蔽模型奠基）
- GEMV²：Boban, Barros, Tonguz, IEEE TVT 2014（車聯網以建築輪廓分 LOS/NLOS）— 見 [[../FR3_BeamMgmt/SPEC.md|SPEC 1.1]]
- 實測佐證轉角效應：哥倫比亞大學曼哈頓 28GHz 量測（轉角 20 m 內掉 >20 dB）

### ④ 檢驗
```matlab
runtests('tests/testIsLos.m')   % 3 測試
% LOS 地圖（最有感）：
[X,Y]=meshgrid(0:2:255,0:2:165);
L=arrayfun(@(x,y) isLos([x y 1.5],s),X,Y);
imagesc(0:2:255,0:2:165,L); axis xy equal
```
**物理問句**：亮區是否為「從 gNB 放射的視線走廊」？有無隔樓卻亮的怪點？

---

## 3. 通道模型（channelModelV2.m）— Part 1 的靈魂

### ① 這是什麼
給定車的位置，產生「多路徑通道」：LOS 一條直達路徑 + NLOS 最多 3 條牆面散射路徑，每條有複數增益（大小=路損+陰影、相位=幾何路程）與離開角。

### ② 完整公式

**(a) CI 路徑損耗（close-in reference）**：

$$
\mathrm{PL}^{CI}(d) = \underbrace{32.4 + 20\log_{10}(f_c/\text{1GHz})}_{\mathrm{FSPL}(1\,\text{m})} + 10\,n\,\log_{10}(d) + X_\sigma \quad [\text{dB}]
$$

FR3 實測參數（Shakya2025, 16.95 GHz, Table III）：

$$
n_{LOS}=1.85,\ \sigma_{LOS}=4.05\ \text{dB};\qquad n_{NLOS}=2.59,\ \sigma_{NLOS}=8.78\ \text{dB}
$$

**(b) 陰影衰落空間相關（Gudmundson 1991 + AR(1) 更新）**：相關係數隨移動距離指數衰減

$$
\rho = \exp\!\left(-\frac{\Delta d}{d_{cor}}\right),\qquad
X_\sigma[t] = \rho\, X_\sigma[t-1] + \sqrt{1-\rho^2}\,\sigma\,\varepsilon,\quad \varepsilon\sim\mathcal{N}(0,1)
$$

$\Delta d$ = 本 TTI 實際位移（不是固定值！靜止時 $\rho=1$ 陰影不變）；$d_{cor}=10$ m（都會 FR3，查證三輪的結果：3GPP 38.901 都會 10–13 m + 去相關距離隨頻率遞減）。

**(c) LOS 路徑複增益**（幾何相位，隨車連續變化 → 都卜勒自然浮現）：

$$
g_{LOS} = \sqrt{10^{-\mathrm{PL}_{LOS}/10}}\; e^{-j 2\pi d_{3D}/\lambda}
$$

**(d) NLOS 散射路徑**（top-3 最近牆面散射點，FR3 稀疏）：

$$
\mathrm{PL}_{NLOS} = \mathrm{FSPL}(1\,\text{m}) + 10\, n_{NLOS} \log_{10}(d_1 + d_2) + L_{refl} + X_\sigma
$$

$$
g_s = \sqrt{10^{-\mathrm{PL}_{NLOS}/10}}\; e^{-j 2\pi (d_1+d_2)/\lambda}
$$

其中 $d_1$=gNB→散射點、$d_2$=散射點→車、$L_{refl}=6$ dB 反射損。

> ⚠️ **重要修正記錄（2026-07-04）**：曾誤寫成兩段各算一次 CI（$\mathrm{FSPL}+10n\log d_1 + \mathrm{FSPL}+10n\log d_2$），使 NLOS 總損 ~200 dB 形成「側街死域」。**CI 的 $n_{NLOS}=2.59$ 是實測「總損耗 vs 距離」的擬合值，反射代價已含其中**，故只能對總路程 $d_1{+}d_2$ 算一次。此 bug 由「動畫中 NLOS 全斷」的質疑觸發，oracle 全碼本掃描實驗證實後修正。

**(e) 合成通道向量**（供波束成形）：

$$
\mathbf{h} = \sum_{p} g_p\, \mathbf{a}(\theta_p, \phi_p) \in \mathbb{C}^{N_t}
$$

### ②½ 程式對應（`env/channelModelV2.m`，2026-07-04 版行號；行號會漂移，以區塊註解為準）
| 公式 | 位置 |
|---|---|
| (a) $\mathrm{FSPL}(1\text{m})$ | 第 33 行 `FSPL_1m = 32.4 + 20*log10(fc/1e9)` |
| (a) CI 參數 $n,\sigma$ | `config.m` §Channel（`nLos/sigmaLos/nNlos/sigmaNlos`，Shakya2025 值） |
| (b) 陰影 AR(1) | 第 38–45 行（`rho=exp(-d_move/dCor)`；$d_{cor}$ 在 `config.m` `dCor`） |
| (c) LOS 增益+幾何相位 | 第 50–58 行 |
| (d) NLOS 單斜率+相位（含修正註解） | 第 60–74 行（top-3 選擇在 61–63） |
| (e) 合成 $\mathbf{h}=\sum g_p \mathbf{a}_p$ | `env/myStep.m`（波束套用段，逐路徑疊加後與 $\mathbf{f}$ 內積） |

### ③ 出處
- PLE/σ 實測：[[notes/Shakya2025_UrbanPropagation]]（原文：`D:\PROJECTS\papers\核心論文\Shakya2025_UrbanPropagation\`）
- 陰影相關：Gudmundson, *Electronics Letters* 1991（奠基）；$d_{cor}$ 佐證 3GPP TR 38.901 Table 7.5-6
- 幾何相位/都卜勒：Clarke 1968 / Jakes 1974；GBSM 幾何隨機通道建模
- FR3 通道背景：[[notes/FR3_Rel19_ChannelModeling]]

### ④ 檢驗
```matlab
runtests('tests/testChannelModelV2.m')  % 5 測試：幾何LOS/陰影靜止/相位決定性/top-3/NLOS單斜率
% 手動：沿主街掃 x，畫路徑增益曲線 → 應「近 gNB 高、遠端低、平滑無跳崖」
```
**物理問句**：SNR 曲線平滑嗎？LOS↔NLOS 切換發生在幾何遮蔽點嗎？

---

## 4. 散射點（generateScatterers.m）

### ① 這是什麼
NLOS 的反射從哪來——散射點**貼在大樓面向街道的牆面上**（每面牆取樣 5 點、向外推 0.5 m、落在路廊內才保留），**決定性生成（無 rand）**。決定性是相位連續 $\varphi = 2\pi(d_1+d_2)/\lambda$ 的前提：散射點若每步亂跳，相位就斷了。

### ② 公式
牆面取樣點 $\mathbf{s} = \mathbf{w}(u) + 0.5\,\hat{\mathbf{n}}$（$u \in \{0.1,\dots,0.9\}$ 牆面參數、$\hat{\mathbf{n}}$ 外法向），保留條件 $\mathbf{s}\in\mathcal{R}$（道路走廊）。通道只取距車最近的 top-3：

$$
\mathcal{S}_{act} = \underset{|\mathcal{S}'|=3}{\arg\min} \sum_{\mathbf{s}\in\mathcal{S}'} \lVert \mathbf{s}-\mathbf{x}_{veh}\rVert
$$

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| 牆面取樣+外推 0.5m+落路保留 | `env/generateScatterers.m` | 主迴圈（每牆 5 點、linspace 0.1–0.9 避角） |
| 路廊判定 | `env/generateScatterers.m` | 內部函式 `pointInRoads` |
| top-3 最近選擇 | `env/channelModelV2.m` | 第 61–63 行（sort 距離取前 3） |

### ③ 出處
- FR3 稀疏（少數主導 MPC、ASA 15.3°）：[[notes/Shakya2025_UrbanPropagation]]
- 群集數變少趨勢：[[notes/FR3_Rel19_ChannelModeling]]
- 牆面散射（GBSM 慣例）：幾何隨機通道模型標準做法

### ④ 檢驗
```matlab
runtests('tests/testGenerateScatterers.m')  % 4 測試：格式/決定性/貼牆/面向道路
sc=generateScatterers(s); % 畫圖：紅點應全貼牆、不在樓內/路中央；跑兩次結果相同
```

---

## 5. 波束成形基礎（upaArray.m + dftCodebook.m）

### ① 這是什麼
gNB 用 128 天線把能量「聚成一根手電筒」。導向向量 $\mathbf{a}$ 描述「訊號從某方向來/去時，各天線收到的相位差」；DFT 碼本 $\mathbf{F}$ 是一組預先造好的手電筒（每束指向一個固定方向），波束管理=從碼本裡挑一束。

### ② 公式

**UPA 導向向量**（$M\times N$ 陣列、間距 $d=\lambda/2$）：

$$
[\mathbf{a}(\theta,\phi)]_{m,n} = \exp\!\Big( j\frac{2\pi d}{\lambda}\big( m\sin\theta_{el} + n\cos\theta_{el}\sin\theta_{az} \big) \Big)
$$

**碼本波束成形增益**（整個專題最核心的量）：

$$
G(\theta) = |\mathbf{a}(\theta)^H \mathbf{f}|^2, \qquad 0 \le G \le N_t
$$

對準（$\mathbf{f} = \mathbf{a}/\sqrt{N_t}$）時 $G=N_t=128$（21 dB）；失配時暴跌——窄波束讓 $G$ 對角度誤差極敏感，**這就是 FR3 波束管理問題的數學根源**。

> ⚠️ **現況與待辦（Part 1 暫定）**：目前碼本 `dftCodebook(16,8)` = **128 束（方位角 8 × 仰角 16）**，方位角間隔約 17°；Part 2 T2.0 將做 3GPP Type I 過取樣（O1=4 → 方位角 32 束 @4.3°）。另有待決問題 Q6：方位角僅 8 元素而 V2X 追蹤關鍵在方位角，天線配置合理性待 Part 2 正面處理。

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| 導向向量 $\mathbf{a}(\theta,\phi)$ | `env/upaArray.m` | 全檔 |
| DFT 碼本 $\mathbf{F}$（128 束 = 方位 8 × 仰角 16） | `env/dftCodebook.m` | azList/elList linspace + 逐束呼叫 upaArray |
| $G=\lvert\mathbf{a}^H\mathbf{f}\rvert^2$ 實際計算 | `env/myStep.m`（通訊）、`env/echoSignalV2.m`（感測） | 內積後取 `abs()^2` |

### ③ 出處
- 陣列訊號處理標準（頻率無關）；任何陣列教科書（如 Van Trees, *Optimum Array Processing*）
- 3GPP Type I codebook 概念（Part 2 將做過取樣）

### ④ 檢驗
```matlab
[F,~,~]=dftCodebook(16,8,fc,dSpacing);
norm(F(:,1))                      % 每束單位功率 → 1
a=upaArray(fc,30,0,16,8,dSpacing);
plot(abs(F'*a).^2)                % 一根主峰=只有指向 30° 的波束增益高
```

---

## 6. ISAC 感測回波（echoSignalV2.m）

### ① 這是什麼
gNB 發射的波束打到車、反彈回 gNB 感測接收機。回波強度隨「波束對準程度」變化 → gNB **不需要車回報**就能感覺波束對不對——這是論文「免回饋感測輔助」賣點的物理基礎。

### ② 完整公式

**理想回波功率**（單站雷達，數位接收對準真實回波角）：

$$
P_{echo} = P_{tx}\, |\mathbf{a}^H \mathbf{f}|^2\, \beta^2, \qquad \beta^2 \propto \frac{\sigma_{RCS}}{d^4}
$$

- $|\mathbf{a}^H\mathbf{f}|^2$：**一個**發射波束對準因子（接收為數位全陣列、增益固定 → 不是 $|\cdot|^4$）
- $1/d^4$：雷達雙向路損（去程 $1/d^2$ × 回程 $1/d^2$），程式中即 $40\log_{10} d$
- $\sigma_{RCS} = 25\ \text{m}^2$：車輛雷達散射截面（光學區 >1–3 GHz 與頻率無關 → FR3 直接適用）

**匹配濾波後回波 SNR**：

$$
\mathrm{SNR}_{echo} = \frac{G_{mf}\, |\mathbf{a}^H \mathbf{f}|^2\, P_{tx}\, \beta^2}{N_0}, \qquad G_{mf}=10
$$

**量測抖動**：實際量測值 $\hat{P}_{echo} = P_{echo}\cdot 10^{\xi/10}$，$\xi\sim\mathcal{N}(0, 1\,\text{dB}^2)$。

> ⚠️ **修正記錄**：$P_{echo}$ 曾誤寫 $\propto|\mathbf{a}^H\mathbf{f}|^4$（誤設同一類比波束收發）。兩篇核心論文皆為數位接收 → 一個波束因子。修正使 $P_{echo}$ 與 $\mathrm{SNR}_{echo}$ 同次方（測試 `testEchoSignalV2` 以「比值不隨波束變」鎖住此性質）。

### ②½ 程式對應（`env/echoSignalV2.m`）
| 公式 | 位置 |
|---|---|
| $P_{echo}=P_{tx}\lvert\mathbf{a}^H\mathbf{f}\rvert^2\beta^2$（一次方，修正處） | 第 34–37 行 `P_echo_ideal = ptx * bfGain * plLin`（含修正註解） |
| $\mathrm{SNR}_{echo}$（匹配濾波 $G_{mf}$） | 第 38 行 `snrEchoIdeal = matchedGain * bfGain * ...` |
| $1/d^4$ 雙向路損 | 同檔 plLin 計算（`40*log10(d)` 形式） |
| 量測抖動 $\xi\sim\mathcal{N}(0,1\text{dB}^2)$ | 同檔（`measStdDb` 乘 randn）；參數 `config.m` `MEAS_NOISE_STD_DB` |
| $\sigma_{RCS},G_{mf}$ 參數 | `config.m` §Sensing（`MATCHED_GAIN_G`、`MEAS_A_COEF`） |

### ③ 出處
- **MultiUser Eq.6/8b/10**（$\beta\propto\sqrt{\sigma_{rcs}/d^4}$、發射增益 $|\mathbf{a}^H\mathbf{f}|^2$）：[[notes/MultiUserBeamforming_DRL_Sensing]]（原文：`D:\PROJECTS\papers\核心論文\MultiUserBeamforming_DRL_Sensing\`）
- **Zhao2024 Eq.2/6**（數位接收 $b(\theta)a^H(\theta)f$ + 匹配濾波）：[[notes/Zhao2024_IBTD_ISAC_V2V]]
- 雷達方程式基礎：Skolnik, *Introduction to Radar Systems*
- FR3 接軌：Multi-Band Sensing in FR3（arXiv 2602.23539，1/d⁴）

### ④ 檢驗
```matlab
runtests('tests/testEchoSignalV2.m')  % 2 測試：P∝|a^Hf|² 一次方、P/SNR 同次方
% 手動：對固定車位掃全碼本 → echo 應單峰（對準最強）
```

---

## 7. CRLB 角度估測下界（computeCrlb.m）

### ① 這是什麼
「用這麼強的回波，角度最準能估到多準」的理論極限（Cramér–Rao 下界）。回波越強 → CRLB 越小 → gNB 越「看得清」車。DRL 用它當「我還追得住嗎」的信心指標。

### ② 公式
簡化比例式（由**量測** echo SNR 推得，不偷看真實角度）：

$$
\mathrm{CRLB}(\hat\theta) = \frac{a_1}{N_t \cdot \widehat{\mathrm{SNR}}_{echo}} \quad [\text{rad}^2], \qquad a_1 = 1
$$

因 $\mathrm{SNR}_{echo} \propto \mathrm{SNR}_{base}\,|\mathbf{a}^H\mathbf{f}|^2$，故

$$
\mathrm{CRLB} \propto \frac{1}{N_t\, \mathrm{SNR}_{base}\, |\mathbf{a}^H\mathbf{f}|^2}
$$

正是計畫書 Eq.5 的形式。感測門檻 $\Gamma_{CRLB} = 3\times10^{-3}\ \text{rad}^2 \approx (\text{HPBW}/2)^2$——CRLB 超過它 ⟺ 角度誤差比半波束還寬 ⟺ 實質失明。

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| $\mathrm{CRLB}=a_1/(N_t\cdot\widehat{\mathrm{SNR}}_{echo})$ | `env/computeCrlb.m` | 第 15 行 `crlb = aCoef/(nAnt*max(snrEchoMeas,1e-12))` |
| 數值夾限 $[10^{-9},10^6]$ | `env/computeCrlb.m` | 第 16 行 |
| 門檻 $\Gamma_{CRLB}=3\times10^{-3}$ | `config.m` | `GAMMA_CRLB` |
| 被誰呼叫 | `env/myStep.m`（每步由量測 echo SNR 算） | — |

### ③ 出處
- **Stoica & Nehorai, IEEE Trans. ASSP 1989**（DOA CRB 奠基，2879 引用）：CRB $\propto 1/(N\cdot\mathrm{SNR})$
- 形式對齊：[[notes/MultiUserBeamforming_DRL_Sensing]] Eq.10、[[notes/Zhao2024_IBTD_ISAC_V2V]]（估測誤差 $\propto a_i/(G\cdot\mathrm{SNR})$）
- FR3 CRB 分析：ISAC meets FR3（arXiv 2605.18120 Fig.3）

### ④ 檢驗
```matlab
% CRLB 應與 echo 完全反向：掃全碼本畫 CRLB → 對準時谷底、失配時峰
% testFR3Env/testEchoSensing 斷言「對準 CRLB < 失配 CRLB」與「CRLB-SNR 相關 < -0.9」
```

---

## 8. 通訊 SNR 與連線判定（myStep.m + linkSuccess.m）

### ① 這是什麼
車收到的實際訊號品質，以及「這一刻能不能傳資料」的判定。注意 echo（感測）**不是**連線閘門——它是預測器；連線看的是通訊 SNR。

### ② 公式

**通訊 SNR**（等效通道 = 通道向量投影到所選波束）：

$$
\mathrm{SNR} = \frac{P_{tx}\,\big|\mathbf{h}^H \mathbf{f}\big|^2}{N_0}\cdot 10^{-L_{sw}/10}
$$

$L_{sw}=0.5$ dB 換波束瞬間損耗（換束後 1 TTI 生效——防「瞬移作弊」）。

**連線成功（outage 標準定義）**：

$$
\text{link} = \mathbb{1}\!\left\{ \log_2(1+\mathrm{SNR}) \ge C_{th} \right\},\qquad C_{th} = 4\ \text{bps/Hz} \Leftrightarrow \mathrm{SNR} \ge 15\ (11.8\ \text{dB})
$$

懸崖式門檻：過線即成功、差一點即失敗，無部分分數。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| 通訊 SNR $=P_{tx}\lvert\mathbf{h}^H\mathbf{f}\rvert^2/N_0$、切換損 $L_{sw}$ | `env/myStep.m`（snrNow 計算段） |
| 連線判定 $\log_2(1+\mathrm{SNR})\ge C_{th}$ | `env/linkSuccess.m`（單行邏輯） |
| 門檻 $C_{th}=4$、$L_{sw}=0.5$ dB、延遲 1 TTI | `config.m`（`cThreshold`、`BEAM_SWITCH_LOSS_DB`、`BEAM_SWITCH_DELAY`） |

### ③ 出處
- Outage 定義奠基：Ozarow, Shamai, Wyner, IEEE TVT 1994；教科書 Goldsmith 2005 / Tse & Viswanath 2005
- $C=4$ bps/Hz：[[notes/MultiUserBeamforming_DRL_Sensing]] 參數表；計畫書 Eq.9

### ④ 檢驗
```matlab
runtests('tests/testLinkSuccess.m')   % 邊界：SNR=15→過、14→不過
linkSuccess(15,4), linkSuccess(14.9,4)
```

---

## 9. 吞吐量（full-buffer 速率式）

### ① 這是什麼
「這一步實際傳了多少資料」。full-buffer = 假設永遠有資料要傳（RL 訓練標準假設），送達量直接反映波束品質，不被人造的封包佇列動態干擾。

### ② 公式

$$
R_{eff} = (1-\rho_a)\, B \log_2(1+\mathrm{SNR})\cdot \mathbb{1}\{\text{link}\}, \qquad \text{served}[t] = R_{eff}\,\Delta t
$$

$\rho_a\in\{0.05, 0.25, 0.50\}$ = 動作 $a$ 的掃描開銷（掃描佔用時頻資源 → 直接砍吞吐——「掃太多會虧」的機制來源）。

次要指標（給導師看封包/可靠度面，不入獎勵）：累積送達 $\sum \text{served}$、連線成功率 $\frac{1}{T}\sum \mathbb{1}\{\text{link}\}$。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| $R_{eff}=(1-\rho)B\log_2(1+\mathrm{SNR})\cdot\mathbb{1}\{\text{link}\}$、served 累積 | `env/myStep.m`（Throughput 區塊） |
| $\rho_a$ 開銷表 | `config.m`（`rhoTable = [0.05 0.25 0.50]`） |
| 次要指標初始化（served_cum/link_success_cum） | `env/myReset.m` |

### ③ 出處
- 波束追蹤 DRL 標準獎勵=SE/速率式：EURASIP JWCN 2022（DOI 10.1186/s13638-022-02191-7）；Ye & Ge 2023, *Nature Sci Reports*（計畫書 ref[7]）
- full-buffer 慣例：arXiv 1611.06497；計畫書 Eq.9/11

### ④ 檢驗
```matlab
runtests('tests/testThroughput.m')  % served_cum ≡ Σ R_eff·dt（不變量守門）
```

---

## 10. RLF 與 Episode（不終止、穿越制）

### ① 這是什麼
Episode = 車穿越場景的全程（**駛離才結束**＝主終止；4000 TTI 只是保險絲上限、非物理量、實質永不觸發，詳見 §1）。連線再爛也**不提前終止**——否則 agent 會學「躲 NLOS」而不是「進 NLOS 後怎麼救」。斷線只記指標。

### ② 公式

$$
\text{rlf\_counter}[t] = \begin{cases} \text{rlf\_counter}[t-1]+1 & \text{link}=0\\ 0 & \text{link}=1 \end{cases}
$$

$$
\text{RLF 事件} \mathrel{+}= \mathbb{1}\{\text{rlf\_counter} = T_{310}\},\qquad T_{310} = 100\ \text{TTI} = 1\ \text{s}
$$

追蹤指標：最長連續斷線 $\max_t \text{rlf\_counter}[t]$、RLF 事件數。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| rlf_counter / longest_outage / rlf_events 更新 | `env/myStep.m`（RLF 指標區塊，不終止） |
| Episode 上限判定 `t >= EPISODE_CAP_TTI` | `env/myStep.m`（末段） |
| 車駛離場景 → isDone | `env/vehicleMobility.m`（is_out）→ `env/myStep.m` |
| $T_{310}=100$、安全上限 4000 | `config.m`（`RLF_THRESHOLD_TTI`、`EPISODE_CAP_TTI`） |

### ③ 出處
- T310≈1s：3GPP TR 38.331
- 「不用 RLF 終止、用吞吐+可靠度指標」：[[notes/MultiUserBeamforming_DRL_Sensing]]、[[notes/Zhao2024_IBTD_ISAC_V2V]]、Ye & Ge 2023 慣例一致

### ④ 檢驗
```matlab
runtests('tests/testRlfEpisode.m')  % 靜止深NLOS 120步不終止；安全上限=4000（>3206 硬邊界）
```

---

## 11. 車輛移動（vehicleMobility.m）

### ① 這是什麼
Manhattan 移動模型：車沿路走、路口擲骰子（直行 0.5 / 左 0.25 / 右 0.25）、轉彎走圓弧減速、直行吸附車道中心線、駛離場景回報 episode 結束。

### ② 公式
位置更新（heading $\psi$、速度 $v$）：

$$
\mathbf{x}[t+1] = \mathbf{x}[t] + v\,\Delta t\,[\cos\psi, \sin\psi, 0]
$$

轉彎：半徑 $r=6$ m 圓弧、減速因子 0.6，每步轉角 $\Delta\psi = \dfrac{0.6\,v\,\Delta t}{r}$。速度集合 $\{30,45,60\}$ km/h（每 episode 抽定）。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| 位置更新、路口轉向決策、轉彎弧、車道吸附、駛離判定 | `env/vehicleMobility.m`（全檔；道路座標內部讀 `sceneGeometry()`） |
| 速度集合/轉向機率 | `config.m`（`vRange`、`pStraight/pLeft/pRight`） |
| 進入點抽選與初始 heading | `env/myReset.m` |

### ③ 出處
- Manhattan Mobility Model（V2X 模擬標準，SUMO 慣例）
- 路口轉向機率：[[notes/MultiUserBeamforming_DRL_Sensing]]；計畫書一致

### ④ 檢驗
```matlab
runtests('tests/testVehicleMobility.m')  % 5 測試：吸附/前進/駛離/進入點/gNB固定
% 手動：跑 3 個 seed 畫軌跡 → 全程貼路、轉彎是弧
```

---

## 12. 總體檢（單元全過之後）

| 檢驗 | 指令 | 通過標準 |
|---|---|---|
| 行為契約 | `runAllTests` | 37/37 綠 |
| 合理策略體檢 | `inspectEnv`（週期掃描 P=15） | 連線 ~90%+、SNR 僅轉彎後掉 |
| 因果鏈目檢 | `liveInspectEnv`（動畫） | 變紅→SNR 掉→斷線 同步發生 |
| 成績階梯 | 亂猜 ~50% < 週期 83–94% < oracle 97.7% | 分數隨策略聰明度單調上升 |
| 對照組基準 | `periodicSweepSim(20)` | P=15：93.7%、1003 Mbps、ρ=0.080 |

**成績階梯的意義**：oracle（每步全掃碼本全部 128 束）97.7% = 環境物理天花板；它與週期掃描之間 ~15 個百分點的縫 = Part 2 DRL 的研究空間。

### 檢驗心法（三層）
1. **它說到做到嗎** → 跑對應測試（測試=行為契約）
2. **輸出長得像物理嗎** → 畫圖找「不該存在的形狀」（跳崖、穿牆、負功率）——NLOS 死域 bug 就是這層抓到的
3. **改壞它測試會叫嗎** → 故意改錯一行跑測試看紅（對測試的信心來自看過它失敗）

---

## 附錄 A：對照組——今日 5G（demo/）

獨立教學檔 `demo/demoTraditional5G.m`、`demo/liveTraditional5G.m`（3.5 GHz n78、8 個 SSB 寬波束、20 ms 週期掃描，經 3GPP 協定查證：FR1 SSB 上限 $L=8$、SSB periodicity ∈ {5,10,20,40,80,160} ms）。

核心對照：同一張地圖、同一個轉彎——

| | 3.5 GHz（今日） | 16.95 GHz（FR3） |
|---|---|---|
| 轉角後 SNR | 50→20 dB（活著） | 崩到門檻下數十 dB |
| 波束行為 | 整趟換 ~5 次 | 波束索引不停滑動 |
| 波束管理 | 協定已解決（8 束+20ms 掃） | **需要智慧決策 = 本論文** |

頻率越高轉角越致命的實測佐證：NYU 140 GHz 都會量測（arXiv 2103.05496）。

## 附錄 B：庫內資源地圖（⭐ = 庫內真的有檔）

> **引用分兩類**：⭐ 庫內有原文+翻譯+筆記（wikilink 點得開）；🌐 外部文獻（庫內**無原檔**，只有引用條目——連結見附錄 C）。

| 資源 | 路徑 |
|---|---|
| 規格唯一真相 | `D:\FR3_BeamMgmt\SPEC.md` |
| 決策/修正歷史 | `D:\FR3_BeamMgmt\docs\待討論與決策.md` |
| ⭐ 核心論文筆記 | [[notes/MultiUserBeamforming_DRL_Sensing]]、[[notes/Shakya2025_UrbanPropagation]]、[[notes/FR3_Rel19_ChannelModeling]] |
| ⭐ 強化學習筆記 | [[notes/Zhao2024_IBTD_ISAC_V2V]]（原文在 `papers\強化學習\`）、[[notes/Salami2025_TransferRL_BeamSelection]]、[[notes/Wang2026_V2X_MARL_Benchmarking]] |
| ⭐ 論文原文/翻譯 | `D:\PROJECTS\papers\核心論文\` 與 `papers\強化學習\`（各 Slug 資料夾含 `*_專題修改建議.md`） |
| 論文總索引 | [[INDEX]] |
| ⏳ pipeline 待收錄 | ISAC_meets_FR3、RoadwayGeometry_ISAC_BeamTracking、BaiHeath2014_BlockageModel（INDEX 已排程，收錄前用附錄 C 外部連結） |

## 附錄 C：外部文獻連結表（庫內無原檔者）

| 文獻 | 用於 | 連結 / 取得方式 |
|---|---|---|
| Bai, Vaze, Heath, IEEE TWC 2014（遮蔽模型奠基） | §2 LOS | https://arxiv.org/abs/1309.4141 |
| Gudmundson 1991（陰影空間相關） | §3 通道 | DOI: 10.1049/el:19911328（IEEE Xplore / 學校圖書館） |
| Clarke 1968 / Jakes 1974（都卜勒奠基） | §3 通道 | 經典教科書級：Google Scholar 搜 "Clarke A statistical theory of mobile-radio reception" |
| Stoica & Nehorai, IEEE ASSP 1989（DOA CRB 奠基） | §7 CRLB | DOI: 10.1109/29.32276 |
| Ozarow, Shamai, Wyner, IEEE TVT 1994（outage 奠基） | §8 連線 | DOI: 10.1109/25.293665 |
| Goldsmith, *Wireless Communications* 2005 | §8 連線 | 教科書（Cambridge；學校圖書館） |
| Tse & Viswanath, *Fundamentals of Wireless Communication* 2005 | §8 連線 | 免費官方版：https://web.stanford.edu/~dntse/wireless_book.html |
| Skolnik, *Introduction to Radar Systems* | §6 echo | 教科書 |
| GEMV²（Boban et al., IEEE TVT 2014，車聯網 LOS/NLOS） | §2 LOS | 官方網站 vehicle2x.net；Google Scholar "GEMV2 geometry-based vehicle-to-vehicle" |
| EURASIP JWCN 2022（波束訓練 SE/EE 獎勵） | §9 吞吐 | https://doi.org/10.1186/s13638-022-02191-7 |
| Ye & Ge 2023, Nature Sci Reports（V2V DRL 波束管理，計畫書 ref[7]） | §9/§10 | https://www.nature.com/articles/s41598-023-47769-3 |
| Multi-Band Sensing in FR3 | §6 echo | https://arxiv.org/abs/2602.23539（⏳ 待收錄） |
| ISAC meets FR3 | §7/§10 | https://arxiv.org/abs/2605.18120（⏳ 待收錄） |
| NYU 140GHz 都會量測（去相關/轉角 vs 頻率） | §3/附錄A | https://arxiv.org/abs/2103.05496 |
| full-buffer RL 慣例 | §9 吞吐 | https://arxiv.org/abs/1611.06497 |
| 3GPP TR 38.901 / 38.331 / 37.885 | §1/§10 | https://www.3gpp.org/specifications（免費下載） |
| 哥倫比亞大學曼哈頓 28GHz 轉角實測 | §2 | https://wimnet.ee.columbia.edu/ （INFOCOM'25 論文 PDF） |

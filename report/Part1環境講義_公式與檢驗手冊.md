# Part 1 環境講義 — 公式、出處與檢驗手冊

> 對象：FR3_BeamMgmt 專題（`D:\FR3_BeamMgmt`）Part 1 環境層。
> 每一節固定四段：**①這是什麼（白話）→ ②完整公式（LaTeX）→ ③出處（論文+庫內路徑）→ ④怎麼檢驗**。
> SPEC.md（**規格值記錄，非正確性依據**）：`D:\FR3_BeamMgmt\SPEC.md`；決策/修正歷史：`D:\FR3_BeamMgmt\docs\待討論與決策.md`。
> 最後更新：2026-07-17（§5 碼本取樣空間修正：方位角改正弦空間均勻 ✅）


---

## 📌 文獻驗證狀態

| 部分                      | 驗證方式               | 狀態     | 備註                                                      |     |
| ----------------------- | ------------------ | ------ | ------------------------------------------------------- | --- |
| **§3 通道模型**             | IEEE Xplore + 直讀原文 | ✅ 完全驗證 | Shakya2025 / Gudmundson 1991 / Clarke 1968 / Jakes 1974 |     |
| **§1–2（場景、LOS/NLOS）**   | SPEC.md + 3GPP 原文  | ✅ 無誤   | Bai-Heath 2014, 3GPP 38.901                             |     |
| **§5–8（波束/回波/CRLB/連線）** | 三核心論文 + 標準核對       | ✅ 完全驗證  | 公式形式與數值均已核實                                    |     |
| **§11 移動**              | —                  | ✅ 已查證  | 轉向機率出處已更正（設計選擇，非 MultiUser）                             |     |

**重點修正（2026-07-16，本輪環境參數三改）：**
- ✅ **反射損 L_refl：6 dB → 0**（已含於 Shakya n_NLOS 斜率，避免雙重計算）
- ✅ **雜訊 N₀：−109 → −86 dBm**（kTB+NF=8dB，3GPP TR 38.820）
- ✅ **連線門檻 C：4 → 主值 2、消融 {1,2,4}**（QoS 設計參數）

**早期修正（2026-07-12）：** Gudmundson DOI 10.1049/el:19911328、Clarke DOI 10.1002/j.1538-7305.1968.tb00069.x、Jakes ISBN 978-0471437208（均 IEEE Xplore 確認）。

---

## 0. 系統參數與符號表

> **這在幹嘛：** 一張速查表，把整個專題會用到的物理參數（頻率、頻寬、天線數、功率、雜訊…）和符號集中列出來，方便隨時對照。

| 符號 | 意義 | 值 | 程式位置 |
|---|---|---|---|
| $f_c$ | 載波頻率 | 16.95 GHz | `config.m` |
| $B$ | 頻寬 | 100 MHz | `config.m` |
| $\lambda$ | 波長 | $c/f_c \approx 17.7$ mm | 衍生 |
| $N_t$ | 天線數（UPA 方位16×仰角8；38.901 記法 $(M,N)=(8,16)$） | 128 | `config.m` |
| $P_{tx}$ | 發射功率 | 30 dBm | `config.m` |
| $N_0$ | 雜訊功率 | −86 dBm（$kTB$+NF, B=100MHz, NF=8dB）| `config.m` |
| $\Delta t$ | **波束決策週期**（程式沿用「TTI」字樣）| 10 ms | `config.m` |

> **⚠️ 命名澄清（2026-07-20）**：3GPP 的 TTI 是 **slot 級**（本專題 SCS 60 kHz → NR slot **0.25 ms**）。本專題的 $\Delta t$ 實為「**波束管理決策週期**」——每個 $\Delta t$ 做一次 RL 動作決策，屬 L3/RRM 時間尺度，與實體層 slot 非同一概念。
>
> **為何不能縮到 slot 粒度（物理必要條件）**：
> $$\Delta t > N_{beams}\cdot T_{sym} = 128 \times 17.84\,\mu s = 2.28\ \text{ms}$$
> 否則一次全掃塞不進單一決策週期 → $\rho = N_{swept}T_{sym}/\Delta t > 1$，開銷模型失去意義。取 10 ms 使 $\rho_{全掃}=0.228$，為合理工作點。
>
> **出源**：**[T1]** MultiUser (Wang2024) Table I 明載 $dt = 10$ ms（同款設定）；計畫書一致。
> **旁證**：實測最佳掃描週期 $P=15 \Rightarrow 150$ ms，與 Kanhere et al., IEEE VTC2021-Spring (arXiv:2103.03434) 28 GHz 車載實測之最佳掃描週期 $\approx 300$ ms 同數量級。
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

> **這在幹嘛：** 定義整個模擬世界的地圖——街道、大樓、基地台位置在哪，是後面所有計算（遮蔽、散射、移動）的唯一取值來源。

![[visualizeScene.png]]
*俯視圖：雙十字路口、6 棟街廓大樓（A–F）、gNB（紅星，主街中央）、6 個車輛進入點（E1–E6）。*

### ① 這是什麼
一張 255×165 m 的曼哈頓式地圖：3 條 15 m 寬街道（主街 y=82.5、直街 x=82.5 / 172.5）、6 棟 75×75×20 m 街廓大樓、gNB 固定於 (127.5, 90, 10)（街廓 B 南牆、主街中央）、車高 1.5 m、6 個進入點。**所有幾何的唯一真相來源**——LOS 判定、散射點、車輛移動全部從這裡取值，避免多處定義不同步。

### ② 公式與 Episode 定義（雙層，2026-07-12 修正）
幾何本身無公式；關鍵是 **episode 怎麼結束**。分兩層講清楚各自角色：

**主終止 = 穿越制（真正在用的）**：
$$
\text{episode 結束} \iff \text{車駛離場景 (is\_out)}
$$
變動長度、**平均 1699 TTI**（中位 1582、最大 3206；200 條軌跡實測 2026-07-21，保險絲 `EPISODE_CAP_TTI=4000`）。軌跡驅動 episode 為標準慣例——[[notes/Zhao2024_IBTD_ISAC_V2V]]（episode=車輛運動序列）、Ye&Ge Nature 2023（計畫書 ref[7]）。

**安全上限 = 4000 TTI（工程保險絲）**：蒙地卡羅三速度各 10000 episodes 確認最長合法 episode：30km/h max=3206、45km/h=2139、60km/h=1604，4000 留餘裕。

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
s = sceneGeometry(); % 畫圖目檢：大樓不壓路、gNB 在街廓 B 南牆（面向主街）、進入點在路口
```
**物理問句**：大樓有沒有蓋到馬路上？gNB 位置合理嗎？

---

## 2. LOS/NLOS 幾何遮蔽（isLos.m）

> **這在幹嘛：** 判斷車有沒有被大樓擋住——gNB 到車的直線若穿過任何一棟樓就是「看不到」(NLOS)，否則是「看得到」(LOS)。

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

> **這在幹嘛：** 算「訊號從基地台走到車，變得多弱、相位差多少」——給定車的位置，產生 LOS 直達 + NLOS 牆面散射的多路徑通道，是決定 SNR 好壞的核心。

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

**(d) NLOS 散射路徑**（top-3 最近牆面散射點，FR3 稀疏；**邊界線性淡入淡出**，2026-07-21 新增）：

$$
\mathrm{PL}_{NLOS} = \mathrm{FSPL}(1\,\text{m}) + 10\, n_{NLOS} \log_{10}(d_{3D}) + X_\sigma
$$

（**損耗用直線距離 $d_{3D}$**（gNB→車），非散射路徑長；**無外加反射常數**。$n_{NLOS}=2.59$ 已含反射與繞路代價，見下方說明。）

$$
g_s = \sqrt{w}\;\sqrt{10^{-\mathrm{PL}_{NLOS}/10}}\; e^{-j 2\pi (d_1+d_2)/\lambda}
$$

其中 $d_1$=gNB→散射點、$d_2$=散射點→車。**損耗錨在直線 $d_{3D}$、相位用幾何路徑 $d_1{+}d_2$（都卜勒來源）——職責分離**（2026-07-16 選項A修正，前身損耗誤用 $d_1{+}d_2$）。**反射損 $L_{\mathrm{refl}}=0$**（理由如下）。

> 🆕 **邊界淡入淡出權重 $w$（2026-07-21 新增，仿 QuaDRiGa cluster birth-death）**：舊版硬取「離車最近 top-3」，車移動使第 3/4 近散射點互換名次時，該路徑增益是**瞬間**出現/消失（不連續），可能在 SNR/波束選擇 reward 上產生偽突變。修正：取第 3、4 近散射點的距離中點 $d_{bound}=(d_{(3)}+d_{(4)})/2$，在 $d_{bound}\pm W_{fade}/2$ 範圍內線性淡入淡出：
> $$w = \mathrm{clip}\!\left(\frac{d_{bound}+W_{fade}/2 - d}{W_{fade}},\,0,\,1\right)$$
> 振幅乘 $\sqrt{w}$（對應功率乘 $w$）。**$W_{fade}=3\,\text{m}$ 為 [T3 暫定設計，未量化驗證]**——QuaDRiGa/Jaeckel et al. 2014 (IEEE TAP) 只提供「群集功率漸增/漸減」的概念，未給固定淡入淡出距離常數；本專題取一車道寬度量級為暫定值，與 $K$ 併入 2026-09 消融驗證清單。名次互換瞬態下 NLOS 路徑數可能暫態達 4（原本恆 ≤3）。

> 📐 **為何損耗用 $d_{3D}$ 不用 $d_1{+}d_2$（2026-07-16 選項A）**：Shakya $n_{NLOS}=2.59$ 是對「總損耗 vs **3D T-R 直線距離 $d_{3D}$**」擬合（已開原文核實 arXiv:2410.17539 v4），斜率**已含繞路/反射代價**（2.59>自由空間 2.0 的超出部分即是）。若把它套在幾何路徑長 $d_1{+}d_2$（比直線長）上=範式混用、重複計繞路（統計 PLE 該配直線距離、幾何射線範式才配路徑長+自由空間 n≈2+顯式反射）。實測影響僅 ~0.2dB（散射點貼近車、$d_1{+}d_2{\approx}d_{3D}$），屬範式一致性修正。$d_1{+}d_2$ 仍用於相位（幾何連續→都卜勒）。

> ✅ **為何不外加反射常數（2026-07-16 修正，前身「6 dB 反射損」）**：
> 
> **核心理由**：Shakya2025 式(1)/表III 的 NLOS PLE $n_{NLOS}=2.59$ 是對「**實測 NLOS 總損耗 vs 距離**」做的單斜率擬合——訊號繞射/反射/穿透到達的所有代價**已隱含在斜率內**（本節 07-04 修正註解已載明）。在其上**再外加**一個固定反射常數 = 對反射**雙重計算**。
> 
> **三篇核心論文皆無此做法**：Shakya / MultiUser / FR3_Rel19 均**無**「實測 PLE 之上再加反射常數」的建模。舊值 6 dB 為 T3 自推（ITU-R P.2040 正入射 8.1dB / Fresnel 反推），**非核心論文**，已移除以對齊三篇的 CI 統計建模。
> 
> **散射點仍保留**：其用途是產生 NLOS 幾何連續相位 $\varphi=2\pi(d_1{+}d_2)/\lambda$（都卜勒來源），與反射損耗無關 → 設 0 不影響相位建模。
> 
> **實測影響（2026-07-16 periodicSweepSim(8)）**：拔 6dB 對 NLOS 路徑 +6dB 增益，連同 N₀ 修正一併驗證（見 §0/§8 N₀ 修正）。

**實裝驗證**：CI 的 $n_{NLOS}=2.59$ 是實測「總損耗 vs 距離」的擬合值，故只能算**一次** CI 單斜率。損耗用直線 $d_{3D}$（Shakya PLE 對直線距離擬合）；相位用幾何路徑 $d_1{+}d_2$（都卜勒來源）。

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

### ③ 出處（已驗證，2026-07-12）

#### 公式 (a)–(b)：CI 路徑損耗 + FSPL 基準
- **✅ Shakya, D., Ying, M., Rappaport, T. S., et al. (2025).** "Urban Outdoor Propagation Measurements and Channel Models at 6.75 GHz FR1(C) and 16.95 GHz FR3 Upper Mid-Band Spectrum for 5G and 6G." NYU WIRELESS. (Submitted to IEEE; DOI pending)
  - 方程式 (1)：CI 路徑損耗公式與 FSPL 基準
  - Table III：16.95 GHz 全向 PLE 與陰影參數
  - 庫內位置：[[notes/Shakya2025_UrbanPropagation]]；原文 PDF：`D:\PROJECTS\papers\核心論文\Shakya2025_UrbanPropagation\`

#### 公式 (b)：AR(1) 陰影相關
- **✅ Gudmundson, M. (1991).** "Correlation model for shadow fading in mobile radio systems." *Electronics Letters*, vol. 27, no. 23, pp. 2145–2146. DOI: 10.1049/el:19911328 （IEEE Xplore 驗證）
  - 相關係數 $\rho = \exp(-\Delta d/d_{cor})$ 指數衰減模型奠基
  - 本實現採實際位移 $\Delta d$（非固定間隔），使陰影更新更動態
  - 去相關距離 $d_{cor}=10$ m 佐證：3GPP TR 38.901 都會 10–13 m

#### 公式 (c)–(d)：幾何相位 & 都卜勒
- **✅ Clarke, R. H. (1968).** "A Statistical Theory of Mobile-Radio Reception." *The Bell System Technical Journal*, vol. 47, no. 6, pp. 957–1000. DOI: 10.1002/j.1538-7305.1968.tb00069.x （IEEE Xplore 驗證）
  - 幾何相位 $\varphi = 2\pi d/\lambda$ 與都卜勒頻移 $f_D = v\cos\alpha/\lambda$ 奠基
  - 原文免費 PDF：https://archive.org/details/bstj47-6-957

- **Jakes, W. C. (Ed.). (1974).** *Microwave Mobile Communications*. Wiley. ISBN: 978-0471437208 （原版；IEEE Classic Reissue: 978-0780310698）
  - Clarke 1968 理論的教科書標準化與系統化呈現

#### 補充參考
- FR3 通道建模背景：[[notes/FR3_Rel19_ChannelModeling]]（3GPP Rel-19 GBSM 框架）
- SPEC.md：`D:\FR3_BeamMgmt\SPEC.md` §1.2（所有出處宣告集中於此）

### ③½ 公式文獻引用速查表

| 公式 | 原始出處 | DOI/ISBN | 程式位置 |
|-----|--------|---------|--------|
| (a) FSPL 基準 | Shakya2025, Eq. (1) | TBD | `channelModelV2.m:33` |
| (a) CI 路徑損耗 | Shakya2025, Table III | TBD | `channelModelV2.m:51,69` |
| (b) $\rho = \exp(-\Delta d/d_{cor})$ | Gudmundson 1991 | 10.1049/el:19911328 | `channelModelV2.m:41` |
| (b) AR(1) 陰影更新 | Gudmundson 1991 | 10.1049/el:19911328 | `channelModelV2.m:42` |
| (c) 幾何相位 $\varphi = 2\pi d/\lambda$ | Clarke 1968 | 10.1002/j.1538-7305.1968.tb00069.x | `channelModelV2.m:52,70` |
| (c)–(d) 都卜勒 $f_D = v\cos\alpha/\lambda$ | Clarke 1968 | 10.1002/j.1538-7305.1968.tb00069.x | 隱含 `myStep.m` |
| (d) NLOS 單斜率損耗 | Shakya2025 | TBD | `channelModelV2.m:69` |

### ④ 檢驗
```matlab
runtests('tests/testChannelModelV2.m')  % 6 測試：幾何LOS/陰影靜止/相位決定性/NLOS單斜率/top-3(4)/邊界淡入淡出連續性
% 手動：沿主街掃 x，畫路徑增益曲線 → 應「近 gNB 高、遠端低、平滑無跳崖」
```
**物理問句**：SNR 曲線平滑嗎？LOS↔NLOS 切換發生在幾何遮蔽點嗎？

---

## 4. 散射點（generateScatterers.m）

> **這在幹嘛：** 決定 NLOS 訊號「從哪面牆反射過來」——在大樓面向街道的牆上固定佈點，讓被擋住的車還能靠牆面反射收到訊號。

### ① 這是什麼
車被大樓擋住（NLOS）時，直線訊號被切斷，但訊號仍會**打到旁邊大樓的牆面、反彈**到車。本單元決定這些「反射點」放哪：在每棟大樓**面向街道的牆面**上固定佈點（每牆取樣 5 點、向外推 `scatterOffsetDist`、只留落在路廊內的），**決定性生成（無 rand）**。通道（§3）再取離車最近的 **top-3** 當 NLOS 反射路徑：gNB $\to(d_1)\to$ 反射點 $\to(d_2)\to$ 車。第3/4近散射點名次互換的邊界改**線性淡入淡出**（2026-07-21新增，見§②(d)），避免路徑瞬間出現/消失。

> 🔑 **兩個關鍵觀念**：
> 1. **散射點只給「幾何」、不給「損耗」**：反射點提供 NLOS 的**離開角**與**路徑長 $d_1{+}d_2$（→相位→都卜勒）**；訊號**衰減多少**由 §3 的 Shakya 統計 CI（用直線 $d_{3D}$）決定。故每點反射損耗 `reflectLoss=0`（反射代價已含於 $n_{NLOS}=2.59$ 斜率，外加會雙重計算＝6dB 被移除的原因）。**反射點 = 決定性單次反射的幾何錨點。**
> 2. **為何要「決定性（固定不動）」——空間一致性（spatial consistency）**：車移動時相位 $\varphi=2\pi(d_1{+}d_2)/\lambda$ 必須**平滑連續**，都卜勒才會自然浮現；反射點若每步亂跳，相位就斷了、都卜勒就假了。
>    這個要求有**正式名稱與標準依據**：**3GPP TR 38.901 §7.6.3「Spatial consistency」**明訂移動性模擬需要空間一致性，並指出 **drop-based（隨機抽樣）模型本身不具備**、必須額外套一道 spatial consistency procedure（Procedure A/B）來補救。
>    → **本專題改採決定性幾何，天生（by construction）滿足空間一致性，不需補救程序**——這就是不用隨機散射的理由。

### ② 公式
散射點就是**牆面上的取樣點** $\mathbf{s} = \mathbf{w}(u)$（$u\in\{0.1,\dots,0.9\}$，避開牆角）。判斷「這面牆是否面向街道」用一個探針：把牆面點往牆外推 $d_{offset}$，看落不落在路廊——$\mathbf{w}(u)+d_{offset}\hat{\mathbf{n}}\in\mathcal{R}$ 才保留該牆面點。**注意：存下來的是牆面點 $\mathbf{w}(u)$，不是推出去的探針**，故 $d_{offset}$ 只影響「哪些牆入選」、對散射點座標零影響。通道取離車最近的 **top-K**：

$$
\mathcal{S}_{act} = \underset{|\mathcal{S}'|=K}{\arg\min} \sum_{\mathbf{s}\in\mathcal{S}'} \lVert \mathbf{s}-\mathbf{x}_{veh}\rVert
$$

**設計旋鈕**：

| 旋鈕 | 現值 | 性質 / 理由 | Q3 敏感度 |
|---|---|---|---|
| $K$（NLOS 路徑數） | 3 | [T3 設計選擇] FR3 稀疏、少數主導 MPC（Shakya NLOS ASA 23.99°≪3GPP 59.54°）；窄波束只收少數路徑。⚠️ K=3 為工程近似，文獻未指定具體路徑數 | Q3 掃 K∈{1,3,5} |
| $W_{fade}$（top-3/4 邊界淡入淡出寬度） | 3 m | [T3 暫定設計，未量化，2026-07-21新增] 修正名次互換瞬間跳變（仿 QuaDRiGa birth-death，Jaeckel 2014 IEEE TAP 無淡入淡出距離常數） | 與 K 併入 Q3 消融清單 |
| 每牆取樣點數 | 5 | [T3 設計選擇] 空間解析 vs 計算成本折衷 | — |
| `reflectHeight`（散射點高度） | 1.5 m | [T3 設計選擇] **街道層反射**：放在車輛平面（車高亦 1.5 m），模擬街道峽谷側牆的街道層反射 | — |
| $d_{offset}$（探針距離） | 0.5 m | ✅ **非物理參數**：只是「測牆是否面向街道」的探針，**對 SNR/相位零影響**（散射點在牆面上）。任何 $0<值<$路寬 等效 → **無需文獻、無需敏感度掃描** | 不適用 |
| `reflectLoss` | 0 | 反射代價已含於 §3 的 $n_{NLOS}$ 斜率（見上 🔑） | — |

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| 牆面取樣 + 探針測朝向 + 落路保留 | `env/generateScatterers.m` | 主迴圈第 49–72 行（每牆 4 面 × 5 點、linspace 0.1–0.9 避角；探針 q 測 `pointInRoads`，存牆面點 px,py） |
| **單次反射損耗 `reflectLoss = 0`**（每點第 4 欄 loss；2026-07-16 6→0，反射已含於 §3 n_NLOS 斜率） | `env/generateScatterers.m` | 第 36 行 |
| **散射點高度 `reflectHeight = 1.5 m`**（每點第 3 欄 z；街道層、影響 NLOS 仰角）[T3 設計選擇] | `env/generateScatterers.m` | 第 27 行 |
| `scatterOffsetDist` 定義（**牆面朝向探針，非物理參數**） | `config.m` | 值 0.5 m；2026-07-16 重新定性：對 SNR/相位零影響、無需文獻與敏感度掃描 |
| 路廊判定 | `env/generateScatterers.m` | 內部函式 `pointInRoads` |
| top-3 最近選擇＋邊界淡入淡出 | `env/channelModelV2.m` | 區塊註解 `--- ③④⑤ NLOS：top-3 主導散射（邊界平滑淡入淡出） ---`（2026-07-21 時為第 86 行起；**行號會漂移，以區塊註解為準**） |

### ③ 出處（2026-07-16 逐一開原文核實）

**方法命名**：本法＝**幾何式決定性單次反射（geometry-based deterministic / single-bounce ray-tracing）**。前身誤標「GBSM」已更正——GBSM 的 S=Stochastic 指**隨機**撒群集，與本專題「決定性、無 rand、貼真實牆面」相反。

| 引用                                                        | 用途                                                                                                  | 核實                                                             |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **Shakya2025**（arXiv:2410.17539 v4 Table VI）              | ASA 15.3°（≪3GPP 47.31°）→ FR3 稀疏、少數主導路徑                                                              | 🟢 已開原文                                                        |
| **GEMV²**（Boban et al., IEEE TVT 2014, arXiv:1305.0124）   | 主要範式依據：真實建物輪廓 + **決定性大尺度 LOS/NLOS 分類**，與本法最貼合                                                       | 🟢 已開原文（註：其小尺度為 stochastic，本法只借「真實幾何+決定性 LOS/NLOS」精神）          |
| **FR3_Rel19**（[[notes/FR3_Rel19_ChannelModeling]]）        | FR3 **稀疏性隨頻率上升**（M_min 捕捉稀疏、群集數改為**帶範圍可變量**、角度/延遲擴展隨頻率變小）                                           | 🟢 已核原文（措辭已更正：**非**「群集數單調變少」）                                  |
| **3GPP TR 38.901 §7.6.3**（Spatial consistency；j40/Rel-18） | **⭐ 採用決定性幾何的正當理由**：標準明訂移動性模擬需空間一致性，且 drop-based 隨機模型**不具備**、需額外 Procedure A/B 補救 → 本法用決定性幾何**天生滿足** | 🟢 已開原文（親自抽 j40 docx；§7.6.3.1/7.6.3.2/7.6.3.4、Table 7.6.3.1-2） |
| ETSI TR 103 257-1                                         | 將 geometry-based deterministic (ray-tracing) 列為 V2X/ITS 通道模型選項之一                                    | 範式參考                                                           |


### ④ 檢驗
```matlab
runtests('tests/testGenerateScatterers.m')  % 4 測試：格式/決定性/貼牆/面向道路
sc=generateScatterers(s); % 畫圖：紅點應全貼牆、不在樓內/路中央；跑兩次結果相同
```

---

## 5. 波束成形基礎（steeringPhase / arrayResponse / beamWeight / elementGain + dftCodebook.m）

> **這在幹嘛：** 讓 128 根天線把能量聚成一根「手電筒」指向某方向——碼本就是一組預先造好、各指一個方向的手電筒，波束管理就是從裡面挑一根對準車。

### ① 這是什麼
gNB 用 128 天線把能量「聚成一根手電筒」。導向向量 $\mathbf{a}$ 描述「訊號從某方向來/去時，各天線收到的相位差」；DFT 碼本 $\mathbf{F}$ 是一組預先造好的手電筒（每束指向一個固定方向），波束管理=從碼本裡挑一束。

### ② 公式

**UPA 導向向量**（$M\times N$ 陣列、間距 $d=\lambda/2$）：

$$
[\mathbf{a}(\theta,\phi)]_{m,n} = \exp\!\Big( j\frac{2\pi d}{\lambda}\big( m\sin\theta_{el} + n\cos\theta_{el}\sin\theta_{az} \big) \Big)
$$

其中 $m$ 索引**仰角（垂直）**維、$n$ 索引**方位（水平）**維。

> **⚠️ 2026-07-20 修正：方位相位原本缺 $\cos\theta_{el}$ 因子**
> 舊版程式為 $\psi_{az}=(2\pi d/\lambda)\sin\theta_{az}$（等於把 UPA 當成兩個獨立 ULA 相乘，**僅在 $\theta_{el}=0$ 時正確**）。
> 物理意義：**仰角越大，水平方向的有效孔徑投影越短**、方位解析度越差；缺此因子則方位波束不會隨仰角展寬。
>
> | 仰角 | 方位相位偏差 | 等效指向誤差（$az=30°$）|
> |---|---|---|
> | 12.6° | 2.4% | +0.9° |
> | 26° | 10.1% | +3.8° |
> | 41° | 24.5% | +11.5° |
> | 50° | **35.7%** | **+21.1°**（$az=50°$ 甚至落到可視區外 → grating）|
>
> ⚠️ **此缺陷因同日的仰角下傾修正（$\pm15°\to0$–$50°$）而大幅惡化**：舊仰角範圍內 $\cos\theta_{el}\ge0.966$、誤差 $\le3.4\%$，幾乎看不出來。**修一個東西會放大另一個潛伏的錯誤。**
>
> 註：碼本與通道皆呼叫同一函式，故修正前模擬**內部自洽**（波束仍可選可追蹤），但該陣列**並非物理上的 $\lambda/2$ 平面陣列**，角度↔波束映射被扭曲。
>
> **出源**：[T1] Balanis, *Antenna Theory*, 4th ed., Wiley 2016, **§6.10**（平面陣列 AF）；[T1] Van Trees, *Optimum Array Processing* Part IV, Wiley 2002, **§4.1**。
>
> **效果**：方位追蹤誤差最大 **15 格 → 1 格**；方位+仰角兩維完全命中 **54% → 68%**；CRLB 最大值 0.0022 → 0.00017。
> ⚠️ **歸因勘誤**：先前把「方位誤差最大 15 格」歸因於 $\pm60°$ 扇區覆蓋洞，實為此 $\cos\theta_{el}$ 混疊所致。

> 📐 **陣列軸向慣例（3GPP TR 38.901）**：38.901 記法 $(M,N)$ 中 **$M$=垂直（仰角）元件數、$N$=水平（方位）元件數**。故本專題「方位16／仰角8」在 38.901 應寫成 **$(M,N)=(8,16)$**（8 垂直 × 16 水平 = 128）。出處：MathWorks `phased.NRRectangularPanelArray`（實作 38.901）；本專題 `steeringPhase.m` 一致（`ntRow=8=M`、`ntCol=16=N`）。

> ### ⚠️ 2026-07-21 架構分離：導向向量 $\mathbf{a}$ 與波束權重 $\mathbf{w}$ 是**兩個不同的東西**
>
> 上面那條相位式，是 $\mathbf{a}$ 與 $\mathbf{w}$ **共用的骨架**。但兩者在骨架之外的處理**完全相反**：
>
> | | $\mathbf{a}(\theta)$ 陣列響應 | $\mathbf{w}(\theta_0)$ 波束權重 |
> |---|---|---|
> | 是什麼 | 「那個方向的波打到陣列上長什麼樣」——一個**事實** | 「我要怎麼調相位把能量送過去」——一個**選擇** |
> | 屬於哪一側 | **通道側** | **發射側** |
> | 元件方向圖 $g(\theta_{el})$ | **含**（天線的物理屬性） | **不含** |
> | 範數 | $\lVert\mathbf{a}\rVert^2 = N_t\, g(\theta_{el})$，**不歸一化** | $\lVert\mathbf{w}\rVert = 1$（發射功率約束） |
> | 程式 | `env/arrayResponse.m` | `env/beamWeight.m` |
>
> $$\mathbf{a}(\theta)=\sqrt{g(\theta_{el})}\,\big[e^{j\psi_1}\cdots e^{j\psi_{N_t}}\big]^{\mathsf T},\qquad \mathbf{w}(\theta_0)=\tfrac{1}{\sqrt{N_t}}\big[e^{j\psi_1}\cdots e^{j\psi_{N_t}}\big]^{\mathsf T}$$
>
> **為什麼一定要分**：2026-07-21 前兩者共用同一個函式 `upaArray.m`，該函式**又歸一化、又乘元件方向圖**。
> 碼本與通道都呼叫它，於是：
>
> 1. **元件方向圖被套兩次** → 對準增益變成 $g^2$ 而非 $N_t g$。$el=50°$ 時人造多損 **7.1 dB**。
> 2. **陣列增益 $10\log_{10}128 = 21.1$ dB 憑空消失** → 兩邊各除一次 $\sqrt{N_t}$，$|\mathbf{w}^H\mathbf{a}|^2=|\alpha|^2$，
>    等於退化成單根天線。實測鏈路預算比第一原理手算**短少 19.6 dB**。
> 3. $\lVert\mathbf{F}(:,k)\rVert \in [0.4415,\,1.0000]$ —— **違反 TR 38.901 eq. (7.3-1)** 的功率約束
>    $w_m=(1/\sqrt M)e^{j\phi_m}$（純相位、式中沒有任何 $A_{\text{dB}}$ 項）。高下傾波束只發出 19.5% 的功率。
>
> **為什麼五輪審查、43/43 單元測試、5/5 物理驗證全都沒抓到**：這三項都是**絕對值**錯、**比值**對。
> 既有檢驗全是「拿模型跟模型自己比」（驗 $R_{\text{eff}}$ 是否等於公式、驗路損**斜率**是否等於 PLE），
> 對「整條鏈路一致地少 21 dB」完全隱形。**這是本專題最重要的方法論教訓**：
> 內部一致性檢驗再多，也驗不出絕對值錯誤——必須有一支**從第一原理手算、完全不引用模擬結果**的稽核
> （`verifyAbsolutePhysics.m`）。
>
> **修正後實測**：鏈路預算誤差 $-19.6 \to \mathbf{+1.9}$ dB；且這 $+1.9$ **完全可解釋**——模擬側平均了 300 次含陰影
> 的實現，對數常態的均值高於中位數 $\exp(\sigma^2/2)$，$\sigma=4.05$ dB $\Rightarrow +1.89$ dB，與實測吻合，**非誤差**。
> $\lVert\mathbf{F}(:,k)\rVert$ 全 128 束 $=1.0000$；`verifyPart1` ⑤ 波束方向性峰值 $0.81 \to \mathbf{1.00}$。
>
> **出源**：[T2] 3GPP TR 38.901 V19.4.0 **eq. (7.3-1)**（權重純相位）、**Table 7.3-1**（元件方向圖 $A_{EV}$）、
> **eq. (7.5-22)**（通道係數含元件場型）；[T1] Balanis 4th ed. §6.10；[T1] Van Trees Part IV §4.1。

**元件方向圖**（`env/elementGain.m`）：單一天線振子本身的指向性，與陣列有幾根天線無關。

$$
A_{EV}(\theta_{el}) = -\min\!\Big[12\Big(\frac{\theta_{el}}{\theta_{3\text{dB}}}\Big)^2,\ \mathrm{SLA}_V\Big]\ \text{dB},\qquad \theta_{3\text{dB}}=65°,\ \mathrm{SLA}_V=30\ \text{dB}
$$

**出源**：[T2] 3GPP TR 38.901 V19.4.0 **Table 7.3-1**（垂直切面 $A_{EV}$ 一列）。

> ✅ **方位切面 $A_{EH}$ 已納入（S1–S2，2026-07-22）**：Table 7.3-1 的水平切面
> $A_{EH}(\phi)=-\min[12(\phi/65)^2,30]$ dB 已透過三扇區接入（見下方「前後模糊性」節的修正結案）。
> `arrayResponse` 新增 opt-in 第 7 參 `applyAzPattern`；`myReset`/`myStep` 對每條 path 以
> **相對服務面板法線的 $\phi=\text{azRel}$** 套用 $A_{EH}$，故 $\pm90°$ 掉增益的問題不再存在——
> 每個 UE 由 $|\text{azRel}|$ 最小的面板服務、恆落在 $\pm60°$ 內。
> （峰值增益 $G_{E,\max}=8$ dBi 為常數偏移，仍未納入，不影響波束選擇與相對比較。）

**碼本波束成形增益**（整個專題最核心的量）：

$$
G(\theta) = |\mathbf{w}^H \mathbf{a}(\theta)|^2, \qquad 0 \le G \le N_t\, g(\theta_{el})
$$

完全對準時 $G = N_t\,g(\theta_{el})$ ——**陣列增益 $N_t$ 與元件增益 $g$ 各恰好乘一次**。
$g\to1$（法線方向）時即 $G=N_t=128$（**21 dB**）；失配時暴跌——窄波束讓 $G$ 對角度誤差極敏感，
**這就是 FR3 波束管理問題的數學根源**。

**碼本的取樣空間**（2026-07-17 更正；2026-07-18 軸向翻正）：波束必須在 **$\sin(\theta_{az})$ 上等間隔**，不是在角度上等間隔。原因見 ②½ 下方的白話說明。目前 `dftCodebook(ntRow=8, ntCol=16)` = **128 束（方位 16 × 仰角 8）**，方位 16 束在 $\sin$ 空間均勻、端點 $\pm 60°$，$\Delta\sin \approx 0.1155$（< 正交間隔 $2/16=0.125$，屬輕微過取樣）。

> **為什麼不能在角度上等間隔（白話）**：陣列因子 AF 是**空間頻率** $\psi = \frac{2\pi d}{\lambda}\sin\theta$ 的函數，不是角度的函數。波束在 $\psi$ 域寬度固定，換算回角度域時會隨掃描角**展寬 $1/\cos\theta$** —— 越靠扇區邊緣的波束越胖。所以角度等間隔會造成「中央過取樣、邊緣欠取樣」，相鄰波束之間破出覆蓋空洞。
>
> **實測（`tests/testDftCodebook.m`）**：舊的角度均勻柵格最差交越損 **−5.77 dB**；改為正弦空間均勻後為 **−3.78 dB**，逼近理論值 −3.92 dB。改善 1.99 dB，波束數不變（仍 128）。

**軸向配置**（2026-07-18）：8V×16H（方位 16 元件 HPBW≈6.36°、仰角 8 束 HPBW≈12.6°）。**依據**：[T1] Shakya2025 表V ASD 22–82° ≫ ZSD 8.5–13°（角度能量集中方位）。

> **⚠️ 仰角改為下傾（2026-07-20 修正）**
> 舊版 `elList = linspace(-15, 15, 8)`（角度等間隔、上下對稱），有**兩個疊在一起的缺陷**：
>
> **(a) 一半波束指向沒有使用者的上空。** 車永遠在 gNB 下方（gNB 10 m、車 1.5 m），符號約定下 $el=\arctan(\Delta h/d)$ **恆為正**。對稱 $\pm15°$ 使近半數波束浪費在 $el<0$。
> **(b) 剩下一半又過度密集、同時涵蓋不足。** 8 元件仰角 HPBW ≈ $102°/8 ≈ 12.75°$，舊版 8 束擠在 $30°$ 內（間距 $4.3°$）→ 約 3 倍過取樣；卻**完全放棄 $15°$–$49°$**。
>
> **實測證據**（20 條軌跡、33,005 個 TTI）：**9.0% 的時間仰角 > 15°，超出碼本涵蓋**；所有 SNR $=-52$ dB 的深度失準壞點皆落於該區。主街距 gNB 最近僅 **7.5 m** → 仰角高達 **49°**（超出上限 3 倍以上）。這也是「LOS 的 SNR 與距離相關竟為正 $(+0.15)$」的成因——**近距離反而有壞點**。
>
> **修正**：
> $$\text{elList} = \arcsin\!\big(\text{linspace}(0,\ \sin 50°,\ 8)\big)\quad\Rightarrow\quad [0°,\,6.3°,\,12.6°,\,19.2°,\,26.0°,\,33.2°,\,41.0°,\,50.0°]$$
>
> **需求範圍推導**：$\Delta h = 10 - 1.5 = 8.5$ m；$d\in[7.5,\,180]$ m $\Rightarrow el\in[2.7°,\,49°]$ → 取 $[0°, 50°]$ 完整涵蓋。
> **取樣密度校驗**：$\sin 50° = 0.766$，8 束 → $\Delta\sin \approx 0.11$；8 元件正交間距 $2/8 = 0.25$ → **仍為過取樣**，交越損只會更好。
> **⇒ 涵蓋範圍擴大 1.67 倍，波束數與天線皆不變 —— 零成本改善。**
>
> **出源**：**[T2] 3GPP TR 38.901 §7.3** BS 天線模型含電下傾（electrical downtilt）參數——蜂巢網路基地台一律下傾，使波束涵蓋地面使用者而非天空。**[T1] Tse & Viswanath §7.3.1；Balanis §6.3**：正交基底取在 directional cosine（sine 空間）上，故一併改為 sine 均勻取樣。
>
> **效果**：仰角追蹤誤差最大 $6 \to 1$ 格、方位+仰角兩維完全命中 $54\% \to 70\%$、LOS 平均 SNR $22.8 \to 23.7$ dB、CRLB 最差值 $0.067 \to 0.0097$。

### ②½ 程式對應
| 內容 | 檔案 | 位置／備註 |
|---|---|---|
| 相位骨架 $e^{j\psi}$（兩側共用，含 $\cos\theta_{el}$ 因子） | `env/steeringPhase.m` | 全檔。**獨立成檔的理由**：兩側若各寫一份相位式必然分歧，2026-07-20 的 $\cos\theta_{el}$ 缺失即為前例 |
| 陣列響應 $\mathbf{a}(\theta)$，$\lVert\mathbf{a}\rVert^2=N_t g$ | `env/arrayResponse.m` | **通道側**，含元件方向圖、不歸一化。共 13 個呼叫點 |
| 波束權重 $\mathbf{w}(\theta_0)$，$\lVert\mathbf{w}\rVert=1$ | `env/beamWeight.m` | **發射側**，純相位。唯一呼叫點：`dftCodebook.m` |
| 元件方向圖 $g(\theta_{el})$ | `env/elementGain.m` | 單一真相；原本在三個地方各寫一份 |
| DFT 碼本 $\mathbf{F}$（128 束 = 方位 16 × 仰角 8） | `env/dftCodebook.m` | `azList = asind(linspace(-sin60°, sin60°, 16))` 正弦空間均勻；`elList = asind(linspace(0, sin50°, 8))` **下傾 0–50°、亦為正弦空間均勻**（2026-07-20 修正，見上方 ⚠️ 框）；逐束呼叫 `beamWeight` |
| 碼本取樣空間與覆蓋驗證 | `tests/testDftCodebook.m` | 全檔（sine 均勻性 + 最差交越損 > −4 dB） |
| $G=\lvert\mathbf{w}^H\mathbf{a}\rvert^2$ 實際計算 | `env/myStep.m`（通訊）、`env/echoSignalV2.m`（感測） | 內積後取 `abs()^2` |
| **絕對值稽核**（第一原理手算，不引用模擬） | `verifyAbsolutePhysics.m` | 雜訊功率／鏈路預算／向量範數／元件方向圖次數／雷達方程式 |

### ③ 出處
| 級      | 引用                                                                                                                                                        | 支持什麼                                                                                                                                                                                                       |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **T1** | 3GPP TS 38.214, *NR; Physical layer procedures for data*, **§5.2.2.2.1**（Type I Single-Panel Codebook, Rel-15 起）                                          | 波束索引 $l$ 進入 $\exp(j2\pi l/(O_1 N_1))$ 的**線性相位項** → 取樣格點在**空間頻率上等間隔**，$\sin\theta$ 步長 $= 2/(O_1 N_1)$。**標準從未在角度域等間隔取樣。**                                                                                    |
| **T1** | Tse & Viswanath, *Fundamentals of Wireless Communication*, Cambridge Univ. Press, 2005, **§7.3.1「Angular domain representation of signals」, pp. 318–322** | ULA 的固定角度基底取在 **directional cosine** $\Omega = k/L_t$ 上，構成空間 DFT **正交**基底。**正交性只在 sine space 等間隔時成立**；λ/2、$N$ 元陣列即 $\sin\theta$ 間隔 $2/N$。                                                                  |
| **T1** | Balanis, *Antenna Theory: Analysis and Design*, 4th ed., Wiley, 2016, **§6.3（N-Element Linear Array: Uniform Amplitude and Spacing）**                     | $AF(\psi) = \frac{\sin(N\psi/2)}{N\sin(\psi/2)}$，$\psi = kd\cos\theta+\beta$。方向圖是 $\psi$ 的函數；主瓣在 $\psi$ 域固定寬、在 $\theta$ 域隨掃描角展寬 $\propto 1/\cos\theta$ ⇒ **角度等間隔必然中央過取樣、邊緣欠取樣**。                           |
| **T2** | Mailloux, *Phased Array Antenna Handbook*, 3rd ed., Artech House, 2018, **Ch. 8（Multiple-Beam Systems / Beam Crossover Loss）**                            | 「3.9 dB 交越損」的出處。**這不是實驗值，是閉式解**：正交間隔 $\Delta\psi = 2\pi/N$，交越點 $\Delta\psi/2 = \pi/N$ 代入 Dirichlet 核得 $AF = \frac{\sin(\pi/2)}{N\sin(\pi/2N)} \to 2/\pi = 0.6366$ ⇒ 功率 $(2/\pi)^2 = 0.405$ ⇒ **−3.92 dB**。 |
| T3     | Van Trees, *Optimum Array Processing*, Wiley, 2002                                                                                                        | 陣列訊號處理通論（頻率無關，故 FR3 適用）。                                                                                                                                                                                   |

> **這個 3.92 dB 本身就證明了命題**：它是 Dirichlet 核在**正交格點**交越處的閉式值，只在 sine space 均勻的臨界取樣 DFT 柵格上成立。角度均勻的柵格**根本算不出一個穩定的 3.92 dB**（我們實測到 −5.77 dB）。

> **例外情形（誠實揭露，均不適用本專題）**：① 部分 mmWave 論文與 3GPP RAN4 測試確實用角度均勻 GoB，但那是**類比波束成形／OTA 量測的掃描規格**，不是碼本正交性設計。② 近場（Fresnel 區）DFT 碼本本就會 beam-split，sine space 均勻性亦不成立；但 SPEC.md L27 已論證 Fraunhofer 距離 $2D^2/\lambda \approx 2.8$ m ≪ 車距 → **遠場成立**，此例外不適用。

### ④ 檢驗
```matlab
config;
% ⚠️ 引數順序是 (ntRow=仰角8, ntCol=方位16)，不是 (16,8)
[F, azList, elList] = dftCodebook(ntRow, ntCol, fc, dSpacing);

% (1) 功率約束：**全部 128 束**都要是 1（只驗第 1 束會漏掉高下傾束的問題）
max(abs(vecnorm(F) - 1))                       % → ~0（<1e-9）

% (2) 方位正弦空間等間隔：Δsin = 2·sin60°/15 = 0.11547
max(abs(diff(sind(azList)) - 2*sind(60)/15))   % → ~0（<1e-9）

% (3) 仰角亦為正弦空間均勻、且為下傾 0–50°
elList                                          % → [0 6.3 12.6 19.2 26.0 33.2 41.0 50.0]

% (4) 對準增益 = N_t·g（陣列增益與元件增益各一次）
k = (ntRow-1)*ntCol + round(ntCol/2);          % 取最大下傾、近 broadside 的那一束
%    ⚠️ 測試方向必須取自碼本格點本身。azList 是 16 點對稱取樣、**不含 0°**，
%       隨手取 az=0° 會沒對準，兩個假說都對不上（2026-07-21 踩過這個坑）
a = arrayResponse(fc, azList(round(ntCol/2)), elList(ntRow), ntRow, ntCol, dSpacing);
abs(F(:,k)'*a)^2 / (nAnt * elementGain(elList(ntRow)))   % → 1.000

plot(abs(F'*a).^2)                              % 一根主峰
```
完整驗證：`runtests('tests/testDftCodebook.m')` → 3/3；
**絕對值稽核** `verifyAbsolutePhysics` → 4/4（①雜訊 ②鏈路預算 ③範數 ④元件方向圖次數）。

---

## 6. ISAC 感測回波（echoSignalV2.m）

> **這在幹嘛：** 基地台發射的波束打到車、反彈回來，靠回波強弱就能「感覺」波束對不對準——不用車回報，這是本論文「免回饋感測輔助」的物理基礎。

### ① 這是什麼
gNB 發射的波束打到車、反彈回 gNB 感測接收機。回波強度隨「波束對準程度」變化 → gNB **不需要車回報**就能感覺波束對不對——這是論文「免回饋感測輔助」賣點的物理基礎。

### ② 完整公式

**理想回波功率**（單站雷達，數位接收對準真實回波角）：

$$
P_{echo} = P_{tx}\, |\mathbf{a}^H \mathbf{f}|^2\, \beta^2, \qquad \boxed{\beta^2 = \frac{N_t^2\,\lambda^2\,\sigma_{RCS}}{(4\pi)^3\, d^{2n_{pl}}}}
$$

（$\beta$ 依 **MultiUser Eq.(8b)** 原文：$\beta = N_t\sqrt{v_c^2\sigma_{rcs}/f_c^2/(4\pi)^3/d^4}$，$v_c/f_c=\lambda$。含**雙向路損 $1/d^4$**、**目標 RCS**、**接收數位陣列增益 $N_t$**。）

- $|\mathbf{a}^H\mathbf{f}|^2$：**一個**發射波束對準因子（接收為數位全陣列、增益固定 → 不是 $|\cdot|^4$）

> **⚠️ 2026-07-20 重大修正：照射方向改為沿「實際傳播通道」，而非直指目標的幾何角**
>
> **白話**：基地台的電波要照到車上，得沿著「電波實際走得通的路」。LOS 時那條路就是直線（＝幾何角）；NLOS 時直線被大樓擋死，電波是繞著散射點過去的。舊版不管有沒有被擋，一律用幾何角算回波 → 等於讓**雷達看穿牆**。
>
> **修正**：以**通訊通道的單位方向**取代真實角的導引向量
> $$\hat{\mathbf{h}} = \frac{\mathbf{h}}{\|\mathbf{h}\|},\qquad \mathbf{h}=\sum_p g_p\,\mathbf{a}(\theta_{az,p},\theta_{el,p}),\qquad |\mathbf{a}^H\mathbf{f}|^2 \;\longrightarrow\; |\hat{\mathbf{h}}^H\mathbf{f}|^2$$
> 其中 $\mathbf{h}$ **就是計算通訊 SNR 用的同一組多路徑**（`myStep` 已把通道計算前移至選束之前）。
>
> **為何必須修——它動搖研究前提**：計畫書 L204 的主張是「**以感測取代 UE 即時回饋**」。舊版讓感測看到的方向與通訊實際能量方向不同，等於**人為破壞該前提**。實測：
>
> | | 修正前 | 修正後 |
> |---|---|---|
> | 感測選束 vs 通訊最佳束落差（NLOS 中位 / P90）| 0.4 / **9.3 dB** | — |
> | $\mathrm{corr}(\log P_{echo}, \log \mathrm{SNR}_{comm})$ — LOS | +0.631 | **+0.630**（幾乎不變）|
> | 同上 — **NLOS** | **+0.284** | **+0.539**（近乎翻倍）|
> | NLOS 平均 SNR | 10.9 dB | **11.9 dB** |
> | 連線率 | 87% | **89%** |
>
> 三點皆符合設計意圖：**(1) LOS 不變** → 修正具針對性（LOS 時 $\mathbf{h}\propto\mathbf{a}$，自動退化為舊行為、向後相容）；**(2) NLOS 大幅改善** → 觀測恢復反映通訊實況；**(3) 未變成 $\approx+1.0$** → **問題沒有被修簡單**（回波是非同調雙程功率＋量測雜訊，通訊是同調複數和，本就不該完全相關）。
> ⚠️ 若修完變成 +0.95 反而是**修歪**——感測退化為完美 CSI，研究問題消失。
>
> **出源**：[T2] 3GPP Rel-19 ISAC **target channel concatenation** —— 感測通道為 (gNB→目標) ⊛ RCS ⊛ (目標→gNB)，**每段皆為標準通道模型**（含其 LOS/NLOS 與角度擴展），而非直線幾何角。本式即該串接語意在單站、共用陣列下的實作。
>
> **量級校準不變**：波束因子仍為 $[0,1]$ 的**對準因子**，陣列增益與雙程路徑損耗全在 $\beta^2$ 內 → 本修正**只改角度結構，不動既有雷達方程式校準**。
>
> **⚠️ 仍存在的簡化（須揭露）**：目標的**真實幾何角**仍用於 CRB 的 $\cos\theta$ 項（那是「被估計的角度」本身，屬合理用法）。

> **⚠️ 2026-07-20：感測接收機雜訊改用 gNB 的雜訊指數**
> 回波在 **gNB** 接收，舊版卻沿用 $N_0^{comm}$（由 **UE** 的 NF=8 dB 算出）。
> 新增 $N_0^{sense}$：BS NF=5 dB（[T2] 3GPP TR 38.104 §7.2 BS 接收機參考靈敏度所隱含之 NF）
> $$N_0^{sense} = -174 + 10\log_{10}(100\times10^6) + 5 = -89\ \text{dBm}\quad(\text{對照 } N_0^{comm}=-86\ \text{dBm})$$
> 影響：感測 SNR 原**偏悲觀 3 dB**、CRB 偏大 3 dB；**選束不受影響**（所有候選同幅偏移，相對比較不變）。
- $1/d^{2n_{pl}}$：雷達雙向路損（去程 $1/d^{n_{pl}}$ × 回程 $1/d^{n_{pl}}$），程式中即 $20 n_{pl}\log_{10} d$。
  ⚠️ **注意這裡不是 $40\log_{10}d$**：$n_{pl}$ 取 LOS/NLOS 的 **CI 實測 PLE**（1.85 / 2.59，Shakya2025），
  只有在自由空間 $n_{pl}=2$ 時才退化為教科書雷達方程式的 $1/d^4$ = $40\log_{10}d$。
- $\sigma_{RCS} = 25\ \text{m}^2$（=**14 dBsm**）：車輛雷達散射截面。**出源 [T1] Motomura et al., IEEE WiSNet 2018**（79 GHz 車輛 RCS 實測；MultiUser 之引用 [17]，25 m² 在其實測範圍內）。⚠️ 實測為 79 GHz，採「光學區（>1–3 GHz）RCS 近似頻率無關」外推至 16.95 GHz（車體 ~4 m ≫ λ≈1.8 cm；理論支撐 Skolnik/Knott，非 FR3 直測）→ Q3 掃 σ∈{1,10,25,100} m²。

**匹配濾波後回波 SNR**：

$$
\mathrm{SNR}_{echo} = \frac{G_{mf}\, |\mathbf{a}^H \mathbf{f}|^2\, P_{tx}\, \beta^2}{N_0}, \qquad G_{mf}=10
$$

**量測抖動**：實際量測值 $\hat{P}_{echo} = P_{echo}\cdot 10^{\xi/10}$，$\xi\sim\mathcal{N}(0, \sigma_{dB}^2)$。

**（2026-07-20 修正）$\sigma_{dB}$ 改由匹配濾波增益 $G$ 推導，不再是自訂值：**

> **白話**：量測到的回波功率，是對 $G$ 個匹配濾波樣本取平均。取平均本身就有隨機起伏，樣本越多越穩——所以「量測有多不準」不是可以自由填的數字，而是**由 $G$ 決定的**。

複高斯訊號的功率估計服從 chi-square（$2G$ 自由度）：

$$\sigma_{rel} = \frac{1}{\sqrt{G}} \quad\Longrightarrow\quad \sigma_{dB} \approx \frac{10}{\ln 10}\sigma_{rel} = \frac{4.343}{\sqrt{G}}$$

$G = 10$（**[T1]** Zhao2024 Table 1 逐值核實）$\Rightarrow \boxed{\sigma_{dB} = 1.373\ \text{dB}}$

程式：`MEAS_NOISE_STD_DB = 4.343 / sqrt(MATCHED_GAIN_G)`

| | 舊 | 新 |
|---|---|---|
| 值 | 1.0 dB | **1.373 dB** |
| 等級 | **[T3]** 無推導的自訂值 | **由 [T1] 參數 $G$ 推導** |
| 耦合 | 與 $G$ 脫節 | $G$ 一改 $\sigma_{dB}$ 自動跟著改 |

⚠️ **與 Zhao2024 Eq.(9) 非重複計算**：Eq.(9) $\sigma^2_i=a_i\sigma^2/(G\cdot\mathrm{SNR}_{echo})$ 給的是**角度估計變異數（CRLB）**，已由 `computeCrlb` 以 `MEAS_A_COEF`/`MATCHED_GAIN_G` 實作；本項是**功率量測本身**的估計誤差。兩者是不同的量。

**NLOS 感測回波衰減（2026-07-19 定版：3GPP Rel-19 標準串接通道）**

感測通道 = (gNB→目標 子通道) ⊛ RCS ⊛ (目標→gNB 子通道)，每段用**與通訊相同的 CI 路徑損耗**（LOS $n{=}1.85$/NLOS $n{=}2.59$）。單站往返同距離 $d$：

$$
\beta^2 = \frac{N_t^2\,\lambda^2\,\sigma_{RCS}}{(4\pi)^3\, d^{\,2\,n_{pl}}}, \qquad n_{pl}=\begin{cases} n_{LOS}=1.85 & \text{LOS}\\ n_{NLOS}=2.59 & \text{NLOS}\end{cases}
$$

- $n_{pl}{=}2$（自由空間）時退化為 MultiUser 雷達方程式 $\beta^2\propto1/d^4$（相容 Eq.8b）。
- **「感測>通訊在 NLOS 脆弱」由往返兩段自然浮現**（通訊只單段 NLOS PLE）——非外加 proxy。
- 出源：[T2] **3GPP Rel-19 ISAC**（target channel concatenation，TR 38.901 對齊）；PLE [T1] Shakya2025（與 channelModelV2 同源）。

> **⚠️ 2026-07-19 重要更正——退掉舊「材質穿透 proxy（Option B）」**
> 舊版用「玻璃/混凝土/磚牆穿透損耗」當 NLOS 感測衰減，機制**錯配**：穿透損耗是「訊號**穿牆進室內**(O2I)」的量；本場景車在街道走「**繞樓** NLOS」，正確機制是 **NLOS PLE**（統計已含材質平均）。故改用上式標準通道。
> **附帶發現**：標準通道下感測 CRLB 中位 LOS 3.2e-7 / NLOS 1.2e-4 **皆 ≪ Γ=3e-3** → **FR3 30dBm/128天線強鏈路感測本就可靠、CRLB 閘門無鑑別力** → 獎勵退為純吞吐（感測輔助改由選束+觀測發揮）。此為**可寫入論文之發現**。
> 下方材質穿透值（玻璃42.3/煤渣磚15.0/混凝土46.7/鋼門58.5 dB，Shakya OJ-COMS 直測＋ITU-R P.2040）是**正確實測事實**，但**退為 Q3 選配**（O2I/額外遮擋敏感度），**不餵 baseline 感測**。

### 材質穿透值（Q3 選配參考，不入 baseline）

**1️⃣ 反射型 NLOS ρ=1.0（無穿牆）**
- **[T2] 3GPP TR 38.901**（Rel-16/18）：NLOS 多徑傳播模型；多路徑能量由反射/散射/繞射主導，無額外穿牆損耗
- **物理依據**：gNB→車線段穿過建物邊界但車位於街道，反射多徑來自街廓內部和周邊；LOS 與 NLOS 的差異已納入 Shakya2025 PLE（LOS=1.85, NLOS=2.59）
- **程式對應**：`isLos()` 判定後，若 NLOS 但車在主街/側街反射區 → 選此值

**2️⃣ 玻璃幕牆 −42.3 dB 單程（ρ ≈ 5.9×10⁻⁵）**
- **[T1] Shakya et al., IEEE Open J. Commun. Soc., vol.5, pp.5192-5213 (2024), DOI 10.1109/OJCOMS.2024.3431686**：*Comprehensive FR1(C) and FR3 ... Material Penetration Loss Measurements ...*（arXiv:2405.01362 / GLOBECOM 版的期刊完整版）
  - Table 8 直接實測：FR3 16.95 GHz，Low-E tinted glass wall 3cm 穿透損耗 **42.3 dB 單程**（σ=0.2）
  - 對比：同材料 6.75 GHz 為 33.7 dB → 頻率越高損耗越大
- **程式對應**：街廓材質為「glass」→ 選此值

**3️⃣ 煤渣磚牆 −15.0 dB 單程（ρ ≈ 3.2×10⁻²）**
- **[T1] Shakya et al., IEEE OJ-COMS 2024**（同上，DOI 10.1109/OJCOMS.2024.3431686）
  - Table 8 直接實測：Cinderblock wall（煤渣空心磚）22cm @ 16.95 GHz 穿透損耗 **15.0 dB 單程**（σ=1.1；6.75 GHz 為 13.4 dB）
  - **2026-07-18 修正**：舊值 −58 dB 有雙重錯誤——① 取自 Vargas-Mello 28 GHz（非 16.95 GHz），② 58 dB 恰為本篇 Table 8「鋼門」值，與磚牆混淆。現改為同頻段直測煤渣磚 15.0 dB。
- **程式對應**：街廓材質為「brick」→ 選此值

**4️⃣ 混凝土 −46.7 dB 單程（ρ ≈ 2.1×10⁻⁵，ITU-R P.2040 標準公式 @ 15cm 牆）**
- **[T2] Rec. ITU-R P.2040-3/-4, Table 3**：混凝土 $\varepsilon_r'=5.24$（$b=0$，頻率無關）、$\sigma=0.0462\,f^{0.7822}$ S/m（$f$ in GHz，有效 1–100 GHz）
  - @ 16.95 GHz：$\sigma=0.423$ S/m → 複介電常數 $\varepsilon_r=5.24-j0.448$ → **吸收率 301.9 dB/m**
  - 穿透損耗 $= 301.9\cdot t\,(\text{m}) + \text{界面反射損}(\approx1.5\text{ dB})$，$t=0.15$ m → **46.7 dB**（計算稿 `scratchpad/p2040_concrete.m`，已驗證）
- **關鍵：混凝土穿透無單一值，完全取決於牆厚**（$t$=15cm 為常見 RC 牆/樓板）：

  | 厚度 | 穿透損耗 |
  |---|---|
  | 10 cm | 31.6 dB |
  | **15 cm** | **46.7 dB** |
  | 20 cm | 61.8 dB |

- **交叉驗證**：舊估 45 dB ≈ 15cm、Liu2024（arXiv:2412.08752）薄片外插 25.9 dB ≈ 8cm，皆與本公式一致 → 三方互證。
- **用途**：此為「穿牆」O2I 值，供 Q3 選配；baseline 感測 NLOS 走標準 CI 通道（見上）。Q3 掃厚度 10/15/20cm（`CONCRETE_SWEEP_THICKNESS_M`）

**5️⃣ 鋼門/高衰減選項 −58.5 dB 單程（ρ ≈ 1.4×10⁻⁶）**
- **[T1] Shakya et al., IEEE OJ-COMS 2024**（同上）：Table 8 Steel door 4.7cm @ 16.95 GHz = **58.5 dB**（全表最大衰減）
- **程式對應**：Q3 需「幾乎完全遮蔽」極端材質時 → 選此（勿與磚牆混用）

### baseline 感測不依材質（標準通道）＋歷程備查

**baseline 感測 NLOS 衰減由標準 CI 通道給出（見上式），不讀街廓材質。** 材質表 `BUILDING_MATERIALS`（config 預設全 concrete）僅供 Q3「若牆體額外遮擋(O2I)」選配。

> **歷程備查（2026-07-19）**：曾有「Option B — NLOS 感測吃逐棟材質穿透衰減」以逼 λ-gate 觸發（材質判定邏輯 myStep §1b），但該 proxy **物理錯配**（穿透=O2I、非繞樓 NLOS）。改標準 CI 通道後揭露**感測本就可靠**（CRLB 恆 ≪ Γ）→ 退掉 Option B 與 λ-gate、獎勵改純吞吐。詳見上方更正框與 SPEC 1.4/1.7。

**SU 寬多波束感測（`env/myStep.m` action=4，MultiUser Wang2024 Eq.17）**
- **用途**：波束**失準**（agent 迷失/NLOS）時，發射「寬多波束」$\mathbf f=\sum_{i\in\text{window}}\mathbf F(:,i)\,/\,\lVert\cdot\rVert$（±`MULTIBEAM_WIN` 束疊加）一次照亮多方向、擴大感測視野，代價=峰值增益降低（通訊變差）。
- **驗證（失準 k 格，單束 vs 多波束 CRLB）**：完美對準 k=0 單束優（7.7e-6 vs 6.2e-5）；失準 k=1–4 **多波束遠優**（5e-5 vs 1.5e-3）——正是「迷失時用寬波束穩定重新捕捉」的用途。
- ⚠️ **Part 2 佈線**：action=4 目前僅 env 支援；要讓 DRL agent 可選需更新 `agent/*.m` 的 `rlFiniteSetSpec`（現為 [0 1 2 3]，未含 4）。

### ②½ 程式對應（`env/echoSignalV2.m`）
| 公式 | 位置 |
|---|---|
| $\beta^2=N_t^2\lambda^2\sigma_{RCS}/((4\pi)^3 d^{2n_{pl}})$ 串接目標通道（來回 CI PLE $n_{pl}$；$n{=}2$ 退化為雷達方程式 $1/d^4$）| `n_pl = nLos*isLos + nNlos*~isLos; betaSq = nAnt^2*lambda^2*sigmaRcs/((4*pi)^3*d^(2*n_pl))` |
| **LOS/NLOS PLE 選擇（3GPP Rel-19 串接）** | `echoSignalV2` §3；PLE 由 myStep 傳入（`nLos=1.85/nNlos=2.59`，與通訊同源）|
| $P_{echo}=P_{tx}\lvert\mathbf{a}^H\mathbf{f}\rvert^2\beta^2$（一次方） | `P_echo_ideal = ptx * bfGain * betaSq` |
| $\mathrm{SNR}_{echo}$（匹配濾波 $G_{mf}$） | `snrEchoIdeal = matchedGain * bfGain * (ptx*betaSq) / n0` |
| 量測抖動 $\xi\sim\mathcal{N}(0,1\text{dB}^2)$ | 同檔（`measStdDb` 乘 randn）；參數 `config.m` `MEAS_NOISE_STD_DB` |
| **$\sigma_{RCS}=25$ 參數** | **`config.m` `SIGMA_RCS`**（經 `myStep.m` 傳入 echoSignalV2） |
| $G_{mf}=10$ 參數 | `config.m` `MATCHED_GAIN_G` |

### ③ 出處
- **MultiUser Eq.6/8b/10**（$\beta\propto\sqrt{\sigma_{rcs}/d^4}$、發射增益 $|\mathbf{a}^H\mathbf{f}|^2$）：[[notes/MultiUserBeamforming_DRL_Sensing]]（原文：`D:\PROJECTS\papers\核心論文\MultiUserBeamforming_DRL_Sensing\`）
- **Zhao2024 Eq.2/6**（數位接收 $b(\theta)a^H(\theta)f$ + 匹配濾波）：[[notes/Zhao2024_IBTD_ISAC_V2V]]
- **NLOS 感測衰減（baseline，標準通道）**：[T2] **3GPP Rel-19 ISAC** target channel concatenation（TR 38.901 對齊）——來回兩段 CI PLE；PLE 值 [T1] Shakya2025（與 channelModelV2 同源）。
- **材質穿透值（Q3 選配，不入 baseline）**：[T1] **Shakya et al. IEEE OJ-COMS 2024**（vol.5, pp.5192-5213, DOI 10.1109/OJCOMS.2024.3431686）Table 8 直測：玻璃 42.3/煤渣磚 15.0/鋼門 58.5 dB @16.95GHz；[T2] **ITU-R P.2040-3/-4** 混凝土 εr'=5.24、σ=0.0462·f^0.7822 → 15cm=46.7dB。供 O2I/額外遮擋敏感度用；[[notes/Shakya2024_Comprehensive_FR3_Penetration]]。
- 雷達方程式基礎：Skolnik, *Introduction to Radar Systems*
- FR3 接軌：Multi-Band Sensing in FR3（arXiv 2602.23539，1/d⁴）

### ④ 檢驗
```matlab
runtests('tests/testEchoSignalV2.m')  % 2 測試：P∝|a^Hf|² 一次方、P/SNR 同次方
% 手動：對固定車位掃全碼本 → echo 應單峰（對準最強）
```

---

## 7. 方位角估計的 Cramér–Rao 下界（computeCrlb.m）

> **這在幹嘛：** 用回波估車的角度，最準能估到多準的理論極限——回波越強估得越準(CRLB 越小)，DRL 用它當「我還追得住車嗎」的信心指標。

### ① 這是什麼
「以目前的回波品質，角度最準能估到多準」的**理論下界**。回波越強 → 界越小 → gNB 越「看得清」車。DRL 用它當「我還追得住嗎」的信心特徵。

> **⚠️ 2026-07-20 重大修正：由 proxy 升級為真正的 CRLB**
> 舊版是「受 CRLB 啟發的簡化 proxy」$\mathrm{CRLB}=a_1/(N_t\cdot\widehat{\mathrm{SNR}}_{echo})$，即 $\propto 1/N$。
> 該式**與理論及所引文獻兩邊都不符**，三個錯疊在一起：
>
> | 缺陷 | 說明 |
> |---|---|
> | (a) **$N$ 的次方錯誤** | 測角 CRB $\propto 1/N^3$——角度解析度來自**孔徑**，Fisher 資訊隨元件位置平方和增長（$\sum m^2=O(N^3)$）。舊式為 $1/N$ |
> | (b) **$N$ 用錯對象** | 舊式代入**總天線數 128**；決定**方位**解析度的是**方位孔徑 $N_{az}=16$**。總陣列增益 128 已含於 $\widehat{\mathrm{SNR}}_{echo}$，再當孔徑是語意重複 |
> | (c) **與所引來源不符** | Zhao2024 Eq.(9) 為 $\sigma_i^2=a_i\sigma^2/(G\cdot\mathrm{SNR})$，**式中無 $N$**；舊式卻多除一個 $N_t$ |
>
> 量級影響：$N_{az}=16$、$d/\lambda=0.5$、broadside 時，舊式較新式**高估約 17 dB**。

### ② 公式

$$
\mathrm{CRB}(\theta) = \frac{6\,a}{\widehat{\mathrm{SNR}}_{echo}\cdot(2\pi d/\lambda)^2\cdot \cos^2\theta\cdot N_{az}(N_{az}^2-1)} \quad [\text{rad}^2]
$$

- $N_{az}=16$：**方位孔徑元件數**（非總天線數 128）
- $\cos^2\theta$：掃描角偏離 broadside 時**有效孔徑投影縮短** → 解析度下降
  ⚠️ 程式須傳 $|\cos\theta|$：$|az|>90°$（陣列後方）時 $\cos(az)$ 為負，直接傳入會被下限夾住而失去角度相依性
- $a$：**實務估計器相對於 CRB 的劣化倍率**（$\ge 1$，預設 1 取理論下界）。保留 Zhao2024 Table 1 的係數介面，語意由「比例常數」改為「劣化倍率」
- 保留原設計：由**量測**回波 SNR 推得（基地台真的量得到的量），非由真實角度反推

**效果**：$\sqrt{\mathrm{CRB}}$ 中位由 0.57° 降為 **0.114°**——16 元件方位孔徑於高 SNR 下的合理量級（舊式偏大 5 倍）。觀測邊界 `CRLB_RANGE` 連帶重新校準為 $[10^{-11},10^{-1}]$。

> **⚠️ 報告禁令**：CRB 是**下界**而非實際估計器精度。可用作**相對**感測品質指標，
> **不得**直接宣稱「本系統測角精度 0.114°」。

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| $\mathrm{CRB}=6a/[\widehat{\mathrm{SNR}}_{echo}(2\pi d/\lambda)^2\cos^2\theta\,N_{az}(N_{az}^2-1)]$ | `env/computeCrlb.m` | 傳入 `ntCol`（方位孔徑）與 `abs(cosd(az))` |
| 數值夾限 $[10^{-14},10^6]$（下界放寬以容納新量級）| `env/computeCrlb.m` | — |
| 門檻 $\Gamma_{CRLB}=3\times10^{-3}$ | `config.m` | `GAMMA_CRLB` |
| 被誰呼叫 | `env/myStep.m`（每步由量測 echo SNR 算） | — |
| 量測 echo SNR 的 log 欄位（2026-07-21 補） | `loggedSignals.snrEchoMeas_t` | 線性、無單位；與 `P_echo_t`（W）不可互代 |
| 儀表板回波 SNR 面板 | `animateEnvDashboard.m` | 讀 `snrEchoMeas_t`；缺欄位改為報錯，不再以回波功率 fallback |

### ③ 出處
- **一般化 CRB 矩陣式**：[T1] P. Stoica & A. Nehorai, MUSIC, maximum likelihood, and Cramér-Rao bound, *IEEE Trans. ASSP*, **37(5):720–741, May 1989**, **DOI 10.1109/29.17564**
  ⚠️ **2026-07-21 勘誤**：舊版標 `10.1109/29.32276` —— 該 DOI 實際指向 **Roy & Kailath, ESPRIT, IEEE Trans. ASSP 37(7):984–995, 1989**，是另一篇論文。
  ⚠️ **避免過度歸因**：該文給的是**一般化矩陣形式**的隨機 CRB，**並未寫出 $1/N^3$ 或 $1/[N(N^2-1)]$ 的顯式標量式**。
- **本節的 ULA 顯式式**：由上述一般式代入 **ULA、單源、$d=\lambda/2$** 特例化推得（關鍵在 $\sum m^2 = N(N-1)(2N-1)/6 = O(N^3)$）。同型結果亦見 [T3] ISAC 綜述 arXiv:2104.09954（$\mathrm{CRB}_\omega \approx 6/(N^3 L\,\mathrm{SNR})$）
- [T1] Van Trees, *Optimum Array Processing* (Part IV), Wiley 2002, **Ch. 8 Parameter Estimation I: Maximum Likelihood** 之 CRB 小節
  ⚠️ 原標「§8.2」**查無一手佐證**（二手資料對節號說法不一），改標章別，待有紙本再補
- 形式對齊：[[notes/MultiUserBeamforming_DRL_Sensing]] Eq.10、[[notes/Zhao2024_IBTD_ISAC_V2V]]（估測誤差 $\propto a_i/(G\cdot\mathrm{SNR})$）
- FR3 CRB 分析：ISAC meets FR3（arXiv 2605.18120 Fig.3）

### ④ 檢驗
```matlab
% CRLB 應與 echo 完全反向：掃全碼本畫 CRLB → 對準時谷底、失配時峰
% testFR3Env/testEchoSensing 斷言「對準 CRLB < 失配 CRLB」與「CRLB-SNR 相關 < -0.9」
```

---

## 8. 通訊 SNR 與連線判定（myStep.m + linkSuccess.m）

> **這在幹嘛：** 算車實際收到的訊號品質(SNR)，並判斷「這一刻算不算連得上」——頻譜效率超過門檻才算連線成功、才能傳資料。

### ① 這是什麼
車收到的實際訊號品質，以及「這一刻能不能傳資料」的判定。注意 echo（感測）**不是**連線閘門——它是預測器；連線看的是通訊 SNR。

### ② 公式

**通訊 SNR**（等效通道 = 通道向量投影到所選波束）：

$$
\mathrm{SNR} = \frac{P_{tx}\,\big|\mathbf{h}^H \mathbf{f}\big|^2}{N_0}\cdot 10^{-L_{sw}/10}
$$

$L_{sw}$ 為換波束瞬間損耗。**本專題 $L_{sw}=0$ dB、切換延遲 $=0$ TTI**（2026-07-20 查證後歸零——
舊值 0.5 dB / 1 TTI 無任何出處，且文獻中並不存在「固定 dB 懲罰」這種形式；詳見下方 ③ 出處說明）。
式中保留該項是為了讓公式結構完整、日後若找到出處可直接接上。

**連線成功（outage 標準定義）**：

$$
\text{link} = \mathbb{1}\!\left\{ \log_2(1+\mathrm{SNR}) \ge C_{th} \right\},\qquad C_{th} = 2\ \text{bps/Hz} \Leftrightarrow \mathrm{SNR} \ge 3\ (4.8\ \text{dB})
$$

懸崖式門檻：過線即成功、差一點即失敗，無部分分數。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| 通訊 SNR $=P_{tx}\lvert\mathbf{h}^H\mathbf{f}\rvert^2/N_0$、切換損 $L_{sw}$ | `env/myStep.m`（snrNow 計算段） |
| 連線判定 $\log_2(1+\mathrm{SNR})\ge C_{th}$ | `env/linkSuccess.m`（單行邏輯） |
| 門檻 $C_{th}=2$【主值】、**$L_{sw}=0$ dB、切換延遲 $=0$ TTI**（2026-07-20 查證後歸零，見下方說明）| `config.m`（`cThreshold`、消融 `cThresholdSweep=[1 2 4]`、`BEAM_SWITCH_LOSS_DB`、`BEAM_SWITCH_DELAY`） |
| 延遲語意（2026-07-21 定義）：request 當下算第 1 個等待 TTI；`delay=N` 時第 N 個 TTI 採用**最新** pending 束且不計損 | `env/myStep.m`；實際損失記於 `loggedSignals.beamSwitchLoss_t` |
| 選束決策 vs 實際服務束（2026-07-21）：`sweepSelectedBeam`＝本 TTI 選出的束；`beam_idx`＝實際服務束，delay>0 時可合法落後 | validator 的 metric 檢查用前者，tracking KPI 用後者 |
| **驗證有兩個互不隸屬的 gate**：`OverallPass`（environment mechanics，逐 seed）與 `PropagationPass`（physics/statistics，跨 seed）。引用結果須同時列出，OverallPass 通過**不代表**傳播物理已驗證 | `validateFr3Environment.m` 結尾同時輸出兩者 |
| 有效速率公式已抽為獨立函式（2026-07-21，純重構、行為不變） | `env/effectiveRate.m`；封閉解錨點測試 `tests/testEffectiveRate.m` |
| **三扇區幾何基礎（S0，2026-07-22）**：面板法線 [0,120,-120]°，服務扇區 `argmin_s |wrapTo180(azWorld-boresight)|`（平手取最小索引），`azRel` 保證 `|azRel|≤60°`。**僅建立 geometry primitives，尚未接線到 channel/beam/element pattern/myStep/oracle/validator**；不使用 hysteresis、不讀通道功率 | `env/sectorGeometry.m`、`env/azToSector.m`、`env/beamIdxToGridIdx.m`；SPEC §1.56 |
| **方位方向圖 opt-in（S1，2026-07-22）**：`arrayResponse` 可選第 7 參 `applyAzPattern`（預設 false）。false 逐元素等同現況；true 時 `‖a‖²=N·gElem(el)·gAz(az)`，`gAz`＝`elementGainAz`（3GPP TR 38.901 Table 7.3-1 A_EH，65°/30dB，公式未改，線性功率）。**預設關閉、未接線環境主路徑，Part 1A/v3 數值不變** | `env/arrayResponse.m`；`tests/testElementGainAz.m`、`tests/testArrayResponse.m`；SPEC §1.56 |
| **三扇區通道合成接線（S2，2026-07-22）**：`myReset`/`myStep` 每 TTI 由 UE 幾何選 serving sector，對每條 path 以 `azRelPath=wrapTo180(az_tx-boresight)` + 方位方向圖合成 `hChan`；`channelModelV2` 仍輸出世界角。`echoSignalV2` 加 `panelBoresightDeg`，CRLB 用 `azRel`。前後鏡像消除（V1：east panel 對 west UE 低 ≥25 dB）。**這是第一個改變 SNR/beam KPI 的階段** | `env/myReset.m`、`env/myStep.m`、`env/echoSignalV2.m`；`tests/testSectorChannel.m`、guard `testElementPattern.m`；SPEC §1.56、登記簿 F-4a |
| **sector-aware validation + post-sector v4（S3，2026-07-22）**：validator oracle 只在 serving `logs.F`（128 束）argmax，無 384 全站；新增三扇區 KPI-only 診斷。**v4（30 seeds）GATE 1 30/30、GATE 2 PASS（CI 下界 9.99 dB、正 gap 100%）、V1–V5 綠**。v3 pre-sector vs v4 post-sector 不可混合：LOS SNR 44.9→41.2、NLOS 32.6→28.2 dB、連線率 99.7→98.8%、Reff 711→688 Mbps（前後鏡像修正的預期代價）。⚠️ **obs range 重校（S4）仍待做，正式 RL 訓練未開始** | `validateFr3Environment.m`；`results/Part1_Validation_v4/`；SPEC §1.56、登記簿 F-4a |

> **⚠️ 波束切換代價歸零（2026-07-20，查證後修正）**
> 舊值 $L_{sw}=0.5$ dB、延遲 1 TTI **無任何出處**，與已修正的 $\rho$ 舊值同性質。查證結果：
> - **三篇核心論文全文 grep 零命中** → 皆未建模波束切換延遲/損失
> - **[T2]** TS 38.214 / 38.306 `beamSwitchTiming` = sym14/28/48（30 kHz SCS ≈ **0.5–1.7 ms**）；
>   TS 38.133 §8.10 TCI state switch delay ≈ **3 ms 級** → 真實延遲 **< 1 TTI**，
>   舊值 1 TTI (10 ms) **量級高估 3–20 倍**
> - **[查無]** 「切換期間打固定 dB 折扣」此形式在規範與文獻中**不存在**；一致做法為
>   **時間軸中斷／作廢符元**（TS 38.133 稱 interruption；機制為切換過渡超出 CP 長度
>   導致 EVM 惡化、該段符元無法解碼），非常數 dB 衰減
>
> → 兩者歸零。⚠️ 日後若把 TTI 細化至 NR slot 粒度（0.5–1 ms），應改以「損失 $N$ 個符元」
> 的形式重新引入，$N$ 依 `beamSwitchTiming` 或 TS 38.133 §8.10 折算並標 T2 出處。
>
> **連帶**：切換代價歸零後，「波束抖動 (ping-pong)」的代價來源消失（選錯束本就會被 SNR
> 自然懲罰），故評估過的 **3GPP L3 量測濾波（TS 38.331 §5.5.3.2）維持關閉**
> （`L3_FILTER_K = 0`，規範 NOTE 1 允許）。兩個獨立實驗亦一致顯示重濾波有害——
> 因**每束的更新率等於掃描週期**而非每 TTI，重平滑等於拿舊資料做決策。
> 程式碼保留但不啟用，記為「已評估、因根因未確立而未採用」。

### ③ 出處
- **判定形式**（outage=SE<門檻）奠基：Ozarow, Shamai, Wyner, IEEE TVT 1994；Goldsmith 2005 / Tse & Viswanath 2005 —— 只支持**定義**，不支持某特定門檻值。
- **門檻值**：**QoS 設計參數，非文獻規定**。主值 $C=2$、消融 $\{1,2,4\}$（2026-07-16，前身 C=4）。無單一權威可強制：3GPP V2X (TS 22.186) 以 PER/Mbps 表達、CQI-1 可解調下限≈0.15 bps/Hz。$C=4$ 原借自 MultiUser Table I（28GHz/15dBm 高品質服務門檻），修正 N₀ 後對本預算偏嚴，改主值 $C=2$（實測 baseline LOS 88%/NLOS 60%，環境合理）。

### ④ 檢驗
```matlab
% 主值 C_th=2 的邊界：SNR=3→過（log2(4)=2）、2.9→不過
linkSuccess(3, 2), linkSuccess(2.9, 2)

% 門檻是傳入參數而非寫死，故測試以 seThreshold=4 驗證邊界同樣合法
runtests('tests/testLinkSuccess.m')   % 邊界：SNR=15→過、14→不過（此處 4 為測資，非主值）
```

---

## 9. 吞吐量（full-buffer 速率式）

> **這在幹嘛：** 算「這一步實際傳了多少資料」——連上才有流量，且要扣掉掃描佔用的時間，這就是獎勵函數裡「掃太多會虧」的來源。

### ① 這是什麼
「這一步實際傳了多少資料」。full-buffer = 假設永遠有資料要傳（RL 訓練標準假設），送達量直接反映波束品質，不被人造的封包佇列動態干擾。

### ② 公式

$$
R_{eff} = (1-\rho_a)\, B \cdot \min\!\big(\log_2(1+\mathrm{SNR}),\ \mathrm{SE}_{\max}\big)\cdot \mathbb{1}\{\text{link}\}, \qquad \text{served}[t] = R_{eff}\,\Delta t
$$

$\rho_a$ = 動作 $a$ 的掃描開銷（掃描佔用時頻資源 → 直接砍吞吐——「掃太多會虧」的機制來源）。

> **⚠️ 頻譜效率上限 $\mathrm{SE}_{\max}$（2026-07-20 新增，原式無上限）**
>
> **白話**：Shannon 公式 $\log_2(1+\mathrm{SNR})$ 沒有天花板——SNR 越高算出來的速率就越高。但真實系統受**調變階數與碼率**限制，再好的通道也只能傳到某個速率為止。不設上限等於「用現實中不存在的調變在傳資料」。
>
> $$\mathrm{SE}_{\max} = 8 \times \frac{948}{1024} = 7.4063\ \text{bps/Hz}$$
>
> **出源**：**[T1] 3GPP TS 38.214 Table 5.1.3.1-2**（MCS index table 2，適用 256QAM）——最高 **MCS 27**：調變階數 8（256QAM）、目標碼率 948/1024。單層單碼字。
>
> **為何必須加（實測 28,699 個連線 TTI）**：
>
> | 指標 | 未截斷時 |
> |---|---|
> | 超過 7.4063 的連線 TTI 比例 | **47.0%** |
> | 平均 SE | 7.03（截斷後 6.23）|
> | **吞吐量高估** | **12.8%** |
> | 最大 SE | **17.7 bps/Hz = 上限的 2.4 倍**（對應 SNR 53.3 dB）|
>
> 亦即：報告中近半數時間的吞吐是靠不存在的調變達成的。峰值 17.7 bps/Hz 一經換算即可拆穿。
>
> **連帶：$R_{\max}$ 改為物理上限**
> 舊值 $R_{\max}=B\log_2(1+10^4)=1329$ Mbps（假設最大 SNR 40 dB），但實測 SNR 達 **53.3 dB**、$R_{eff}$ 達 1762 Mbps → 獎勵的吞吐項 $R_{eff}/R_{\max}$ 曾達 **1.33**，正規化溢出且與註解宣稱不符。
> 現改為
> $$R_{\max} = B\cdot \mathrm{SE}_{\max} = 740.6\ \text{Mbps}$$
> 使 $R_{\max}$ **同時是物理上限與正規化基準**（本就該是同一個值）→ $R_{eff}/R_{\max}\le 1$ 由建構保證（實測最大 0.995）。
>
> **⚠️ 一致性要求**：`env/myStep.m` 與 `agent/rewardFn.m` **兩處都要截斷且必須用同一個上限**，否則 agent 會為一個環境不會給的速率而優化（獎勵與指標脫節，是 RL 最難查的 bug 之一）。防回歸測試：`tests/testFR3Env.m::testSpectralEfficiencyCap`。

**（2026-07-19 修正）$\rho$ 改由 3GPP 符元預算推導，不再是硬填值**：

$$
\rho_a = \frac{N_{\text{swept}}(a)\cdot T_{\text{sym}}}{T_{\text{TTI}}},\qquad T_{\text{sym}} = \frac{1+\text{CP}}{\text{SCS}} \approx 17.84\ \mu s\ (\text{SCS}=60\ \text{kHz}),\quad T_{\text{TTI}}=10\ \text{ms}
$$

| 動作 | 語意 | 掃描束數 $N_{\text{swept}}$ | $\rho_a$ |
|---|---|---|---|
| 0 | Keep | 0（僅週期 CSI-RS 量測）| 0.005 |
| 1 | Scan ±1（P-2/P-3 細化）| 3 | 0.0054 |
| 2 | Scan ±2（P-2/P-3 細化）| 5 | 0.0089 |
| 3 | 全掃 128 束（P-1）| 128 | **0.2284** |
| 4 | 寬多波束 | 5 | 0.0089 |

**出處**（2026-07-19 查證修正）：

| 成分 | 級別 | 出處 |
|---|---|---|
| 開銷折扣式 $(1-T_{train}/T_{frame})\cdot R$ | **T1** | Hussain & Michelusi, "Throughput Optimal Beam Alignment in Millimeter Wave Networks", arXiv:1702.06152, **§III 式(2)**：$r_L=\frac{N-L}{N}\int_{A_L}S_L(\theta)d\theta\log_2(1+\mathrm{SNR})$，$(N-L)/N$ = frame 內扣掉 $L$ 個對準 slot 後剩給資料的比例 |
| 每束 = 1 OFDM 符元 | **T2** | 3GPP TS 38.211 §7.4.1.5.3 **Table 7.4.1.5.3-1 Row 1**（1-port CSI-RS、density 3 RE/PRB、noCDM → 佔 1 符元）＋ TS 38.331 `repetition='ON'`（set 內各 resource 同 spatial filter、分佈於不同符元）|
| 全掃時間預算校驗 | **T2** | TS 38.213 §4.1（SSB burst ≤5 ms、$L\le$64）＋ TS 38.211 §7.4.3（1 SSB = 4 符元）→ 每束 78 µs ≈ 4 符元 71 µs ✓ |
| 碼本 128 束 | **T1** | $N_{beams}=N_{ant}=128$ 為 128 元件 UPA 的**正交 DFT 基底大小**，由陣列自由度決定、非自由參數。Tse & Viswanath, *Fundamentals of Wireless Communication*, Cambridge 2005, §7.3.1, pp.318–322；Balanis, *Antenna Theory* 4th ed., Wiley 2016, §6.3（程式定義：`dftCodebook.m:15-18`）|
| 128 束 vs 3GPP $L\le64$ | **T2** | **不同量，非衝突**：$L\le64$（TS 38.213 §4.1）約束「一個 SSB burst 於 5 ms 內塞得下幾個**廣播**波束」＝**時間預算**；陣列波束數＝**空間自由度**。細化用 CSI-RS 可配多個 `NZP-CSI-RS-ResourceSet`（每 set ≤64、RRM ≤96，TS 38.331），不受 $L$=64 封頂。業界佐證：Verizon 28 GHz 每 cell **8 SSB + 64 CSI-RS 細化波束** |
| 128 元件於 FR3 的合理性 | **T2** | 16.95 GHz → $\lambda$=1.77 cm、$\lambda/2$=8.85 mm，16×8 陣列實體僅 **14.2 × 7.1 cm**（小於手機面板）；上中頻段主打「極大規模 MIMO（數百天線）」(arXiv:2502.17914；Nokia 7–15 GHz 白皮書) → **128 屬保守下緣** |

> **⚠️ 引用勘誤（本文件初版）**：曾將開銷折扣式歸於 Giordani et al., *IEEE COMST* 2019 (DOI 10.1109/COMST.2018.2869411)。經查證**無法證實該 tutorial 含此式**，已改引 Hussain & Michelusi arXiv:1702.06152 式(2)。另曾誤引 TS 38.214 §5.1.6 為「每束 1 符元」出處，正確為 TS 38.211 Table 7.4.1.5.3-1。

> **⚠️ 已知建模限制（報告須揭露）**
> **(i) 分母取單一 TTI（10 ms）而非 Hussain 原式的 frame** —— 與本專題「每 TTI 一個 RL 決策」的粒度自洽，但若實際 CSI-RS 週期為 20/40 ms，本式**高估** $\rho$。查無文獻指定分母，此選擇屬 T3。
> **(ii) 未計 P-3（UE 側 Rx 波束掃描）** —— 每個 Tx 束應再乘 $N_{rx}$ 個符元，方向相反，使本式**低估** $\rho$。
> (i)(ii) 部分互相抵消，整體視為同數量級的合理估計。

**⚠️ 為何改**：舊值 $\{0.05,0.25,0.50\}$ 出自計畫書 Table 3 之**自訂設定，無任何文獻依據**（知識庫查證：三篇核心論文均無顯式 $\rho$ 式；`關鍵公式與參數總表` 出處欄為空），違反本專題「禁止無出源經驗值」規範。更關鍵的是舊值把「±2 細化(5 束)」與「全掃(128 束)」同定價 0.50，**抹平了輕量追蹤與重量重獲取的代價差**，使「以感測換取開銷下降」的研究命題在環境層面不成立。新式下全掃比細化貴 **25 倍**，恢復了該取捨。

次要指標（給導師看封包/可靠度面，不入獎勵）：累積送達 $\sum \text{served}$、連線成功率 $\frac{1}{T}\sum \mathbb{1}\{\text{link}\}$。

### ②½ 程式對應
| 內容                                                                                | 檔案                                        |
| --------------------------------------------------------------------------------- | ----------------------------------------- |
| $R_{eff}=(1-\rho)B\log_2(1+\mathrm{SNR})\cdot\mathbb{1}\{\text{link}\}$、served 累積 | `env/myStep.m`（Throughput 區塊）             |
| $\rho_a$ 開銷表（由 `BEAMS_SWEPT*T_SYM/dt` 推導）                                      | `config.m`（`BEAMS_SWEPT = [0 3 5 128 5]`）  |
| 次要指標初始化（served_cum/link_success_cum）                                              | `env/myReset.m`                           |

### ③ 出處
- 波束追蹤 DRL 標準獎勵=SE/速率式：EURASIP JWCN 2022（DOI 10.1186/s13638-022-02191-7）；Ye & Ge 2023, *Nature Sci Reports*（計畫書 ref[7]）
- full-buffer 慣例：arXiv 1611.06497；計畫書 Eq.9/11

### ④ 檢驗
```matlab
runtests('tests/testThroughput.m')  % served_cum ≡ Σ R_eff·dt（不變量守門）
```

---

## 10. RLF 與 Episode（不終止、穿越制）

> **這在幹嘛：** 定義「一回合什麼時候結束」（車駛離場景才結束，斷線不提前終止）並統計連線失敗事件——避免 agent 學會「躲 NLOS」而不是「進去後怎麼救」。

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

> **這在幹嘛：** 讓車在地圖上動——沿街走、路口擲骰子決定直行/左轉/右轉、轉彎走圓弧減速、駛離場景就結束回合。

### ① 這是什麼
Manhattan 移動模型：車沿路走、路口擲骰子（直行 0.5 / 左 0.25 / 右 0.25）、轉彎走圓弧減速、直行吸附車道中心線、駛離場景回報 episode 結束。

### ② 公式
位置更新（heading $\psi$、速度 $v$）：

$$
\mathbf{x}[t+1] = \mathbf{x}[t] + v\,\Delta t\,[\cos\psi, \sin\psi, 0]
$$

轉彎：**近似圓弧**（實作為「以 $\Delta\psi=\dfrac{0.6\,v\,\Delta t}{r}$ 的速率旋轉 heading 並前進」，$r=6$ m 只設定轉向快慢，非繞固定圓心的精確 6m 半徑圓）、減速因子 0.6。速度集合 $\{30,45,60\}$ km/h。⚠️ 轉彎半徑 6m/減速 0.6/轉向機率 0.5-0.25-0.25 皆為 **T3 設計選擇**（非標準規定；經典 Manhattan 慣例）。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| 位置更新、路口轉向決策、轉彎弧、車道吸附、駛離判定 | `env/vehicleMobility.m`（全檔；道路座標內部讀 `sceneGeometry()`） |
| 本步是否採轉彎速度（第 5 輸出 `isTurningStep`，2026-07-21 補） | `env/vehicleMobility.m` → `loggedSignals.isTurning_t` |
| 轉彎兩階段閉合（2026-07-21）：均勻 `dTheta=90/N` 主轉彎 + 半正弦出口修正段，每步位移恆為 `vCurr·dt`，無吸附/teleport | `env/vehicleMobility.m`；測試 `tests/testTurnClosure.m` |
| 轉彎階段單一判定 `isTurnPhase(state)`（1000–3999，含修正段） | 同上；兩階段皆減速、皆記 `isTurningStep=true`、皆不吸附 |
| 速度集合/轉向機率 | `config.m`（`vRange`、`pStraight/pLeft/pRight`） |
| 進入點抽選與初始 heading | `env/myReset.m` |

### ③ 出處
- Manhattan Mobility Model（廣泛採用的都會移動模型；⚠️ 非 SUMO/3GPP 強制標準，路口轉向為抽象化）
- 路口轉向機率 0.5/0.25/0.25：[T3] 經典 Manhattan 模型慣例值，計畫書 §步驟一(L130)一致（直行偏多符合真實車流）

### ④ 檢驗
```matlab
runtests('tests/testVehicleMobility.m')  % 5 測試：吸附/前進/駛離/進入點/gNB固定
% 手動：跑 3 個 seed 畫軌跡 → 全程貼路、轉彎是弧
```

---

## 11½. 獎勵權重、動作空間、觀測初值（2026-07-20 補記）

> **這在幹嘛：** 這節收的是「不在任何物理公式裡、但會直接決定 DRL 學到什麼」的三個設定。

### ① 獎勵權重 $w_1, w_2$ —— 先釐清一件常被誤解的事

$$r = w_1\!\left(\frac{R_{eff}}{R_{\max}}\right) - w_2\,\rho,\qquad w_1=2.0,\ w_2=0.05$$

> **⚠️ 開銷的主要定價不在 $w_2$，而在 $R_{eff}$ 的 $(1-\rho)$ 因子。**
> $\rho$ 已由 3GPP 符元預算推導（見 §9）；P 曲線實測證明**取捨完全由 $(1-\rho)$ 產生**——$P=1$ 輸給 $P=15$ 純因開銷，與 $w_2$ 無關。

**尺度分析**：

| 項 | 範圍 |
|---|---|
| 吞吐項 $w_1(R_{eff}/R_{\max})$ | $[0,\ 2.0]$（$R_{\max}$ 為物理上限 → 比值 $\le1$）|
| 開銷項 $w_2\rho$ | $\le 0.05\times0.228 = 0.0114$ |

→ **開銷項僅為吞吐項上限的 0.57%**，實質是「輕微推力」而非主要機制。

> **關鍵認知**：**PPO 對獎勵的正仿射縮放不變**，故 $w_1,w_2$ 兩數中真正有意義的只有**比值 $w_2/w_1 = 0.025$**；$w_1$ 僅決定絕對尺度（影響學習率／優勢正規化的數值範圍，不影響最優策略）。
> → **需要調的是一個超參數，不是兩個。**

**出源分級**：**[T3] 設計選擇**。RL 獎勵權重本質上無文獻可引（沒有論文會規定 $w_1=2.0$）；誠實作法是標明設計選擇、給出尺度分析、並於 Part 2 以敏感度掃描驗證結論對 $w_2/w_1$ 不敏感（建議掃 $\{0,\,0.0125,\,0.025,\,0.05,\,0.1\}$）。
⚠️ 不刪 $w_2$：計畫書 Eq.11 明列此項，刪除會偏離計畫書形式。

### ② 動作空間：`numActions` 由 3 改為 4

**DRL 動作集 $=\{0,1,2,3\}$**（0=Keep、1=Scan±1、2=Scan±2、3=全掃 128 束）。

> **⚠️ 這是方法論要求，不是偏好**：baseline 策略為「每 $P$ 個 TTI 執行一次 action 3（全掃）」。若 DRL 拿不到 action 3，它**連 baseline 的策略都無法重現**，只能靠 ±1/±2 慢慢爬 → **結構性讓分**。如此比較下 DRL 輸了不說明任何事、贏了也會被質疑不公。
> **DRL 的動作集必須是 baseline 的超集**，才構成有效對照。

不含 action 4（寬多波束，MultiUser Eq.17）：SU 專用、未做效益驗證、未納入任何 baseline；`env` 保留實作但不開放給 agent，待有需求再以消融評估。

⚠️ **修正前的三方不一致（真實 bug）**：`config.numActions=3` 推導出 $\{0,1,2\}$；3 個 agent 腳本寫死 `rlFiniteSetSpec([0 1 2 3])`；`env/myStep` 支援 0–4 → 同一程式庫中 agent 的動作集有兩種版本，**訓練結果不可比、無法重現**。
**規則**：所有 `actInfo` 一律寫 `rlFiniteSetSpec(0:numActions-1)`，**不得寫死清單**。

### ③ 觀測歷史初值：改為真實量測

舊版 `myReset` 填 $P_{echo}=0$、$\mathrm{CRLB}=\Gamma=3\times10^{-3}$。CRB 公式修正（§7）後典型值約 $4\times10^{-6}$，舊初值**比典型大 750 倍**：

| 觀測（對數正規化後）| 舊初值 | 典型值 |
|---|---|---|
| CRLB | **0.85** | 0.56 |
| $P_{echo}$ | **0.00**（被夾在最小值）| 中段 |

→ **每個 episode 的前 $k_{Hist}=4$ 步，agent 都看到「完全沒有回波、感測極差」的假訊號**。RL 對初始狀態特別敏感（每一集都由此出發），屬**系統性偏誤**而非隨機雜訊。

**修法**：reset 時做一次**真實量測**填滿歷史——物理上對應計畫書 step1 的初始波束掃描，且與 `myStep` **完全同源**（同一組 paths → $\hat{\mathbf{h}}$ → `echoSignalV2`），確保 $t=0$ 與 $t>0$ 一致。

---

## 11¾. 已知簡化與限制（**報告必須揭露**）

> 以下為經查證後**確認保留**的簡化。每項皆說明理由與影響方向，供報告直接引用。

| # | 簡化 | 影響 | 理由 |
|---|---|---|---|
| 1 | **陣列方位面為全向元件**（僅實作 3GPP 垂直切面 $-\min[12(\theta/65)^2,30]$，未含水平切面與 $G_{E,\max}=8$ dBi）| 方位覆蓋**高估**；絕對 SNR **低估** 8 dB（方向相反、部分抵消）| 加水平方向圖必須與 3 扇區綁在一起做；單獨加會使主街東西兩端掉 25 dB，模型變得**不真實地差** |
| 2 | **碼本方位掃描限 $\pm60°$** | 車有 **5.4–7.1%** 時間落在 endfire 盲區 | $\pm60°$ 是 $\lambda/2$ 陣列的**標準掃描限**（超出後波束展寬、增益下降）；endfire 盲區是**所有線性陣列**的固有性質 |
| 3 | **決策週期 $\gg$ 相干時間** — 10 ms vs 0.45–0.90 ms（**11–22 倍**），每週期取單一快衰落樣本 | **平均吞吐正確**（$\mathbb{E}[\log_2(1+\mathrm{SNR})]$ 即遍歷容量）；**連線率偏保守**（瞬時 SNR 門檻會產生假中斷）| **文獻通例**——MultiUser、Zhao2024 皆每決策時隙抽一次通道 |
| 4 | **陰影 AR(1) 跨 LOS/NLOS 不完全收斂** | 影響 **7.9%** 的 TTI（切換後 5 m 內）；為**變異數**偏差非均值偏差 | 已改為 3GPP TR 38.901 §7.6.3.1 的**雙獨立過程**（LOS/NLOS 各一），殘餘為過渡期效應 |
| 5 | **genie 初始波束獲取** — reset 以真實幾何角選初始束、不付掃描代價 | $<0.02\%$（1/1700 TTI × $\rho$=0.228）| 計畫書 step1 本即假設初始掃描已完成；研究聚焦**追蹤**而非**獲取** |
| 6 | **感測角度**仍用真實幾何角於 CRB 的 $\cos\theta$ 項 | 該項為「被估計的角度」本身，屬合理用法 | 照射方向已於 2026-07-20 改為沿實際傳播通道（見 §6）|
| 7 | **單 gNB、無鄰細胞干擾** | — | 三篇核心論文皆同 |
| 8 | **full-buffer** | — | RL 訓練標準（arXiv 1611.06497）|
| 9 | K=3 主導散射、散射點高度 1.5 m、$\sigma_{RCS}$=25 m²、$\sigma_{dB}$ | [T3] | 已排 Q3（2026-09）消融掃描 |

### 單扇區覆蓋的定稿寫法

> 🚩 **本節的定稿寫法已於 2026-07-21 作廢，不可寫進報告。** 理由見下方。
>
> ~~本模型採單扇區 gNB⋯⋯車輛有約 **5.4–7.1%** 的時間位於扇區外，此時無波束可對準⋯⋯~~

#### ⚠️ 為什麼作廢：平面陣列的前後模糊性（2026-07-21 發現）

上面那段話有**兩個獨立的錯誤**，而且互相掩護：

**錯誤一：5.4–7.1% 是錯的，真實值是 68.3%。**

舊值是以「$\sin\theta_{az}$ 有沒有落在碼本涵蓋範圍內」為判準量出來的。但
$\sin 120° = \sin 60° = 0.866$ **落在碼本範圍內**，於是位於 $120°$ 的車被算成「扇區內」。
**這個判準本身就對前後模糊視而不見**——等於拿一把有同樣缺陷的尺，去量這個缺陷。
按真實角度重量，扇區外佔 **68.3%**。

**錯誤二：「扇區外無波束可對準」在程式中根本不成立。**

陣列響應只透過 $\sin\theta_{az}$ 進入相位，而 $\sin\theta = \sin(180°-\theta)$
⇒ **陣列在數學上分不出正面與背面**。實測：

| 方位角 | $\sin\theta_{az}$ | 最佳束增益 |
|---|---|---|
| $30°$ | $+0.5000$ | 20.13 dB |
| $150°$ | $+0.5000$ | **20.13 dB**（與 $30°$ 完全相同）|
| $60°$ | $+0.8660$ | 20.27 dB |
| $120°$ | $+0.8660$ | **20.27 dB**（與 $60°$ 完全相同）|
| $170°$（幾乎在面板正後方）| $+0.1736$ | **20.32 dB**（仍拿滿陣列增益）|

車子繞到基地台正後方，還是拿到完整的 20 dB 陣列增益。
旁證：實測扇區外 SNR 中位 39.5 dB $\approx$ 扇區內 39.8 dB——本應相差極大。

**根因**：唯一能打破這個對稱的，是天線元件本身的**方位切面方向圖** $A_{EH}$
（[T2] 3GPP TR 38.901 V19.4.0 **Table 7.3-1**）。前身只建了垂直切面 $A_{EV}$
⇒ 模型裡的 gNB 是全向天線。

> ✅ **已修正（S0–S3，2026-07-22）**：見本節末「修正結案」。三扇區 + panel-relative 合成
> 讓 $A_{EH}$ 依相對面板法線的 azRel 生效，前後對稱被打破；下表「現況」欄為**修正前**的歷史數字。

**影響**（`diagnoseLinkSaturation.m` 反事實推算，非重跑模擬）：

| | 現況 | 補上 $A_{EH}$ |
|---|---|---|
| 連線率 | 99.7% | **78.4%** |
| SNR 中位 | 39.6 dB | **14.6 dB** |
| SE 截斷比例 | 92.4% | **39.7%** |

⇒ 先前判定的「環境已飽和」有**相當部分源自本缺陷**，不是「FR3 鏈路太好」。

#### 連帶：gNB 朝向的座標系不一致（已隨三扇區一併解決）

前身 `channelModelV2` 產生世界座標 `az_tx`，直接當「相對法線的角度」餵給 `arrayResponse`，
無任何旋轉 ⇒ 陣列法線隱含為世界 $+x$，與文件描述的面板朝向不一致。
**S2 已解決**：座標轉換移到合成點，每條 path 以 `azRelPath = wrapTo180(az_tx - boresight(sector))`
明確轉為相對服務面板法線的角度，`channelModelV2` 維持輸出世界角（單一職責）。

#### ✅ 修正結案：三扇區 panel-relative 合成（S0–S3，2026-07-22，方案 A 已實作）

當初列的三個修法選項中**採用方案 A（$A_{EH}$ + 三扇區）**，並已分四階段入庫：

| 階段 | 做了什麼 | commit |
|---|---|---|
| S0 | 扇區幾何 primitives：`sectorGeometry` boresight $[0,120,-120]°$、`azToSector`（argmin、平手取最小 index、保證 $\lvert\text{azRel}\rvert\le60°$） | `a057a02` |
| S1 | `arrayResponse` opt-in 方位方向圖 `applyAzPattern`（預設 false，$A_{EH}$ 依 azRel） | `22462b4` |
| S2 | `myReset`/`myStep` 每 TTI 由 UE 幾何選 serving sector、每條 path 轉 azRel + 套 $A_{EH}$ 合成 `hChan`；`echoSignalV2` 加 `panelBoresightDeg`、CRLB 用 azRel | `77a5419` |
| S3 | sector-aware validator（oracle 只在 serving 128 束 argmax，無 384 全站）+ post-sector v4 | `834364e` |

**前後對稱如何被打破**：UE 由 $\lvert\text{azRel}\rvert$ 最小的面板服務，`azRel` 恆 $\le60°$；
面板背向（相對法線 $>60°$）的方向由 $A_{EH}$ 衰減、且本來就交給鄰扇區。
故上文「$120°$ 與 $60°$ 拿相同增益」「$170°$ 仍拿滿 20 dB」的鏡像**不再發生**。

**驗收（V1–V5，於 `runAllTests` 全綠）**：V1 east-facing panel 對 west UE 最佳增益低 **≥25 dB**
（關 $A_{EH}$ 時兩者差 <1 dB，證明差異源自方向圖）；V2 360° 每點 $\lvert\text{azRel}\rvert\le60$、三扇區皆命中；
V4 邊界 $60/180/-60°$ 確定性切換、平手取最小 index；V5 CRLB 用 azRel（含非零 boresight 反證）。

**post-sector v4（30 seeds，`results/Part1_Validation_v4/`）**：GATE 1 OverallPass **30/30**、
GATE 2 PropagationPass **PASS**（gap 95%CI 下界 9.99 dB、正 gap 100%）。
方位方向圖使偏離法線方向衰減，故 LOS SNR $44.9\to41.2$ dB、NLOS $32.6\to28.2$ dB、
連線率 $99.7\to98.8\%$、$R_\text{eff}$ $711\to688$ Mbps——這是**前後鏡像修正的預期物理代價，非退步**。

> ⚠️ **仍待 S4**：obs 正規化範圍 `P_ECHO_RANGE`/`CRLB_RANGE` 需依 v4 分布重校，
> 正式 RL 訓練在此之前不宜開始。

#### $A_{EH}$ 方位切面（`env/elementGainAz.m`，S1–S2 已接入）

$$A_{EH}(\varphi) = -\min\!\Big[12\Big(\frac{\varphi}{65°}\Big)^2,\ 30\Big]\ \text{dB}$$

**接線歷程**：此函式在 2026-07-21 先寫好但刻意未接（F-4 待定案）。
2026-07-22 定案採方案 A 後，S1 讓 `arrayResponse` 以 opt-in 方式套用它、S2 於
`myReset`/`myStep` 的 panel-relative 合成正式啟用。接入點以 guard
`tests/testElementPattern.m:testAzPatternOnlySanctionedOptIn` 白名單化（只允許 myReset/myStep）。

已驗證性質（`tests/testElementPattern.m`、`tests/testElementGainAz.m`）：

| 性質 | 實測 |
|---|---|
| $\varphi = 65°/2 = 32.5°$（HPBW 半寬）| 恰為 $-3.00$ dB ✓ |
| 法線 $\varphi=0$ | 增益 $=1$ ✓ |
| 左右對稱 $A_{EH}(-\varphi)=A_{EH}(\varphi)$ | ✓ |
| 飽和於 $\mathrm{SLA}_H = 30$ dB | $\varphi > 102.8°$ 後 ✓ |
| 飽和前單調遞減 | ✓ |
| 回傳線性功率增益（`arrayResponse` 取 $\sqrt{g_\text{el}\cdot g_\text{az}}$）| ✓ |

接入後環境物理已依設計改變（見上方「修正結案」的 v3→v4 對照）——這是**有意的**前後鏡像修正，
非誤差；S1 的 opt-in 預設關閉階段則確認過零行為改變。

測試 `testAzNotYetWiredIn` 會在它被接入時**主動失敗**——這是刻意的，
用來強制「接入者必須同步三件組並重跑 baseline」，避免有人順手接進去就走。

> 💡 **順帶記一個教訓**：撰寫上表的 $\mathrm{SLA}_V$ 測試時，我一開始斷言 $\theta=90°$
> 就該飽和到 $-30$ dB，被測試抓出來是錯的——飽和門檻其實是
> $12(\theta/65)^2 = 30 \Rightarrow \theta = 65\sqrt{30/12} = 102.8°$，$90°$ 時只有 $-23.01$ dB。
> **測試寫錯、程式正確。** 這正是為未使用的程式先寫測試的價值。

⚠️ 原本就成立、修完後仍成立的一條紀律：**不得宣稱「由鄰扇區服務」同時又把這些 TTI 計為中斷**——兩句話無法同時成立，會被直接質疑。亦不採「排除扇區外樣本」，以免構成選樣偏誤。

---

## 12. 總體檢（單元全過之後）

> **這在幹嘛：** 所有單元都過之後的整體健康檢查——跑全部測試、看策略成績階梯（亂猜<週期掃描<oracle）、目視動畫確認因果鏈合理。

| 檢驗 | 指令 | 通過標準 |
|---|---|---|
| 行為契約 | `runAllTests` | 37/37 綠 |
| 合理策略體檢 | `inspectEnv`／`periodicSweepSim`（週期掃描 P=15） | 連線 ~78%（C=2 修正後）、SNR 僅轉彎後掉 |
| 因果鏈目檢 | `liveInspectEnv`（動畫） | 變紅→SNR 掉→斷線 同步發生 |
| 成績階梯 | 亂猜 ~17% < 週期 69–78% < oracle ~97%（2026-07-16 修正後 verifyPart1） | 分數隨策略聰明度單調上升 |
| 對照組基準 | `periodicSweepSim(20)` | P=15：**81.6%、432 Mbps、ρ=0.080**（C=2、選項A後）|

**成績階梯的意義**：oracle（每步全掃碼本全部 128 束）~97% = 環境物理天花板；它與（受限動作空間的）週期掃描 ~82% 之間 ~15 個百分點的縫 = Part 2 DRL 的研究空間。

**oracle（exhaustive 上限）出處**：以「窮舉波束搜尋」當效能上限是波束管理標準慣例——核心論文 **MultiUser 即以 AoD-Genie 為效能上限**（[[notes/MultiUserBeamforming_DRL_Sensing]] 譯文 §V）；3GPP NR 波束管理 P-1/P-2 **exhaustive beam sweeping**（TR 38.802、TS 38.213/214）；tutorial：Giordani et al., *IEEE Commun. Surveys & Tutorials*, 2019（arXiv:1804.01908）。

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
| Stoica & Nehorai, IEEE Trans. ASSP 37(5):720–741, 1989 | §7 CRLB（一般化矩陣式）| DOI: **10.1109/29.17564** ⚠️ 2026-07-21 勘誤，舊值 10.1109/29.32276 實為 ESPRIT 論文 |
| ISAC 綜述（ULA 顯式 CRB_ω ≈ 6/(N³L·SNR)）| §7 CRLB（ULA 特例化）| arXiv:2104.09954 |
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

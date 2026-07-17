# Part 1 環境講義 — 公式、出處與檢驗手冊

> 對象：FR3_BeamMgmt 專題（`D:\FR3_BeamMgmt`）Part 1 環境層。
> 每一節固定四段：**①這是什麼（白話）→ ②完整公式（LaTeX）→ ③出處（論文+庫內路徑）→ ④怎麼檢驗**。
> SPEC.md（**規格值記錄，非正確性依據**）：`D:\FR3_BeamMgmt\SPEC.md`；決策/修正歷史：`D:\FR3_BeamMgmt\docs\待討論與決策.md`。
> 最後更新：2026-07-17（§5 碼本取樣空間修正：方位角改正弦空間均勻 ✅）

> ⚠️ **驗證原則（2026-07-13 立）**：本手冊每項參數/公式一律以**原始出處（論文 / ITU / 3GPP / 物理推導 / 自算檢驗）**自證正確性——**不以 SPEC.md 當作正確性依據**。SPEC.md 只是本專題**衍生的規格值記錄**，它本身可能含錯（實例：§0 的 R1，N0=−109 dBm 是 SPEC 級的物理錯誤；§5 的碼本取樣則是 SPEC **從未規定**、程式自行 linspace 的規格空白）。凡本手冊與 SPEC.md 不符，一律**以原始文獻/物理為準**，並回頭修正 SPEC，而非反過來拿 SPEC 壓手冊。

---

## 📌 文獻驗證狀態

| 部分                      | 驗證方式               | 狀態     | 備註                                                      |     |
| ----------------------- | ------------------ | ------ | ------------------------------------------------------- | --- |
| **§3 通道模型**             | IEEE Xplore + 直讀原文 | ✅ 完全驗證 | Shakya2025 / Gudmundson 1991 / Clarke 1968 / Jakes 1974 |     |
| **§1–2（場景、LOS/NLOS）**   | SPEC.md + 3GPP 原文  | ✅ 無誤   | Bai-Heath 2014, 3GPP 38.901                             |     |
| **§5–8（波束/回波/CRLB/連線）** | 三核心論文 + 標準核對       | ✅ 已查證  | 公式形式對得上；部分數值標「待對回原表」                                    |     |
| **§11 移動**              | —                  | ✅ 已查證  | 轉向機率出處已更正（設計選擇，非 MultiUser）                             |     |

**重點修正（2026-07-16，本輪環境參數三改，詳見 `docs/待討論與決策.md`）：**
- ✅ **反射損 L_refl：6 dB → 0**（已含於 Shakya n_NLOS 斜率，避免雙重計算）
- ✅ **雜訊 N₀：−109 → −86 dBm**（kTB+NF=8dB，3GPP TR 38.820；舊值違反熱雜訊底）
- ✅ **連線門檻 C：4 → 主值 2、消融 {1,2,4}**（QoS 設計參數，非文獻規定值）
- 🟡 已標註：§11 轉向機率出處更正、§6/§7 Zhao2024 數值待對回原表、§7 Γ_CRLB/§6 抖動/§11 轉彎為設計選擇

**早期修正（2026-07-12）：** Gudmundson DOI 10.1049/el:19911328、Clarke DOI 10.1002/j.1538-7305.1968.tb00069.x、Jakes ISBN 978-0471437208（均 IEEE Xplore 確認）。

---

## 0. 系統參數與符號表

> **這在幹嘛：** 一張速查表，把整個專題會用到的物理參數（頻率、頻寬、天線數、功率、雜訊…）和符號集中列出來，方便隨時對照。

| 符號 | 意義 | 值 | 程式位置 |
|---|---|---|---|
| $f_c$ | 載波頻率 | 16.95 GHz | `config.m` |
| $B$ | 頻寬 | 100 MHz | `config.m` |
| $\lambda$ | 波長 | $c/f_c \approx 17.7$ mm | 衍生 |
| $N_t$ | 天線數（UPA $16\times 8$） | 128 | `config.m` |
| $P_{tx}$ | 發射功率 | 30 dBm | `config.m` |
| $N_0$ | 雜訊功率 | −86 dBm（$kTB$+NF, B=100MHz, NF=8dB）| `config.m` |
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
變動長度、平均 ~1250 TTI。**出處**：軌跡驅動 episode 為 2023–2025 主流——[[notes/Zhao2024_IBTD_ISAC_V2V]]（episode=車輛運動序列 起點→終點）、Ye&Ge Nature 2023（接近模擬邊界即提早終止，計畫書 ref[7]）、arXiv 2511.02260（2025，軌跡自然結束=駛離範圍）；Lane-change RL「軌跡結束 or 逾時」雙層。

**安全上限 = 4000 TTI（工程保險絲，非物理量）**：防移動狀態機 bug / 未來改碼 / 新速度導致無限繞圈。蒙地卡羅**三速度各 10000 episodes（共 30000）**確認最長合法 episode 全部 <4000、0 次觸及：30km/h max=**3206**（最慢=最長，硬邊界，99.9th=max）、45km/h=2139、60km/h=1604（2 路口、轉彎狀態用完必離場 → 路徑集合幾何有界），4000 留餘裕、實質永不觸發。

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

**(d) NLOS 散射路徑**（top-3 最近牆面散射點，FR3 稀疏）：

$$
\mathrm{PL}_{NLOS} = \mathrm{FSPL}(1\,\text{m}) + 10\, n_{NLOS} \log_{10}(d_{3D}) + X_\sigma
$$

（**損耗用直線距離 $d_{3D}$**（gNB→車），非散射路徑長；**無外加反射常數**。$n_{NLOS}=2.59$ 已含反射與繞路代價，見下方說明。）

$$
g_s = \sqrt{10^{-\mathrm{PL}_{NLOS}/10}}\; e^{-j 2\pi (d_1+d_2)/\lambda}
$$

其中 $d_1$=gNB→散射點、$d_2$=散射點→車。**損耗錨在直線 $d_{3D}$、相位用幾何路徑 $d_1{+}d_2$（都卜勒來源）——職責分離**（2026-07-16 選項A修正，前身損耗誤用 $d_1{+}d_2$）。**反射損 $L_{\mathrm{refl}}=0$**（理由如下）。

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

> ⚠️ **重要修正記錄（2026-07-04）**：曾誤寫成兩段各算一次 CI（$\mathrm{FSPL}+10n\log d_1 + \mathrm{FSPL}+10n\log d_2$），使 NLOS 總損 ~200 dB 形成「側街死域」。**CI 的 $n_{NLOS}=2.59$ 是實測「總損耗 vs 距離」的擬合值**，故只能算**一次** CI 單斜率，不可兩段 FSPL 疊加。此 bug 由「動畫中 NLOS 全斷」的質疑觸發，oracle 全碼本掃描實驗證實後修正。（🔎 註：當時距離取 $d_1{+}d_2$，後於 2026-07-16 **選項A** 精修為**直線 $d_{3D}$**——因 Shakya PLE 對直線距離擬合，見上方 📐 說明。）

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
  - ⚠️ **修正記錄**：舊 ISBN 978-0471439226 已更正為 978-0471437208

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
runtests('tests/testChannelModelV2.m')  % 5 測試：幾何LOS/陰影靜止/相位決定性/top-3/NLOS單斜率
% 手動：沿主街掃 x，畫路徑增益曲線 → 應「近 gNB 高、遠端低、平滑無跳崖」
```
**物理問句**：SNR 曲線平滑嗎？LOS↔NLOS 切換發生在幾何遮蔽點嗎？

---

## 4. 散射點（generateScatterers.m）

> **這在幹嘛：** 決定 NLOS 訊號「從哪面牆反射過來」——在大樓面向街道的牆上固定佈點，讓被擋住的車還能靠牆面反射收到訊號。

### ① 這是什麼
車被大樓擋住（NLOS）時，直線訊號被切斷，但訊號仍會**打到旁邊大樓的牆面、反彈**到車。本單元決定這些「反射點」放哪：在每棟大樓**面向街道的牆面**上固定佈點（每牆取樣 5 點、向外推 `scatterOffsetDist`、只留落在路廊內的），**決定性生成（無 rand）**。通道（§3）再取離車最近的 **top-3** 當 NLOS 反射路徑：gNB $\to(d_1)\to$ 反射點 $\to(d_2)\to$ 車。

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
| $K$（NLOS 路徑數） | 3 | [T3 設計選擇] FR3 稀疏、少數主導 MPC（Shakya ASA 15.3°≪3GPP 47°）；窄波束只收少數路徑。⚠️ Shakya 未給「K=?」 | $K\in\{1,3,5\}$ |
| 每牆取樣點數 | 5 | [T3 設計選擇] 空間解析 vs 計算成本折衷 | — |
| `reflectHeight`（散射點高度） | 1.5 m | [T3 設計選擇] **街道層反射**：放在車輛平面（車高亦 1.5 m），模擬街道峽谷側牆的街道層反射。⚠️ **會影響 NLOS 仰角** $\text{el}_{tx}=\arctan\frac{10-1.5}{\cdot}$ → 影響仰角波束選擇（非零影響參數）。無文獻規定散射點高度 | $\in\{1.5, 5, 10\}$ m |
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
| top-3 最近選擇 | `env/channelModelV2.m` | 第 61–69 行（sort 距離取前 3；標註 [T3 暫定設計，Q3 待驗證]） |

### ③ 出處（2026-07-16 逐一開原文核實）

**方法命名**：本法＝**幾何式決定性單次反射（geometry-based deterministic / single-bounce ray-tracing）**。前身誤標「GBSM」已更正——GBSM 的 S=Stochastic 指**隨機**撒群集，與本專題「決定性、無 rand、貼真實牆面」相反。

| 引用 | 用途 | 核實 |
|---|---|---|
| **Shakya2025**（arXiv:2410.17539 v4 Table VI） | ASA 15.3°（≪3GPP 47.31°）→ FR3 稀疏、少數主導路徑 | 🟢 已開原文 |
| **GEMV²**（Boban et al., IEEE TVT 2014, arXiv:1305.0124） | 主要範式依據：真實建物輪廓 + **決定性大尺度 LOS/NLOS 分類**，與本法最貼合 | 🟢 已開原文（註：其小尺度為 stochastic，本法只借「真實幾何+決定性 LOS/NLOS」精神） |
| **FR3_Rel19**（[[notes/FR3_Rel19_ChannelModeling]]） | FR3 **稀疏性隨頻率上升**（M_min 捕捉稀疏、群集數改為**帶範圍可變量**、角度/延遲擴展隨頻率變小） | 🟢 已核原文（措辭已更正：**非**「群集數單調變少」） |
| **3GPP TR 38.901 §7.6.3**（Spatial consistency；j40/Rel-18） | **⭐ 採用決定性幾何的正當理由**：標準明訂移動性模擬需空間一致性，且 drop-based 隨機模型**不具備**、需額外 Procedure A/B 補救 → 本法用決定性幾何**天生滿足** | 🟢 已開原文（親自抽 j40 docx；§7.6.3.1/7.6.3.2/7.6.3.4、Table 7.6.3.1-2） |
| ETSI TR 103 257-1 | 將 geometry-based deterministic (ray-tracing) 列為 V2X/ITS 通道模型**選項之一** | 🟡 範式佐證（註：5.9GHz DSRC，非 FR3；非數值依據） |

> ⚠️ **本組合為本專題自建（最重要的誠實揭露）**：「決定性固定散射點 + single-bounce + isotropic」這個**完整組合**，三篇核心論文**皆無**——Shakya=純 CI 統計（無幾何）、MultiUser=Rician **隨機**衰落、3GPP GBSM=**隨機**群集/射線。本法是**為滿足空間一致性（38.901 §7.6.3）而自行設計的簡化**，借 GEMV² 的「真實幾何＋決定性」精神。
> → **論文措辭**：只能寫「參考 GEMV² 精神、依 38.901 §7.6.3 空間一致性需求**自行設計**」，**不可**寫成「依循 Shakya/MultiUser/GBSM 的方法」。
>
> ⚠️ **isotropic（全向散射）是簡化，非最佳物理**：$f_c=16.95$ GHz（$\lambda\approx1.8$ cm）下牆面遠大於波長，物理上 **specular（鏡面反射，入射角=反射角）更貼近真實**；isotropic 一般用於粗糙面的次要散射分量。本專題基於**計算成本**與**研究重點**（波束策略的相對比較，非通道保真度）仍採 isotropic，並已用 SNR 熱圖驗證此簡化仍產生**合理物理**（LOS 中位 22 dB > NLOS 13 dB、隨遮蔽深度衰減、無異常亮點）。升級路徑：改鏡像法（specular image method），屬 Part 2+ 選項。
>
> ⚠️ **point-scatterer 單點近似的已知侷限**：Delbari et al.（arXiv:2511.20572, 2025）明確指出點散射假設對牆/地/天花板等**大反射體不完全成立**。本專題仍採，屬**簡化假設（非依據）**。
>
> ✅ **`scatterOffsetDist=0.5m` 非物理參數（2026-07-16 讀碼釐清）**：它只是「把牆面點往外推、測是否落在馬路」的**探針距離**，用來判斷牆面向街道；散射點本身**存在牆面上**，故對 SNR/相位**零物理影響**。任何 $0<$值$<$路寬 等效 → **無需文獻、無需敏感度掃描**。（前版誤當物理參數、還掛「IEEE 多採固定偏移 ✅」查無來源，已更正。）

### ④ 檢驗
```matlab
runtests('tests/testGenerateScatterers.m')  % 4 測試：格式/決定性/貼牆/面向道路
sc=generateScatterers(s); % 畫圖：紅點應全貼牆、不在樓內/路中央；跑兩次結果相同
```

---

## 5. 波束成形基礎（upaArray.m + dftCodebook.m）

> **這在幹嘛：** 讓 128 根天線把能量聚成一根「手電筒」指向某方向——碼本就是一組預先造好、各指一個方向的手電筒，波束管理就是從裡面挑一根對準車。

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

**碼本的取樣空間**（2026-07-17 更正）：波束必須在 **$\sin(\theta_{az})$ 上等間隔**，不是在角度上等間隔。原因見 ②½ 下方的白話說明。目前 `dftCodebook(16,8)` = **128 束（方位角 8 × 仰角 16）**，方位角柵格為 $\{\pm 60°, \pm 38.2°, \pm 21.8°, \pm 7.1°\}$，$\Delta\sin \equiv 0.2474$。

> **為什麼不能在角度上等間隔（白話）**：陣列因子 AF 是**空間頻率** $\psi = \frac{2\pi d}{\lambda}\sin\theta$ 的函數，不是角度的函數。波束在 $\psi$ 域寬度固定，換算回角度域時會隨掃描角**展寬 $1/\cos\theta$** —— 越靠扇區邊緣的波束越胖。所以角度等間隔會造成「中央過取樣、邊緣欠取樣」，剛好跟波束的實際行為相反，相鄰波束之間就**破出覆蓋空洞**：車停在兩束中間時，碼本裡最好的那束也對不準它。
>
> **實測（`tests/testDftCodebook.m`）**：舊的 `linspace(-60,60,8)` 角度均勻柵格，方位角掃描全程最差交越損 **−5.77 dB**；改為正弦空間均勻後為 **−3.78 dB**，逼近臨界取樣正交 DFT 柵格的**閉式理論值 −3.92 dB**（$(2/\pi)^2$，見 ③ 出處）。**改善 1.99 dB，且波束數不變（仍 128）、動作空間不變。**

> ⚠️ **現況與待辦（Part 1 暫定）**
> - **仰角仍用角度等間隔**，這是**刻意保留、非疏漏**：扇區僅 ±15°，此範圍內 $\sin(el)\approx el$，角度均勻與 sine 均勻差異 <1%，無實質影響。
> - **Q12（原講義誤標為 Q6）**：仰角 16 束擠在 ±15° 內 → 波束寬約 6.35° 卻以 2° 取樣，約 **3.6 倍冗餘**；128 束的**有效自由度僅約 40**。而 V2X 追蹤關鍵在方位角，方位角卻只有 8 元素。這是**天線元件配置**問題（16V×8H 對車輛場景可能裝反），**不是取樣空間問題** —— 改取樣空間解決不了它。待 Part 2 正面處理。
> - **Part 2 T2.0**：3GPP Type I 過取樣（O1=4 → 方位角 32 束 @ Δsin≈0.062，交越損降至約 −0.25 dB）。因柵格已在正確的框架上，屆時只需把束數改為 `ntCol*O1`。（決策表 Q7，已採用）

### ②½ 程式對應
| 內容 | 檔案 | 位置 |
|---|---|---|
| 導向向量 $\mathbf{a}(\theta,\phi)$ | `env/upaArray.m` | 全檔 |
| DFT 碼本 $\mathbf{F}$（128 束 = 方位 8 × 仰角 16） | `env/dftCodebook.m` | `azList = asind(linspace(...))` 正弦空間均勻；`elList` 角度均勻（±15° 內誤差<1%）；逐束呼叫 upaArray |
| 碼本取樣空間與覆蓋驗證 | `tests/testDftCodebook.m` | 全檔（sine 均勻性 + 最差交越損 > −4 dB） |
| $G=\lvert\mathbf{a}^H\mathbf{f}\rvert^2$ 實際計算 | `env/myStep.m`（通訊）、`env/echoSignalV2.m`（感測） | 內積後取 `abs()^2` |

### ③ 出處
| 級 | 引用 | 支持什麼 |
|---|---|---|
| **T1** | 3GPP TS 38.214, *NR; Physical layer procedures for data*, **§5.2.2.2.1**（Type I Single-Panel Codebook, Rel-15 起） | 波束索引 $l$ 進入 $\exp(j2\pi l/(O_1 N_1))$ 的**線性相位項** → 取樣格點在**空間頻率上等間隔**，$\sin\theta$ 步長 $= 2/(O_1 N_1)$。**標準從未在角度域等間隔取樣。** |
| **T1** | Tse & Viswanath, *Fundamentals of Wireless Communication*, Cambridge Univ. Press, 2005, **§7.3.1「Angular domain representation of signals」, pp. 318–322** | ULA 的固定角度基底取在 **directional cosine** $\Omega = k/L_t$ 上，構成空間 DFT **正交**基底。**正交性只在 sine space 等間隔時成立**；λ/2、$N$ 元陣列即 $\sin\theta$ 間隔 $2/N$。 |
| **T1** | Balanis, *Antenna Theory: Analysis and Design*, 4th ed., Wiley, 2016, **§6.3（N-Element Linear Array: Uniform Amplitude and Spacing）** | $AF(\psi) = \frac{\sin(N\psi/2)}{N\sin(\psi/2)}$，$\psi = kd\cos\theta+\beta$。方向圖是 $\psi$ 的函數；主瓣在 $\psi$ 域固定寬、在 $\theta$ 域隨掃描角展寬 $\propto 1/\cos\theta$ ⇒ **角度等間隔必然中央過取樣、邊緣欠取樣**。 |
| **T2** | Mailloux, *Phased Array Antenna Handbook*, 3rd ed., Artech House, 2018, **Ch. 8（Multiple-Beam Systems / Beam Crossover Loss）** | 「3.9 dB 交越損」的出處。**這不是實驗值，是閉式解**：正交間隔 $\Delta\psi = 2\pi/N$，交越點 $\Delta\psi/2 = \pi/N$ 代入 Dirichlet 核得 $AF = \frac{\sin(\pi/2)}{N\sin(\pi/2N)} \to 2/\pi = 0.6366$ ⇒ 功率 $(2/\pi)^2 = 0.405$ ⇒ **−3.92 dB**。 |
| T3 | Van Trees, *Optimum Array Processing*, Wiley, 2002 | 陣列訊號處理通論（頻率無關，故 FR3 適用）。 |

> **這個 3.92 dB 本身就證明了命題**：它是 Dirichlet 核在**正交格點**交越處的閉式值，只在 sine space 均勻的臨界取樣 DFT 柵格上成立。角度均勻的柵格**根本算不出一個穩定的 3.92 dB**（我們實測到 −5.77 dB）。

> **例外情形（誠實揭露，均不適用本專題）**：① 部分 mmWave 論文與 3GPP RAN4 測試確實用角度均勻 GoB，但那是**類比波束成形／OTA 量測的掃描規格**，不是碼本正交性設計。② 近場（Fresnel 區）DFT 碼本本就會 beam-split，sine space 均勻性亦不成立；但 SPEC.md L27 已論證 Fraunhofer 距離 $2D^2/\lambda \approx 2.8$ m ≪ 車距 → **遠場成立**，此例外不適用。

### ④ 檢驗
```matlab
[F,azList,elList]=dftCodebook(16,8,fc,dSpacing);
norm(F(:,1))                      % 每束單位功率 → 1
max(abs(diff(sind(azList))-0.2474))    % 正弦空間等間隔 → ~0（<1e-9）
a=upaArray(fc,30,1,16,8,dSpacing);
plot(abs(F'*a).^2)                % 一根主峰=只有指向 30° 的波束增益高
```
完整驗證（含最差交越損掃描）：`runtests('tests/testDftCodebook.m')` → 3/3 通過。

---

## 6. ISAC 感測回波（echoSignalV2.m）

> **這在幹嘛：** 基地台發射的波束打到車、反彈回來，靠回波強弱就能「感覺」波束對不對準——不用車回報，這是本論文「免回饋感測輔助」的物理基礎。

### ① 這是什麼
gNB 發射的波束打到車、反彈回 gNB 感測接收機。回波強度隨「波束對準程度」變化 → gNB **不需要車回報**就能感覺波束對不對——這是論文「免回饋感測輔助」賣點的物理基礎。

### ② 完整公式

**理想回波功率**（單站雷達，數位接收對準真實回波角）：

$$
P_{echo} = P_{tx}\, |\mathbf{a}^H \mathbf{f}|^2\, \beta^2, \qquad \boxed{\beta^2 = \frac{N_t^2\,\lambda^2\,\sigma_{RCS}}{(4\pi)^3\, d^4}}
$$

（$\beta$ 依 **MultiUser Eq.(8b)** 原文：$\beta = N_t\sqrt{v_c^2\sigma_{rcs}/f_c^2/(4\pi)^3/d^4}$，$v_c/f_c=\lambda$。含**雙向路損 $1/d^4$**、**目標 RCS**、**接收數位陣列增益 $N_t$**。）

- $|\mathbf{a}^H\mathbf{f}|^2$：**一個**發射波束對準因子（接收為數位全陣列、增益固定 → 不是 $|\cdot|^4$）
- $1/d^4$：雷達雙向路損（去程 $1/d^2$ × 回程 $1/d^2$），程式中即 $40\log_{10} d$
- $\sigma_{RCS} = 25\ \text{m}^2$：車輛雷達散射截面。**值出自 MultiUser Table I（引 [17]）**；「光學區 >1–3 GHz 與頻率無關 → FR3 直接適用」為**標準雷達光學推論**（非論文原句，但為雷達常識可辯護）

**匹配濾波後回波 SNR**：

$$
\mathrm{SNR}_{echo} = \frac{G_{mf}\, |\mathbf{a}^H \mathbf{f}|^2\, P_{tx}\, \beta^2}{N_0}, \qquad G_{mf}=10
$$

**量測抖動**：實際量測值 $\hat{P}_{echo} = P_{echo}\cdot 10^{\xi/10}$，$\xi\sim\mathcal{N}(0, 1\,\text{dB}^2)$（**1 dB 抖動為設計選擇**，代表感測殘餘量測不確定，無文獻直接出處）。

> ⚠️ **修正記錄**：$P_{echo}$ 曾誤寫 $\propto|\mathbf{a}^H\mathbf{f}|^4$（誤設同一類比波束收發）。兩篇核心論文皆為數位接收 → 一個波束因子。修正使 $P_{echo}$ 與 $\mathrm{SNR}_{echo}$ 同次方（測試 `testEchoSignalV2` 以「比值不隨波束變」鎖住此性質）。

### ②½ 程式對應（`env/echoSignalV2.m`）
| 公式 | 位置 |
|---|---|
| $\beta^2=N_t^2\lambda^2\sigma_{RCS}/((4\pi)^3d^4)$ 雷達方程式（含 $1/d^4$、RCS、$N_t$） | `betaSq = nAnt^2*lambda^2*sigmaRcs/((4*pi)^3*d^4)`（含 2026-07-17 修正註解） |
| $P_{echo}=P_{tx}\lvert\mathbf{a}^H\mathbf{f}\rvert^2\beta^2$（一次方） | `P_echo_ideal = ptx * bfGain * betaSq` |
| $\mathrm{SNR}_{echo}$（匹配濾波 $G_{mf}$） | `snrEchoIdeal = matchedGain * bfGain * (ptx*betaSq) / n0` |
| 量測抖動 $\xi\sim\mathcal{N}(0,1\text{dB}^2)$ | 同檔（`measStdDb` 乘 randn）；參數 `config.m` `MEAS_NOISE_STD_DB` |
| **$\sigma_{RCS}=25$ 參數** | **`config.m` `SIGMA_RCS`**（2026-07-17 新增；經 `myStep.m` 傳入 echoSignalV2 第 13 引數） |
| $G_{mf}=10$ 參數 | `config.m` `MATCHED_GAIN_G` |

### ③ 出處
- **MultiUser Eq.6/8b/10**（$\beta\propto\sqrt{\sigma_{rcs}/d^4}$、發射增益 $|\mathbf{a}^H\mathbf{f}|^2$）：[[notes/MultiUserBeamforming_DRL_Sensing]]（原文：`D:\PROJECTS\papers\核心論文\MultiUserBeamforming_DRL_Sensing\`）
- **Zhao2024 Eq.2/6**（數位接收 $b(\theta)a^H(\theta)f$ + 匹配濾波）：[[notes/Zhao2024_IBTD_ISAC_V2V]]
- 雷達方程式基礎：Skolnik, *Introduction to Radar Systems*
> ✅ **$G_{mf}=10$ 已核原文（2026-07-16 更新，前版標「庫內查無」已過時）**：開 Zhao2024 原始論文 PDF（**IET Communications 2024, DOI 10.1049/cmu2.12835**）Table 1「Parameter settings」逐字比對——**Matched filter gain G = 10** 與 **a1=1, a2=a3=200** 皆逐值一致（原文該兩值標引用 [5]）。⚠️ 勿與計畫書 ref[8]（He et al., IEEE TWC 2026, V2I）混淆：那是另一篇不同論文。
>
> ⚠️ **2026-07-17 重大修正：code 先前未實作 σ_RCS 與 $N_t$**。舊 code 用臨時雙向 FSPL `32.4+20log10(fc)+40log10(d)`，**缺 $\sigma_{RCS}$（−14.0 dB）與接收陣列增益 $N_t^2$（−42.1 dB）、常數差 +11.0 dB → 合計較正確物理低 45.1 dB**（純常數偏移，故不改 argmax → 波束選擇與連線率 baseline 不受影響）。此為「文件宣稱物理、code 未實作」的反向破口，由全環境三件組稽核抓出。修正後：CRLB 中位 2.9e−2 → **1.7e−6**、CRLB<Γ 占比 3.6% → **100%**。
> **連帶待處理（Part 2）**：① $\Gamma_{CRLB}=3\times10^{-3}$ 現已**永遠滿足、失去鑑別力**（角度誤差 σ≈0.075° ≪ HPBW 3.5°）→ 見 Q2／Q11（系統參數比 MultiUser 強：30dBm vs 15dBm、128 vs 32 天線）；② `P_ECHO_SCALE=1e-13` 偏差 **40.6 dB**，實測 P_echo 中位 1.1e−9，需重校。
- FR3 接軌：Multi-Band Sensing in FR3（arXiv 2602.23539，1/d⁴）

### ④ 檢驗
```matlab
runtests('tests/testEchoSignalV2.m')  % 2 測試：P∝|a^Hf|² 一次方、P/SNR 同次方
% 手動：對固定車位掃全碼本 → echo 應單峰（對準最強）
```

---

## 7. CRLB 角度估測下界（computeCrlb.m）

> **這在幹嘛：** 用回波估車的角度，最準能估到多準的理論極限——回波越強估得越準(CRLB 越小)，DRL 用它當「我還追得住車嗎」的信心指標。

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

> 🟡 **出處註記**：$\Gamma_{CRLB}=3\times10^{-3}$ 為**自推設計門檻**（由陣列 HPBW/2 推得，無文獻直接規定值）。⚠️ 專案報告另寫 2.7e-3，兩者略有出入待調和；此為感測「失明」判準的設計旋鈕，屬 Part 2 可調參數。

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
> 🟡 **數值出處註記**：量測係數 $a_1{=}1,a_2{=}a_3{=}200,G{=}10$ 的**公式形式**出自 Zhao2024 Eq.9，但**三個數值在庫內查無**（Table 1 為圖片）——僅專案報告自述，**待對回 Zhao2024 原表核實**。
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

$L_{sw}=0.5$ dB 換波束瞬間損耗（換束後 1 TTI 生效——防「瞬移作弊」）。

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
| 門檻 $C_{th}=2$【主值】、$L_{sw}=0.5$ dB、延遲 1 TTI | `config.m`（`cThreshold`、消融 `cThresholdSweep=[1 2 4]`、`BEAM_SWITCH_LOSS_DB`、`BEAM_SWITCH_DELAY`） |

### ③ 出處
- **判定形式**（outage=SE<門檻）奠基：Ozarow, Shamai, Wyner, IEEE TVT 1994；Goldsmith 2005 / Tse & Viswanath 2005 —— 只支持**定義**，不支持某特定門檻值。
- **門檻值**：**QoS 設計參數，非文獻規定**。主值 $C=2$、消融 $\{1,2,4\}$（2026-07-16，前身 C=4）。無單一權威可強制：3GPP V2X (TS 22.186) 以 PER/Mbps 表達、CQI-1 可解調下限≈0.15 bps/Hz。$C=4$ 原借自 MultiUser Table I（28GHz/15dBm 高品質服務門檻），修正 N₀ 後對本預算偏嚴，改主值 $C=2$（實測 baseline LOS 88%/NLOS 60%，環境合理）。

### ④ 檢驗
```matlab
runtests('tests/testLinkSuccess.m')   % 邊界：SNR=15→過、14→不過
linkSuccess(15,4), linkSuccess(14.9,4)
```

---

## 9. 吞吐量（full-buffer 速率式）

> **這在幹嘛：** 算「這一步實際傳了多少資料」——連上才有流量，且要扣掉掃描佔用的時間，這就是獎勵函數裡「掃太多會虧」的來源。

### ① 這是什麼
「這一步實際傳了多少資料」。full-buffer = 假設永遠有資料要傳（RL 訓練標準假設），送達量直接反映波束品質，不被人造的封包佇列動態干擾。

### ② 公式

$$
R_{eff} = (1-\rho_a)\, B \log_2(1+\mathrm{SNR})\cdot \mathbb{1}\{\text{link}\}, \qquad \text{served}[t] = R_{eff}\,\Delta t
$$

$\rho_a\in\{0.05, 0.25, 0.50\}$ = 動作 $a$ 的掃描開銷（掃描佔用時頻資源 → 直接砍吞吐——「掃太多會虧」的機制來源）。

次要指標（給導師看封包/可靠度面，不入獎勵）：累積送達 $\sum \text{served}$、連線成功率 $\frac{1}{T}\sum \mathbb{1}\{\text{link}\}$。

### ②½ 程式對應
| 內容                                                                                | 檔案                                        |
| --------------------------------------------------------------------------------- | ----------------------------------------- |
| $R_{eff}=(1-\rho)B\log_2(1+\mathrm{SNR})\cdot\mathbb{1}\{\text{link}\}$、served 累積 | `env/myStep.m`（Throughput 區塊）             |
| $\rho_a$ 開銷表                                                                      | `config.m`（`rhoTable = [0.05 0.25 0.50]`） |
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

轉彎：半徑 $r=6$ m 圓弧、減速因子 0.6，每步轉角 $\Delta\psi = \dfrac{0.6\,v\,\Delta t}{r}$。速度集合 $\{30,45,60\}$ km/h（每 episode 抽定）。

> 🟡 **出處註記**：$r=6$ m、減速 0.6 為**工程設計選擇**（無文獻直接出處，取都會路口典型轉彎半徑量級）；速度 30–60 km/h 依計畫書。轉角式為圓弧運動學標準。

### ②½ 程式對應
| 內容 | 檔案 |
|---|---|
| 位置更新、路口轉向決策、轉彎弧、車道吸附、駛離判定 | `env/vehicleMobility.m`（全檔；道路座標內部讀 `sceneGeometry()`） |
| 速度集合/轉向機率 | `config.m`（`vRange`、`pStraight/pLeft/pRight`） |
| 進入點抽選與初始 heading | `env/myReset.m` |

### ③ 出處
- Manhattan Mobility Model（V2X 模擬標準，SUMO 慣例）
- 路口轉向機率 0.5/0.25/0.25：**設計選擇**，計畫書 §步驟一(L130)一致（直行偏多符合真實車流）。⚠️ **非出自 MultiUser**——MultiUser 原文為「三向各 1/3」，前版誤引已更正（2026-07-16）

### ④ 檢驗
```matlab
runtests('tests/testVehicleMobility.m')  % 5 測試：吸附/前進/駛離/進入點/gNB固定
% 手動：跑 3 個 seed 畫軌跡 → 全程貼路、轉彎是弧
```

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

**成績階梯的意義**：oracle（每步全掃碼本全部 128 束）~97% = 環境物理天花板；它與（受限動作空間的）週期掃描 ~82% 之間 ~15 個百分點的縫 = Part 2 DRL 的研究空間。（數字為 2026-07-16 修正參數 N₀=−86/C=2/L_refl=0/NLOS損耗用d3D 下實測；舊值 93.7%/97.7% 為修正前，已淘汰。）

> **oracle（exhaustive/genie 上限）出處**：以「窮舉波束搜尋／genie-aided」當效能上限是波束管理標準慣例——核心論文 **MultiUser 即以 AoD-Genie 為效能上限**（[[notes/MultiUserBeamforming_DRL_Sensing]] 譯文 §V，完美 AoD）；3GPP NR 波束管理 P-1/P-2 **exhaustive beam sweeping**（TR 38.802、TS 38.213/214）；tutorial：Giordani et al., *IEEE Commun. Surveys & Tutorials*, 2019（arXiv:1804.01908）。⚠️ 誠實區分：MultiUser 上限=完美 AoD；本專題 oracle=全掃 128 碼本波束（exhaustive），同類但實作不同。97% 為本專題實測值（非文獻值）。

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

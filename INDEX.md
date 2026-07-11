# INDEX — 論文總索引（Claude 召回層）

> 一行一篇，先讀這份決定要翻哪篇 `notes/`。新論文由 `scripts/ingest.ps1` 自動 append。

## 核心論文
- [[notes/MultiUserBeamforming_DRL_Sensing]] — DRL+回波多波束波束成形，免回饋、吞吐量獎勵、3 動作。relevance:high #核心論文
- [[notes/Shakya2025_UrbanPropagation]] — FR3(16.95GHz) 實測通道：PLE/DS/AS 均低於 3GPP，可修正 `config.m`。relevance:high #核心論文
- [[notes/FR3_Rel19_ChannelModeling]] — 3GPP Rel-19 FR3(7–24GHz) 通道建模教學：反內插、SMa、群集數可變、絕對ToA；近場/SNS 因遠場128不採用。relevance:high #核心論文

## 強化學習
- [[notes/Salami2025_TransferRL_BeamSelection]] — 點雲指紋+Chamfer 距離選模型做遷移 RL，省 16×。relevance:high #強化學習
- [[notes/Zhao2024_IBTD_ISAC_V2V]] — DRQN+LSTM 無導頻 ISAC 波束追蹤、封包齡、乘法獎勵。relevance:high #強化學習
- [[notes/Wang2026_V2X_MARL_Benchmarking]] — MARL 干擾博弈基準，PPO 家族最佳、局部觀測勝全域。relevance:high #強化學習

## 其他
- [[notes/Palenik2024_OFDM_Interference]] — OFDM 干擾分類(ISI/IBI/ICI)+OTFS 都卜勒韌性。relevance:med #其他

## ⏳ 待完整收錄（僅索引+連結，尚未跑 MinerU 翻譯；2026-07-12 加入）
> Episode 定義 / V2X 波束追蹤 episode 長度佐證（先庫後網對照時查到）。收錄前只用摘要層級引用，勿當已驗證全文。
- **DL-Based Beam Management for mmWave Vehicular Networks Exploring Temporal Correlation** — arXiv:2511.02260（2025）— **episode = 一條連續車輛軌跡，車駛離範圍/邊界自然終止**（直接佐證本專題穿越制）。https://arxiv.org/abs/2511.02260 #待收錄 #episode
- **Machine Learning-Based mmWave MIMO Beam Tracking in V2I** — arXiv:2412.05427（2024）— V2I 波束追蹤、軌跡含轉彎/變速。https://arxiv.org/abs/2412.05427 #待收錄
- **DRL Algorithms for Hybrid V2X Communication** — arXiv:2310.03767（2024）— V2X DRL 基準比較。https://arxiv.org/abs/2310.03767 #待收錄

## 標準與教科書（參考條目，無 notes；SPEC.md Part 1 引用）
- Goldsmith, *Wireless Communications*, Cambridge 2005 — outage=P(SNR<門檻) 教科書定義（用於 1.6 link_success）
- Tse & Viswanath, *Fundamentals of Wireless Communication*, Cambridge 2005 — outage/目標速率（1.6）
- Ozarow, Shamai, Wyner, IEEE TVT 1994 — 蜂巢 outage capacity 奠基（1.6）
- Stoica & Nehorai, IEEE Trans. ASSP 1989 — DOA Cramér-Rao Bound 奠基（1.5 CRLB）
- Gudmundson, Electronics Letters 1991 — 陰影衰落指數空間相關（1.2）
- Clarke 1968 / Jakes 1974 — 都卜勒衰落（1.2 NLOS 相位）
- Skolnik, *Introduction to Radar Systems* — 雷達方程式（1.4 echo）
- GEMV² — Boban, Barros, Tonguz, IEEE TVT 2014 — 車聯網幾何 LOS/NLOS（1.1）
- 3GPP TR 38.901（UMi 通道）、TR 37.885（V2X 流量）、TR 38.331（T310 RLF）
- ITU-R M.2135-1 — IMT-Advanced 評估指引（場景，注意 UMi 為六角形 ISD 200m）
- ETSI 23–27GHz 車輛 RCS 實測；3GPP ISAC RCS 模型（arXiv 2505.20673）（1.9 RCS）

---
共 7 篇 notes + 標準/教科書參考。relevance:high ×6、med ×1。（FR3 pipeline 收錄中：ISAC_meets_FR3、RoadwayGeometry_ISAC_BeamTracking、BaiHeath2014_BlockageModel）

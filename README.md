# 專案研究報告：ET-BERT 與通用 LLM 在加密流量分析之比較
**Comparative Study: ET-BERT vs. General LLMs for Encrypted Traffic Analysis**

---

## 1. 摘要 (Abstract)
隨著傳輸層安全性協定 (TLS 1.3) 的普及，傳統深度封包檢測 (DPI) 已無法有效識別惡意流量。本報告對比了兩種主流的 AI 解決方案：**領域專用模型 ET-BERT** 與 **通用大型語言模型 (General LLMs)**。分析顯示，雖然通用 LLM 在文本理解上表現卓越，但在處理原始網路封包 (Raw PCAP/Hex) 時，ET-BERT 憑藉其專用的 **分詞策略 (Tokenization)** 與 **雙向編碼架構 (Bi-directional Encoder)**，在識別遠端存取工具 (RATs) 與 C2 通訊特徵上展現出更高的效率與準確度。

---

## 2. 模型架構與原理比較 (Architecture & Mechanism)

在加密流量分析的場景中，這兩種模型的根本差異在於它們「如何閱讀」與「如何理解」封包數據。

### 2.1 ET-BERT (Encrypted Traffic BERT)
* **本質：** 屬於 **Encoder-only** 架構（基於 BERT）。
* **核心邏輯：** 專為「理解與分類」設計。它將網路封包的 Hex 序列視為一種「外星語言」，並試圖理解其文法結構。
* **分詞方式 (Tokenization)：** 採用 **Burst-based** 策略。它通常將兩個 Hex 字符（如 `ff`）視為一個 Token（即 1 Byte = 1 Token）。這與計算機網路的最小單位（Byte）完美對齊。
* **注意力機制：** **雙向注意力 (Bi-directional Attention)**。模型在判斷某個 Byte 的意義時，會同時參考它「前面」與「後面」的 Byte。這對於理解加密握手 (Handshake) 的完整上下文至關重要。

### 2.2 通用 LLM (General Large Language Models)
* **本質：** 大多屬於 **Decoder-only** 架構（如 GPT 系列）或 Encoder-Decoder 架構。
* **核心邏輯：** 專為「生成與接龍」設計。它們主要學習的是人類自然語言（英語、程式碼）的概率分佈。
* **分詞方式 (Tokenization)：** 採用 **BPE (Byte Pair Encoding)** 或 WordPiece。這在處理 Hex 數據時效率極低，例如 `a1b2` 可能被切分成不規則的片段，破壞了網路協定的結構完整性。
* **注意力機制：** 通常是 **單向/自回歸注意力 (Uni-directional / Auto-regressive)**。模型習慣預測「下一個字」，這對於分類任務來說，不如雙向關注整體結構來得直接有效。

---

## 3. 針對特定目標的適用性分析

我們針對您關注的四類目標進行理論效能評估：

### 3.1 遠端桌面工具 (TeamViewer, AnyDesk)
* **特徵：** 這類合法工具的流量具有高度規律的「心跳包 (Heartbeat)」與「螢幕更新突發 (Screen Update Bursts)」。
* **ET-BERT 優勢：** ET-BERT 能夠捕捉這些微小的 Payload 長度變化與 Byte 分佈模式。由於它在預訓練階段看過數百萬計的 TLS 握手，它能輕易區分 TeamViewer 的專有加密協議與標準 HTTPS 的差異。
* **LLM 劣勢：** 通用 LLM 缺乏對 TeamViewer 專有二進位協定的先驗知識 (Prior Knowledge)。除非經過極大量的指令微調 (Instruction Tuning)，否則 LLM 很難從一堆 Hex 亂碼中「推理」出這是 AnyDesk。

### 3.2 勒索軟體 (Ransomware)
* **特徵：** 勒索軟體在加密前會進行「密鑰交換」與「C2回報」，這通常發生在極短的時間窗口內，且封包大小固定。
* **ET-BERT 優勢：** 擅長處理 **Sequence Classification**（序列分類）。即使 Payload 被加密，ET-BERT 也能偵測到特定的 TLS Client Hello 指紋 (JA3 fingerprints) 變體，這是勒索軟體的典型特徵。
* **LLM 劣勢：** 勒索軟體流量屬於「少樣本 (Few-shot)」場景。通用 LLM 需要大量的 Context Window 來放入 Hex 數據，且容易產生幻覺 (Hallucination)，誤將正常的加密備份流量判讀為勒索行為。

### 3.3 Cobalt Strike (C2 & Malleable Profiles)
* **特徵：** Cobalt Strike 最強大的功能是 Malleable C2，可以偽裝成 Google、Amazon 的流量。
* **ET-BERT 優勢：** **這是不實作比較中的決勝點。** 攻擊者可以偽裝統計特徵（改封包大小、時間），但很難完全偽裝協定的「深層語言結構」。ET-BERT 透過 Masked Language Modeling (MLM) 訓練，能發現那些「看起來像 Google 但文法怪怪的」封包序列，進而識破偽裝。
* **LLM 劣勢：** LLM 更擅長語義層面的分析（例如分析 HTTP Log 中的 User-Agent 字串），但面對純粹的加密二進位流 (Encrypted Binary Stream)，LLM 的推理能力會大幅下降。

---

## 4. 綜合比較表 (Comparison Matrix)

| 比較維度 | ET-BERT (領域專用模型) | General LLM (通用模型) |
| :--- | :--- | :--- |
| **輸入數據型態** | **Raw Hex / Bytes** (原始封包) | **Text / Logs** (文字日誌) |
| **知識領域** | **網路協定、加密指紋、流量行為** | 人類語言、程式碼、常識 |
| **架構優勢** | **雙向理解 (Bi-directional)**<br>能看清封包的前後文關係 | **生成能力 (Generative)**<br>能解釋為什麼，但對Hex敏感度低 |
| **Tokenization** | **Byte-level** (精準對齊協定) | **Subword-level** (容易切碎協定特徵) |
| **運算成本** | **低** (通常 < 1億參數，推論快) | **極高** (通常 > 70億參數，推論慢) |
| **對抗 Cobalt Strike** | **強** (能識別深層偽裝特徵) | **弱** (易受偽裝干擾，除非有解析後的 Log) |
| **部署場景** | 邊緣設備、防火牆、IDS | 雲端分析中心、資安報告生成 |

---

## 5. 結論與建議 (Conclusion)

### 5.1 結論
在 **加密流量分類 (Encrypted Traffic Classification)** 這一具體任務上，**ET-BERT 顯著優於通用 LLM**。
* **ET-BERT** 就像是一位專精於閱讀二進位代碼的**密碼學專家**，它不需要「說話」，只需要精準地判斷「這是好是壞」。
* **通用 LLM** 就像是一位**通識教授**，它能讀懂防火牆的日誌報告並告訴你發生了什麼，但如果你直接丟給它一堆加密後的亂碼 (Hex)，它無法像 ET-BERT 那樣敏銳地察覺其中的微小模式。

### 5.2 建議策略
在實際的資安防禦體系中，建議採用 **分層式架構**：
1.  **第一線 (偵測層)：** 使用 **ET-BERT** 處理海量的原始流量 (PCAP)，快速過濾出 TeamViewer、AnyDesk 或潛在的 Cobalt Strike 流量。因為它速度快、精度高。
2.  **第二線 (分析層)：** 當 ET-BERT 發出警報後，將相關的 Metadata（來源 IP、解析後的 Header 資訊）丟給 **通用 LLM**，讓 LLM 結合威脅情資 (CTI) 生成人類可讀的資安事件報告。

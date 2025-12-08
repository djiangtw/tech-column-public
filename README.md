# Tech Column - 技術專欄

**深入淺出的系統架構與硬體設計**

**作者**: Danny Jiang  
**授權**: CC BY 4.0 International  
**最後更新**: 2025 年 12 月

---

## 📖 關於本專欄

Tech Column 是一個技術寫作專案，專注於系統架構、硬體設計和性能優化領域。本專欄的目標是用生動的比喻和真實的案例，將複雜的技術概念解釋得清晰易懂，讓讀者不僅知道「是什麼」，更理解「為什麼」。

**關於案例**：本專欄文章中的案例場景均為**模擬場景**，基於業界先進經驗並隱去機敏資訊撰寫。所有內容符合職業道德和保密協議要求，不涉及任何公司的專有技術或商業機密。

### 本專欄的特色

- **生動的比喻**：用圖書館理解 Cache、用停車場理解 Associativity、用城市交通理解 NoC
- **真實的案例**：來自 20+ 年產業經驗的實際問題和解決方案
- **循序漸進**：從入門到進階，系統性地建立知識體系
- **實務導向**：不只是理論，更提供可執行的優化建議和設計原則

---

## 📚 文章系列

### Cache Architecture 系列

深入探討 CPU Cache 的設計與優化，從基礎概念到實戰應用。

1. [Cache 基礎概念入門：用圖書館理解 CPU Cache](topics/cache-architecture/01-cache-basics.md)
2. [理解 Cache Associativity：停車場的智慧](topics/cache-architecture/02-cache-associativity.md)
3. [現代 CPU Cache 架構設計：從 L1 到 L3 的設計哲學](topics/cache-architecture/03-modern-cache-design.md)
4. [Cache Coherency 與 MESI 協議：多核心時代的一致性挑戰](topics/cache-architecture/04-cache-coherency-mesi.md)
5. [Cache 性能優化實戰：從理論到實踐](topics/cache-architecture/05-cache-optimization.md)
6. [False Sharing 與多執行緒優化：看不見的性能殺手](topics/cache-architecture/06-false-sharing.md)

**系列特色**：
- 📖 從入門到進階，循序漸進
- 🎯 用生動的比喻解釋複雜概念（圖書館、停車場）
- �� 結合真實案例與實務經驗
- 🔧 提供可執行的優化建議

**總字數**：約 20,800 字

---

### Network-on-Chip 系列

探索晶片內部的通訊架構，從 Bus 到 Network 的演進。

1. [Network-on-Chip 入門：從 Bus 到 Network 的演進](topics/network-on-chip/01-noc-introduction.md)
2. [NoC 拓撲結構的圖論分析：從數學到硬體](topics/network-on-chip/02-topology-graph-theory.md)
3. [NoC 路由演算法與死鎖避免：從理論到實作](topics/network-on-chip/03-routing-deadlock.md)
4. [Router 微架構設計：從 Pipeline 到硬體實作](topics/network-on-chip/04-router-microarchitecture.md)
5. [NoC 與 Cache Coherency 整合：多核心的協調藝術](topics/network-on-chip/05-noc-cache-coherency.md)
6. [NoC 與先進封裝：突破物理邊界](topics/network-on-chip/06-noc-advanced-packaging.md)

**系列特色**：
- 🌐 從基礎到前沿技術
- 📊 結合圖論與硬體設計
- 🔬 深入探討 Router 微架構
- 🚀 涵蓋先進封裝技術（Chiplet、UCIe、CoWoS）

**總字數**：約 14,100 字

---

## 🎯 目標讀者

本專欄適合：

- **系統軟體工程師**：想要理解硬體如何影響軟體性能
- **硬體工程師**：從事 CPU、SoC 設計和驗證
- **性能優化工程師**：需要深入理解 Cache 和 NoC 行為
- **電腦架構學生**：在真實世界情境中學習系統架構
- **技術愛好者**：對計算機底層原理感興趣

**先備知識**：
- 基本的計算機組織概念
- 理解 CPU、記憶體、匯流排等基本元件
- 有程式設計經驗（有幫助但非必需）

---

## 📊 統計資訊

- **文章總數**：12 篇
- **總字數**：約 35,000 字
- **系列數**：2 個
- **涵蓋主題**：Cache、NoC、性能優化、多核心架構、先進封裝
- **最後更新**：2025 年 12 月

---

## 📄 授權

**版權所有 © 2025 Danny Jiang**

本專欄所有文章採用 **Creative Commons Attribution 4.0 International License (CC BY 4.0)** 授權。

**您可以自由地：**

- **分享** — 以任何媒介或格式複製及散布本素材
- **修改** — 重混、轉換本素材，及依本素材建立新素材，且為任何目的，包含商業性質之使用

**惟需遵守下列條件：**

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更

**授權條款**：https://creativecommons.org/licenses/by/4.0/

---

## �� 如何使用本專欄

### 線上閱讀

直接在 GitHub 上瀏覽 Markdown 檔案：
- **Cache Architecture 系列**：從 `topics/cache-architecture/01-cache-basics.md` 開始
- **Network-on-Chip 系列**：從 `topics/network-on-chip/01-noc-introduction.md` 開始

### 離線閱讀

Clone 此 repository：
```bash
git clone https://github.com/djiangtw/tech-column-public.git
cd tech-column-public
```

使用任何 Markdown 閱讀器或文字編輯器閱讀檔案。

### 推薦閱讀順序

**初學者**：
1. 先讀 Cache Architecture 系列（01-06）
2. 再讀 Network-on-Chip 系列（01-06）

**有經驗的工程師**：
- 可以根據興趣選擇特定主題
- 每篇文章都相對獨立，可以單獨閱讀

---

## 🤝 貢獻

這是一個唯讀的公開 repository。本專欄在私有 repository 中開發。

**歡迎回饋**：
- 針對錯字、錯誤或建議開 issue
- 鼓勵討論和提問
- 分享您的閱讀心得

**注意**：無法接受 pull request，因為這是從私有開發 repository 單向同步的。

---

## 👨‍💻 關於作者

**Danny Jiang**

系統軟體工程師，專注於 RISC-V 架構、嵌入式系統、性能優化。20+ 年產業經驗，熱愛用生動的比喻解釋複雜的技術概念。

**專業領域**：
- RISC-V 架構與系統軟體
- CPU Cache 與性能優化
- Network-on-Chip 設計
- 嵌入式系統開發

**其他作品**：
- [See RISC-V Run: Fundamentals](https://github.com/djiangtw/see-riscv-run-public) - RISC-V 架構完整指南
- [Data Structures in Practice](https://github.com/djiangtw/data-structures-in-practice-public) - 硬體導向的資料結構

---

## 🔗 連結

- **GitHub**: https://github.com/djiangtw/tech-column-public
- **作者**: [Danny Jiang](https://github.com/djiangtw)
- **Email**: djiang.tw@gmail.com
- **LinkedIn**: [linkedin.com/in/danny-jiang-26359644](https://www.linkedin.com/in/danny-jiang-26359644/)

---

## 📝 引用

如果您在研究、教學或文章中引用本專欄，請使用：

```
Danny Jiang. (2025). Tech Column - 技術專欄：深入淺出的系統架構與硬體設計.
採用 CC BY 4.0 授權. https://github.com/djiangtw/tech-column-public
```

---

## 🙏 致謝

本專欄得以完成，感謝：

- **技術社群**：RISC-V、ARM、開源硬體社群的知識分享
- **經典教材**：*Computer Architecture: A Quantitative Approach*、*See MIPS Run* 等啟發
- **早期讀者**：提供寶貴的反饋和建議
- **家人和朋友**：在寫作過程中給予支持

---

## 📅 更新計劃

**已完成**：
- ✅ Cache Architecture 系列（6 篇）
- ✅ Network-on-Chip 系列（6 篇）

**規劃中**：
- 🔄 Lock-Free Programming 系列
- 🔄 System Architecture 系列
- 🔄 更多主題持續更新中...

---

**祝閱讀愉快！** 📖

如有任何問題或建議，歡迎透過 GitHub Issues 與我聯繫。

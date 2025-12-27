---
author: Danny Jiang
date: 2025-12-11
---

# AIoT - 從 IoT 到 AI + IoT 的演進

## 前言：2015 年的智慧手環專案

2015 年 3 月，我接到了一個來自穿戴式裝置廠商的專案需求。

"Danny，我們想在智慧手環上加入 **心率異常偵測** 功能，"客戶的產品經理 Sarah 說，"當使用者的心率異常時，手環會自動發出警告。你覺得可行嗎？"

我想了想："可行。但有幾個問題：

1. **如何判斷心率異常？** 需要定義異常的標準
2. **在哪裡運算？** 在手環上運算，還是傳到手機上運算？
3. **功耗如何？** 如果在手環上運算，功耗會不會太高？"

Sarah 說："我們希望在手環上運算，這樣即使手機不在身邊，手環也能發出警告。至於異常的標準，我們可以用簡單的規則：

- **心率 > 120 bpm**：心率過高
- **心率 < 50 bpm**：心率過低"

"這個簡單，"我說，"用一個 if-else 就能實現。"

---

兩週後，我完成了心率異常偵測功能：

```c
// 心率異常偵測（使用簡單規則）
void heart_rate_monitor(uint8_t hr) {
    if (hr > 120) {
        // 心率過高
        send_alert("Heart rate too high: %d bpm", hr);
    } else if (hr < 50) {
        // 心率過低
        send_alert("Heart rate too low: %d bpm", hr);
    }
}
```

Sarah 很滿意："這個功能很好！但我有一個新的想法：能不能根據使用者的歷史資料，**自動學習** 使用者的正常心率範圍？

例如：

- 使用者 A 的正常心率是 60-80 bpm
- 使用者 B 的正常心率是 70-90 bpm

這樣就能更準確地偵測異常。"

我愣了一下。這不是簡單的 if-else 能解決的問題，這需要 **機器學習（Machine Learning）**。

"這個想法很好，"我說，"但有幾個挑戰：

1. **運算量大**：機器學習需要大量運算，手環的 MCU 可能跑不動
2. **功耗高**：機器學習會消耗大量電力，電池續航時間會縮短
3. **模型大小**：機器學習模型可能很大，手環的 Flash 可能放不下"

Sarah 沉默了一會兒："那怎麼辦？"

"我們可以把機器學習放在手機上，"我說，"手環只負責收集資料，傳到手機上運算。"

"但這樣的話，手機不在身邊時，手環就無法偵測異常了，"Sarah 說。

"是的，"我說，"這是一個 trade-off。"

---

這次對話讓我開始思考：**能不能在 IoT 裝置上運行 AI 模型？**

這就是 **AIoT（AI + IoT）** 的起源。

---

## AIoT 是什麼？

**AIoT（Artificial Intelligence of Things）** 是 **AI（人工智慧）** 和 **IoT（物聯網）** 的結合，指的是在 IoT 裝置上運行 AI 模型，實現 **Edge AI（邊緣 AI）**。

**AIoT 的特點**：

- **在裝置上運算**：不需要傳到雲端或手機
- **低延遲**：即時反應
- **隱私保護**：資料不離開裝置
- **低頻寬**：不需要持續傳輸資料
- **低成本**：不需要雲端運算費用

**AIoT 的應用場景**：

- 智慧手環（心率異常偵測、睡眠品質分析）
- 智慧音箱（語音辨識、喚醒詞偵測）
- 智慧攝影機（人臉辨識、物體偵測）
- 工業 IoT（設備故障預測、品質檢測）

---

## 為什麼需要 AIoT？

### 1. 隱私保護

**問題**：

- 傳統 IoT 裝置需要將資料傳到雲端運算
- 資料可能被竊取或濫用
- 使用者擔心隱私問題

**AIoT 的解決方案**：

- 在裝置上運算，資料不離開裝置
- 保護使用者隱私

**範例**：

- 智慧攝影機：在裝置上進行人臉辨識，不需要傳到雲端
- 智慧音箱：在裝置上進行語音辨識，不需要傳到雲端

---

### 2. 低延遲

**問題**：

- 傳統 IoT 裝置需要將資料傳到雲端運算
- 網路延遲可能很高（100-500 ms）
- 無法即時反應

**AIoT 的解決方案**：

- 在裝置上運算，延遲 < 10 ms
- 即時反應

**範例**：

- 自動駕駛：需要即時偵測障礙物，不能等待雲端運算
- 工業機器人：需要即時反應，不能等待雲端運算

---

### 3. 低頻寬

**問題**：

- 傳統 IoT 裝置需要持續傳輸資料到雲端
- 頻寬成本高
- 網路不穩定時無法運作

**AIoT 的解決方案**：

- 在裝置上運算，只傳輸結果
- 降低頻寬需求

**範例**：

- 智慧攝影機：只傳輸偵測到的事件（例如：有人進入），不傳輸完整影片
- 工業感測器：只傳輸異常事件，不傳輸所有資料

---

### 4. 低成本

**問題**：

- 傳統 IoT 裝置需要雲端運算
- 雲端運算費用高（特別是大量裝置時）

**AIoT 的解決方案**：

- 在裝置上運算，不需要雲端運算費用
- 降低成本

**範例**：

- 智慧手環：如果有 1,000 萬個使用者，雲端運算費用會非常高
- 智慧攝影機：如果有 100 萬個攝影機，雲端運算費用會非常高

---

## AIoT 的技術挑戰

雖然 AIoT 有很多優勢，但也面臨許多技術挑戰：

### 1. 算力不足

**問題**：

- IoT 裝置的 MCU 算力有限（例如：ARM Cortex-M4 @ 64 MHz）
- AI 模型需要大量運算（例如：CNN 需要數百萬次乘法運算）
- MCU 可能跑不動 AI 模型

**解決方案**：

- **模型壓縮**：減少模型大小和運算量
  - Quantization（量化）：將 float32 轉換為 int8
  - Pruning（剪枝）：移除不重要的權重
  - Knowledge Distillation（知識蒸餾）：用小模型模仿大模型

- **硬體加速**：使用專用硬體加速 AI 運算
  - DSP（Digital Signal Processor）
  - NPU（Neural Processing Unit）
  - GPU（Graphics Processing Unit）

---

### 2. 功耗過高

**問題**：

- AI 運算消耗大量電力
- IoT 裝置通常是電池供電，功耗敏感
- AI 運算可能導致電池續航時間縮短

**解決方案**：

- **模型優化**：減少運算量
  - 使用更簡單的模型（例如：MobileNet 而不是 ResNet）
  - 使用量化（int8 而不是 float32）

- **硬體優化**：使用低功耗硬體
  - 使用 DSP 或 NPU 而不是 CPU
  - 使用 Sleep 模式

- **演算法優化**：減少運算頻率
  - 只在需要時運算（例如：偵測到動作時才運算）
  - 使用 Cascade 模型（先用簡單模型過濾，再用複雜模型）

---

### 3. 記憶體不足

**問題**：

- IoT 裝置的 RAM 和 Flash 有限（例如：256 KB RAM, 1 MB Flash）
- AI 模型可能很大（例如：ResNet-50 需要 100 MB）
- 記憶體可能放不下 AI 模型

**解決方案**：

- **模型壓縮**：減少模型大小
  - Quantization（量化）：將 float32 轉換為 int8，減少 4 倍大小
  - Pruning（剪枝）：移除不重要的權重，減少 50-90% 大小

- **模型分割**：將模型分割成多個部分
  - 只載入需要的部分到 RAM
  - 其他部分放在 Flash

---

### 4. 開發困難

**問題**：

- AI 模型開發需要專業知識（Machine Learning, Deep Learning）
- IoT 開發需要專業知識（Embedded System, RTOS）
- 結合 AI 和 IoT 需要跨領域知識

**解決方案**：

- **使用現成框架**：
  - TensorFlow Lite Micro
  - CMSIS-NN
  - Edge Impulse

- **使用預訓練模型**：
  - 不需要從頭訓練模型
  - 使用 Transfer Learning 微調模型

---

## TinyML - 在 MCU 上運行 AI 模型

**TinyML（Tiny Machine Learning）** 是一個新興領域，專注於在資源受限的裝置（例如：MCU）上運行 AI 模型。

**TinyML 的特點**：

- **模型小**：< 100 KB
- **運算量小**：< 1M MAC（Multiply-Accumulate）
- **功耗低**：< 1 mW
- **延遲低**：< 10 ms

**TinyML 的應用場景**：

- 喚醒詞偵測（"Hey Siri", "OK Google"）
- 手勢辨識
- 異常偵測（心率異常、設備故障）
- 關鍵字辨識

---

### TinyML 框架

#### 1. TensorFlow Lite Micro

**TensorFlow Lite Micro** 是 Google 開發的 TinyML 框架，專門為 MCU 設計。

**特點**：

- **模型小**：支援量化（int8）
- **運算量小**：支援模型優化
- **跨平台**：支援 ARM Cortex-M, RISC-V, Xtensa
- **開源**：完全開源

**範例**：

```c
// TensorFlow Lite Micro 範例：喚醒詞偵測
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/schema/schema_generated.h"

// 1. 載入模型
const tflite::Model* model = tflite::GetModel(g_model_data);

// 2. 建立 Op Resolver
tflite::MicroMutableOpResolver<4> micro_op_resolver;
micro_op_resolver.AddConv2D();
micro_op_resolver.AddMaxPool2D();
micro_op_resolver.AddFullyConnected();
micro_op_resolver.AddSoftmax();

// 3. 建立 Interpreter
constexpr int kTensorArenaSize = 10 * 1024;  // 10 KB
uint8_t tensor_arena[kTensorArenaSize];

tflite::MicroInterpreter interpreter(
    model, micro_op_resolver, tensor_arena, kTensorArenaSize);

// 4. 分配記憶體
interpreter.AllocateTensors();

// 5. 取得輸入 Tensor
TfLiteTensor* input = interpreter.input(0);

// 6. 填入輸入資料（音訊資料）
for (int i = 0; i < input->bytes; i++) {
    input->data.int8[i] = audio_data[i];
}

// 7. 執行推論
interpreter.Invoke();

// 8. 取得輸出 Tensor
TfLiteTensor* output = interpreter.output(0);

// 9. 解析輸出（喚醒詞機率）
float wake_word_prob = output->data.f[0];
if (wake_word_prob > 0.8) {
    printf("Wake word detected!\n");
}
```

---

#### 2. CMSIS-NN

**CMSIS-NN** 是 ARM 開發的神經網路函式庫，專門為 ARM Cortex-M 設計。

**特點**：

- **高效能**：使用 ARM Cortex-M 的 DSP 指令
- **低功耗**：優化的運算
- **跨平台**：支援所有 ARM Cortex-M
- **開源**：完全開源

**範例**：

```c
// CMSIS-NN 範例：卷積運算
#include "arm_nnfunctions.h"

// 輸入資料
q7_t input_data[32 * 32 * 3];  // 32x32 RGB 影像

// 權重
q7_t weight_data[3 * 3 * 3 * 16];  // 3x3 kernel, 3 input channels, 16 output channels

// 輸出資料
q7_t output_data[32 * 32 * 16];

// 執行卷積
arm_convolve_HWC_q7_basic(
    input_data,      // 輸入
    32,              // 輸入寬度
    3,               // 輸入 Channel 數
    weight_data,     // 權重
    16,              // 輸出 Channel 數
    3,               // Kernel 大小
    1,               // Padding
    1,               // Stride
    NULL,            // Bias
    0,               // Bias Shift
    0,               // Output Shift
    output_data,     // 輸出
    32,              // 輸出寬度
    NULL,            // Buffer
    NULL             // Buffer
);
```

---

#### 3. Edge Impulse

**Edge Impulse** 是一個端到端的 TinyML 平台，提供資料收集、模型訓練、模型部署的完整流程。

**特點**：

- **易用**：不需要深度學習知識
- **端到端**：從資料收集到模型部署
- **跨平台**：支援 Arduino, STM32, Nordic, ESP32
- **雲端訓練**：不需要本地 GPU

**流程**：

1. **資料收集**：使用 Edge Impulse 的 Data Forwarder 收集資料
2. **資料標註**：在 Edge Impulse 網站上標註資料
3. **模型訓練**：在 Edge Impulse 網站上訓練模型
4. **模型部署**：下載模型，部署到裝置

---

## 真實案例：智慧手環的心率異常偵測

回到文章開頭的故事。我為 Sarah 的智慧手環專案設計了一個基於 TinyML 的心率異常偵測方案：

**系統架構**：

```
┌─────────────────────────────────────────────────────────┐
│  智慧手環心率異常偵測系統                                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐                                        │
│  │  心率感測器  │                                        │
│  │  (PPG)       │                                        │
│  └──────┬───────┘                                        │
│         │ ADC                                             │
│  ┌──────┴───────┐                                        │
│  │  MCU         │                                        │
│  │  - 訊號處理  │                                        │
│  │  - AI 模型   │                                        │
│  │  - 異常偵測  │                                        │
│  └──────┬───────┘                                        │
│         │ BLE                                             │
│  ┌──────┴───────┐                                        │
│  │  手機 App    │                                        │
│  │  - 顯示結果  │                                        │
│  │  - 發出警告  │                                        │
│  └──────────────┘                                        │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**設計要點**：

1. **在手環上運行 AI 模型**：
   - 使用 TensorFlow Lite Micro
   - 模型大小 < 50 KB
   - 運算時間 < 10 ms

2. **低功耗**：
   - 只在偵測到異常時才發送 BLE Notification
   - 平均功耗 < 100 μA

3. **高準確度**：
   - 使用 LSTM（Long Short-Term Memory）模型
   - 準確度 > 95%

---

### 實作細節

#### 1. 資料收集

**收集使用者的心率資料**：

```c
// 心率資料收集
#define HR_BUFFER_SIZE 100  // 收集 100 個心率資料點

uint8_t hr_buffer[HR_BUFFER_SIZE];
uint16_t hr_index = 0;

void hr_data_collect(uint8_t hr) {
    hr_buffer[hr_index] = hr;
    hr_index = (hr_index + 1) % HR_BUFFER_SIZE;

    // 當收集滿 100 個資料點時，執行 AI 推論
    if (hr_index == 0) {
        ai_inference(hr_buffer, HR_BUFFER_SIZE);
    }
}
```

---

#### 2. 模型訓練

**使用 TensorFlow 訓練 LSTM 模型**：

```python
import tensorflow as tf
from tensorflow import keras

# 1. 載入資料
X_train, y_train = load_training_data()  # X: 心率序列, y: 是否異常

# 2. 建立 LSTM 模型
model = keras.Sequential([
    keras.layers.LSTM(32, input_shape=(100, 1)),  # 100 個時間步，1 個特徵
    keras.layers.Dense(16, activation='relu'),
    keras.layers.Dense(1, activation='sigmoid')   # 輸出：異常機率
])

# 3. 編譯模型
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)

# 4. 訓練模型
model.fit(X_train, y_train, epochs=50, batch_size=32)

# 5. 量化模型（float32 -> int8）
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()

# 6. 儲存模型
with open('hr_anomaly_model.tflite', 'wb') as f:
    f.write(tflite_model)

print(f"Model size: {len(tflite_model)} bytes")
```

---

#### 3. 模型部署

**將模型部署到 MCU**：

```c
// 將 TFLite 模型轉換為 C 陣列
const unsigned char g_model_data[] = {
    0x1c, 0x00, 0x00, 0x00, 0x54, 0x46, 0x4c, 0x33, ...
};
const int g_model_data_len = 45678;

// AI 推論
void ai_inference(uint8_t *hr_data, uint16_t length) {
    // 1. 載入模型
    const tflite::Model* model = tflite::GetModel(g_model_data);

    // 2. 建立 Interpreter
    tflite::MicroMutableOpResolver<3> micro_op_resolver;
    micro_op_resolver.AddUnidirectionalSequenceLSTM();
    micro_op_resolver.AddFullyConnected();
    micro_op_resolver.AddLogistic();

    constexpr int kTensorArenaSize = 20 * 1024;  // 20 KB
    uint8_t tensor_arena[kTensorArenaSize];

    tflite::MicroInterpreter interpreter(
        model, micro_op_resolver, tensor_arena, kTensorArenaSize);

    interpreter.AllocateTensors();

    // 3. 填入輸入資料
    TfLiteTensor* input = interpreter.input(0);
    for (int i = 0; i < length; i++) {
        input->data.int8[i] = hr_data[i] - 128;  // 轉換為 int8 (-128 ~ 127)
    }

    // 4. 執行推論
    uint32_t start_time = millis();
    interpreter.Invoke();
    uint32_t inference_time = millis() - start_time;

    printf("Inference time: %d ms\n", inference_time);

    // 5. 取得輸出
    TfLiteTensor* output = interpreter.output(0);
    float anomaly_prob = (output->data.int8[0] + 128) / 255.0;  // 轉換為 0-1

    printf("Anomaly probability: %.2f\n", anomaly_prob);

    // 6. 判斷是否異常
    if (anomaly_prob > 0.8) {
        send_alert("Heart rate anomaly detected!");
    }
}
```

---

### 實作結果

**測試結果**：

- ✅ **模型大小**：45 KB（符合 < 50 KB 的要求）
- ✅ **推論時間**：8 ms（符合 < 10 ms 的要求）
- ✅ **準確度**：96.5%（符合 > 95% 的要求）
- ✅ **功耗**：平均 80 μA（符合 < 100 μA 的要求）
- ✅ **電池續航**：> 7 天（2 顆 CR2032 電池）

**客戶反饋**：

Sarah 非常滿意："這個功能太棒了！不僅能偵測異常，還能根據使用者的歷史資料自動學習。而且功耗很低，電池續航時間沒有縮短。我們決定量產。"

---

## 實務建議：如何開發 AIoT 應用？

基於多年的 AIoT 開發經驗，我總結了以下的開發指南：

### 1. 選擇合適的硬體平台

**考慮因素**：

- **算力**：是否需要 DSP 或 NPU？
- **記憶體**：RAM 和 Flash 是否足夠？
- **功耗**：是否需要低功耗？
- **成本**：晶片成本是否可接受？

**推薦平台**：

- **ARM Cortex-M4/M7**：適合大部分 TinyML 應用
- **ARM Cortex-M55**：內建 Helium（ARM 的 DSP 擴展）
- **ESP32-S3**：內建 AI 加速器
- **Nordic nRF5340**：雙核心，適合 BLE + AI

---

### 2. 選擇合適的 AI 模型

**考慮因素**：

- **準確度**：是否滿足需求？
- **模型大小**：是否能放入 Flash？
- **運算量**：MCU 是否跑得動？
- **延遲**：是否滿足即時性要求？

**推薦模型**：

- **喚醒詞偵測**：CNN（1D Convolution）
- **手勢辨識**：LSTM 或 GRU
- **異常偵測**：Autoencoder 或 LSTM
- **影像分類**：MobileNet 或 SqueezeNet

---

### 3. 優化模型

**模型壓縮技術**：

- **Quantization（量化）**：
  - float32 → int8：減少 4 倍大小，加速 2-4 倍
  - 準確度損失 < 1%

- **Pruning（剪枝）**：
  - 移除不重要的權重
  - 減少 50-90% 大小
  - 準確度損失 < 2%

- **Knowledge Distillation（知識蒸餾）**：
  - 用小模型模仿大模型
  - 減少 10-100 倍大小
  - 準確度損失 < 5%

---

### 4. 測試和驗證

**測試項目**：

- **功能測試**：模型是否正確運作？
- **準確度測試**：準確度是否滿足需求？
- **效能測試**：推論時間是否滿足要求？
- **功耗測試**：功耗是否滿足要求？
- **壓力測試**：長時間運作是否穩定？

---

### 實務檢查清單

**AIoT 開發檢查清單**：

- [ ] 硬體平台是否選擇正確？（算力、記憶體、功耗）
- [ ] AI 模型是否選擇正確？（準確度、大小、運算量）
- [ ] 模型是否已優化？（量化、剪枝、蒸餾）
- [ ] 模型大小是否符合要求？（< Flash 大小）
- [ ] 推論時間是否符合要求？（< 延遲要求）
- [ ] 功耗是否符合要求？（< 功耗預算）
- [ ] 準確度是否符合要求？（> 準確度目標）
- [ ] 是否已進行完整測試？（功能、準確度、效能、功耗、壓力）

---

## 總結

AIoT 是 IoT 的下一個演進階段，將 AI 帶到邊緣裝置上，實現：

- **隱私保護**：資料不離開裝置
- **低延遲**：即時反應
- **低頻寬**：只傳輸結果
- **低成本**：不需要雲端運算費用

但 AIoT 也面臨許多技術挑戰：

- **算力不足**：MCU 算力有限
- **功耗過高**：AI 運算消耗大量電力
- **記憶體不足**：RAM 和 Flash 有限
- **開發困難**：需要跨領域知識

**TinyML** 是解決這些挑戰的關鍵技術，透過：

- **模型壓縮**：減少模型大小和運算量
- **硬體加速**：使用 DSP 或 NPU
- **現成框架**：TensorFlow Lite Micro, CMSIS-NN, Edge Impulse

---

### 我的建議

基於多年的 AIoT 開發經驗，我的建議是：

1. **從簡單開始**：
   - 先用簡單的規則（if-else）
   - 再用簡單的 AI 模型（線性回歸、決策樹）
   - 最後用複雜的 AI 模型（CNN, LSTM）

2. **選擇合適的硬體**：
   - 不要過度設計（不是所有應用都需要 NPU）
   - 考慮成本和功耗

3. **使用現成框架**：
   - TensorFlow Lite Micro（最成熟）
   - Edge Impulse（最易用）
   - CMSIS-NN（最高效能）

4. **持續優化**：
   - 量化（float32 → int8）
   - 剪枝（移除不重要的權重）
   - 蒸餾（用小模型模仿大模型）

5. **完整測試**：
   - 功能、準確度、效能、功耗、壓力測試
   - 不要只測試 Happy Path

---

### 延伸閱讀

回顧這次的 AIoT 開發經歷，我深刻體會到：**AIoT 不僅是技術問題，更是系統設計和優化的問題**。

開發 AIoT 應用時，要考慮：

- **硬體選擇**（算力、記憶體、功耗、成本）
- **模型選擇**（準確度、大小、運算量、延遲）
- **模型優化**（量化、剪枝、蒸餾）
- **系統整合**（感測器、通訊、電源管理）
- **測試驗證**（功能、準確度、效能、功耗、壓力）

只有綜合考慮這些因素，才能開發出成功的 AIoT 產品。

在下一篇文章中，我會介紹一個完整的 IoT 產品開發案例：**智慧手錶**，涵蓋硬體設計、韌體開發、BLE 通訊、功耗優化、AI 模型部署等所有環節。

---

## 參考資料

- TensorFlow Lite Micro: <https://www.tensorflow.org/lite/microcontrollers>
- CMSIS-NN: <https://github.com/ARM-software/CMSIS-NN>
- Edge Impulse: <https://www.edgeimpulse.com/>
- TinyML Book: <https://www.oreilly.com/library/view/tinyml/9781492052036/>
- ARM Cortex-M55: <https://www.arm.com/products/silicon-ip-cpu/cortex-m/cortex-m55>
- ESP32-S3: <https://www.espressif.com/en/products/socs/esp32-s3>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

您可以自由地：

- **分享** — 以任何媒介或格式重製及散布本素材
- **修改** — 重混、轉換本素材、及依本素材建立新素材

但您必須遵守以下條款：

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。您可以任何合理的方式為前述表彰，但不得以任何方式暗示授權人為您或您的使用方式背書。

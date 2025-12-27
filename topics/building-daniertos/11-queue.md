# Queue 實作

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：生產者與消費者的經典問題

你有沒有在餐廳的廚房後面看過那個「出菜口」？

廚師做好一道菜，放在出菜口的架子上，然後繼續做下一道。服務生從另一邊取走菜，端給客人。廚師和服務生不需要面對面交接——架子就是他們之間的緩衝區。

如果沒有這個架子呢？廚師做好菜後，必須站在那邊等服務生來拿。如果服務生正在忙，廚師就只能乾等。整個廚房的效率會大幅下降。

這就是電腦科學中經典的 **Producer-Consumer Problem**（生產者-消費者問題）。

在 RTOS 中，我們經常遇到類似的場景：

- **Sensor Task**：不斷讀取感測器資料（Producer）
- **Process Task**：處理資料並做出決策（Consumer）

如果 Sensor Task 必須等 Process Task 處理完才能繼續讀取，那讀取頻率就會被拖慢。

**Queue（佇列）** 就是 RTOS 版本的「出菜口架子」。它是一個 FIFO 緩衝區，可以暫存多筆資料，讓 Producer 和 Consumer 以不同的速度運作，同時內建同步機制。

> 💡 **設計哲學**：Queue 是 RTOS 中最常用的 IPC（Inter-Process Communication）機制，因為它同時解決了「同步」和「資料傳遞」兩個問題。

---

## 一、Ring Buffer 設計

### 1.1 為什麼用 Ring Buffer？

Queue 需要一個緩衝區來存放資料。最簡單的實作是 **Ring Buffer（環形緩衝區）**：

- 固定大小的陣列
- 兩個指標：`head`（讀取位置）和 `tail`（寫入位置）
- 當指標到達陣列尾端時，繞回開頭

```
初始狀態：
┌───┬───┬───┬───┬───┐
│   │   │   │   │   │
└───┴───┴───┴───┴───┘
  ↑
head = tail = 0（空）

寫入 A, B, C：
┌───┬───┬───┬───┬───┐
│ A │ B │ C │   │   │
└───┴───┴───┴───┴───┘
  ↑           ↑
head=0      tail=3

讀取 A, B：
┌───┬───┬───┬───┬───┐
│   │   │ C │   │   │
└───┴───┴───┴───┴───┘
          ↑   ↑
        head=2 tail=3

寫入 D, E, F（繞回）：
┌───┬───┬───┬───┬───┐
│ F │   │ C │ D │ E │
└───┴───┴───┴───┴───┘
      ↑   ↑
   tail=1 head=2
```

### 1.2 資料結構

```c
typedef struct {
    uint8_t *buffer;          // 緩衝區
    size_t item_size;         // 每個項目的大小
    size_t capacity;          // 最多可存放幾個項目
    volatile size_t count;    // 目前有幾個項目
    size_t head;              // 讀取位置
    size_t tail;              // 寫入位置

    // 同步機制
    tcb_t *send_waiting_list;     // 等待送出的 Task
    tcb_t *receive_waiting_list;  // 等待接收的 Task
} queue_t;
```

### 1.3 初始化

```c
bool queue_init(queue_t *q, void *buffer, size_t item_size, size_t capacity) {
    q->buffer = (uint8_t *)buffer;
    q->item_size = item_size;
    q->capacity = capacity;
    q->count = 0;
    q->head = 0;
    q->tail = 0;
    q->send_waiting_list = NULL;
    q->receive_waiting_list = NULL;
    return true;
}

// 使用靜態緩衝區
#define QUEUE_SIZE 10
typedef struct { int x, y; } point_t;

point_t point_buffer[QUEUE_SIZE];
queue_t point_queue;

void init(void) {
    queue_init(&point_queue, point_buffer, sizeof(point_t), QUEUE_SIZE);
}
```

---

## 二、基本操作

### 2.1 Send（送入資料）

```c
static void buffer_write(queue_t *q, const void *item) {
    uint8_t *dst = q->buffer + (q->tail * q->item_size);
    memcpy(dst, item, q->item_size);
    q->tail = (q->tail + 1) % q->capacity;
    q->count++;
}

bool queue_send(queue_t *q, const void *item, uint32_t timeout_ticks) {
    critical_enter();

    // 1. 如果有 Task 在等待接收，直接傳給它
    if (q->receive_waiting_list != NULL) {
        tcb_t *receiver = waiting_list_remove_first(&q->receive_waiting_list);

        // 直接複製到接收者的緩衝區
        memcpy(receiver->wait_buffer, item, q->item_size);

        // 喚醒接收者
        wake_task(receiver, WAKE_REASON_SIGNALED);

        critical_exit();
        return true;
    }

    // 2. 如果 Queue 有空間，寫入
    if (q->count < q->capacity) {
        buffer_write(q, item);
        critical_exit();
        return true;
    }

    // 3. Queue 滿了，需要等待
    if (timeout_ticks == 0) {
        critical_exit();
        return false;
    }

    // 4. Block 等待空間
    current_tcb->wait_buffer = (void *)item;  // 暫存要送的資料
    block_current_task(&q->send_waiting_list, timeout_ticks);

    critical_exit();

    return (current_tcb->wake_reason == WAKE_REASON_SIGNALED);
}
```

### 2.2 Receive（取出資料）

```c
static void buffer_read(queue_t *q, void *item) {
    uint8_t *src = q->buffer + (q->head * q->item_size);
    memcpy(item, src, q->item_size);
    q->head = (q->head + 1) % q->capacity;
    q->count--;
}

bool queue_receive(queue_t *q, void *item, uint32_t timeout_ticks) {
    critical_enter();

    // 1. 如果 Queue 有資料，讀取
    if (q->count > 0) {
        buffer_read(q, item);

        // 如果有 Task 在等待送出，讓它送
        if (q->send_waiting_list != NULL) {
            tcb_t *sender = waiting_list_remove_first(&q->send_waiting_list);

            // 把 sender 的資料寫入 Queue
            buffer_write(q, sender->wait_buffer);

            // 喚醒 sender
            wake_task(sender, WAKE_REASON_SIGNALED);
        }

        critical_exit();
        return true;
    }

    // 2. Queue 是空的，需要等待
    if (timeout_ticks == 0) {
        critical_exit();
        return false;
    }

    // 3. Block 等待資料
    current_tcb->wait_buffer = item;  // 接收到的資料會放這裡
    block_current_task(&q->receive_waiting_list, timeout_ticks);

    critical_exit();

    return (current_tcb->wake_reason == WAKE_REASON_SIGNALED);
}
```

---

## 三、ISR 安全版本

### 3.1 為什麼需要 ISR 版本？

ISR 中不能 Block。如果 Queue 滿了/空了，必須立即返回失敗。

### 3.2 Send from ISR

```c
bool queue_send_from_isr(queue_t *q, const void *item, bool *need_switch) {
    *need_switch = false;

    // 如果有 Task 在等待接收
    if (q->receive_waiting_list != NULL) {
        tcb_t *receiver = waiting_list_remove_first(&q->receive_waiting_list);
        memcpy(receiver->wait_buffer, item, q->item_size);

        receiver->wake_reason = WAKE_REASON_SIGNALED;
        receiver->state = TASK_READY;
        ready_list_add(receiver);

        if (receiver->priority > current_tcb->priority) {
            *need_switch = true;
        }

        return true;
    }

    // 如果有空間
    if (q->count < q->capacity) {
        buffer_write(q, item);
        return true;
    }

    // Queue 滿了
    return false;
}
```

### 3.3 Receive from ISR

```c
bool queue_receive_from_isr(queue_t *q, void *item, bool *need_switch) {
    *need_switch = false;

    // 如果有資料
    if (q->count > 0) {
        buffer_read(q, item);

        // 如果有 Task 在等待送出
        if (q->send_waiting_list != NULL) {
            tcb_t *sender = waiting_list_remove_first(&q->send_waiting_list);
            buffer_write(q, sender->wait_buffer);

            sender->wake_reason = WAKE_REASON_SIGNALED;
            sender->state = TASK_READY;
            ready_list_add(sender);

            if (sender->priority > current_tcb->priority) {
                *need_switch = true;
            }
        }

        return true;
    }

    // Queue 是空的
    return false;
}
```

---

## 四、使用範例

### 4.1 生產者-消費者

```c
queue_t sensor_queue;
int sensor_buffer[16];

void sensor_task(void) {
    queue_init(&sensor_queue, sensor_buffer, sizeof(int), 16);

    while (1) {
        int reading = read_sensor();

        if (!queue_send(&sensor_queue, &reading, 100)) {
            // 超時：Queue 滿了太久
            uart_puts("Queue full, dropping data\n");
        }

        danie_delay(10);
    }
}

void process_task(void) {
    while (1) {
        int data;

        if (queue_receive(&sensor_queue, &data, WAIT_FOREVER)) {
            process_sensor_data(data);
        }
    }
}
```

### 4.2 ISR 到 Task 的通訊

```c
queue_t uart_rx_queue;
char rx_buffer[64];

void uart_isr(void) {
    char c = UART->RX_DATA;
    bool need_switch;

    queue_send_from_isr(&uart_rx_queue, &c, &need_switch);

    if (need_switch) {
        trigger_pendsv();  // 觸發 Context Switch
    }
}

void uart_handler_task(void) {
    while (1) {
        char c;
        if (queue_receive(&uart_rx_queue, &c, WAIT_FOREVER)) {
            handle_uart_char(c);
        }
    }
}
```

### 4.3 傳遞結構體

```c
typedef struct {
    uint32_t timestamp;
    int16_t x, y, z;
} accel_data_t;

queue_t accel_queue;
accel_data_t accel_buffer[8];

void accel_task(void) {
    queue_init(&accel_queue, accel_buffer, sizeof(accel_data_t), 8);

    while (1) {
        accel_data_t data = {
            .timestamp = get_tick_count(),
            .x = read_accel_x(),
            .y = read_accel_y(),
            .z = read_accel_z()
        };

        queue_send(&accel_queue, &data, WAIT_FOREVER);
        danie_delay(10);
    }
}
```

---

## 五、進階：Peek 和 Reset

### 5.1 Peek（看但不取）

```c
bool queue_peek(queue_t *q, void *item, uint32_t timeout_ticks) {
    critical_enter();

    if (q->count > 0) {
        // 複製但不移動 head
        uint8_t *src = q->buffer + (q->head * q->item_size);
        memcpy(item, src, q->item_size);
        critical_exit();
        return true;
    }

    // 如果要等待，邏輯和 receive 類似...
    // （省略）

    critical_exit();
    return false;
}
```

### 5.2 Reset（清空 Queue）

```c
void queue_reset(queue_t *q) {
    critical_enter();

    q->count = 0;
    q->head = 0;
    q->tail = 0;

    // 喚醒所有等待送出的 Task（它們會收到失敗）
    while (q->send_waiting_list != NULL) {
        tcb_t *task = waiting_list_remove_first(&q->send_waiting_list);
        wake_task(task, WAKE_REASON_TIMEOUT);  // 或定義一個 WAKE_REASON_RESET
    }

    critical_exit();
}
```

---

## 總結

本文實作了 danieRTOS 的 Queue 機制：

1. **Ring Buffer**：固定大小的環形緩衝區
2. **Send/Receive**：支援 Block 和 Timeout
3. **ISR 安全**：FromISR 版本不會 Block
4. **直接傳遞**：當有 Task 等待時，繞過 Buffer 直接傳
5. **結構體支援**：可以傳遞任意大小的資料

Queue 是 RTOS 中最常用的 IPC 機制。它結合了 Semaphore 的同步能力和緩衝區的資料儲存能力。

---

## 參考資料

**經典教科書**

- **The Art of Computer Programming, Volume 1: Fundamental Algorithms**
  Knuth, D. E.
  Section 2.2: Linked Allocation，Queue 和 Circular Buffer 的經典講解。

- **Operating Systems: Three Easy Pieces**
  Arpaci-Dusseau, R. H. and Arpaci-Dusseau, A. C.
  https://pages.cs.wisc.edu/~remzi/OSTEP/
  Chapter 30: Condition Variables，Producer-Consumer 問題的深入討論。

**RTOS 參考實作**

- **FreeRTOS Kernel - queue.c**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  Queue 的完整實作，包含 xQueueSend() 和 xQueueReceive()。

**延伸閱讀**

- **Data Structures in Practice**
  Danny Jiang
  Ring Buffer 的實作細節。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

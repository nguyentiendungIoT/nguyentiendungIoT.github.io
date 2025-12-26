---
title: Bài 6 RTOS: Queue, Mutex, Semaphore (Đường ống vạn năng)
date: 2025-12-26 10:00:00 +0700
categories: [FreeRTOS, RTOS]
tags: [freertos, queue, semaphore, mutex, synchronization, kernel]
---

Bài này **tập trung vào phân tích mã nguồn kernel** để hiểu **cấu tạo và cơ chế hoạt động** của các primitive đồng bộ trong FreeRTOS: **queue, semaphore, mutex**.

- Mục tiêu là trả lời câu hỏi: *nó hoạt động như thế nào ở tầng kernel?* (block/unblock, danh sách chờ, ownership, priority inheritance…)
- Bài này **không** đi theo hướng “học cách sử dụng API” (how-to dùng `xQueueSend`, `xSemaphoreTake`, …).

Mã nguồn tham chiếu: https://github.com/FreeRTOS/FreeRTOS-Kernel.git (trọng tâm là `queue.c`).

Mình từng nghĩ FreeRTOS có “nhiều loại đồ chơi đồng bộ”: queue, semaphore, mutex… mỗi thứ một kiểu. Nhưng khi mình đọc vào mã nguồn kernel thì thấy một sự thật khá thú vị:

- **Queue mới là “lõi” (primitive) vạn năng**.
- **Semaphore và Mutex thực chất đều chạy trên cùng một lõi Queue**.

Bài này mình viết theo hướng **first-principles**: bắt đầu từ “đồng bộ rốt cuộc là gì”, rồi đi vào `Queue_t`, và cuối cùng bám sát đường đi trong `xQueueGenericSend()` để thấy vì sao mọi thứ quy về queue.

---

## 1) First principles: Đồng bộ (synchronization) rốt cuộc là gì?

Trong RTOS, dù tên gọi có hoành tráng đến đâu, đa phần cơ chế đồng bộ đều quay về 2 nhu cầu nền tảng:

1. **Truyền trạng thái** giữa các task
   - Có/không có dữ liệu
   - Có/không có “token”
   - Ai đang sở hữu tài nguyên

2. **Điều phối thời gian chạy**
   - Task nào phải chờ (block)
   - Khi nào được đánh thức (unblock)
   - Ai được chạy trước (phụ thuộc scheduler + priority)

Nếu trong kernel có một cấu trúc mà:

- giữ được **trạng thái**,
- cho phép task A **thay đổi trạng thái**,
- và có cơ chế **đánh thức** task B “đúng lúc”,

thì cấu trúc đó có thể đóng vai **message queue**, **semaphore**, **mutex**… và thậm chí nhiều primitive khác.

FreeRTOS chọn cấu trúc “lõi” đó chính là: **Queue**.

---

## 2) Nhìn từ gốc: Một `Queue_t` chứa những gì?

Trong `queue.c`, kiểu chính là `Queue_t` (typedef từ `xQUEUE`). Mình tóm lại những phần “đinh” cần nhìn:

### 2.1) Ring buffer (khi dùng như queue dữ liệu)

- `pcHead`: đầu vùng storage
- `u.xQueue.pcTail`: mốc cuối storage (marker)
- `pcWriteTo`: vị trí sẽ write item tiếp theo
- `u.xQueue.pcReadFrom`: vị trí “đã đọc lần trước” (lần đọc kế tiếp sẽ dịch con trỏ rồi copy)

Đây là **circular buffer**: chạm `pcTail` thì **wrap** về `pcHead`.

### 2.2) Hai danh sách task đang chờ (đây mới là phần “đồng bộ”)

- `xTasksWaitingToSend`: task đang block vì queue full
- `xTasksWaitingToReceive`: task đang block vì queue empty

Đây là “đường ống” đúng nghĩa: khi trạng thái queue đổi (có chỗ trống / có data), kernel sẽ unblock task phù hợp.

### 2.3) Trạng thái số lượng

- `uxMessagesWaiting`: số item hiện có trong queue
- `uxLength`: sức chứa (tính theo số item)
- `uxItemSize`: kích thước mỗi item

### 2.4) Copy-by-value (một câu mà rất dễ dính bẫy)

Trong comment của struct, FreeRTOS nói thẳng:

- **Items are queued by copy, not reference**.

Nghĩa là queue sẽ **copy dữ liệu vào storage của kernel** (thường là `memcpy()`), chứ không giữ “tham chiếu” tới dữ liệu bên ngoài.

Theo tài liệu @tailieu, đây là cơ chế “queue by copy”: dữ liệu được sao chép từng byte vào hàng đợi, và đọc ra theo thứ tự FIFO.

### 2.5) “Bí mật”: Semaphore/Mutex mượn chung struct bằng union

`Queue_t` có union `u`:

- `u.xQueue` dùng khi là queue data
- `u.xSemaphore` dùng khi là semaphore/mutex

Hai mode loại trừ nhau nên dùng union để tiết kiệm RAM. Và đây là nền tảng để FreeRTOS “gói” nhiều primitive lên cùng một lõi.

---

## 3) ELI5: Queue = băng chuyền vòng tròn + hai hàng người chờ

Mình hay hình dung thế này:

- Ring buffer là **băng chuyền vòng tròn** chở “món hàng”
- `uxMessagesWaiting` là số món đang nằm trên băng chuyền
- `pcWriteTo` là điểm đặt món mới lên băng chuyền
- `pcReadFrom` là điểm nhặt món ra

Và quan trọng nhất:

- Nếu băng chuyền **đầy** → người gửi phải đứng xếp hàng: `xTasksWaitingToSend`
- Nếu băng chuyền **trống** → người nhận phải đứng xếp hàng: `xTasksWaitingToReceive`

Cái làm nên “đồng bộ” không phải `memcpy()`.
Cái làm nên “đồng bộ” chính là: **ai bị block và ai được unblock khi trạng thái thay đổi**.

---

## 4) `xQueueGenericSend()` — “cửa vào” vạn năng

Hàm mình thấy đáng đọc nhất trong `queue.c` là:

```c
BaseType_t xQueueGenericSend(
    QueueHandle_t xQueue,
    const void *pvItemToQueue,
    TickType_t xTicksToWait,
    BaseType_t xCopyPosition
);
```

Nếu nhìn theo first principles, “send vào queue” phải đảm bảo:

1) **Đúng** (không race condition)
2) **Đúng thứ tự/đúng chính sách đánh thức** (task nào chờ thì được xử lý đúng)
3) **Có thể block** khi queue full (nếu user cho phép chờ)

### 4.1) Fast path: làm nhanh trong critical section

Trong vòng `for(;;)` của `xQueueGenericSend()` thường có nhánh “đường nhanh”:

- Vào `taskENTER_CRITICAL()`
- Nếu queue **còn chỗ** (hoặc case overwrite) thì:
  - `prvCopyDataToQueue()` để “đặt item” (hoặc tăng count nếu là semaphore/mutex)
  - Nếu có task đang chờ receive thì `xTaskRemoveFromEventList()` để unblock
  - Thoát critical và trả về `pdPASS`

Ý nghĩa: nếu điều kiện đủ đơn giản, kernel làm **atomic** và **nhanh**.

### 4.2) Slow path: phải block thì chuyển sang cơ chế điều phối

Nếu queue full mà `xTicksToWait > 0`:

- Thoát critical
- `vTaskSuspendAll()`
- khóa queue nội bộ (`prvLockQueue(pxQueue)`)
- nếu vẫn full: đưa task hiện tại vào `xTasksWaitingToSend` rồi nhường scheduler

Ý nghĩa: khi đã “phải chờ”, thì không còn là chuyện copy data nữa; nó trở thành chuyện **quản lý danh sách chờ + scheduler**.

### 4.3) Ring buffer thật sự nằm ở đâu?

Phần “băng chuyền vòng tròn” thường nằm trong `prvCopyDataToQueue()`:

- Send-to-back: copy vào `pcWriteTo`, tăng con trỏ, chạm `pcTail` thì wrap về `pcHead`
- Send-to-front: dịch `pcReadFrom` ngược lại, wrap nếu cần

Đọc đoạn này thường sẽ “đã mắt” vì nó đúng kiểu circular buffer.

### 4.4) Mô phỏng: Queue (interactive)

<div style="width: 100%; margin: 20px 0;">
  <iframe
    src="/assets/sims/freertos_queue_visualizer.html"
    width="100%"
    height="1600px"
    style="border: 1px solid #ccc; border-radius: 8px;"
    title="Mô phỏng FreeRTOS Queue"
    scrolling="yes"
  >
  </iframe>
</div>

---

## 5) Semaphore: Queue không chở đồ, chỉ chở trạng thái

Điểm cốt lõi (và rất đẹp) là:

- Semaphore **không cần payload**.
- Nó chỉ cần biểu diễn “có token / không có token” (binary) hoặc “đếm token” (counting).

Trong FreeRTOS, semaphore thường tương ứng với queue có:

- `uxItemSize == 0`

Khi `uxItemSize == 0`:

- Không có `memcpy()` payload
- `uxMessagesWaiting` trở thành **số token hiện có**

Theo @tailieu:

- FreeRTOS dùng semaphore để đồng bộ: một nơi *give*, nơi khác *take*.
- Binary semaphore được tạo bằng `xSemaphoreCreateBinary()` thường bắt đầu ở trạng thái **empty** (phải give trước rồi mới take được).

Nói cách khác, “take semaphore” nhìn từ kernel giống như: **receive một queue mà item size = 0**.

### 5.1) Counting semaphore

Counting semaphore có thể hiểu cực gọn:

- max count = queue length
- current count = `uxMessagesWaiting`

### 5.2) Mô phỏng: Semaphore (interactive)

<div style="width: 100%; margin: 20px 0;">
  <iframe
    src="/assets/sims/freertos_semaphore_debugger%20-%20Copy.html"
    width="100%"
    height="1600px"
    style="border: 1px solid #ccc; border-radius: 8px;"
    title="Mô phỏng FreeRTOS Semaphore"
    scrolling="yes"
  >
  </iframe>
</div>

---

## 6) Mutex: vẫn là queue, nhưng thêm ownership + priority inheritance

Mutex khác semaphore ở 2 điểm:

1) **Ownership**: mutex có “ai đang giữ”
2) **Priority inversion**: cần cơ chế priority inheritance/disinherit

Trong FreeRTOS, mutex thường được tạo giống “queue length = 1, item size = 0” (tức là về mặt storage, nó cũng không chở payload).

### 6.1) Ownership nằm ở đâu?

Mutex holder nằm trong union (vùng semaphore/mutex) của `Queue_t`:

- `u.xSemaphore.xMutexHolder`

Khi task take mutex thành công, kernel sẽ ghi lại holder. Nhờ đó mutex có thể phân biệt “ai là chủ”.

### 6.2) Priority inheritance xảy ra lúc nào?

Bài toán priority inversion (mình tóm tắt theo first principles):

- Task Low giữ mutex
- Task High cần mutex → bị block
- Task Medium (không liên quan mutex) cứ chạy → Low không có CPU để nhả mutex
- High bị kẹt gián tiếp bởi Medium

Priority inheritance giải bằng cách:

- Khi High chờ mutex, Low tạm thời được **nâng priority** để có CPU chạy, nhả mutex sớm.

Theo @tailieu, mutex trong FreeRTOS có cơ chế **priority inheritance**, và khi mutex được trả lại thì sẽ **disinherit** (trả priority về phù hợp).

### 6.3) Disinherit xảy ra khi give mutex

Khi mutex được give, nó vẫn “đi qua” đường send/return token (tùy implementation), và tại thời điểm nhả, kernel sẽ:

- disinherit priority cho holder
- reset `xMutexHolder`

### 6.4) Mô phỏng: Mutex (interactive)

<div style="width: 100%; margin: 20px 0;">
  <iframe
    src="/assets/sims/freertos-mutex-visualizer.html"
    width="100%"
    height="1600px"
    style="border: 1px solid #ccc; border-radius: 8px;"
    title="Mô phỏng FreeRTOS Mutex"
    scrolling="yes"
  >
  </iframe>
</div>

---

## 7) Copy-by-value: hiểu đúng để khỏi dính bẫy

Đây là chỗ mình thấy nhiều người (kể cả mình trước đây) hay hiểu lệch.

### 7.1) Gửi số nguyên: queue copy giá trị

```c
int a = 10;
xQueueSend(q, &a, ...);

a = 20; // đổi bên ngoài
```

Do queue **copy-by-value**, item nằm trong queue vẫn là **10**.

### 7.2) Dữ liệu lớn: gửi pointer (queue copy địa chỉ, không copy buffer)

Nếu payload lớn (frame camera, buffer UART…), mình thường không gửi nguyên mảng.
Mình gửi **con trỏ**:

```c
uint8_t *p = frame;
xQueueSend(q, &p, ...);
```

Lưu ý:

- Queue vẫn copy-by-value, nhưng thứ nó copy là **giá trị con trỏ** (địa chỉ)
- Receiver nhận được địa chỉ đó và tự đọc buffer

Điều này kéo theo trách nhiệm thiết kế:

- buffer phải còn sống tới khi receiver xử lý xong
- không bị ghi đè giữa chừng
- thường cần pool/refcount/double-buffer… tùy bài

---

## 8) Kết luận: “Tất cả bọn chúng đều là Queue!”

Nếu mình gom lại ngắn gọn:

- **Queue data**: ring buffer + copy payload
- **Semaphore**: `uxItemSize == 0`, “data” chính là `uxMessagesWaiting` (token count)
- **Mutex**: cũng `uxItemSize == 0`, thường length = 1, nhưng có thêm:
  - `xMutexHolder`
  - priority inheritance khi task ưu tiên cao bị block
  - priority disinherit khi mutex được trả

Vậy nên, nhìn đúng ở tầng kernel, queue thật sự là một “đường ống” vừa chở dữ liệu, vừa chở trạng thái, vừa điều phối việc block/unblock.

---

## 9) Gợi ý tự soi bằng debugger (mình hay làm vậy)

Nếu bạn có source FreeRTOS kernel trong project, mình gợi ý đặt breakpoint ở:

- `xQueueGenericSend()` (để thấy fast path vs block path)
- `prvCopyDataToQueue()` (để thấy wrap con trỏ + `memcpy()`)
- `xQueueSemaphoreTake()` (để thấy take semaphore/mutex và logic ưu tiên nếu là mutex)

Watch các field:

- `uxMessagesWaiting`, `uxLength`, `uxItemSize`
- `pcWriteTo`, `u.xQueue.pcReadFrom`, `u.xQueue.pcTail`
- `u.xSemaphore.xMutexHolder`
- danh sách chờ: `xTasksWaitingToSend`, `xTasksWaitingToReceive`

Nếu cần, mình có thể viết thêm một bài follow-up: “đọc trace của queue step-by-step” dựa trên một ví dụ 2 task + 1 ISR cho dễ hình dung.

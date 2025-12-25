---
title: Bài viết đầu tiên - Hello Embedded World
date: 2025-12-24 18:00:00 +0700
categories: [General, Life]
tags: [hello, embedded]
---

## Chào thế giới

Đây là bài viết test blog kỹ thuật của mình. Blog này được xây dựng dựa trên tư duy **Docs-as-Code**.

### Ví dụ về Code C
Mình là dân Embedded :

```c
#include <stdio.h>

int main() {
    printf("Hello World from Github Pages!");
    return 0;
}
```

## Mô phỏng 8051 Interactiv

Dưới đây là mô phỏng tương tác của 2 task khi truy cập queue, bạn có thể test code trực tiếp:

<div style="width: 100vw; margin: 20px calc(-50vw + 50%); position: relative; left: 50%; right: 50%; margin-left: -50vw; margin-right: -50vw;">
    <iframe
        src="/assets/sims/mo_phong_stm32_queue.html"
        width="100%"
        height="1000px"
        style="border: 1px solid #ccc; border-radius: 8px;"
        title="Mô phỏng QUEUE RTOS"
        scrolling="no"
    >
    </iframe>
</div>
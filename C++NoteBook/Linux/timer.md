# Linux 定时：相关系统调用说明

  
> 适用场景：用户态定时器队列 + `timerfd` + `epoll` 一类 Reactor 实现。  


---

## 0. 相关头文件与类型

### `struct timespec`

**声明**

```c
// s + ns 构成完整时间
struct timespec {
    time_t tv_sec;   /* 秒 */
    long   tv_nsec;  /* 纳秒，范围 [0, 999999999] */
};
```

**作用**

- 表示一个时间点或一段时间。
- 与 `clock_gettime` 配合读时钟；与 `itimerspec` 中的字段类型一致。

---

### `struct itimerspec`

**声明**

```c
struct itimerspec {
    struct timespec it_interval;  /* 周期性间隔 */
    struct timespec it_value;     /* 首次到期 / 一次性超时 */
};
```

**字段含义**


| 字段            | 作用                                                 |
| ------------- | -------------------------------------------------- |
| `it_value`    | 第一次触发还要等多久（相对或绝对，由 `timerfd_settime` 的 `flags` 决定） |
| `it_interval` | 第一次触发后，是否周期性重复；**全 0** 表示一次性定时器                    |


---

## 1. `clock_gettime`

### 声明

```c
#include <time.h>

int clock_gettime(clockid_t clockid, struct timespec *tp);
```

### 参数


| 参数        | 含义                                                |
| --------- | ------------------------------------------------- |
| `clockid` | 时钟源。定时器场景常用 `**CLOCK_MONOTONIC**`（单调递增，不受系统调时间影响） |
| `tp`      | 输出：当前时钟读数写入此结构体                                   |


**常用 `clockid`**


| 值                 | 作用                                    |
| ----------------- | ------------------------------------- |
| `CLOCK_MONOTONIC` | 自系统启动以来单调递增；**相对超时、idle、定时队列首选**      |
| `CLOCK_REALTIME`  | 墙钟时间，可被 NTP / 手动调整；**不适合**「再过 N 秒」类逻辑 |


### 返回值


| 值    | 含义              |
| ---- | --------------- |
| `0`  | 成功              |
| `-1` | 失败，错误码在 `errno` |


### 作用

- 在用户态获取 **当前时间**，用于：
  - 计算逻辑定时器的绝对到期时刻（`now + delay`）；
  - 到期扫表时取 `now`，判定 `expiration <= now`。
- **不创建 fd**，不参与 `epoll`；仅提供时间基准。

### 示例

```c
struct timespec ts;
if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0) {
    perror("clock_gettime");
}
// ts.tv_sec, ts.tv_nsec 即当前单调时间
```

---

## 2. `timerfd_create`

### 声明

```c
#include <sys/timerfd.h>

int timerfd_create(int clockid, int flags);
```

### 参数


| 参数        | 含义                                                      |
| --------- | ------------------------------------------------------- |
| `clockid` | 定时器使用的时钟，通常 `**CLOCK_MONOTONIC**`（与 `clock_gettime` 一致） |
| `flags`   | 创建选项，可按位或                                               |


**常用 `flags`**


| 值                            | 作用                                          |
| ---------------------------- | ------------------------------------------- |
| `TFD_NONBLOCK`               | 非阻塞：`read` 在未到期时返回 `-1` 且 `errno == EAGAIN` |
| `TFD_CLOEXEC`                | `exec` 系列调用时自动关闭 fd，避免泄漏                    |
| `TFD_NONBLOCK | TFD_CLOEXEC` | Reactor 场景常见组合                              |


### 返回值


| 值      | 含义                                                    |
| ------ | ----------------------------------------------------- |
| `>= 0` | 成功，返回 **timerfd 文件描述符**                               |
| `-1`   | 失败，`errno` 如 `EINVAL`（非法 clock/flags）、`EMFILE`（fd 用尽） |


### 作用

- 创建一个 **特殊的文件描述符**。
- 该 fd **不传输业务数据**；到期后变为 **可读**（`EPOLLIN`），供 `epoll_wait` 与 socket 一并等待。
- **一个 fd 可对应用户态多条逻辑定时器**：内核只负责「何时叫醒」；具体哪几条到期，由用户态有序表 + `expiration <= now` 扫表决定。
- 典型用法：**每个 I/O 线程 / 每个 EventLoop 一个 timerfd**。

---

## 3. `timerfd_settime`

### 声明

```c
#include <sys/timerfd.h>

int timerfd_settime(int fd, int flags,
                    const struct itimerspec *new_value,
                    struct itimerspec *old_value);
```

### 参数


| 参数          | 含义                                                    |
| ----------- | ----------------------------------------------------- |
| `fd`        | `timerfd_create` 返回的描述符                               |
| `flags`     | `0`：相对时间；`**TFD_TIMER_ABSTIME**`：按绝对单调时间解释 `it_value` |
| `new_value` | 新的定时参数（见 `itimerspec`）                                |
| `old_value` | 可选输出旧参数；不需要时传 `**NULL**`                              |


`**new_value` 常见填法**


| 意图          | `it_value`    | `it_interval` |
| ----------- | ------------- | ------------- |
| 一次性 N 秒后    | `{N 秒, 0 纳秒}` | 全 0           |
| 取消 / disarm | 全 0           | 全 0           |
| 尽快触发（扫表用）   | `{0, 1}` 纳秒级  | 全 0           |
| 周期性         | 首次 delay      | 重复间隔          |


### 返回值


| 值    | 含义                         |
| ---- | -------------------------- |
| `0`  | 成功                         |
| `-1` | 失败，如 `EINVAL`（非法 timespec） |


### 作用

- **设置或修改** timerfd 的下一次到期时间。
- 用户态维护多条逻辑定时器时：每次增删改后，用 **表中最早到期时间 − now** 换算成 `it_value`，再调用本函数 **重设内核闹钟**。
- `it_value` 全 0：**解除**定时（不再产生可读事件，直到再次 settime）。

---

## 4. `timerfd_gettime`

### 声明

```c
#include <sys/timerfd.h>

int timerfd_gettime(int fd, struct itimerspec *curr_value);
```

### 参数


| 参数           | 含义                |
| ------------ | ----------------- |
| `fd`         | timerfd           |
| `curr_value` | 输出：当前内核中尚未触发的定时参数 |


### 返回值


| 值    | 含义  |
| ---- | --- |
| `0`  | 成功  |
| `-1` | 失败  |


### 作用

- **查询** timerfd 当前 armed 状态（还剩多久、是否 periodic）。
- 生产逻辑 **不依赖** 此调用；多用于 **调试** 或自检 `settime` 是否生效。

---

## 5. `read`（针对 timerfd）

### 声明

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

### 参数（timerfd 专用约定）


| 参数      | 含义                             |
| ------- | ------------------------------ |
| `fd`    | timerfd                        |
| `buf`   | 指向 `**uint64_t**` 的缓冲区         |
| `count` | 必须 `**>= 8**`（sizeof uint64_t） |


### 返回值


| 值    | 含义                                                          |
| ---- | ----------------------------------------------------------- |
| `8`  | 成功读到一个到期通知                                                  |
| `-1` | 失败；非阻塞且未到期时 `**errno == EAGAIN**`；信号中断 `**errno == EINTR**` |


`**buf` 中 uint64_t 的含义**

- 表示 **自上次 read 以来，累计到期次数**（可能 `> 1`，例如 event loop 繁忙未及时 poll）。
- **读一次即消费** 当前可读状态；配合 epoll 时，通常读清后 EPOLLIN 才恢复正常沿。

### 作用

- timerfd 到期后必须 **read** 才能清掉内核计数；否则可能反复触发 `EPOLLIN`。
- **read 不告诉你是哪一条逻辑定时器到期**；只表示「该扫表了」。
- 用户态应用 `**clock_gettime` 取 now**，再对定时器表执行 `expiration <= now` 批量处理。

### 示例（while非阻塞读清）

```cpp
// 先清空 fd 的内容
    uint64_t buf = 0;
    //  同 eloopFd 一致，ET 模式，必须用 while 循环榨干至 EAGAIN
    while (true) {
        auto n = ::read(timeFd, &buf, sizeof(buf));
        if (n == sizeof(buf)) {
            break;  // 成功清除内核计数器，进入处理逻辑
        }
        if (n < 0) {
            // 此处和 eloopFd 不太一样，代表没有到时间，直接退出
            if (errno == EAGAIN || errno == EWOULDBLOCK) return;
            if (errno == EINTR) continue;  // 信号中断，继续
            errif(true, "eloopFd read error");
        }
    }

```

---

## 6. `close`（针对 timerfd）

### 声明

```c
#include <unistd.h>

int close(int fd);
```

### 参数


| 参数   | 含义           |
| ---- | ------------ |
| `fd` | 要关闭的 timerfd |


### 返回值


| 值    | 含义  |
| ---- | --- |
| `0`  | 成功  |
| `-1` | 失败  |


### 作用

- 释放 timerfd 资源。
- **建议顺序**：先从 `epoll` 移除该 fd（`EPOLL_CTL_DEL` 或等价封装）→ 再 `close`，避免 epoll 仍引用已关闭 fd。

---

## 7. `epoll_wait`（与 timerfd 配合时）

### 声明

```c
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```

### 参数


| 参数          | 含义                                                        |
| ----------- | --------------------------------------------------------- |
| `epfd`      | `epoll_create1` 返回的 epoll 实例                              |
| `events`    | 输出：就绪事件数组                                                 |
| `maxevents` | 数组容量                                                      |
| `timeout`   | 毫秒；`**-1**` 表示一直阻塞到有事件；使用 timerfd 时仍可设 `-1`（由 timerfd 唤醒） |


### 返回值


| 值     | 含义                                         |
| ----- | ------------------------------------------ |
| `> 0` | 就绪 fd 个数                                   |
| `0`   | 超时（`timeout != -1` 时）                      |
| `-1`  | 失败；`**errno == EINTR**` 表示被信号中断，通常应 **重试** |


### 作用

- 同时等待 **socket、timerfd、eventfd** 等 fd 的就绪事件。
- timerfd 到期 → 返回其 fd 的 `**EPOLLIN`** → 上层再 `read` + 扫用户态定时器表。

---

## 8. `epoll_ctl`（将 timerfd 加入 epoll）

### 声明

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

### 参数


| 参数      | 含义                              |
| ------- | ------------------------------- |
| `epfd`  | epoll 实例                        |
| `op`    | `EPOLL_CTL_ADD` / `MOD` / `DEL` |
| `fd`    | timerfd（或任意要监听的 fd）             |
| `event` | 关注的事件；timerfd 通常设 `**EPOLLIN**` |


### 返回值


| 值    | 含义  |
| ---- | --- |
| `0`  | 成功  |
| `-1` | 失败  |


### 作用

- 把 timerfd **注册** 到 epoll，使定时到期与网络 I/O **统一在一个 wait 循环** 中处理。

---

## 9. 调用关系速览

```text
clock_gettime(CLOCK_MONOTONIC)     → 用户态 now / expiration
timerfd_create                     → 得到一个 timerfd
epoll_ctl(ADD, timerfd, EPOLLIN)   → 纳入 epoll
timerfd_settime                    → 设「最早逻辑到期」对应的 delay
epoll_wait(-1)                     → 阻塞；到期或 I/O 就绪返回
read(timerfd, &uint64_t, 8)        → 消费到期通知
clock_gettime + 扫表               → 处理 expiration <= now 的回调
timerfd_settime                    → 重设下一个最早到期
close(timerfd)                     → 析构
```

---

## 10. 常见 errno（简要）


| 调用                    | errno                    | 含义 / 处理                        |
| --------------------- | ------------------------ | ------------------------------ |
| `read` (非阻塞 timerfd)  | `EAGAIN` / `EWOULDBLOCK` | 未到期，正常退出读循环                    |
| `read` / `epoll_wait` | `EINTR`                  | 信号中断，**重试**                    |
| `timerfd_settime`     | `EINVAL`                 | timespec 非法或 flags 与 clock 不匹配 |
| `timerfd_create`      | `EINVAL`                 | 不支持的 clockid 或 flags           |


---

## 11. 最小可编译验证（独立于任何项目）

```c
#include <sys/timerfd.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <stdint.h>
#include <stdio.h>
#include <errno.h>

int main(void) {
    int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK | TFD_CLOEXEC);
    if (tfd < 0) { perror("timerfd_create"); return 1; }

    struct itimerspec spec = {};
    spec.it_value.tv_sec = 2;
    if (timerfd_settime(tfd, 0, &spec, NULL) < 0) {
        perror("timerfd_settime"); return 1;
    }

    int epfd = epoll_create1(0);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = tfd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, tfd, &ev);

    struct epoll_event out;
    if (epoll_wait(epfd, &out, 1, -1) > 0) {
        uint64_t n;
        read(tfd, &n, sizeof(n));
        printf("timer fired, count=%lu\n", (unsigned long)n);
    }

    close(tfd);
    close(epfd);
    return 0;
}
```

```bash
g++ -std=c++17 -O2 -o test_timerfd test_timerfd.c
./test_timerfd
```

---

## 12. man 页索引

```text
man 2 clock_gettime
man 2 timerfd_create
man 2 timerfd_settime
man 2 timerfd_gettime
man 2 read
man 2 close
man 2 epoll_wait
man 2 epoll_ctl
man 7 time
```


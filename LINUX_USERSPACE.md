---
title: Linux 用户态编程规范补充
---

# Linux 用户态编程规范补充

本补充面向 Linux 用户态应用开发，与 `SKILL.md` 配合使用。通用规则（命名、注释、函数设计、整数与数据表示、模块组织）仍以 `SKILL.md` 为唯一权威来源；本文件只说明平台差异和用户态特有规则。规则冲突时，本文件中明确覆盖的条目以本文件为准，其余仍以 `SKILL.md` 为准。

## 1. 定位与优先级

本文件是 Linux 用户态平台补充，不重复通用规则。裸机/RTOS 固件的平台规则见 `MCU_RTOS.md`，其中的 ISR、`volatile`、中断栈预算等条款不适用于用户态。通用核心与本文件的衔接关系：

- 命名、格式、注释、函数设计、模块组织、通用并发原则：直接遵循 `SKILL.md`。
- 错误处理、资源配对、整数与数据表示、构建验证：在 `SKILL.md` 通用规则上叠加本文件第 2～9 节。
- 公共 API 注释除 `SKILL.md` 要求外，还需说明返回值约定、线程安全性和所有权。

## 2. 系统调用与错误处理

**Rules**:
- 按接口文档检查返回值：系统调用通常为 `-1` 并设置 `errno`；部分库函数返回 `NULL`；pthread 系列直接返回错误码且不设置 `errno`。
- 调用失败后立即保存 `errno`，再执行日志、清理等可能覆盖它的操作。
- 库和模块内部不直接 `exit()`/`abort()`，错误返回给调用者；致命退出集中在应用入口层决策。
- 错误日志包含操作、对象和错误描述；多线程环境使用 `strerror_r` 等线程安全接口。

**Examples**:

❌ **Bad**:
```c
int fd = open(path, O_RDONLY);
read(fd, buf, len);          // open 失败未检查，read 返回值未检查
log_error("open failed");    // 丢失路径与 errno 信息
```

✅ **Good**:
```c
int fd = open(p_path, O_RDONLY | O_CLOEXEC);
if (fd < 0)
{
    // 为什么：先保存 errno，日志和清理调用可能修改它
    int saved_errno = errno;
    log_error("open %s failed: %s", p_path, strerror(saved_errno));
    return ERROR_IO;
}
```

**pthread 错误码约定**:
```c
int rc = pthread_mutex_lock(&s_mutex);
if (rc != 0)
{
    // pthread 系列直接返回错误码，不读取 errno
    log_error("mutex lock failed: %d", rc);
    return ERROR_INVALID_STATE;
}
```

## 3. 资源管理

**Rules**:
- 每个 `open/fopen/socket/accept` 都有对应 `close/fclose`；`opendir/closedir`、`pthread_create` 与 `join/detach` 同样配对。
- 新建 fd 默认使用 `O_CLOEXEC` 或 `fcntl(FD_CLOEXEC)`，避免 `exec` 后泄漏到子进程。
- `malloc/calloc/realloc` 与 `free` 配对；`realloc` 必须用临时变量接收，失败时原指针仍有效。
- 多资源获取失败按逆序释放；推荐单一出口的 `goto cleanup` 模式。
- 释放后置空相关指针，防止 double free 和 use-after-free。

**realloc 示例**:

❌ **Bad**:
```c
p_buf = realloc(p_buf, new_size);   // 失败时原指针丢失，内存泄漏
```

✅ **Good**:
```c
uint8_t *p_new = realloc(p_buf, new_size);
if (p_new == NULL)
{
    // p_buf 仍然有效，按原生命周期处理
    return ERROR_NO_MEMORY;
}
p_buf = p_new;
```

**统一清理示例**:
```c
ErrorCode_t load_config(const char *p_path)
{
    int fd = -1;
    uint8_t *p_buf = NULL;
    ErrorCode_t result = ERROR_IO;

    fd = open(p_path, O_RDONLY | O_CLOEXEC);
    if (fd < 0)
    {
        goto cleanup;
    }

    p_buf = malloc(CONFIG_MAX_SIZE);
    if (p_buf == NULL)
    {
        result = ERROR_NO_MEMORY;
        goto cleanup;
    }

    // ... 读取与解析 ...
    result = ERROR_SUCCESS;

cleanup:
    free(p_buf);
    if (fd >= 0)
    {
        close(fd);
    }
    return result;
}
```

## 4. I/O 模型与超时

**Rules**:
- `read/write` 可能短读短写，必须按返回值循环处理；返回 `-1` 且 `errno == EINTR` 时按约定重试。
- 非阻塞 fd 返回 `EAGAIN/EWOULDBLOCK` 表示暂时无数据，不是致命错误。
- 多路复用使用 `poll/epoll`；epoll 边缘触发（ET）模式必须把 fd 设为非阻塞，并循环读到 `EAGAIN`。
- 计算超时和间隔使用 `clock_gettime(CLOCK_MONOTONIC)`；不要用 `time()`/`gettimeofday()` 计算耗时，避免系统时间回拨影响。

**完整读取示例**:
```c
// 读满 len 或遇到 EOF；返回实际读取字节数，出错返回 -1
static ssize_t read_full(int fd, uint8_t *p_buf, size_t len)
{
    size_t done = 0U;

    while (done < len)
    {
        ssize_t n = read(fd, p_buf + done, len - done);
        if (n > 0)
        {
            done += (size_t)n;
            continue;
        }
        if (n == 0)
        {
            break;  // 对端关闭或文件结束
        }
        if (errno == EINTR)
        {
            continue;  // 被信号打断，重试
        }
        return -1;
    }

    return (ssize_t)done;
}
```

**单调时钟示例**:
```c
struct timespec start;
if (clock_gettime(CLOCK_MONOTONIC, &start) != 0)
{
    return ERROR_INVALID_STATE;
}
```

## 5. 字符串与数据表示安全

**Rules**:
- 禁用 `gets/strcpy/strcat/sprintf/vsprintf`；使用 `snprintf` 等带容量接口，`strncpy` 不保证 NUL 结尾，不作为默认替代。
- `snprintf` 返回值大于等于缓冲区大小表示截断，必须显式处理。
- 格式化输出外部数据时使用固定格式串，禁止把用户输入作为格式串。
- 数组传参会退化为指针，`sizeof` 不再表示数组长度，长度必须显式传递。
- 打印定宽整数使用 `<inttypes.h>` 的 `PRIu64`/`PRId64` 等宏；网络多字节字段用 `htonl/ntohl/htons/ntohs` 转换，协议编解码仍遵循 `SKILL.md` 第 15 节。

**Examples**:

❌ **Bad**:
```c
char buf[64];
sprintf(buf, "%s", user_input);   // 可能溢出
printf(user_input);               // 格式化字符串注入
```

✅ **Good**:
```c
char buf[64];
int written = snprintf(buf, sizeof(buf), "%s", user_input);
if ((written < 0) || ((size_t)written >= sizeof(buf)))
{
    return ERROR_BUFFER_OVERFLOW;  // 编码错误或结果被截断
}

printf("%s", user_input);
```

**字节序与格式化示例**:
```c
uint32_t net_value = htonl(host_value);
uint16_t host_port = ntohs(net_port);

uint64_t counter = 42U;
printf("counter=%" PRIu64 "\n", counter);
```

## 6. 线程与同步

**Rules**:
- 共享可变状态用 `pthread_mutex_t` 保护整个不变量，而不是只保护单个字段；全模块约定统一的锁获取顺序，防止死锁。
- 不在持锁期间执行阻塞 I/O、耗时计算或调用未知回调；等待条件使用 `pthread_cond_t`，并用 `while` 循环防止虚假唤醒。
- 线程间标志使用 C11 `<stdatomic.h>`；`volatile` 不提供线程同步，仅保留给信号标志（第 7 节）等特定语义。
- 优先使用线程安全的 `_r` 接口（`localtime_r`、`strerror_r` 等），避免返回静态缓冲的接口。

**条件变量示例**:
```c
pthread_mutex_lock(&s_mutex);
while (!s_data_ready)
{
    // 为什么：必须用 while 重查条件，处理虚假唤醒
    pthread_cond_wait(&s_cond, &s_mutex);
}
// 在锁内消费数据
pthread_mutex_unlock(&s_mutex);
```

**原子标志示例**:
```c
#include <stdatomic.h>

static atomic_bool s_stop_requested = false;

void request_stop(void)
{
    atomic_store(&s_stop_requested, true);
}
```

**锁顺序注释示例**:
```c
// 锁顺序约定：先 s_config_mutex，后 s_io_mutex；反向获取会导致死锁
```

## 7. 信号与进程

**Rules**:
- 信号处理器只做极短操作：设置 `volatile sig_atomic_t` 标志，或调用 `write()` 等 async-signal-safe 函数；禁止 `printf`、`malloc`、加锁及大多数库函数。
- 使用 `sigaction` 而不是 `signal`；`SA_RESTART` 的选择要与第 4 节的 `EINTR` 策略一致。
- 多线程进程优先用专用线程 `sigwait` 同步处理信号，减少异步处理器。
- 网络程序处理 `SIGPIPE`：忽略该信号或使用 `send(..., MSG_NOSIGNAL)`，防止写已关闭连接导致进程退出。
- `fork` 的子进程必须回收：`waitpid` 或为 `SIGCHLD` 设置 `SIG_IGN`/`SA_NOCLDWAIT`，避免僵尸进程。
- 由 systemd 管理的服务保持前台运行，日志输出到 stderr/syslog，不自行 daemonize。

**信号标志示例**:
```c
static volatile sig_atomic_t s_sigterm_received = 0;

static void on_sigterm(int signo)
{
    (void)signo;
    s_sigterm_received = 1;  // 仅设置标志，实际清理由主循环完成
}

// 主循环
while (!s_sigterm_received)
{
    // ...
}
```

## 8. 安全编码

**Rules**:
- 不使用 `system()/popen()` 执行拼接命令；需要子进程时用 `fork/exec` 传递参数数组，不经过 shell 展开。
- 外部输入一律视为不可信：路径、长度、格式串、环境变量都要验证，防止 `../` 路径穿越。
- 避免检查与使用之间的 TOCTOU 竞态：尽量对 fd 操作（`fstat`、`O_NOFOLLOW`、`O_EXCL`），而不是先 `stat` 再 `open`。
- 最小权限：只绑定需要的地址和端口，敏感文件权限显式设置；setuid 程序不在本补充范围内，需专项评审。

**exec 示例**:
```c
char *const args[] = { "/usr/bin/tool", "--input", p_path, NULL };
execv(args[0], args);  // 参数不经 shell 展开，避免命令注入
```

## 9. 构建与验证

**Rules**:
- 在包含任何系统头文件之前定义 feature test macro（如 `_POSIX_C_SOURCE=200809L` 或 `_GNU_SOURCE`），全工程统一。
- 告警基线 `-Wall -Wextra -Werror`，按项目补充 `-Wconversion -Wshadow` 等；新增告警修复或局部说明抑制理由。
- 开发构建常规启用 ASan/UBSan（`-fsanitize=address,undefined`），泄漏检查使用 LSan 或 valgrind；发布构建与 sanitizer 构建分离。
- 纯逻辑单元测试覆盖边界、畸形输入和错误路径；协议解析入口增加 libFuzzer/AFL 模糊测试。
- 运行时诊断使用 `strace/gdb`，性能分析使用 `perf`；工具版本与验证命令记录在构建脚本中，保证可复现。

**构建示例**:
```sh
gcc -std=c11 -D_POSIX_C_SOURCE=200809L -Wall -Wextra -Werror \
    -fsanitize=address,undefined -g src/main.c -o app_debug
```

## 10. 用户态审查清单

**错误处理 (4 项)**
- [ ] 系统调用和库函数返回值全部按约定检查
- [ ] `errno` 在日志/清理前已保存；pthread 错误码未误用 `errno`
- [ ] 库内部无 `exit()/abort()`，错误向调用者传播
- [ ] 错误日志包含操作、对象与错误描述，线程环境使用线程安全接口

**资源与 I/O (6 项)**
- [ ] fd、`FILE *`、线程等资源获取/释放配对，失败路径逆序释放
- [ ] 新建 fd 默认 `O_CLOEXEC`
- [ ] `realloc` 使用临时变量接收
- [ ] `read/write` 处理短读短写、`EINTR` 和 `EAGAIN`
- [ ] epoll ET 模式 fd 为非阻塞并循环处理到 `EAGAIN`
- [ ] 超时与耗时计算使用 `CLOCK_MONOTONIC`

**字符串与数据 (4 项)**
- [ ] 无 `gets/strcpy/strcat/sprintf`，`snprintf` 截断已处理
- [ ] 用户输入不作为格式串
- [ ] 数组长度显式传递，未对退化指针使用 `sizeof`
- [ ] 网络字段使用字节序转换；定宽整数使用 `PRIu64` 等格式化宏

**并发与信号 (5 项)**
- [ ] 锁保护整个不变量，锁顺序有约定
- [ ] 持锁期间无阻塞 I/O 和未知回调
- [ ] 条件变量使用 `while` 防虚假唤醒
- [ ] 线程间标志使用 C11 原子操作，未把 `volatile` 当线程同步
- [ ] 信号处理器仅执行 async-signal-safe 操作，使用 `sigaction`

**进程与安全 (4 项)**
- [ ] 子进程已回收，无僵尸进程；`SIGPIPE` 有明确策略
- [ ] 无 `system()/popen()` 拼接命令
- [ ] 外部输入已验证，文件操作避免 TOCTOU
- [ ] systemd 服务前台运行，日志走 stderr/syslog

**构建验证 (4 项)**
- [ ] feature test macro 在头文件前统一定义
- [ ] `-Wall -Wextra -Werror` 通过，无未说明的新增告警
- [ ] ASan/UBSan 与泄漏检查已运行且干净
- [ ] 单元测试覆盖边界与错误路径，协议解析有模糊测试

---

**通用规范：** `SKILL.md`。日常速查：`QUICK_REFERENCE.md`。

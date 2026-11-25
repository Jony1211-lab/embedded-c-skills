---
title: 单片机/RTOS 平台规范补充
---

# 单片机/RTOS 平台规范补充

本补充面向裸机和 RTOS 固件开发，与 `SKILL.md` 配合使用。通用规则（常量、格式、命名、注释、函数设计、整数与数据表示、模块组织、通用并发原则）仍以 `SKILL.md` 为唯一权威来源；本文件只写固件平台特有规则，明确覆盖的条目以本文件为准。Linux 用户态开发见 `LINUX_USERSPACE.md`。

## 1. 共享对象与 `volatile`

**Rules**:
- 内存映射寄存器按芯片和编译器契约使用 `volatile`。
- ISR 与前台代码共享的对象默认声明为 `volatile`，除非有文档化的工具链/平台依据说明不需要，以防止编译器缓存或消除访问。
- `volatile` 只约束特定对象的访问可见性，不提供原子性、互斥、多步操作的顺序、多字段一致性或 DMA 缓存一致性；所需的原子操作、临界区、RTOS 同步或缓存维护必须独立处理。
- `volatile` 的读-改-写表达式不自动原子；多字节对象在目标 CPU 上的单次访问也可能不原子。
- 每个共享对象用注释说明写者、读者、执行上下文和实际同步机制；不能仅凭注释推断同步已充分。

**示例**:
```c
/**
 * @brief UART 接收缓冲写指针
 * @isr: uart_rx_isr() 写入，主任务读取
 * @thread-safe: ISR 独占写，主任务只读；平台保证该宽度单次访问原子
 *               volatile 只处理访问可见性，不替代同步
 */
static volatile uint16_t s_uart_rx_write_idx = 0;

/**
 * @brief 上一个 ADC 采样值
 * @isr: adc_complete_isr() 更新
 * @thread-safe: ISR 原子写，主任务无保护读
 */
static volatile uint16_t s_adc_value = 0;

// ISR 中
void uart_rx_isr(void)
{
    uint8_t byte = UART->DR;
    s_uart_rx_buffer[s_uart_rx_write_idx] = byte;
    s_uart_rx_write_idx = (s_uart_rx_write_idx + 1) % UART_RX_BUFFER_SIZE;
}

// 主任务中
void main_task(void)
{
    uint16_t write_idx = s_uart_rx_write_idx;  // 读取 volatile 变量
    if (write_idx != s_uart_rx_read_idx)
    {
        // 有新数据
        uint8_t byte = s_uart_rx_buffer[s_uart_rx_read_idx];
    }
}
```

## 2. 按执行上下文选择同步机制

**Rules**:
- **ISR 与任务**：ISR 工作保持有界。平台天然支持的原子状态用原子操作；小的共享更新用短小的 ISR-safe 临界区或中断屏蔽 API；移交工作用 RTOS ISR-safe 队列、信号量或通知。禁止在 ISR 中调用 task-only RTOS API。
- **任务与任务**：用 RTOS 互斥锁/信号量或文档化的同步原语保护共享可变状态；原子操作仅在语义和内存序足够时使用，原子操作不能让多步不变量整体原子。
- **DMA 与 CPU**：遵循 MCU 和缓存文档处理 DMA 可访问内存、缓存 clean/invalidate、对齐、所有权移交和内存屏障；`volatile` 不能保证 DMA 缓冲一致性。
- **多写者**：尽量定义单一所有者；否则同步机制必须保护整个不变量，而不是只保护单个字段。
- 不要假设关中断能保护任务/任务共享、DMA 访问或多核上下文。

**示例**:
```c
/**
 * @brief 通过 DMA 发送数据
 * @param p_data 数据指针
 * @param len 数据长度
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 *
 * @details
 * 竞态条件说明：
 * - 修改 s_dma_ptr 和 s_dma_len 时可能被 DMA ISR 中断
 * - 解决方案：平台临界区保证多字段更新一致
 */
ErrorCode_t uart_send_dma(const uint8_t *p_data, uint16_t len)
{
    if (!s_inited || p_data == NULL || len == 0)
        return ERROR_INVALID_PARAM;

    // platform_critical_enter/exit 代表平台/RTOS 临界区 API
    uint32_t saved_state = platform_critical_enter();
    {
        s_dma_ptr = (uint8_t *)p_data;
        s_dma_len = len;
        s_dma_status = DMA_STATUS_BUSY;

        // 启动 DMA
        DMA->CR1 = DMA_ENABLE | DMA_FROM_MEMORY;
    }
    platform_critical_exit(saved_state);  // 恢复进入前的中断状态

    return ERROR_SUCCESS;
}
```

## 3. 临界区

**Rules**:
- 使用平台或 RTOS 提供的临界区 API；不要自造没有定义契约的通用 `DISABLE_INTERRUPTS()`/`ENABLE_INTERRUPTS()` 宏。
- API 必须保存并恢复进入前的中断屏蔽状态，或正确支持嵌套；退出时盲目开中断会错误地打开调用者本已关闭的中断。
- 临界区保持短且有界；持有临界区或互斥锁期间不阻塞、不分配内存、不同步写日志、不做慢速外设操作。
- 用注释说明被保护的数据，以及所选临界区对参与上下文为何充分。

## 4. 栈、堆与实时性

### 4.1 栈与堆

**Rules**:
- 为每个 RTOS 任务和中断/主栈（如适用）定义栈预算；测试中检查高水位或 stack canary，并为最坏调用深度和中断嵌套保留余量。
- 受限栈上避免递归和大自动数组；优先使用静态或调用者提供的缓冲区，并文档化其内存开销和共享规则。
- 明确是否允许动态分配；若允许，定义允许的生命周期阶段、分配器约束、碎片策略、最大分配尺寸和失败处理。
- 默认禁止在 ISR 中分配或释放内存，除非分配器明确文档化为 ISR-safe、有界且适用于本系统。
- 每次分配失败都必须处理；不要假设初始化阶段的分配总能成功。

### 4.2 ISR 与时序约束

**Rules**:
- ISR 执行时间必须有界；禁止在 ISR 中阻塞、睡眠/延时、等待外设完成、执行无界循环或调用非 ISR-safe API。
- 耗时处理、解析、日志和协议/业务工作通过 ISR-safe 移交机制交给任务执行。
- 时序关键路径要规定并验证执行时间、阻塞时间预算、截止期和超时行为；每次等待硬件或其他任务都必须有定义的超时或文档化的有界完成保证。
- 可能饿死调度器或违反时序预算的场景不使用忙等；优先使用事件、中断或 RTOS 等待。

## 5. 目标构建与硬件验证

**Rules**:
- 在通用构建可追溯要求（`SKILL.md` 第 16 节）之上，记录目标架构、ABI、链接脚本/链接配置和关键构建参数。
- 外设、中断时序、DMA、内存布局或编译器/ABI 相关行为必须在目标板或 HIL 上验证；主机测试不足以证明这些属性。
- 栈高水位/canary 检查、ISR 执行时间测量纳入目标测试流程，并保留余量结论。
- 记录烧录、测试使用的目标和命令，保证结果可复现。

## 6. MCU/RTOS 审查清单

**并发安全 (6 项)**
- [ ] 同步机制匹配上下文：ISR/任务、任务/任务、DMA/CPU 分别评估
- [ ] ISR 与前台共享对象默认使用 `volatile`；另行保证原子性、互斥、内存顺序及 DMA 一致性
- [ ] 临界区使用平台/RTOS API，支持正确恢复中断状态及嵌套
- [ ] 临界区和锁持有时间短且有界
- [ ] ISR 使用 ISR-safe API，并将耗时工作移交任务
- [ ] DMA 缓存维护、屏障和所有权按芯片文档处理

**栈、堆与实时性 (5 项)**
- [ ] 任务栈/中断栈有预算并通过高水位或 canary 检查
- [ ] 受限路径没有不受控递归或大栈数组
- [ ] 动态内存策略明确，分配失败有处理
- [ ] ISR 无阻塞、延时、无界循环或非 ISR-safe 调用
- [ ] 关键路径有执行/阻塞预算及超时策略

**目标验证 (3 项)**
- [ ] 目标架构、ABI、链接配置与关键构建参数可追溯
- [ ] 硬件相关行为已在目标板或 HIL 验证
- [ ] 栈余量与 ISR 时序测量结果有记录

---

**通用核心：** `SKILL.md`。Linux 用户态：`LINUX_USERSPACE.md`。

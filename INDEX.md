---
title: Skill 文件索引
---

# embedded-c-conventions 文件导航

## 核心文件

| 文件 | 用途 |
|---|---|
| `SKILL.md` | 平台无关的通用核心规范与通用审查清单：接口注释、整数与数据表示安全、通用并发原则、模块组织和构建验证。 |
| `MCU_RTOS.md` | 裸机/RTOS 平台补充：ISR 与 `volatile`、按上下文选同步机制、临界区、DMA 一致性、栈堆预算与实时性、目标板/HIL 验证。 |
| `LINUX_USERSPACE.md` | Linux 用户态补充：系统调用与 `errno`、fd/内存资源配对、字符串安全、短读写与 `EINTR`、线程与信号、安全编码和 sanitizer 验证。 |
| `QUICK_REFERENCE.md` | 与完整规范同步的日常速查表。 |
| `LAYERING_AND_FUNCTION_TABLE.md` | 分层、`switch`、函数表、设备接口和状态机的选择指南。 |
| `README.md` | 技能包概览、安装和文件导航。 |

## 按任务查找

- 快速检查编码规则：打开 `QUICK_REFERENCE.md`。
- 了解规则背景和完整审查清单：打开 `SKILL.md`。
- 编写或审查裸机/RTOS 固件：在通用规范基础上叠加 `MCU_RTOS.md`。
- 编写或审查 Linux 用户态应用：在通用规范基础上叠加 `LINUX_USERSPACE.md`。
- 设计模块边界或选择分发结构：打开 `LAYERING_AND_FUNCTION_TABLE.md`。
- 在 AI 工具中应用：安装本 Skill 后调用 `/embedded-c-conventions`（或按所用工具的方式触发），并说明要审查或编写的代码范围及目标平台。

## 关键原则

- 有意义或可配置的常量应命名，不机械替换清楚的局部字面量。
- 头文件注释按接口契约需要提供；避免为简单声明重复写无信息量的注释。
- MCU/RTOS：ISR 与前台共享对象默认使用 `volatile`，同时独立设计原子性和同步机制；`volatile` 不等于锁、原子操作或 DMA 一致性。
- 根据边界、所有权和目标架构验证整数计算、转换、缓冲和并发行为。
- MCU/RTOS：为栈、堆、ISR 执行时间和关键等待定义预算并验证。
- 对有限稳定的状态/事件优先考虑清晰、可穷举的 `switch`；满足扩展、配置或隔离测试需求时再考虑函数表。
- 工具链告警、静态分析、测试和平台验证应与项目风险相匹配。
- Linux 用户态不套用 ISR/`volatile` 裸机条款；线程同步用 pthread/C11 原子操作，信号处理器仅执行 async-signal-safe 操作。

细节和优先级以 `SKILL.md` 通用核心为准；平台文档（`MCU_RTOS.md`、`LINUX_USERSPACE.md`）明确覆盖的条目在其平台内优先，其余规则仍以通用核心为准。

# Embedded C Conventions

面向 C 开发的通用编码规范与 AI Skill：以平台无关的通用编码规范为核心，按目标平台叠加 MCU/RTOS 或 Linux 用户态规则。覆盖代码可读性、接口契约、整数与数据表示安全、并发、内存和实时性，以及构建验证。

本仓库不绑定特定工具：既可以作为 AI 编码助手的 Skill 加载（任何支持 `SKILL.md` 约定的 Agent 工具均可），也可以直接作为团队的人工编码规范和评审清单使用。规则旨在帮助评审和实现，不替代项目约定、芯片手册或适用的安全标准。

## 文件

- `SKILL.md`：通用核心规范和通用审查清单，是规则的权威来源。
- `MCU_RTOS.md`：裸机/RTOS 平台补充规范（ISR、`volatile`、临界区、DMA、栈堆与实时性、目标验证）。
- `LINUX_USERSPACE.md`：Linux 用户态应用开发补充规范，只写平台特有规则。
- `QUICK_REFERENCE.md`：日常速查（通用 + 两个平台分块）。
- `LAYERING_AND_FUNCTION_TABLE.md`：架构分层与分发结构选择示例。
- `INDEX.md`：文件导航与关键原则。

## 用法一：作为 AI Skill 使用

1. **安装**：将整个目录放到所用工具的 skills 加载路径，目录名即技能名 `embedded-c-conventions`。例如：
   - Cursor：放入 Cursor 可识别的 skills 路径，通过 `/embedded-c-conventions` 调用。
   - Claude Code：放入项目 `.claude/skills/` 或用户级 skills 目录。
   - 其他支持 `SKILL.md` 的 Agent 工具：按各自文档放置；`SKILL.md` 的 `description` 已包含触发关键词，工具可自动匹配。
2. **使用时指定目标平台**，让 AI 叠加对应平台文档。提示词示例：

   裸机/RTOS 固件：
   ```text
   请按 embedded-c-conventions 审查这个模块：通用规则用 SKILL.md，平台规则用
   MCU_RTOS.md，重点关注整数边界、ISR 共享状态和错误路径。
   ```

   Linux 用户态应用：
   ```text
   请按 embedded-c-conventions 审查这个模块：通用规则用 SKILL.md，平台规则用
   LINUX_USERSPACE.md，重点关注 fd 泄漏、短读写/EINTR 处理和线程同步。
   ```
3. **生成代码时同样指定平台**。AI 应按通用核心 + 平台文档生成，并遵循 `SKILL.md` 第 8 节的 AI 工作流约定：代码带行内注释、变更总结留在对话中、不主动生成额外 .md 文档。

## 用法二：作为团队人工规范使用

- **新人入门**：按 `README.md` → `INDEX.md` → `QUICK_REFERENCE.md` 的顺序阅读。
- **日常编写**：查 `QUICK_REFERENCE.md`，其中第 1～9 节为通用规则，第 10、11 节按平台选用。
- **代码评审**：使用 `SKILL.md` 末尾的通用审查清单，叠加对应平台文档（`MCU_RTOS.md` / `LINUX_USERSPACE.md`）末尾的平台清单逐项检查。
- **规则有争议时**：以 `SKILL.md` 通用核心为准；平台文档明确覆盖的条目在其平台内优先。

## 项目定制

- 编写或审查时先尊重代码库现有风格、编译器和目标约束。规范中的示例用于说明思路，不应不加验证地复制到目标固件或生产代码。
- 可按项目约定调整命名前缀、大括号风格、告警集等条目；修改通用规则时，同步更新 `QUICK_REFERENCE.md`，保持单一权威来源。

## 适用原则

- 对公共 API 按调用契约提供必要说明；简单且自解释的声明无需重复注释。
- Linux 用户态开发在通用规则上叠加 `LINUX_USERSPACE.md`；裸机 ISR、`volatile` 和中断栈预算条款不直接套用。
- ISR/前台共享对象默认使用 `volatile`（MCU/RTOS），并另行处理原子性、互斥、顺序和 DMA 一致性。
- 用定宽类型表达有明确宽度的数据；显式验证整数运算、窄化转换、缓冲边界和协议表示。
- 明确任务栈、堆策略、ISR/关键路径预算及其验证方式（MCU/RTOS）。
- 根据可读性、穷举性和扩展需求选择 `switch` 或函数表，不将函数表作为通用替代。
- 结合工具链告警、静态分析和测试；按平台完成目标板/HIL（MCU/RTOS）或 sanitizer 等（Linux 用户态）验证。

规则冲突时以 `SKILL.md` 通用核心为准；平台文档（`MCU_RTOS.md`、`LINUX_USERSPACE.md`）明确覆盖的条目在其平台内优先。架构文档中的抽象和示例需按项目实际需求取舍。

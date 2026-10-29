# 嵌入式固件技能仓库

本仓库提供可复用的嵌入式固件工程技能。

所有技能直接存放在 `skills/` 下：

```text
skills/
├── embedded-architecture
├── embedded-cstyle
├── embedded-documentation
├── embedded-driver-design
├── embedded-isr-design
└── embedded-rtos-design
```

## 项目文件

- [CONTEXT.md](CONTEXT.md)：项目理念、技术栈和设计原则
- [CLAUDE.md](CLAUDE.md)：仓库规则和开发指南
- 每个技能都有独立的 `SKILL.md`，包含详细规则和检查清单

## 技能职责

- `embedded-architecture`
  - 固件架构：项目结构、分层、依赖方向和平台隔离

- `embedded-cstyle`
  - C 编码规范：命名、类型、函数、变量、宏、头文件、内存和错误处理

- `embedded-documentation`
  - 文档生成：项目、模块、接口、RTOS/ISR、控制共五级文档体系

- `embedded-driver-design`
  - 驱动分层设计：总线抽象、设备上下文、寄存器封装和模拟测试

- `embedded-isr-design`
  - 中断设计：快进快出、事件捕获、FromISR API 和延后处理

- `embedded-rtos-design`
  - RTOS 系统架构：任务划分、调度、优先级、任务间通信和看门狗

## 仓库规则

每个技能必须：

1. 直接存放在 `skills/` 下
2. 包含有效的 `SKILL.md`
3. 在 `.claude-plugin/plugin.json` 中注册
4. 在本文档中列出

## 技能开发指南

创建或修改技能时：

- 保持职责聚焦、单一
- 避免技能之间功能重叠
- 优先编写可复用的工程规则，而非项目专用逻辑
- 在有帮助时提供实用示例
- 与现有技能的结构和命名保持一致

目标是为 AI 辅助开发构建可维护、可扩展且专业的嵌入式固件技能库。

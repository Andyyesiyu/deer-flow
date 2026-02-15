# DeerFlow: From Deep Research to Super Agent Harness

> 架构演进大纲 — 从 1.0 到 2.0 的核心变革

---

## 一、开篇：为什么要重写

### 1.1 1.0 的成功与局限

- DeerFlow 1.0 定位：**Deep Research Framework**（深度研究框架）
- 社区超预期使用：数据管道、PPT 生成、Dashboard 搭建、内容工作流……
- 核心洞察：DeerFlow 不只是研究工具，而是一个 **Agent 运行时容器（Harness）**

### 1.2 重写的决心

- "DeerFlow 2.0 is a ground-up rewrite. It shares no code with v1."
- 不是渐进式重构，而是**从零开始的架构革新**
- 1.x 分支继续维护，2.0 成为主线

---

## 二、架构对比：1.0 vs 2.0

### 2.1 Agent 模型

| | 1.0 | 2.0 |
|---|---|---|
| 架构模式 | 多 Agent 工作流图 | 单 Lead Agent + 动态 Subagent |
| Agent 角色 | Researcher / Coder / Planner / Coordinator / Reviewer | Lead Agent（全能编排者）+ general-purpose / bash Subagent |
| 编排方式 | 预定义图结构，固定路径 | 运行时动态分解，按需生成 |
| 扩展方式 | 新增 Agent 类型 | 新增 Skill / Tool / MCP Server |

**核心转变**：从 "定义 Agent 是谁" → "定义 Agent 能做什么"

### 2.2 执行模型

**1.0：顺序流水线**
```
用户 → Planner → Researcher → Coder → Reviewer → 输出
                    ↑              ↓
                    └── 循环修正 ──┘
```

**2.0：动态分解 + 并行执行**
```
用户 → Lead Agent → 分析 → 分解为 N 个子任务
                         → 并行启动 ≤3 个 Subagent
                         → 收集结果 → 综合输出
                    （如果 >3 个子任务，分批执行）
```

- 关键优势：不再受限于预设流程，任何任务都能处理

### 2.3 部署架构

**1.0**：单体服务

**2.0**：四服务架构
```
Nginx (2026) — 统一入口
├── LangGraph Server (2024)  — Agent 运行时
├── Gateway API (8001)       — REST API（模型/MCP/技能/文件/记忆）
└── Frontend (3000)          — Next.js 完整 Web 应用
```

---

## 三、2.0 的五大核心创新

### 3.1 中间件链架构（Middleware Chain）

- **1.0 的问题**：跨 Agent 的通用逻辑（记忆、上下文压缩、文件处理）分散在各处
- **2.0 的方案**：11 个中间件按严格顺序组成流水线，覆盖 Agent 完整生命周期

**中间件清单**（按执行顺序）：
1. ThreadDataMiddleware — 线程隔离目录
2. UploadsMiddleware — 文件上传感知
3. SandboxMiddleware — 沙箱获取
4. DanglingToolCallMiddleware — 中断修复
5. SummarizationMiddleware — 上下文压缩（可选）
6. TodoListMiddleware — 任务跟踪（可选）
7. TitleMiddleware — 自动标题生成
8. MemoryMiddleware — 异步记忆更新
9. ViewImageMiddleware — 视觉能力注入（可选）
10. SubagentLimitMiddleware — 并发硬限制（可选）
11. ClarificationMiddleware — 澄清中断（必须最后）

**核心优势**：
- 关注点分离，每个中间件只做一件事
- 条件组合：根据运行时配置动态注入
- 不靠提示词约束行为，靠中间件**物理保证**（如并发限制通过截断 tool call 实现）

### 3.2 沙箱执行环境（Sandbox）

- **1.0**：Agent 只有"工具调用"能力
- **2.0**：Agent 拥有**一台完整的计算机**

```
/mnt/user-data/
├── uploads/      ← 用户上传的文件
├── workspace/    ← 工作区
└── outputs/      ← 最终交付物
```

关键设计点：
- Provider 模式：Local / Docker / Kubernetes 可插拔
- 虚拟路径转译：Agent 看到的路径与物理路径解耦
- 懒初始化：不调工具不创建沙箱
- 上下文继承：Subagent 复用父 Agent 的沙箱

### 3.3 Skill 系统（渐进式能力扩展）

- **1.0**：能力硬编码在 Agent 角色中
- **2.0**：能力模块化为可插拔的 Skill

```
skills/
├── public/                  ← 内置技能
│   ├── research/SKILL.md
│   ├── slide-creation/SKILL.md
│   └── web-page/SKILL.md
└── custom/                  ← 用户自定义
    └── my-skill/SKILL.md
```

核心设计：
- **渐进加载**：启动时只注入名称+描述（几十 tokens），需要时才读取完整指令
- **声明式定义**：SKILL.md = YAML 元数据 + Markdown 指令
- **运行时管理**：通过 UI 或 API 动态启用/禁用/安装

### 3.4 长期记忆系统（Memory）

- **1.0**：对话结束即遗忘
- **2.0**：跨会话持久记忆

```json
{
  "user": {
    "workContext": "用户正在做数据分析项目...",
    "personalContext": "偏好 Python，简洁风格...",
    "topOfMind": "近期关注 RAG 优化..."
  },
  "facts": [
    { "content": "用户的项目用 FastAPI", "confidence": 0.9, "category": "knowledge" }
  ]
}
```

核心设计：
- **全自动**：Agent 无需主动"保存"，中间件静默完成
- **异步防抖**：30 秒批量处理，不阻塞对话
- **LLM 提取**：不存原文，用 LLM 提取结构化 facts + context
- **启动注入**：下次对话时 top-15 facts 自动注入系统提示词

### 3.5 工具生态（三层扩展）

- **1.0**：固定工具集
- **2.0**：三层可扩展工具体系

```
┌─────────────────────────────────────────┐
│  Layer 1: 配置工具 (config.yaml)        │
│  bash / read_file / write_file /        │
│  web_search / web_fetch / ...           │
├─────────────────────────────────────────┤
│  Layer 2: MCP 工具 (extensions_config)  │
│  任何 MCP Server（stdio/SSE/HTTP）      │
│  UI 可视化管理，热更新无需重启           │
├─────────────────────────────────────────┤
│  Layer 3: 内置工具 (always on)          │
│  present_files / ask_clarification /    │
│  view_image / task（Subagent 委托）     │
└─────────────────────────────────────────┘
```

---

## 四、2.0 独有的工程亮点

### 4.1 澄清优先（Clarify-First）

- 不是"先做再问"，而是"不确定就先问"
- ClarificationMiddleware 通过 `wrap_tool_call` 拦截执行，返回 `Command(goto=END)` 中断 Agent 循环
- 用户回答后 Agent 恢复执行
- **区别于所有其他框架**：执行流程中的优雅中断机制

### 4.2 双保险行为约束

- 提示词层面："最多 3 个并行子任务"
- 中间件层面：`SubagentLimitMiddleware` 物理截断超额调用
- **Prompt 是建议，Middleware 是保证**

### 4.3 上下文工程

- SummarizationMiddleware：上下文接近 token 限制时自动压缩
- Subagent 上下文隔离：子任务不会污染主 Agent 的上下文窗口
- 渐进式 Skill 加载：按需加载，不预占上下文
- 文件系统充当"外部记忆"：中间结果写入文件，而非堆积在上下文中

### 4.4 完整的前端体验

- Next.js 16 + React 19 全栈 Web 应用
- 实时流式渲染（LangGraph SDK SSE）
- Artifact 双视图（代码/预览）
- Chain of Thought 可折叠展示
- Subagent 任务卡片实时进度
- 文件上传（自动转换 PDF/PPT/Excel/Word → Markdown）
- Skill 安装/管理 UI
- MCP Server 配置 UI
- 记忆查看器
- 中英双语 + 暗色模式

---

## 五、与竞品的定位差异

### 5.1 vs. 其他 Agent 框架（CrewAI / AutoGen / AutoGPT）

> 其他框架给你 Agent 的骨架，DeerFlow 给你 Agent 的整个身体。

- 其他框架：定义 Agent + 编排协作 → 你自己搭基础设施
- DeerFlow：开箱即用的完整平台 → 沙箱、记忆、前端、API 全部就绪

### 5.2 vs. LangChain Deep Agents SDK

> Deep Agents 是 SDK，DeerFlow 是 Platform。

- Deep Agents：`create_deep_agent()` → 你需要自己搭建其余一切
- DeerFlow：`make serve` → 一个可部署的完整产品

| 维度 | Deep Agents | DeerFlow 2.0 |
|------|-------------|--------------|
| 本质 | Python 库 | 全栈平台 |
| 前端 | 无 | 完整 Next.js 应用 |
| REST API | 无 | 40+ 端点的 Gateway |
| MCP 集成 | 基础 | 多服务器、多协议、UI 管理 |
| Skill 系统 | 无 | 渐进加载、声明式定义 |
| 自研中间件 | 0 | 8 个生产级中间件 |
| 部署方案 | 自己搭 | Docker 一键启动 |

### 5.3 vs. 原生 LangGraph

> LangGraph 是引擎，DeerFlow 是整车。

- LangGraph：提供图结构和状态机原语 → 最大灵活性，最少内置
- DeerFlow：在 LangGraph 之上构建完整的 Agent 运行时 → 牺牲部分灵活性，换取开箱即用

---

## 六、适用场景

### 选 DeerFlow 2.0 的理由

- 需要 Agent **真正执行代码和操作文件**（不只是聊天）
- 任务复杂，需要**多步骤分解和并行执行**
- 需要**生产级部署**（沙箱隔离、持久记忆、完整 UI）
- 想要**开箱即用**但保留完全可扩展性
- 需要接入 **MCP 生态**
- 团队或用户需要 **Web 界面**交互

### 一句话定位

> **DeerFlow 2.0 — 不只是一个 Agent 框架，而是一个让 Agent 真正能干活的运行时平台。**

---

## 七、未来展望

> （根据项目规划补充）

- [ ] 更多内置 Skill
- [ ] 多用户协作
- [ ] Agent 间通信协议
- [ ] 更丰富的沙箱环境
- [ ] 企业级权限与审计

---

*文档版本：Draft v0.1 | 基于 DeerFlow 2.0 代码库分析*

# LADD 实施策略：编排器-执行器模型

本文档提出了一种技术实现方案，用于构建前文所述的“分层代理驱动开发 (LADD)”模式。方案的核心是避免直接合并 `BMAD-METHOD` 和 `claude-code-spec-workflow` 的代码库，而是采用一个更健壮、更灵活的 **“编排器-执行器 (Orchestrator-Executor)”** 架构。

## 一、 反面模式：为什么不能直接整合代码库？

直接将两个项目合并为一个单一的代码库是不可行的，原因如下：

1.  **理念与目标的冲突:** `BMAD-METHOD` 是一个通用战略框架，关心“规划的正确性”。而 `claude-code-spec-workflow` 是一个专用的战术执行引擎，关心“执行的效率”。强行合并会导致一个目标混乱、难以维护的单体应用。
2.  **技术债与复杂性:** 处理不同的依赖、编码风格和架构假设会引入巨大的技术债，使项目迅速变得臃肿和脆弱。
3.  **丧失灵活性和可替换性:** LADD 模式的威力在于分层。战略层应该能够指挥**任何**战术执行引擎，反之亦然。紧密的耦合会破坏这种模块化能力，阻碍未来的技术演进。

## 二、 推荐架构：编排器-执行器模型

最佳实践是将两个系统解耦，通过清晰的 API 进行通信。

### 1. 执行器 (Executor): `claude-code-spec-workflow` 的服务化

- **角色:** 战术代码执行引擎，一个“代码生成即服务”。
- **改造任务:**
    1.  保持 `claude-code-spec-workflow` 现有代码库不变，其 CLI 工具依然有价值。
    2.  在其之上，利用项目已有的 `fastify` 依赖，添加一个轻量级的 **API 层**。
    3.  将核心的 `/` 命令功能暴露为 RESTful API 端点。

- **示例 API 端点:**
    - `POST /api/v1/specs`: 创建一个新的 spec，取代 `/spec-create`。
        - **Body:** `{ "name": "feature-name", "description": "..." }`
    - `POST /api/v1/specs/{name}/orchestrate`: 启动自动化执行，取代 `/spec-orchestrate`。
    - `GET /api/v1/specs/{name}/status`: 获取任务状态，取代 `/spec-status`。
    - `GET /api/v1/specs/{name}/files`: 获取该 spec 生成的文件列表和内容。

### 2. 编排器 (Orchestrator): 一个全新的战略规划应用

- **角色:** 战略规划中心，是 `BMAD-METHOD` 理念的具体实现。
- **实现任务:**
    1.  创建一个**全新的、独立的项目**（例如，一个使用 React/Vue 和 Node.js/Python 的 Web 应用）。
    2.  提供一个图形用户界面，让用户（如游戏总监）与各种“规划代理”进行交互，完成 LADD 流程的**战略规划阶段**。
    3.  负责将最终的设计方案转化为结构化的 **技术任务蓝图 (Tech Task Blueprint)**，通常是一个 JSON 对象。
    4.  通过调用**执行器**的 API 来管理和监控整个开发流程。

## 三、 端到端工作流程示例

1.  **访问编排器:** `游戏总监` 访问 `ladd-game-planner.com` (编排器应用)。
2.  **战略规划:** 他与 `主策划代理` 和 `主美术代理` 对话，完成“宠物系统”的创意和设计。
3.  **生成任务蓝图:** 编排器在后台生成一份 JSON 格式的任务蓝图：
    ```json
    {
      "feature": "Pet System",
      "tasks": [
        { "id": "pet-data", "description": "Create data structures for pets..." },
        { "id": "pet-following-ai", "description": "Implement pet following logic..." },
        { "id": "pet-combat-system", "description": "Implement pet combat abilities..." }
      ]
    }
    ```
4.  **API 调用与执行:** 编排器开始按顺序执行任务。
    - 它向 `claude-code-spec-workflow` 服务 (执行器) 发起 API 调用: `POST http://localhost:3000/api/v1/specs`，请求体为 `{ "name": "pet-data", "description": "..." }`。
    - 接着调用 `POST http://localhost:3000/api/v1/specs/pet-data/orchestrate`。
5.  **后台工作:** `claude-code-spec-workflow` 服务在后台运行，创建文件、生成代码、运行测试。
6.  **进度监控:** 编排器通过轮询 `GET /api/v1/specs/pet-data/status` 来获取进度，并在其自身的 Dashboard 上实时更新进度条。
7.  **循环执行:** 一个任务完成后，编排器自动开始下一个任务，直到所有任务完成。
8.  **人工介入:** 在需要人工审核或集成非代码资产的环节，编排器会通知相关的人类开发者。

## 四、 结论与优势

**不要合并，要解耦和集成。** 这种“编排器-执行器”模型是实现 LADD 模式的正确路径，它带来了诸多好处：

- **关注点分离:** 战略和战术各司其职，系统边界清晰。
- **高度灵活性:** 可以独立升级或替换任何一个组件。未来可以为编排器添加新的执行器（如 Unity 执行器），或让执行器接收来自其他规划工具的指令。
- **可扩展性:** 每个服务都可以独立扩展，以应对更高的负载。
- **健壮性:** 系统间的松耦合降低了连锁故障的风险，使整体架构更加稳健。

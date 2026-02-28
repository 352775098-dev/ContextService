

```mermaid
graph TB
    subgraph "智能Agent系统"
        User((用户))

        subgraph "核心服务层"
            ContextService["上下文服务 ContextService\n(负责Agent会话生命周期与动态上下文组装)"]
            KMM["记忆服务 KMM\n(负责记忆的存储、加工与压缩)"]
            AgentGov["Agent治理服务 AgentGov\n(负责Agent流程定义与分发)"]
            ToolService["工具服务 ToolService\n(提供各种外部能力调用)"]
        end

        subgraph "Agent运行层"
            BusinessAgent["业务Agent\n(承载具体业务逻辑，运行Agent Flow)"]
        end

        subgraph "辅助组件"
            LLM[(大语言模型)]
            LocalCache["本地缓存 (Local Cache)\n(业务Agent侧的会话上下文快照)"]
        end
    end

    %% 关系与数据流
    User -->|1. 发起对话/请求| BusinessAgent

    %% AgentGov 到 ContextService 的流程下发
    AgentGov -.->|2. 下载Agent Flow定义| ContextService

    %% ContextService 内部及与其他服务的交互
    ContextService -->|3. 根据Flow定义，获取所需参数| KMM
    ContextService -->|4. 调用工具获取动态信息| ToolService
    ContextService -->|5. 调用模型（按需）| LLM
    ContextService -->|6. 维护会话上下文| ContextService

    %% 业务Agent与上下文服务的交互
    BusinessAgent -->|7. 创建/获取会话实例\n读取上下文内容| ContextService
    BusinessAgent -->|8. 写入/更新本地缓存| LocalCache
    BusinessAgent -->|9. 写入交互过程中的上下文信息| KMM

    %% KMM的异步处理
    KMM -.->|10. 异步加工、压缩记忆| KMM

    %% Agent运行时的最终调用
    BusinessAgent -.->|11. 在Flow中调用LLM（携带组装好的Prompt）| LLM

    %% 样式
    classDef core fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef agent fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef external fill:#f5f5f5,stroke:#9e9e9e,stroke-dasharray: 5 5;

    class ContextService,KMM,AgentGov,ToolService core;
    class BusinessAgent agent;
    class User,LLM external;
```

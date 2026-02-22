# ContextService
上下文工程实现

# 上下文工程解决的问题场景
- **上下文窗口限制**：LLM的上下文窗口有限，需要对历史对话进行智能压缩
- **上下文持久化**：将对话历史保存到持久化存储，支持跨会话恢复
- **上下文选择**：从海量历史中选择最相关的上下文片段
- **工具编排**：管理对话中的工具调用链和状态
- **长期记忆**：实现跨会话的信息保留和检索
1. Agent内部的上下文整合：时间推理、内容整合、冲突更新、记忆召回、工具输出信息整合
2. 跨Agent的上下文共享和隔离
3. 跨端的agent上下文共享和隔离

# 上下文工程设计原则
1. 所有模型调用前都应该有上下文，prompt动态调整也在上下文范围，对于不需要上下文内容的经过上下文调用后可以不填内容
2. 写原始对话应放到记忆服务，写对话的时机应由对话系统负责；上下文维护的是会话的记忆、摘要、工具的输出、 不是原始对话
3. 对模型固定配一个上下文窗口，每个模型的上下文窗口隔离；窗口中动态内容持续刷新，
4. 把上下文成消费者和生产者模式，填充上下文的过程和Agent的流程异步，业务路径不允许随意加流程
5. 尽量不要搞显性调用，建议用消息或事件方式作为接口
6. 上下文应该做成模型感知的，场景的诉求通过prompt模板和skill来实现，要用开放算法机制，针对不同场景，去动态调整


# 上下文工程的逻辑架构
@startuml
!define RECTANGLE class

skinparam componentStyle rectangle
left to right direction

package "上下文工程" {

  rectangle "接口层" {
    component "写入QA" as writeQA
    component "查询上下文" as queryContext
    component "重建上下文" as rebuildContext
    component "删除上下文" as deleteContext
  }

  rectangle "上下文管理层" {
    component "保存" as save
    component "压缩" as compress
    component "老化" as aging
    component "关联" as relate
    component "记忆" as memorize
    component "组装" as assemble
    component "结构化模板" as template
  }

  rectangle "上下文数据层" {
    component "私有上下文" as privateCtx
    component "公共上下文" as publicCtx
    component "原始对话" as rawChat
    component "压缩内容" as compressed
    component "长期记忆" as longTermMemory
    component "工具结果" as toolResult
  }

  rectangle "基础组件层" {
    component "RAG" as rag
    component "长期记忆" as longTermBase
    component "工具" as tool
    component "模型推理" as inference
  }

}

writeQA --> save
queryContext --> assemble
rebuildContext --> save
deleteContext --> save

save --> privateCtx
save --> publicCtx
save --> rawChat
compress --> compressed
aging --> longTermMemory
relate --> longTermMemory
memorize --> longTermMemory
assemble --> privateCtx
assemble --> publicCtx
assemble --> compressed
assemble --> longTermMemory
template --> assemble

privateCtx --> rag
publicCtx --> rag
rawChat --> compress
compressed --> longTermBase
longTermMemory --> longTermBase
toolResult --> tool

rag --> inference
longTermBase --> inference
tool --> inference
@enduml
图示说明：
接口层：提供对上下文的外部操作入口（写入、查询、重建、删除）。

上下文管理层：内部核心逻辑，负责上下文的保存、压缩、老化、关联、记忆、组装、结构化模板。

上下文数据层：存储不同类型的数据（私有/公共、原始/压缩、长期记忆、工具结果）。

基础组件层：底层支撑能力（RAG、长期记忆系统、工具、模型推理）。

箭头表示数据流或调用关系，展示层与层之间的协作。


## 接口层

## 上下文管理层

## 上下文组装层

## 上下文组装层

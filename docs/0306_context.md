**原则**
规则类任务（比如foodie和shoping的定制、高铁、机场的选择）进skill，不进prompt
工具全部封装为skill，分为意图类和非意图类（非意图类为能力，被意图依赖，比如时间等）模型只判断意图，依赖关系由DAG引导
依赖类的skill按需并行处理
广义的问答场景（以文本输出）单意图和多意图统一合并为多意图，由GRM判定概率偏低的意图，剪枝掉无效意图，最终由大模型总结
**workflow**
```mermaid
graph TD
    User[用户] -- "原始消息" --> InputGate{输入网关}
    NewSkill[技能] -- "技能离线准备" --> SkillRegistryMethod{技能注册}

    %% === 并行感知层 ===
    subgraph PerceptionLayer ["并行感知层 (Parallel Pre-processing)"]
        direction TB
        InputGate --> TempoSpatialTask[时空标准化<br>解决: 日期计算/指代消解<br><b>标准化输出，对接高频skills<b>]
        InputGate --> EntityTask[实体提取与映射<br>解决: 实体泛化/POI纠偏<br><b>检索子句、关键词<b>]
        InputGate --Graph检索--> MemoryTask[记忆检索<br>GRAG: 偏好/记忆<br><b>个人信息图谱检索<b>]
        InputGate --> ContextTask[对话历史检索<br>RAG: BM25+Emb+ReRank<br><b>排序填充prompt<b>]
        InputGate --> ExceptTask[例外检索<br>RAG：BM25+Emb<br><b>shot填充prompt<b>]

        %% 并行结果汇聚
        TempoSpatialTask--> StructuredCtx[结构化上下文<br><b>按需获取<b>]
        EntityTask --> StructuredCtx
        MemoryTask --> StructuredCtx
        ContextTask --> StructuredCtx
        ExceptTask--> StructuredCtx
    end
    
    subgraph DataBase ["<font color='red'><b>外置信息库<b></font>"]
        PersonalGraph1[(个人信息图谱，长期)]-->MemoryTask
        ConversationHist1[(对话历史，短期)]-->ContextTask
        Synonym[(同义词库)]
        Exceptional[(例外示例/非标场景示例)]-->ExceptTask
        Synonym-->Exceptional
        Synonym-->ConversationHist1
        Synonym-->PersonalGraph1
    end

    %% === 动态路由层 ===
    subgraph RouterLayer ["意图识别层"]
        StructuredCtx -- "按需获取" --> RouterModel["轻量化意图识别<br><b>多意图输出，不考虑参数和依赖<b>"]
        RouterModel-->Top1["意图1+概率"]
        RouterModel-->Top2["意图2+概率"]
        RouterModel-->Top3["意图3+概率"]
        RouterModel-->Other["......"]
        Top2-->Judge{"GRM+概率阈值<br><b>保留判断<b>"}
        Top3-->Judge
        Other-->Judge
        Top1-->IntentList[意图列表]
        Judge-->IntentList
    end

    %% === skill能力中心 ===
    subgraph SkillRegistry ["Skill 注册"]
        SkillDB[(技能注册表)]
        SkillMerge[技能合并]
        SkillList[意图相关技能列表]
        SkillFunc[技能能力]
        SkillPara[技能参数]
        SkillPrePara[可预置技能参数库]
        SkillDAG[技能依赖DAG]
        SkillRegistryMethod-->SkillPara
        SkillRegistryMethod-->SkillFunc
        SkillPara-->SkillDB
        SkillFunc-->SkillDB
        SkillDB-->SkillMerge
        SkillMerge--更新-->SkillDAG
        SkillMerge--更新-->SkillList
        SkillMerge--更新-->SkillPrePara
        SkillList--"检索(可选)"-->RouterModel
    end

    SkillPrePara-->TempoSpatialTask
    SkillPrePara-->EntityTask

    %% === DAG 编排执行层 ===
    subgraph ExecutionLayer ["并行编排执行层"]
        Orchestrator[任务编排器]

        %% 示例：购票Skill的DAG
        subgraph TrainSkill ["Skill: 票务服务(并行示例1)"]
            SlotCheck{参数完整性检查}
            ClarifyNode[澄清交互节点]
            ParaReAct[参数重获取]
            ToolCall_12306[工具调用: 12306]

            %% 通用能力
            TimeUtil[通用能力: 时间校准]
            LocUtil[通用能力: 地点解析]
        end

        %% 示例：搜索 Skill
        subgraph SearchSkill ["Skill: 出行检索推荐(并行示例2)"]
            POISlotCheck{参数检查}
            ToolCall_Poi[工具调用: 并行POI搜索]
            POIReRank["重排(可选)"]
        end
        ToolCall_Poi-->POIReRank
        POISlotCheck--> ToolCall_Poi
        Orchestrator --> TrainSkill
        Orchestrator --> SearchSkill
    end

    %% === 连接线逻辑 ===
    IntentList-- "多意图（能力）优先" --> Orchestrator
    SkillDAG-- "技能依赖" -->Orchestrator 
    StructuredCtx -.-&gtSlotCheck
    %% DAG内部流转逻辑
    SlotCheck -- "缺失/模糊" --> ParaReAct
    ParaReAct --> ClarifyNode
    SlotCheck -- "参数齐全" --> LocUtil --> ToolCall_12306
    SlotCheck -- "参数齐全" --> TimeUtil --> ToolCall_12306
    ClarifyNode --> User

    %% 结果返回
    ToolCall_12306 --> SummaryAgent[总结生成<br><b>强模型/慢思考<b>]
    POIReRank--> SummaryAgent

    SummaryAgent --答复--> User

    %% === 问题卡片 ===
    subgraph NewCard ["<font color='blue'><b>新问题卡片生成<b></font>"]
        NewCardEntray{卡片入口}-->NewCardInstance
        subgraph NewCardInstance["新问题卡片生成"]
            NewCardQuery[新问题]
            NewCardIntent[确定意图]
        end
        User1[用户]--选择-->NewCardInstance--无需意图流程-->NewCardSkillProcedure[Skill流程]-->NewCardSum[总结]-->FollowUp[后续]
        NewCardSum--答复-->User1
    end
    SummaryAgent --> NewCardEntray

    %% === 回话历史和记忆管理 ===
    subgraph MemoryAndHistory ["回话历史和记忆管理"]
        MHEntray{记忆入口}
        subgraph Memory[记忆管理]
            PersonalSave[个人信息提取]
            PersonalGraph[个人信息图谱]
            PersonalSave--图谱更新-->PersonalGraph
            PersonalGraph-->GraphSearch[记忆检索待用]
        end
        subgraph History[对话历史管理]
            HistoryCompress[历史压缩]
            Scenario[分场景压缩]
            HistoryCompress-->Scenario
            CompressedHist[压缩上下文]
            CompressedKey[关键词提取]
            Scenario-->CompressedHist
            Scenario-->CompressedKey
        end
        History-->Multi[多轮对话待用]
    end
    SummaryAgent --过程信息（意图/skill等）--> MHEntray
    MHEntray--> Memory
    MHEntray--> History
```

### 格式示例
{
  "request_meta": {
    "request_id": "req-20260227-travelshop",
    "timestamp": "2026-02-27T09:45:00+08:00",
    "raw_query": "帮我订明天下午从新加坡飞北京的机票，要求后天上午前到达。这周末我要去逛逛北京的非标网红打卡地，顺便帮我查查附近的商场哪里买国潮服饰比较好，还是按我以前的喜好买。",
    "session_id": "sess-998877"
  },

  "1_tempo_spatial_info": {
    "_comment": "时空标准化增强版：严格区分源与目的，支持时间范围",
    "time_anchors": {
      "current_time": "2026-02-27T09:45:00+08:00",
      "source_time": {
        "raw": "明天下午",
        "standardized": "2026-02-28T14:00:00/18:00:00",
        "type": "departure_time",
        "flexibility": "hard"
      },
      "destination_time": {
        "raw": "后天上午前到达",
        "standardized": "<= 2026-03-01T12:00:00",
        "type": "arrival_deadline",
        "flexibility": "soft"
      },
      "time_range": {
        "raw": "这周末",
        "start_time": "2026-02-28T00:00:00+08:00",
        "end_time": "2026-03-01T23:59:59+08:00",
        "duration_hours": 48,
        "type": "activity_period"
      }
    },
    "spatial_anchors": {
      "current_location": {"city": "新加坡", "coord": "1.3521, 103.8198"},
      "source_address": {
        "raw": "新加坡",
        "standardized_city": "Singapore",
        "poi_type": "airport",
        "inferred_poi": "新加坡樟宜机场 (SIN)"
      },
      "destination_address": {
        "raw": "北京",
        "standardized_city": "北京市",
        "poi_type": "airport",
        "inferred_poi": "北京首都/大兴机场 (PEK/PKX)"
      },
      "activity_radius": {
        "center_coord": "TBD (基于最终目的地)",
        "radius_km": 10,
        "type": "shopping_and_poi_area"
      }
    }
  },

  "2_entity_and_index_info": {
    "_comment": "新增业务线索引：出行索引与购物索引",
    "domain_indexes": {
      "travel_index": {
        "transport_mode": "flight",
        "trip_type": "international_leisure",
        "poi_preference": ["trending", "hidden_gem", "non_standard"],
        "route_type": "one_way"
      },
      "shopping_index": {
        "category": "clothing",
        "sub_category": "guochao (国潮/national trend)",
        "venue_type": "shopping_mall",
        "purchase_mode": "offline_in_store"
      }
    },
    "segmented_clauses":[
      "帮我订明天下午从新加坡飞北京的机票要求后天上午前到达",
      "这周末我要去逛逛北京的非标网红打卡地",
      "顺便帮我查查附近的商场哪里买国潮服饰比较好"
    ],
    "keywords":["新加坡", "北京", "机票", "非标网红", "商场", "国潮服饰"],
    "skill_pre_params": {
      "matched_skills":["flight_booking", "poi_recommendation", "shopping_guide"]
    }
  },

  "3_memory_info": {
    "_comment": "融合出行与购物的双重个人图谱偏好",
    "user_profile": {
      "user_id": "u-10086",
      "vip_level": "gold"
    },
    "preferences":[
      {
        "domain": "travel",
        "attributes": {
          "flight_seat": "aisle (过道)",
          "airline_alliance": "Star Alliance (星空联盟)",
          "passport_no": "E12345678"
        }
      },
      {
        "domain": "shopping",
        "trigger": "以前的喜好",
        "attributes": {
          "clothing_size": "L",
          "favorite_brands": ["李宁", "中国李宁", "Randomevent", "FMACM"],
          "price_sensitivity": "medium-high"
        }
      }
    ]
  },

  "4_context_history": {
    "_comment": "对话历史状态",
    "carried_over_slots": {},
    "recent_turns":[]
  },

  "5_exceptional_shots": {
    "_comment": "复杂/非标意图的 Few-shot 检索结果",
    "matched_scenarios": ["非标网红打卡地推荐", "国潮服饰线下导购"],
    "few_shots":[
      {
        "query": "商场哪里买国潮服饰",
        "example_action": "Search_ShoppingGuide(tags=['国潮', '设计师品牌'], location_context='destination_address')"
      }
    ]
  }
}


### 结构化上下文使用说明（待讨论优化）
1. 意图识别层 (RouterLayer) & 编排器 (Orchestrator)
通过读取 2_entity_and_index_info.domain_indexes，路由层可以直接获得业务线的上帝视角：

看到 travel_index.transport_mode = flight，立即拉起 Skill: 国际机票预订 DAG。
看到 shopping_index.category = clothing，立即拉起 Skill: 线下购物导览 DAG。
由于存在时间与空间的跨度，Orchestrator 知道这两个 DAG 需要按照时间线先后执行（先订机票，后推荐北京本地商场）。
2. 参数完整性检查与工具调用 (SlotCheck -> ToolCall)
在执行特定的 Skill DAG 时，参数的提取变得极其精准：

对于机票系统 (Flight_Tool)：

Departure: 直接取 1_tempo_spatial_info.spatial_anchors.source_address -> 新加坡。
Arrival: 直接取 destination_address -> 北京。
出港时间界限: 读取 source_time.standardized（2026-02-28 14:00~18:00）。
到港时间界限: 读取 destination_time.standardized（最晚 2026-03-01 12:00）。
系统价值：以前如果只有单一的 "time"，遇到“明天飞，后天到”会产生混淆；现在有了 source 和 destination 的明确区分，机票 API 的请求参数（dep_time 和 arr_time）可以直接映射，不出错。
对于购物与 POI 推荐系统 (Search_Poi_Tool)：

生效范围: 读取 time_range（这周末）和 destination_address（北京）。这意味着系统不会去推荐新加坡的商场，也不会推荐工作日才开门的集市。
检索过滤条件: 将 shopping_index.sub_category (国潮) 与 3_memory_info 中的 favorite_brands (中国李宁、Randomevent等) 结合，传递给本地商场数据库进行交叉比对。
3. 记忆与异常处理层 (Memory & ExceptTask)
因为 Query 中出现了“按我以前的喜好买”，MemoryTask 被触发，不仅检索了出行偏好，还检索了购物偏好（尺码 L、偏好品牌）。这些信息落入 3_memory_info 后，Shopping DAG 在执行时就不需要发起 ClarifyNode（澄清交互节点）去询问用户买什么牌子，而是直接静默应用这些偏好。
prompt
模板


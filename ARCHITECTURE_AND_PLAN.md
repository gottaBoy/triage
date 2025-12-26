# L4 自动驾驶数据闭环与 Triage 系统架构设计

本设计文档基于行业标准的数据飞轮（Data Flywheel）理念，提供从车端数据采集到模型迭代更新的端到端架构视图。

## 1. 系统架构图 (System Architecture)

以下是基于 Mermaid 的数据流转架构图，展示了数据如何在车端、云端、分诊系统和模型训练之间闭环流动。

```mermaid
graph TD
    %% 1. 车端系统
    subgraph Vehicle [Onboard System: 车端基础设施]
        Sensors[传感器群: Lidar/Cam] --> Compute[计算平台]
        Compute -->|实时推理| Model_Run[模型运行]
        
        subgraph Monitor [监控与触发]
            Model_Run -->|状态流| Logger[实时日志 RingBuffer]
            Model_Run -->|异常检测| Trigger[Trigger 引擎]
            Trigger -->|触发信号| Clipper[数据截取 Clip]
        end
        
        Clipper -->|分级策略| Upload[上传管理器]
    end

    %% 2. 数据接入
    subgraph Cloud_Ingestion [Ingestion: 数据接入层]
        Upload -->|Level 0: 4G/5G| HotStore[热存储 (S3/OSS)]
        Upload -->|Level 1: Wi-Fi| WarmStore[温存储]
        Upload -->|Level 2: 硬盘| ColdStore[冷存储 (Archive)]
    end

    %% 3. 数据平台
    subgraph Data_Platform [Data Platform: 标签与索引]
        HotStore & WarmStore --> ETL[ETL 清洗]
        ETL -->|元数据提取| MetaDB[元数据库]
        ETL -->|特征向量化| VectorDB[向量数据库]
        MetaDB & VectorDB --> FastDM[FastDM: 秒级检索中枢]
    end

    %% 4. 智能分诊
    subgraph Triage_System [Triage: 智能分诊]
        Trigger -->|上报事件| AutoTriage[自动分诊服务]
        AutoTriage -->|去重 & 聚合| Cluster[问题聚类]
        Cluster -->|自动路由| IssueTracker[Jira/工单系统]
        IssueTracker -->|根因分析| RootCause[人工/自动分析]
    end

    %% 5. 闭环迭代
    subgraph Model_Loop [Closed Loop: 模型迭代]
        RootCause -->|定义长尾场景| Mining_Spec[挖掘需求]
        Mining_Spec -->|Query| FastDM
        FastDM -->|相似场景数据| Dataset[训练数据集]
        Dataset -->|标注| Labeling[自动/人工标注]
        Labeling -->|训练| Training[模型训练]
        Training -->|验证| Sim[仿真 & 评测]
    end

    %% 闭环回归
    Sim -->|OTA 更新| Vehicle
```

---

## 2. 核心实现方案 (Implementation Strategy)

### 2.1 核心原则
*   **Data-Driven**: 一切以数据为准绳，用数据定义问题，用数据验证效果。
*   **Automation First**: 尽可能减少人工介入，从发现问题到生成数据集全流程自动化。
*   **Tiered Management**: 针对不同价值的数据采用不同的存储和传输策略，平衡成本与效率。

### 2.2 关键模块设计

#### A. 车端智能采集 (Smart Logging)
*   **Ring Buffer**: 维护一个 30-60秒 的内存环形缓冲区，确保 Trigger 触发时能回溯过去的数据。
*   **Trigger Engine**: 支持动态下发 Lua/Python 规则脚本，无需发版即可更新采集策略（例如：临时抓取所有“路口急刹”数据）。

#### B. 云端数据中枢 (FastDM)
*   **多模态索引**: 结合结构化标签（时间、地点、速度）和非结构化特征（CLIP 模型提取的图像语义），实现“以图搜图”和复杂语义检索。
*   **数据血缘**: 记录每一帧数据用于了哪个版本的模型训练，确保可追溯。

#### C. 自动化分诊 (Auto-Triage)
*   **指纹算法**: 为每个 Issue 生成唯一的“指纹”（基于位置、错误类型、周边障碍物分布），用于去重。
*   **智能路由**: 基于历史处理记录，训练分类器将 Issue 自动分配给最合适的开发者。

---

## 3. 详细实施步骤 (Step-by-Step Guide)

### Phase 1: 基础设施搭建 (Month 1-3)
1.  **定义 Loss Function**: 确定 MPI (Miles Per Intervention) 计算口径。
2.  **车端日志**: 实现高性能 Logger，确保不丢帧、不阻塞主线程。
3.  **基础上传**: 打通 4G/5G 上传通道，实现 Level 0 (接管事件) 自动回传。
4.  **可视化**: 搭建 Web 端可视化工具，支持在线查看回传的 Bag/Log。

### Phase 2: 自动化分诊与检索 (Month 4-6)
1.  **Trigger 框架**: 部署车端规则引擎，实现常见场景（急刹、急转、规划超时）的自动捕获。
2.  **自动建单**: 对接 Jira API，实现 Trigger -> Jira 的自动化流程。
3.  **FastDM v1**: 建立基础的元数据索引（时间、地点、Trigger类型），支持 SQL 查询。

### Phase 3: 数据闭环与主动挖掘 (Month 7-9)
1.  **问题聚类**: 引入 NLP 和聚类算法，自动识别共性问题（如“特定红绿灯识别错误”）。
2.  **主动挖掘**: 基于聚类结果，在 FastDM 中挖掘相似的历史数据。
3.  **仿真闭环**: 打通“挖掘 -> 训练 -> 仿真”流水线，实现每日构建（Daily Build）的自动化评测。

### Phase 4: 智能化与大模型 (Month 10+)
1.  **VLM 标注**: 引入视觉大模型进行离线数据的语义打标。
2.  **端到端挖掘**: 利用大模型直接从原始视频中挖掘复杂长尾场景。
3.  **生成式仿真**: 利用 World Model 生成合成数据，补充训练集。

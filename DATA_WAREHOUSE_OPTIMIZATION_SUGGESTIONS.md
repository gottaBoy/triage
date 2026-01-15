# L4数据闭环数仓方案优化建议

> **基于L4自动驾驶数据闭环完整方案的数仓优化建议**
> 
> 本文档对比L4数据闭环方案与当前数仓方案，提出关键优化建议，确保数仓方案完整支持L4数据闭环的所有核心需求。

---

## 目录

1. [优化概览](#一优化概览)
2. [缺失的关键组件](#二缺失的关键组件)
3. [标签系统优化](#三标签系统优化)
4. [Trigger系统集成](#四trigger系统集成)
5. [CaseID映射系统](#五caseid映射系统)
6. [数据分级上传策略](#六数据分级上传策略)
7. [问题聚类与主动挖数](#七问题聚类与主动挖数)
8. [多层验证流程](#八多层验证流程)
9. [实施优先级](#九实施优先级)

---

## 一、优化概览

### 1.1 当前数仓方案覆盖情况

**已覆盖的核心能力**：
- ✅ 流批一体计算（Flink）
- ✅ 数据湖存储（Paimon）
- ✅ 流式存储（Fluss）
- ✅ OLAP查询（Doris）
- ✅ 对象存储（MinIO）
- ✅ 工作流调度（Airflow）
- ✅ Flink开发平台（Dinky）
- ✅ 基础数据分层（ODS/DWD/DWS/ADS）

**缺失的关键能力**：
- ❌ **标签系统（FreeDM → FastDM）**：秒级标签生成、标签竖表/宽表、ADB加速
- ❌ **Trigger系统**：三端统一Trigger框架、Trigger配置中心
- ❌ **CaseID映射系统**：Case逻辑映射、文件分片管理
- ❌ **数据分级上传**：Hot/Warm/Cold数据分级策略
- ❌ **问题聚类算法**：规则分桶 + Embedding聚类
- ❌ **主动挖数策略**：基于Trigger的主动数据挖掘
- ❌ **多层验证流程**：Model Eval → Log Replay → World Sim → Field Test

### 1.2 优化目标

1. **完整性**：补齐L4数据闭环所需的所有核心组件
2. **性能**：支持秒级标签查询、亚秒级实时分析
3. **自动化**：减少人工介入，提升自动化程度
4. **可扩展性**：支持未来业务扩展和技术演进

---

## 二、缺失的关键组件

### 2.1 标签系统（Label Hub）

**当前状态**：数仓方案中缺少完整的标签系统设计

**L4方案要求**：
- **FreeDM 1.0**：ODPS + Python UDF的批处理标签生成
- **秒级标签体系**：标签竖表（Tall Table）→ 标签宽表（Wide Table）
- **FastDM**：基于ADB的秒级检索引擎
- **标签注册中心**：标签元数据管理

**影响**：
- 无法实现秒级数据检索
- 无法支持FastDM的交互式数据挖掘
- 无法实现"每车每秒的特征空间"

### 2.2 Trigger系统

**当前状态**：数仓方案中缺少Trigger框架设计

**L4方案要求**：
- **三端统一Trigger**：车端/云端/仿真端统一规则
- **Trigger配置中心**：动态下发、版本管理
- **自动问题单生成**：Trigger事件自动生成Case

**影响**：
- 无法实现自动化问题发现
- 无法统一三端异常检测逻辑
- 无法实现"异常事件自动长成问题单"

### 2.3 CaseID映射系统

**当前状态**：数仓方案中缺少Case逻辑映射设计

**L4方案要求**：
- **Case逻辑映射**：Road Case → Bad Case → 文件分片
- **文件分片管理**：20秒窗口、物理文件去重
- **Case ↔ 文件映射表**：支持多Case复用同一文件

**影响**：
- 无法建立Case与数据文件的关联
- 无法实现文件去重和复用
- 无法统计数据使用率和成本

### 2.4 数据分级上传

**当前状态**：数仓方案中缺少数据分级上传策略

**L4方案要求**：
- **Hot Data**：4G/5G立即上传（碰撞、接管、急刹）
- **Warm Data**：Wi-Fi回场站上传（Trigger事件）
- **Cold Data**：物理硬盘回收（全量Raw数据）

**影响**：
- 无法优化数据上传成本
- 无法实现"智能行车记录仪"策略
- 无法平衡实时性和成本

---

## 三、标签系统优化

### 3.1 秒级标签体系设计

**目标**：构建"每车每秒的特征空间"，实现秒级数据检索

#### 3.1.1 标签竖表（Tall Table）设计

**表结构**：`dwd_tag_tall_table`

```sql
CREATE TABLE dwd_tag_tall_table (
    car_id STRING,
    ts_second BIGINT,
    tag_id STRING,
    tag_name STRING,
    tag_value_type STRING,  -- enum/number/bool/string/array/table
    tag_value STRING,       -- JSON格式存储
    tag_category STRING,    -- perception/prediction/pnc/scene/weather
    create_time TIMESTAMP(3),
    PRIMARY KEY (car_id, ts_second, tag_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/tag_tall',
    'bucket' = '8',
    'changelog-producer' = 'input'
);
```

**标签类型示例**：
- **数值型**：速度、加速度、距离、密度
- **枚举型**：天气、场景类型、任务类型
- **布尔型**：是否急刹、是否大转向、是否road_case窗口
- **Table型**：障碍物列表、车道线列表、地图要素列表

#### 3.1.2 标签宽表（Wide Table）设计

**表结构**：`dws_tag_wide_table`

```sql
CREATE TABLE dws_tag_wide_table (
    car_id STRING,
    ts_second BIGINT,
    -- 数值型标签（示例）
    tag_speed DOUBLE,
    tag_acceleration DOUBLE,
    tag_ped_density INT,
    -- 枚举型标签（示例）
    tag_weather STRING,
    tag_scene_type STRING,
    tag_task_type STRING,
    -- 布尔型标签（示例）
    tag_emergency_brake BOOLEAN,
    tag_large_turn BOOLEAN,
    tag_road_case BOOLEAN,
    -- 扩展表引用（Table型标签）
    tag_obstacles_table_id STRING,
    tag_lane_markings_table_id STRING,
    PRIMARY KEY (car_id, ts_second) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'fenodes' = 'doris-fe-service:8030',
    'table.identifier' = 'default_cluster.dws.dws_tag_wide',
    'username' = 'root',
    'password' = 'password'
);
```

**Pivot转换逻辑**（Flink SQL）：

```sql
-- 竖表转宽表（使用Flink SQL）
INSERT INTO dws_tag_wide_table
SELECT 
    car_id,
    ts_second,
    -- 数值型标签聚合
    MAX(CASE WHEN tag_name = 'speed' THEN CAST(tag_value AS DOUBLE) END) as tag_speed,
    MAX(CASE WHEN tag_name = 'acceleration' THEN CAST(tag_value AS DOUBLE) END) as tag_acceleration,
    MAX(CASE WHEN tag_name = 'ped_density' THEN CAST(tag_value AS INT) END) as tag_ped_density,
    -- 枚举型标签
    MAX(CASE WHEN tag_name = 'weather' THEN tag_value END) as tag_weather,
    MAX(CASE WHEN tag_name = 'scene_type' THEN tag_value END) as tag_scene_type,
    -- 布尔型标签
    MAX(CASE WHEN tag_name = 'emergency_brake' THEN CAST(tag_value AS BOOLEAN) END) as tag_emergency_brake,
    MAX(CASE WHEN tag_name = 'large_turn' THEN CAST(tag_value AS BOOLEAN) END) as tag_large_turn
FROM dwd_tag_tall_table
GROUP BY car_id, ts_second;
```

#### 3.1.3 标签注册中心设计

**表结构**：`dim_tag_registry`

```sql
CREATE TABLE dim_tag_registry (
    tag_id STRING,
    tag_name STRING,
    tag_value_type STRING,
    tag_category STRING,
    generation_method STRING,  -- cloud_sql_only/car_rule+cloud_rule/car_python+cloud_python
    data_source STRING,         -- SLS/microlog/map/business
    description STRING,
    create_time TIMESTAMP(3),
    update_time TIMESTAMP(3),
    PRIMARY KEY (tag_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dim.dim_tag_registry'
);
```

### 3.2 标签生成流程

#### 3.2.1 批处理标签生成（FreeDM 1.0）

**使用Flink批处理 + Python UDF**：

```python
# 标签生成UDF（Python）
@udf(input_types=[DataTypes.STRING()], result_type=DataTypes.ARRAY(DataTypes.STRING()))
def generate_tags(microlog_json):
    """从microlog JSON中提取秒级标签"""
    import json
    data = json.loads(microlog_json)
    
    tags = []
    # 急刹检测
    if data.get('deceleration', 0) > 3.0:
        tags.append({
            'tag_id': 'emergency_brake',
            'tag_name': 'emergency_brake',
            'tag_value': 'true',
            'tag_value_type': 'bool'
        })
    
    # 速度标签
    tags.append({
        'tag_id': 'speed',
        'tag_name': 'speed',
        'tag_value': str(data.get('speed', 0)),
        'tag_value_type': 'number'
    })
    
    return json.dumps(tags)
```

**Flink SQL作业**：

```sql
-- 从Paimon读取microlog数据
CREATE TABLE source_microlog (
    car_id STRING,
    ts_second BIGINT,
    microlog_json STRING
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/ods/microlog'
);

-- 生成标签
CREATE TABLE sink_tag_tall (
    car_id STRING,
    ts_second BIGINT,
    tag_id STRING,
    tag_name STRING,
    tag_value_type STRING,
    tag_value STRING
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/tag_tall'
);

INSERT INTO sink_tag_tall
SELECT 
    car_id,
    ts_second,
    tag_id,
    tag_name,
    tag_value_type,
    tag_value
FROM source_microlog,
LATERAL TABLE(generate_tags(microlog_json)) AS T(tag_id, tag_name, tag_value_type, tag_value);
```

#### 3.2.2 实时标签生成（基于Trigger）

**使用Flink流处理**：

```sql
-- 实时从SLS心跳数据生成标签
CREATE TABLE source_heartbeat (
    car_id STRING,
    ts_second BIGINT,
    speed DOUBLE,
    driving_mode STRING,
    road_type STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'heartbeat',
    'properties.bootstrap.servers' = 'kafka:9092'
);

-- 实时生成标签
INSERT INTO dwd_tag_tall_table
SELECT 
    car_id,
    ts_second,
    'speed' as tag_id,
    'speed' as tag_name,
    'number' as tag_value_type,
    CAST(speed AS STRING) as tag_value,
    'pnc' as tag_category,
    CURRENT_TIMESTAMP as create_time
FROM source_heartbeat;
```

### 3.3 FastDM检索引擎集成

#### 3.3.1 FastDM查询接口设计

**基于Doris宽表的秒级查询**：

```sql
-- FastDM查询示例：查找"雨天 + 园区 + 高行人密度 + 急刹"的场景
SELECT 
    car_id,
    ts_second,
    tag_speed,
    tag_acceleration
FROM dws_tag_wide_table
WHERE tag_weather = 'rain'
  AND tag_scene_type = '园区'
  AND tag_ped_density >= 3
  AND tag_emergency_brake = TRUE
ORDER BY ts_second;
```

#### 3.3.2 Session聚合

**Flink SQL实现Session聚合**：

```sql
-- Session聚合：将连续的秒聚合成Case
CREATE TABLE dws_tag_sessions (
    session_id STRING,
    car_id STRING,
    start_ts BIGINT,
    end_ts BIGINT,
    duration_seconds INT,
    tag_count INT,
    PRIMARY KEY (session_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dws.dws_tag_sessions'
);

-- Session聚合逻辑
INSERT INTO dws_tag_sessions
SELECT 
    CONCAT(car_id, '_', CAST(start_ts AS STRING)) as session_id,
    car_id,
    MIN(ts_second) as start_ts,
    MAX(ts_second) as end_ts,
    MAX(ts_second) - MIN(ts_second) as duration_seconds,
    COUNT(*) as tag_count
FROM dws_tag_wide_table
WHERE tag_road_case = TRUE
GROUP BY 
    car_id,
    TUMBLE(ts_second, INTERVAL '1' MINUTE)  -- 按时间窗口分组
HAVING duration_seconds >= 10;  -- 过滤短Session
```

### 3.4 实施步骤

1. **Phase 1：标签注册中心**（Week 1）
   - 创建标签注册表
   - 实现标签注册API
   - 初始化常用标签（50+）

2. **Phase 2：标签生成**（Week 2-3）
   - 实现批处理标签生成（FreeDM 1.0）
   - 实现实时标签生成
   - 标签竖表存储（Paimon）

3. **Phase 3：宽表构建**（Week 4）
   - 实现竖表转宽表（Flink SQL）
   - 同步到Doris
   - 性能优化（索引、分区）

4. **Phase 4：FastDM集成**（Week 5-6）
   - 实现FastDM查询接口
   - Session聚合逻辑
   - 查询性能优化（< 1分钟）

---

## 四、Trigger系统集成

### 4.1 三端统一Trigger框架

**目标**：实现车端/云端/仿真端统一的异常检测逻辑

#### 4.1.1 Trigger配置中心设计

**表结构**：`dim_trigger_config`

```sql
CREATE TABLE dim_trigger_config (
    trigger_id STRING,
    trigger_name STRING,
    trigger_type STRING,  -- car/cloud/sim
    trigger_category STRING,  -- emergency_brake/large_turn/parking/...
    rule_definition STRING,  -- JSON格式的规则定义
    python_code STRING,  -- Python代码（可选）
    version INT,
    status STRING,  -- active/inactive/deprecated
    create_time TIMESTAMP(3),
    update_time TIMESTAMP(3),
    PRIMARY KEY (trigger_id, version) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dim.dim_trigger_config'
);
```

**Trigger规则定义示例**（JSON）：

```json
{
  "trigger_id": "emergency_brake_v1",
  "trigger_name": "急刹车检测",
  "rule_type": "rule",  // rule/python
  "conditions": [
    {
      "field": "deceleration",
      "operator": ">",
      "value": 3.0,
      "duration_ms": 500
    },
    {
      "field": "speed",
      "operator": ">",
      "value": 10.0
    }
  ],
  "output_tags": ["emergency_brake", "hard_brake"]
}
```

#### 4.1.2 车端Trigger执行

**Flink流处理实现车端Trigger逻辑**：

```sql
-- 从Pose专线读取数据
CREATE TABLE source_pose (
    car_id STRING,
    ts_ms BIGINT,
    speed DOUBLE,
    acceleration DOUBLE,
    deceleration DOUBLE,
    steering_angle DOUBLE
) WITH (
    'connector' = 'kafka',
    'topic' = 'pose',
    'properties.bootstrap.servers' = 'kafka:9092'
);

-- Trigger事件输出表
CREATE TABLE sink_trigger_events (
    trigger_event_id STRING,
    car_id STRING,
    trigger_id STRING,
    trigger_name STRING,
    ts_ms BIGINT,
    trigger_value STRING,  -- JSON格式
    PRIMARY KEY (trigger_event_id) NOT ENFORCED
) WITH (
    'connector' = 'kafka',
    'topic' = 'trigger_events',
    'properties.bootstrap.servers' = 'kafka:9092'
);

-- 急刹车Trigger检测
INSERT INTO sink_trigger_events
SELECT 
    CONCAT(car_id, '_', CAST(ts_ms AS STRING), '_emergency_brake') as trigger_event_id,
    car_id,
    'emergency_brake_v1' as trigger_id,
    '急刹车检测' as trigger_name,
    ts_ms,
    JSON_OBJECT(
        'deceleration', CAST(deceleration AS STRING),
        'speed', CAST(speed AS STRING),
        'duration_ms', 500
    ) as trigger_value
FROM source_pose
WHERE deceleration > 3.0
  AND speed > 10.0;
```

#### 4.1.3 云端Trigger回刷

**使用Flink批处理回刷历史数据**：

```sql
-- 从Paimon读取历史Pose数据
CREATE TABLE source_pose_history (
    car_id STRING,
    ts_ms BIGINT,
    speed DOUBLE,
    deceleration DOUBLE
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/ods/pose',
    'scan.mode' = 'latest'  -- 或指定时间范围
);

-- 回刷Trigger事件
INSERT INTO sink_trigger_events
SELECT 
    CONCAT(car_id, '_', CAST(ts_ms AS STRING), '_emergency_brake') as trigger_event_id,
    car_id,
    'emergency_brake_v1' as trigger_id,
    '急刹车检测' as trigger_name,
    ts_ms,
    JSON_OBJECT(
        'deceleration', CAST(deceleration AS STRING),
        'speed', CAST(speed AS STRING)
    ) as trigger_value
FROM source_pose_history
WHERE deceleration > 3.0
  AND speed > 10.0;
```

#### 4.1.4 Trigger事件自动生成Case

**Flink SQL实现自动Case生成**：

```sql
-- Trigger事件表
CREATE TABLE dwd_trigger_events (
    trigger_event_id STRING,
    car_id STRING,
    trigger_id STRING,
    ts_ms BIGINT,
    trigger_value STRING,
    PRIMARY KEY (trigger_event_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/trigger_events'
);

-- Case表
CREATE TABLE dwd_cases (
    case_id STRING,
    car_id STRING,
    trigger_id STRING,
    start_ts_ms BIGINT,
    end_ts_ms BIGINT,
    case_type STRING,  -- road_case/bad_case
    status STRING,  -- pending/processing/resolved
    PRIMARY KEY (case_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/cases'
);

-- 自动生成Case（窗口聚合）
INSERT INTO dwd_cases
SELECT 
    CONCAT(car_id, '_', CAST(MIN(ts_ms) AS STRING)) as case_id,
    car_id,
    trigger_id,
    MIN(ts_ms) - 30000 as start_ts_ms,  -- 前30秒
    MAX(ts_ms) + 10000 as end_ts_ms,     -- 后10秒
    'road_case' as case_type,
    'pending' as status
FROM dwd_trigger_events
GROUP BY 
    car_id,
    trigger_id,
    TUMBLE(ts_ms, INTERVAL '1' MINUTE)  -- 1分钟窗口
HAVING COUNT(*) >= 1;  -- 至少1个Trigger事件
```

### 4.2 Trigger配置管理API

**使用Airflow DAG管理Trigger配置下发**：

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def update_trigger_config(**context):
    """更新Trigger配置并下发到车端"""
    # 1. 从Doris读取最新Trigger配置
    # 2. 验证配置有效性
    # 3. 下发到车端（通过Kafka或HTTP API）
    # 4. 记录下发日志
    pass

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'trigger_config_sync',
    default_args=default_args,
    description='同步Trigger配置到车端',
    schedule_interval=timedelta(hours=1),
    catchup=False
)

sync_task = PythonOperator(
    task_id='sync_trigger_config',
    python_callable=update_trigger_config,
    dag=dag
)
```

### 4.3 实施步骤

1. **Phase 1：Trigger配置中心**（Week 1-2）
   - 创建Trigger配置表
   - 实现配置管理API
   - 初始化常用Trigger规则（20+）

2. **Phase 2：云端Trigger**（Week 3-4）
   - 实现Flink流处理Trigger检测
   - 实现批处理回刷逻辑
   - Trigger事件存储（Paimon）

3. **Phase 3：自动Case生成**（Week 5）
   - 实现窗口聚合逻辑
   - Case自动创建流程
   - Case状态管理

4. **Phase 4：车端集成**（Week 6-8）
   - 车端Trigger SDK集成
   - 配置动态下发
   - 车端事件上报

---

## 五、CaseID映射系统

### 5.1 Case逻辑映射设计

**目标**：建立Case与物理文件分片的逻辑映射关系

#### 5.1.1 Case映射表设计

**表结构**：`dwd_case_file_mapping`

```sql
CREATE TABLE dwd_case_file_mapping (
    case_id STRING,
    chunk_id STRING,
    car_id STRING,
    file_id STRING,  -- 物理文件唯一ID
    file_path STRING,  -- OSS路径
    file_type STRING,  -- microlog/minilog/raw
    case_start_ts_ms BIGINT,  -- Case逻辑开始时间
    case_end_ts_ms BIGINT,    -- Case逻辑结束时间
    file_start_ts_ms BIGINT,   -- 文件物理开始时间
    file_end_ts_ms BIGINT,     -- 文件物理结束时间
    file_size BIGINT,  -- 文件大小（字节）
    priority INT,  -- 优先级
    upload_time TIMESTAMP(3),
    PRIMARY KEY (case_id, chunk_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/case_file_mapping',
    'bucket' = '8'
);
```

**文件索引表**：`dwd_file_index`

```sql
CREATE TABLE dwd_file_index (
    file_id STRING,
    car_id STRING,
    file_path STRING,
    file_type STRING,
    file_start_ts_ms BIGINT,
    file_end_ts_ms BIGINT,
    file_size BIGINT,
    upload_status STRING,  -- pending/uploading/completed/failed
    upload_time TIMESTAMP(3),
    PRIMARY KEY (file_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/file_index',
    'bucket' = '8'
);
```

#### 5.1.2 Case到文件映射逻辑

**Flink SQL实现文件分片和映射**：

```sql
-- 从Case表读取Case信息
CREATE TABLE source_cases (
    case_id STRING,
    car_id STRING,
    start_ts_ms BIGINT,
    end_ts_ms BIGINT,
    trigger_type STRING
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/cases'
);

-- 从文件索引表读取文件信息
CREATE TABLE source_files (
    file_id STRING,
    car_id STRING,
    file_path STRING,
    file_start_ts_ms BIGINT,
    file_end_ts_ms BIGINT,
    file_type STRING
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/file_index'
);

-- 生成Case到文件的映射（时间窗口匹配）
INSERT INTO dwd_case_file_mapping
SELECT 
    c.case_id,
    CONCAT(c.case_id, '_', f.file_id) as chunk_id,
    c.car_id,
    f.file_id,
    f.file_path,
    f.file_type,
    c.start_ts_ms as case_start_ts_ms,
    c.end_ts_ms as case_end_ts_ms,
    f.file_start_ts_ms,
    f.file_end_ts_ms,
    f.file_size,
    1 as priority,  -- 默认优先级
    CURRENT_TIMESTAMP as upload_time
FROM source_cases c
JOIN source_files f ON c.car_id = f.car_id
WHERE f.file_start_ts_ms <= c.start_ts_ms
  AND f.file_end_ts_ms >= c.end_ts_ms;  -- 文件时间窗完全覆盖Case时间窗
```

#### 5.1.3 文件去重和复用

**统计文件复用率**：

```sql
-- 文件复用统计表
CREATE TABLE dws_file_reuse_stats (
    file_id STRING,
    car_id STRING,
    file_path STRING,
    case_count BIGINT,  -- 被多少个Case引用
    total_size BIGINT,
    reuse_rate DOUBLE,  -- 复用率
    PRIMARY KEY (file_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dws.dws_file_reuse_stats'
);

-- 计算文件复用统计
INSERT INTO dws_file_reuse_stats
SELECT 
    file_id,
    car_id,
    file_path,
    COUNT(DISTINCT case_id) as case_count,
    MAX(file_size) as total_size,
    COUNT(DISTINCT case_id) * 1.0 / 1.0 as reuse_rate  -- 简化计算
FROM dwd_case_file_mapping
GROUP BY file_id, car_id, file_path;
```

### 5.2 Road Case → Bad Case映射

**表结构**：`dwd_case_hierarchy`

```sql
CREATE TABLE dwd_case_hierarchy (
    road_case_id STRING,
    bad_case_id STRING,
    car_id STRING,
    module STRING,  -- perception/prediction/pnc/localization/hardware
    root_cause STRING,  -- 根因分析结果
    confidence DOUBLE,  -- 置信度
    create_time TIMESTAMP(3),
    PRIMARY KEY (road_case_id, bad_case_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/case_hierarchy',
    'bucket' = '8'
);
```

### 5.3 实施步骤

1. **Phase 1：文件索引系统**（Week 1-2）
   - 创建文件索引表
   - 实现文件上传时自动索引
   - 文件去重逻辑

2. **Phase 2：Case映射逻辑**（Week 3-4）
   - 实现时间窗口匹配算法
   - Case到文件映射表
   - 文件复用统计

3. **Phase 3：Case层级管理**（Week 5）
   - Road Case → Bad Case映射
   - 根因分析集成
   - Case关联查询

---

## 六、数据分级上传策略

### 6.1 Hot/Warm/Cold数据分级

**目标**：优化数据上传成本，平衡实时性和成本

#### 6.1.1 数据分级策略表

**表结构**：`dim_data_upload_strategy`

```sql
CREATE TABLE dim_data_upload_strategy (
    strategy_id STRING,
    trigger_type STRING,
    data_priority STRING,  -- hot/warm/cold
    upload_channel STRING,  -- 4g/5g/wifi/physical
    data_types STRING,  -- JSON数组：["microlog", "minilog", "raw"]
    retention_days INT,
    compression_level STRING,  -- none/low/medium/high
    PRIMARY KEY (strategy_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dim.dim_data_upload_strategy'
);
```

**策略配置示例**：

| trigger_type | priority | upload_channel | data_types | retention_days |
|-------------|----------|----------------|------------|----------------|
| collision | hot | 4g/5g | ["microlog", "minilog", "raw"] | 365 |
| emergency_brake | hot | 4g/5g | ["microlog", "minilog"] | 180 |
| large_turn | warm | wifi | ["microlog", "minilog"] | 90 |
| parking | warm | wifi | ["microlog"] | 30 |
| normal | cold | physical | ["raw"] | 7 |

#### 6.1.2 上传管理器设计

**Flink流处理实现上传决策**：

```sql
-- Trigger事件表
CREATE TABLE source_trigger_events (
    trigger_event_id STRING,
    car_id STRING,
    trigger_type STRING,
    ts_ms BIGINT,
    location STRING  -- wifi/4g/5g
) WITH (
    'connector' = 'kafka',
    'topic' = 'trigger_events'
);

-- 上传策略表
CREATE TABLE source_upload_strategy (
    trigger_type STRING,
    data_priority STRING,
    upload_channel STRING,
    data_types STRING
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dim.dim_data_upload_strategy'
);

-- 上传任务表
CREATE TABLE sink_upload_tasks (
    upload_task_id STRING,
    car_id STRING,
    case_id STRING,
    trigger_type STRING,
    data_priority STRING,
    upload_channel STRING,
    data_types STRING,
    status STRING,  -- pending/uploading/completed/failed
    create_time TIMESTAMP(3),
    PRIMARY KEY (upload_task_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/upload_tasks'
);

-- 生成上传任务
INSERT INTO sink_upload_tasks
SELECT 
    CONCAT(car_id, '_', CAST(ts_ms AS STRING), '_', trigger_type) as upload_task_id,
    car_id,
    NULL as case_id,  -- Case ID稍后关联
    t.trigger_type,
    s.data_priority,
    CASE 
        WHEN s.upload_channel = '4g/5g' AND location IN ('4g', '5g') THEN '4g/5g'
        WHEN s.upload_channel = 'wifi' AND location = 'wifi' THEN 'wifi'
        ELSE 'physical'
    END as upload_channel,
    s.data_types,
    'pending' as status,
    CURRENT_TIMESTAMP as create_time
FROM source_trigger_events t
JOIN source_upload_strategy s ON t.trigger_type = s.trigger_type;
```

#### 6.1.3 Ring Buffer管理

**车端Ring Buffer设计**（概念设计）：

```python
# 车端Ring Buffer实现（伪代码）
class RingBuffer:
    def __init__(self, duration_seconds=60):
        self.buffer = []
        self.duration_seconds = duration_seconds
    
    def append(self, data, ts_ms):
        """添加数据到Ring Buffer"""
        self.buffer.append((ts_ms, data))
        # 清理过期数据
        current_ts = time.time() * 1000
        self.buffer = [
            (ts, d) for ts, d in self.buffer 
            if current_ts - ts < self.duration_seconds * 1000
        ]
    
    def extract_window(self, start_ts_ms, end_ts_ms):
        """提取时间窗口的数据"""
        return [
            data for ts, data in self.buffer
            if start_ts_ms <= ts <= end_ts_ms
        ]
```

### 6.2 上传流量管理

**统计表**：`dws_upload_statistics`

```sql
CREATE TABLE dws_upload_statistics (
    date_key DATE,
    trigger_type STRING,
    data_priority STRING,
    upload_channel STRING,
    case_count BIGINT,
    total_size_gb DOUBLE,
    avg_size_mb DOUBLE,
    PRIMARY KEY (date_key, trigger_type, data_priority) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dws.dws_upload_statistics'
);
```

### 6.3 实施步骤

1. **Phase 1：策略配置**（Week 1）
   - 创建上传策略表
   - 配置Hot/Warm/Cold策略
   - 策略管理API

2. **Phase 2：上传管理器**（Week 2-3）
   - 实现上传决策逻辑
   - 上传任务生成
   - 任务状态管理

3. **Phase 3：流量统计**（Week 4）
   - 上传流量统计
   - 成本分析
   - 策略优化建议

---

## 七、问题聚类与主动挖数

### 7.1 两阶段聚类算法

**目标**：从大量Case中识别典型问题场景

#### 7.1.1 规则分桶（第一阶段）

**使用FastDM进行规则分桶**：

```sql
-- 规则分桶：按现象和场景分桶
CREATE TABLE dws_case_buckets (
    bucket_id STRING,
    bucket_name STRING,
    trigger_type STRING,
    scene_tags STRING,  -- JSON数组
    case_count BIGINT,
    create_time TIMESTAMP(3),
    PRIMARY KEY (bucket_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dws.dws_case_buckets'
);

-- 示例：急刹车 + 高速干线
INSERT INTO dws_case_buckets
SELECT 
    'bucket_001' as bucket_id,
    '急刹车_高速干线' as bucket_name,
    'emergency_brake' as trigger_type,
    JSON_ARRAY('highway', 'dry') as scene_tags,
    COUNT(*) as case_count,
    CURRENT_TIMESTAMP as create_time
FROM dws_tag_wide_table
WHERE tag_emergency_brake = TRUE
  AND tag_scene_type = 'highway'
GROUP BY bucket_id, bucket_name, trigger_type, scene_tags;
```

#### 7.1.2 Embedding聚类（第二阶段）

**使用Flink + Python UDF进行Embedding聚类**：

```python
# Embedding聚类UDF（Python）
@udf(input_types=[DataTypes.STRING()], result_type=DataTypes.INT())
def cluster_cases(case_features_json):
    """基于特征向量进行聚类"""
    import json
    import numpy as np
    from sklearn.cluster import DBSCAN
    
    features = json.loads(case_features_json)
    feature_vector = np.array(features['vector'])
    
    # DBSCAN聚类
    clustering = DBSCAN(eps=0.5, min_samples=3)
    cluster_labels = clustering.fit_predict([feature_vector])
    
    return int(cluster_labels[0])
```

**特征提取**：

```sql
-- Case特征表
CREATE TABLE dws_case_features (
    case_id STRING,
    car_id STRING,
    feature_vector STRING,  -- JSON数组
    cluster_id INT,
    PRIMARY KEY (case_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dws/case_features'
);

-- 提取Case特征（从标签宽表）
INSERT INTO dws_case_features
SELECT 
    case_id,
    car_id,
    JSON_ARRAY(
        CAST(tag_speed AS STRING),
        CAST(tag_acceleration AS STRING),
        CAST(tag_ped_density AS STRING),
        -- ... 更多特征
    ) as feature_vector,
    NULL as cluster_id
FROM dws_tag_wide_table
JOIN dwd_cases ON car_id = car_id AND ts_second BETWEEN start_ts_ms/1000 AND end_ts_ms/1000;
```

### 7.2 场景Profile管理

**表结构**：`dim_scene_profile`

```sql
CREATE TABLE dim_scene_profile (
    profile_id STRING,
    profile_name STRING,
    cluster_id INT,
    tag_rules STRING,  -- JSON格式的标签规则
    trigger_template STRING,  -- Trigger模板
    data_types STRING,  -- JSON数组：所需数据类型
    priority INT,  -- 采数优先级
    target_samples INT,  -- 目标样本数
    current_samples INT,  -- 当前样本数
    PRIMARY KEY (profile_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dim.dim_scene_profile'
);
```

**场景Profile示例**：

```json
{
  "profile_id": "profile_001",
  "profile_name": "雨夜光斑误检",
  "tag_rules": {
    "tag_weather": "rain",
    "tag_time": "night",
    "tag_emergency_brake": true,
    "tag_perception_error": true
  },
  "trigger_template": "emergency_brake_v1",
  "data_types": ["microlog", "minilog"],
  "priority": 10,
  "target_samples": 200,
  "current_samples": 50
}
```

### 7.3 主动挖数策略

**采数优先级计算**：

```sql
-- 场景优先级计算表
CREATE TABLE dws_scene_priority (
    profile_id STRING,
    risk_weight DOUBLE,  -- 风险权重
    mpd_contribution DOUBLE,  -- MPD贡献
    current_samples INT,
    priority_score DOUBLE,  -- 优先级分数
    PRIMARY KEY (profile_id) NOT ENFORCED
) WITH (
    'connector' = 'doris',
    'table.identifier' = 'default_cluster.dws.dws_scene_priority'
);

-- 计算优先级
INSERT INTO dws_scene_priority
SELECT 
    profile_id,
    risk_weight,
    mpd_contribution,
    current_samples,
    (risk_weight * mpd_contribution) / GREATEST(current_samples, 1) as priority_score
FROM dim_scene_profile
JOIN (
    SELECT 
        profile_id,
        AVG(risk_score) as risk_weight,
        SUM(mpd_count) as mpd_contribution
    FROM dws_case_metrics
    GROUP BY profile_id
) metrics ON dim_scene_profile.profile_id = metrics.profile_id
ORDER BY priority_score DESC;
```

### 7.4 实施步骤

1. **Phase 1：规则分桶**（Week 1-2）
   - 实现FastDM规则分桶
   - 创建Case桶表
   - 桶统计和可视化

2. **Phase 2：Embedding聚类**（Week 3-4）
   - 特征提取逻辑
   - Embedding生成（图像/点云）
   - 聚类算法集成

3. **Phase 3：场景Profile**（Week 5）
   - Profile管理表
   - Profile配置API
   - Profile与Case关联

4. **Phase 4：主动挖数**（Week 6-7）
   - 优先级计算逻辑
   - 采数策略生成
   - 策略下发和执行

---

## 八、多层验证流程

### 8.1 验证流程设计

**目标**：确保问题修复的有效性，通过多层验证

#### 8.1.1 验证阶段表

**表结构**：`dwd_validation_pipeline`

```sql
CREATE TABLE dwd_validation_pipeline (
    validation_id STRING,
    case_id STRING,
    model_version STRING,
    stage STRING,  -- model_eval/log_replay/world_sim/field_test
    status STRING,  -- pending/running/passed/failed
    result STRING,  -- JSON格式结果
    start_time TIMESTAMP(3),
    end_time TIMESTAMP(3),
    PRIMARY KEY (validation_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/validation_pipeline',
    'bucket' = '8'
);
```

#### 8.1.2 Model Eval（模型评估）

**使用Airflow DAG调度模型评估**：

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def run_model_eval(**context):
    """运行模型评估"""
    case_id = context['dag_run'].conf.get('case_id')
    model_version = context['dag_run'].conf.get('model_version')
    
    # 1. 准备评估数据集
    # 2. 运行模型推理
    # 3. 计算评估指标（mAP、IoU等）
    # 4. 记录结果到validation_pipeline表
    pass

default_args = {
    'owner': 'ml-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
}

dag = DAG(
    'model_eval_pipeline',
    default_args=default_args,
    description='模型评估流程',
    schedule_interval=None,
    catchup=False
)

eval_task = PythonOperator(
    task_id='run_model_eval',
    python_callable=run_model_eval,
    dag=dag
)
```

#### 8.1.3 Log Replay（日志回放）

**Flink SQL实现日志回放**：

```sql
-- 日志回放任务表
CREATE TABLE dwd_log_replay_tasks (
    replay_id STRING,
    case_id STRING,
    model_version STRING,
    log_path STRING,
    status STRING,
    result STRING,
    PRIMARY KEY (replay_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/log_replay_tasks'
);

-- 从Case映射表获取日志路径
INSERT INTO dwd_log_replay_tasks
SELECT 
    CONCAT(case_id, '_replay_', model_version) as replay_id,
    case_id,
    model_version,
    file_path as log_path,
    'pending' as status,
    NULL as result
FROM dwd_case_file_mapping
WHERE file_type = 'microlog';
```

#### 8.1.4 World Sim（世界仿真）

**仿真任务管理**：

```sql
-- 仿真任务表
CREATE TABLE dwd_sim_tasks (
    sim_id STRING,
    case_id STRING,
    model_version STRING,
    scene_config STRING,  -- JSON格式场景配置
    status STRING,
    result STRING,
    PRIMARY KEY (sim_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/sim_tasks'
);
```

#### 8.1.5 Field Test（实车测试）

**实车测试任务表**：

```sql
-- 实车测试任务表
CREATE TABLE dwd_field_test_tasks (
    test_id STRING,
    case_id STRING,
    model_version STRING,
    car_id STRING,
    test_route STRING,
    status STRING,
    result STRING,
    PRIMARY KEY (test_id) NOT ENFORCED
) WITH (
    'connector' = 'paimon',
    'path' = 's3://minio-bucket/dwd/field_test_tasks'
);
```

### 8.2 验证流程编排

**使用Airflow实现验证流程编排**：

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'validation-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
}

dag = DAG(
    'multi_stage_validation',
    default_args=default_args,
    description='多层验证流程',
    schedule_interval=None,
    catchup=False
)

# Stage 1: Model Eval
model_eval = PythonOperator(
    task_id='model_eval',
    python_callable=run_model_eval,
    dag=dag
)

# Stage 2: Log Replay
log_replay = PythonOperator(
    task_id='log_replay',
    python_callable=run_log_replay,
    dag=dag
)

# Stage 3: World Sim
world_sim = PythonOperator(
    task_id='world_sim',
    python_callable=run_world_sim,
    dag=dag
)

# Stage 4: Field Test
field_test = PythonOperator(
    task_id='field_test',
    python_callable=run_field_test,
    dag=dag
)

# 定义依赖关系
model_eval >> log_replay >> world_sim >> field_test
```

### 8.3 实施步骤

1. **Phase 1：验证流程设计**（Week 1）
   - 创建验证流程表
   - 定义验证阶段
   - 状态管理逻辑

2. **Phase 2：Model Eval**（Week 2-3）
   - 模型评估集成
   - 评估指标计算
   - 结果记录

3. **Phase 3：Log Replay**（Week 4）
   - 日志回放系统集成
   - 回放任务管理
   - 结果对比

4. **Phase 4：World Sim**（Week 5-6）
   - 仿真平台集成
   - 场景配置管理
   - 仿真结果分析

5. **Phase 5：Field Test**（Week 7-8）
   - 实车测试任务管理
   - 测试结果收集
   - 验证报告生成

---


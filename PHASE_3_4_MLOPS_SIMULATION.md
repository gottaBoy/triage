# 阶段三与四实战指南：MLOps 自动化与仿真验证

本指南是数据闭环系列的最终章，涵盖了将标注好的数据通过版本控制（DVC），送入训练流水线（Kubeflow），最后通过仿真（CARLA）进行验证的完整流程。

---

## Part 1: 数据版本管理 (Data Version Control)

在导出 CVAT 数据集后，我们需要解决“模型与数据对应关系”的难题。

### 1. DVC 初始化
我们将使用 Git 来管理代码，用 DVC 来管理大文件。

```bash
# 安装 DVC
pip install dvc dvc-s3

# 初始化项目
git init
dvc init

# 配置 DVC 远端存储 (指向我们的 MinIO)
dvc remote add -d myminio s3://autonomous-data/dvc-store
dvc remote modify myminio endpointurl http://localhost:9000
dvc remote modify myminio access_key_id admin
dvc remote modify myminio secret_access_key password123
```

### 2. 将数据集纳入版本控制
假设从 CVAT 导出的数据集解压到了 `dataset/train` 目录。

```bash
# 1. 只有这一步会把大文件计算哈希
dvc add dataset/train

# 2. 上述命令会生成 dataset/train.dvc，这是一个很小的文本索引文件
# 我们把它提交到 Git
git add dataset/train.dvc .gitignore
git commit -m "Add new training data from CVAT task #101"

# 3. 将真实数据推送到 MinIO
dvc push
```

**此时的价值**：你的 Git 仓库非常轻量。任何同事想要复现这次训练，只需要 `git pull` 后运行 `dvc pull`，数据就会从 MinIO 自动下载下来。

---

## Part 2: 自动化训练流水线 (Kubeflow Pipeline)

当新数据 `dvc push` 之后，我们希望自动触发训练。这里展示一个基于 Kubeflow Pipeline (KFP) 的 Python 定义文件。

```python
# pipeline.py
import kfp
from kfp import dsl

def data_download_op():
    return dsl.ContainerOp(
        name='Pull Data',
        image='my-dvc-image:latest',
        command=['sh', '-c'],
        arguments=['dvc pull dataset/train'],
        file_outputs={'data': '/app/dataset/train'}
    )

def train_model_op(data_path):
    return dsl.ContainerOp(
        name='Train YOLO',
        image='my-training-image:latest',
        command=['python', 'train.py'],
        arguments=['--data', data_path, '--epochs', '100'],
        file_outputs={'model': '/app/runs/best.pt'} # 导出模型
    )

@dsl.pipeline(
    name='Auto-Drive Training Loop',
    description='Triggered when new labeled data is available'
)
def training_pipeline():
    # 步骤1：下载数据
    download = data_download_op()
    
    # 步骤2：训练 (依赖步骤1)
    train = train_model_op(download.outputs['data'])
    
    # 这里可以添加步骤3：评估
    # evaluate = eval_op(train.outputs['model'])

if __name__ == '__main__':
    kfp.compiler.Compiler().compile(training_pipeline, 'autodrive_pipeline.yaml')
```

**操作步骤**：
1.  运行 `python pipeline.py` 生成 YAML。
2.  上传 YAML 到 Kubeflow Dashboard。
3.  点击 Run，集群将自动拉起 Pod 进行训练。

---

## Part 3: 仿真验证 (Simulation with CARLA)

模型训练好后，不能直接上车。我们需要在 CARLA 中“复现”当时导致接管的场景，验证新模型是否能通过。

### 1. 核心概念：OpenSCENARIO
为了让仿真通用，我们使用 OpenSCENARIO XML 标准来描述场景。

**场景描述示例 (XML片段)**:
例如，我们在 Foxglove 中发现，车在“左转遇到行人”时失败了。我们在 XML 中定义这个逻辑：
```xml
<Story name="LeftTurnPedestrian">
    <Action>
        <!-- 设定本车路径 -->
        <PrivateAction>
            <RoutingAction>...</RoutingAction>
        </PrivateAction>
        <!-- 设定行人触发器：当车距离路口 10米 时，行人开始横穿 -->
        <EntityAction entityRef="Pedestrian1">
            <StartTrigger>
                <Condition name="DistanceToIntersection" value="10.0" />
            </StartTrigger>
        </EntityAction>
    </Action>
</Story>
```

### 2. 闭环验证流程

1.  **场景泛化**: 编写 Python 脚本，读取上述 XML，随机调整行人的速度 (1m/s ~ 3m/s) 和出现时机，生成 100 个变种 Case。
2.  **Headless 运行**: 在服务器上以无头模式启动 CARLA。
    ```bash
    ./CarlaUE4.sh -RenderOffScreen
    ```
3.  **加载模型**: 将 Kubeflow 产出的 `best.pt` 加载到 CARLA 的 Python Agent 中。
4.  **自动化评分**: 脚本自动统计 100 次测试中，发生碰撞的次数。
    *   如果碰撞率 < 1%，则视为通过。
    *   如果失败，重新将这些失败的 Case 录制下来（回到 Phase 1），进入下一个闭环迭代。

---

## 总结

至此，你已经拥有了一套完整的开源自动驾驶数据闭环方案：

1.  **Ingestion**: Apollo/ROS2 + MCAP
2.  **Storage**: MinIO
3.  **Triage**: Foxglove
4.  **Labeling**: CVAT
5.  **Versioning**: DVC
6.  **Training**: Kubeflow
7.  **Validation**: CARLA

这套架构虽然基础，但它涵盖了特斯拉、Waymo 等顶级车企闭环系统的所有核心逻辑要素。

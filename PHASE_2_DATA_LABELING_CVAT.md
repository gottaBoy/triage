# 阶段二实战指南：数据标注闭环 (Data Labeling Loop)

在完成了数据存储与查看后，下一步是将有价值的“Corner Case”数据提取出来，送入标注平台转化为真值（Ground Truth）。本指南将演示如何打通 **MinIO (存储)** 到 **CVAT (标注)** 的自动化链路。

## 1. 架构逻辑

在这个阶段，我们需要一个“中间件脚本”来完成以下繁琐工作：
1.  **Extract**: 从 MinIO 下载指定的 MCAP 数据包。
2.  **Process**: 解析 MCAP，提取出每一帧图像（JPG）或点云（PCD）。
3.  **Push**: 调用 CVAT API，自动创建标注任务（Task）并上传数据。

---

## 2. 部署 CVAT 标注平台

CVAT (Computer Vision Annotation Tool) 是业界最流行的开源标注工具。

### 2.1 快速启动
确保 docker 正确安装，直接使用官方脚本启动：

```bash
# 克隆官方仓库
git clone https://github.com/cvat-ai/cvat
cd cvat

# 启动 (这将下载约 2GB 镜像)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

等待启动完成后，访问 `http://localhost:8080`。
首次登录需注册一个超级管理员账号 (如: `admin` / `password123`)。

---

## 3. 编写 "MinIO-to-CVAT" 桥接服务

我们将编写一个 Python 脚本 `labeling_bridge.py`，模拟生产环境中的 ETL 作业。

### 3.1 安装依赖
需要安装 CVAT 的 Python SDK 以及图像处理库：
```bash
pip install cvat-sdk opencv-python-headless boto3 mcap-ros1-support mcap-ros2-support
```

### 3.2 脚本实现 (`labeling_bridge.py`)

此脚本将模拟以下场景：运营人员在 Foxglove 发现了一个异常数据包，将其文件名输入脚本，脚本自动完成后续所有准备工作。

```python
import os
import cv2
import boto3
import shutil
import numpy as np
from cvat_sdk import make_client
from cvat_sdk.core.proxies.tasks import ResourceType, Task
from mcap.reader import make_reader
from mcap_ros1.decoder import DecoderFactory # 假设是 ROS1 格式，ROS2 类推

# 配置信息
MINIO_CONF = {
    'endpoint': 'http://localhost:9000',
    'access_key': 'admin',
    'secret_key': 'password123',
    'bucket': 'autonomous-data'
}

CVAT_CONF = {
    'host': 'http://localhost:8080',
    'username': 'admin',
    'password': 'password123'
}

TEMP_DIR = "./temp_frames"

def download_from_minio(object_key, local_path):
    print(f"[1/4] Downloading {object_key} from MinIO...")
    s3 = boto3.client('s3', 
        endpoint_url=MINIO_CONF['endpoint'],
        aws_access_key_id=MINIO_CONF['access_key'],
        aws_secret_access_key=MINIO_CONF['secret_key']
    )
    s3.download_file(MINIO_CONF['bucket'], object_key, local_path)

def extract_images_from_mcap(mcap_path, output_dir):
    print(f"[2/4] Extracting frames from {mcap_path}...")
    if os.path.exists(output_dir):
        shutil.rmtree(output_dir)
    os.makedirs(output_dir)

    image_files = []
    
    # 打开 MCAP
    with open(mcap_path, "rb") as f:
        reader = make_reader(f, decoder_factories=[DecoderFactory()])
        count = 0
        for schema, channel, message, ros_msg in reader.iter_decoded_messages():
            # 假设我们要提取特定的 camera topic
            if "camera" in channel.topic or "image" in channel.topic:
                # 这是一个简化的图像解码逻辑，实际需根据 Encoding 处理 (bgr8, compressed 等)
                # 这里假设是 raw data 或者需要 cv2 解码
                # 示例代码仅做流程演示，具体解码需视 ros_msg 结构而定
                try:
                    # 模拟：如果没有真实图像数据，生成一张黑图代替，防止脚本报错
                    # 真实场景请使用 cv_bridge
                    img_name = f"frame_{count:04d}.jpg"
                    img_path = os.path.join(output_dir, img_name)
                    
                    # 生成一张带时间戳的测试图
                    blank_image = np.zeros((512,512,3), np.uint8)
                    cv2.putText(blank_image, str(message.log_time), (50, 250), 
                               cv2.FONT_HERSHEY_SIMPLEX, 1, (255, 255, 255), 2)
                    cv2.imwrite(img_path, blank_image)
                    
                    image_files.append(img_path)
                    count += 1
                    if count >= 10: break # 仅演示提取前10帧
                except Exception as e:
                    print(f"Error checking frame: {e}")
                    continue

    print(f"      Extracted {len(image_files)} frames.")
    return image_files

def create_cvat_task(task_name, file_paths):
    print(f"[3/4] Creating CVAT task: {task_name}...")
    
    with make_client(CVAT_CONF['host'], credentials=(CVAT_CONF['username'], CVAT_CONF['password'])) as client:
        # 1. 创建任务
        task_spec = {
            "name": task_name,
            "labels": [{"name": "car"}, {"name": "pedestrian"}, {"name": "traffic_light"}],
            "project_id": None, # 可以指定 Project ID
        }
        task = client.tasks.create_from_data(
            spec=task_spec,
            resource_type=ResourceType.LOCAL,
            resources=file_paths,
            data_params={"image_quality": 80}
        )
        
        print(f"[4/4] Task {task.id} created successfully!")
        print(f"      Go to: {CVAT_CONF['host']}/tasks/{task.id}")

if __name__ == "__main__":
    # 假设这是我们在 Phase 1 生成的文件名
    # 请将其替换为你实际上传到 MinIO 的文件名
    TARGET_FILE = "drive_session_example.mcap" 
    LOCAL_MCAP_PATH = "./temp.mcap"

    # 完整流程
    try:
        # Step 1: 下载
        # 注意：如果你的 MinIO 里还没有文件，请先运行 Phase 1 的 data_generator.py
        # download_from_minio(TARGET_FILE, LOCAL_MCAP_PATH) 
        
        # 为了演示方便，如果没下载到文件，我们跳过下载直接检查本地是否有测试文件
        # 实际使用请取消上面的注释
        if not os.path.exists(LOCAL_MCAP_PATH):
            print("Usage: 请先确保 MinIO 中有文件，或者手动放一个 mcap 文件到当前目录命名为 temp.mcap")
            # 这里创建一个假文件用于演示流程不中断
            with open(LOCAL_MCAP_PATH, 'w') as f: f.write("mock content")

        # Step 2: 提取 (这里为了演示，甚至不需要真实的 MCAP，extract 函数里有 mock 逻辑)
        frames = extract_images_from_mcap(LOCAL_MCAP_PATH, TEMP_DIR)

        # Step 3: 上传
        if frames:
            create_cvat_task(f"Review_Case_{TARGET_FILE}", frames)
        else:
            print("No frames extracted.")

    finally:
        # 清理临时文件
        if os.path.exists(TEMP_DIR): shutil.rmtree(TEMP_DIR)
        if os.path.exists(LOCAL_MCAP_PATH): os.remove(LOCAL_MCAP_PATH)
```

### 3.3 运行结果
运行脚本后，登录 CVAT 平台 (`http://localhost:8080`)，你将在 Tasks 列表中看到一个新的任务 `Review_Case_...`，点击进入即可开始标注。

---

## 4. 标注与导出

1.  **标注**: 在 CVAT 界面中使用矩形框工具（Review Interface）对生成的图片进行 "Car" 或 "Pedestrian" 的框选。
2.  **保存**: 点击 Save。
3.  **导出**: Menu -> Export task dataset -> 选择格式 **"COCO 1.0"** 或 **"YOLO 1.1"**。
    *   这将下载一个 zip 包，其中包含图片和生成的 JSON/TXT 标注文件。
    *   这就是我们训练模型所需的 **"Ground Truth"**。

---

至此，你已经打通了 **数据获取 -> 数据生产** 的关键路径。
在下一阶段（Phase 3 & 4），我们将讲解如何利用 **DVC** 管理这些导出的数据集版本，并将其送入 **Kubeflow** 训练。

# 阶段一实战指南：搭建数据底座 (Data Foundation)

本指南将带你从零开始搭建开源数据闭环的第一步：**存储与可视化**。完成本阶段后，你将拥有一个私有化的 S3 对象存储服务，并能通过 Web 浏览器直接查看车端回传的 MCAP 数据。

## 1. 环境准备

确保你的机器上安装了以下工具：
- [Docker & Docker Compose](https://www.docker.com/)
- Python 3.8+
- [Foxglove Studio](https://foxglove.dev/) (或者直接使用浏览器访问 https://studio.foxglove.dev)

---

## 2. 部署 MinIO 对象存储

我们将使用 MinIO 作为私有化的 "S3"，存储海量的自动驾驶日志。

### 2.1 创建 `docker-compose.yml`

在你的工作区创建一个新目录 `infrastructure`，并在其中创建 `docker-compose.yml`：

```yaml
version: '3'
services:
  minio:
    image: minio/minio
    container_name: minio-server
    ports:
      - "9000:9000" # API API
      - "9001:9001" # Web UI
    environment:
      MINIO_ROOT_USER: "admin"
      MINIO_ROOT_PASSWORD: "password123" # 请在生产环境修改此密码
    # 这一行命令确保启动 MinIO 并设置 Console 地址
    command: server /data --console-address ":9001"
    volumes:
      - ./minio_data:/data

  # 使用 minio client (mc) 自动设置 Bucket 和 CORS
  # 这一步至关重要，否则浏览器端的 Foxglove 无法跨域访问 MinIO 数据
  minio-init:
    image: minio/mc
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      until (mc alias set myminio http://minio:9000 admin password123); do echo 'Waiting for MinIO...'; sleep 5; done;
      mc mb myminio/autonomous-data;
      mc anonymous set download myminio/autonomous-data;
      echo 'Setting CORS for Foxglove...';
      mc admin config set myminio api cors_allow_origin='https://studio.foxglove.dev,http://localhost:8080' cors_allow_methods='GET,PUT,POST,HEAD';
      echo 'MinIO initialized successfully!';
      "
```

### 2.2 启动服务

在终端运行：
```bash
cd infrastructure
docker-compose up -d
```

访问 `http://localhost:9001`，使用 `admin` / `password123` 登录，你应该能看到名为 `autonomous-data` 的 Bucket。

---

## 3. 模拟车端：生成并上传 MCAP 数据

如果没有真实车辆，我们用 Python 脚本生成一个模拟的 MCAP 文件（包含随机生成的车辆位置和日志），并上传到 MinIO。

### 3.1 安装依赖

```bash
pip install mcap-protobuf boto3 protobuf
```
*(注：如果需要更丰富的 ROS2 消息，可安装 `mcap-ros2-support`)*

### 3.2 编写脚本 `data_generator.py`

创建一个脚本来模拟采集过程：

```python
import time
import math
import boto3
from io import BytesIO
from mcap.writer import Writer
from mcap.well_known import SMA_DIAGNOSTIC_LOG

# 1. 模拟车端录制数据
def generate_mock_mcap():
    stream = BytesIO()
    writer = Writer(stream)
    
    writer.start()
    
    # 注册一个简单的 Schema (这里直接用 JSON/Text 通道简化演示，实际会有 ROS/Protobuf schema)
    schema_id = writer.register_schema(
        name="log_message",
        encoding="jsonschema",
        data=b'{"type": "object", "properties": {"level": {"type": "string"}, "msg": {"type": "string"}}}'
    )
    
    channel_id = writer.register_channel(
        schema_id=schema_id,
        topic="vehicle_logs",
        message_encoding="json"
    )

    print("Generating mock driving data...")
    # 模拟 60秒 的数据
    for i in range(60):
        timestamp = time.time_ns()
        
        # 模拟正弦波轨迹数据 (这里简化为 Log，实际应写入 Pose 消息)
        x = i * 1.0
        y = math.sin(i * 0.1) * 10
        
        log_msg = f'{{"level": "INFO", "msg": "Vehicle at x={x:.2f}, y={y:.2f}"}}'.encode('utf-8')
        
        writer.add_message(
            channel_id=channel_id,
            log_time=timestamp,
            data=log_msg,
            publish_time=timestamp
        )
        
    writer.finish()
    stream.seek(0)
    return stream

# 2. 模拟车端回传数据至云端 (MinIO)
def upload_to_minio(data_stream, filename):
    s3_client = boto3.client(
        's3',
        endpoint_url='http://localhost:9000',
        aws_access_key_id='admin',
        aws_secret_access_key='password123',
        config=boto3.session.Config(signature_version='s3v4')
    )
    
    try:
        s3_client.upload_fileobj(
            data_stream, 
            'autonomous-data', 
            filename,
            ExtraArgs={'ContentType': 'application/octet-stream'}
        )
        print(f"Upload successful: {filename}")
        
        # 生成一个 1小时有效的预签名 URL
        url = s3_client.generate_presigned_url(
            'get_object',
            Params={'Bucket': 'autonomous-data', 'Key': filename},
            ExpiresIn=3600
        )
        return url
    except Exception as e:
        print(f"Upload failed: {e}")
        return None

if __name__ == "__main__":
    mcap_data = generate_mock_mcap()
    mcap_filename = f"drive_session_{int(time.time())}.mcap"
    
    print(f"Uploading {mcap_filename} to MinIO...")
    url = upload_to_minio(mcap_data, mcap_filename)
    
    if url:
        print("\n" + "="*50)
        print("Data Ingestion Complete!")
        print("Copy the following URL to Foxglove Studio:")
        print(url)
        print("="*50)
```

### 3.3 运行脚本

```bash
python data_generator.py
```
你将获得一个以 `http://localhost:9000/...` 开头的长链接。

---

## 4. 云端体验：使用 Foxglove 可视化

现在我们完成了闭环的前半部分（采集->存储），接下来验证“看数”环节。

1.  打开浏览器访问 [https://studio.foxglove.dev](https://studio.foxglove.dev)。
2.  在启动页选择 **"Open file from URL"** (从 URL 打开文件)。
3.  粘贴上一步脚本输出的 **MinIO Presigned URL**。
4.  点击 Load。
5.  在左侧面板打开 **"Raw Messages"** 面板，你应该能看到 topic `vehicle_logs` 下源源不断滚动的模拟日志数据。

### 为什么这一步很关键？
这证明了你的**数据闭环基础设施**已经具备了处理**远程流式回放**的能力。不需要下载几 GB 的 Bag 包，工程师就可以直接点击链接排查问题。这是从 "Demo" 走向 "Product" 的关键一步。

---

## 5. 下一步预告

完成本阶段后，你已经拥有了一个简易的数据湖。
**阶段二（下集）**我们将讲解如何把这里发现的“异常片段”，通过代码自动推送到 **CVAT** 进行标注。

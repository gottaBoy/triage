# triage

```
graph TD
    A[自动驾驶感知] --> B[目标检测]
    A --> C[语义分割]
    A --> D[3D检测]
    A --> E[多目标跟踪]
    
    B --> B1[YOLOv9] 
    B --> B2[DETR系列]
    B --> B3[YOLO-World]
    
    C --> C1[Segment Anything SAM]
    C --> C2[Mask2Former]
    C --> C3[BEVFormer-Seg]
    
    D --> D1[BEVDet]
    D --> D2[BEVFormer]
    D --> D3[PointPillars]
    
    E --> E1[ByteTrack]
    E --> E2[OC-SORT]
    E --> E3[StrongSORT]
```

- [【L4自动驾驶数据闭环实战01】最重要的第一步：选对整个组织的LossFunction]（https://zhuanlan.zhihu.com/p/1973693169792213913）
- [【L4自动驾驶数据闭环实战02】一级指标需要什么样的数据：L4 无人车的实时打点与业务心跳]（https://zhuanlan.zhihu.com/p/1974241571554731279）
- [【L4自动驾驶数据闭环实战03】自动驾驶数据闭环的“地基工程”：数据分级上传与Case逻辑映射设计]（https://zhuanlan.zhihu.com/p/1974632071449314206）
- [【L4自动驾驶数据闭环实战04】云端标签中枢：从 FreeDM 到 FastDM 的秒级特征空间]（https://zhuanlan.zhihu.com/p/1975018121740972539）
- [【【L4自动驾驶数据闭环实战05】三端统一 Trigger 框架：让异常事件自动“长成”问题单]（https://zhuanlan.zhihu.com/p/1975401888179586737）
- [【L4自动驾驶数据闭环实战06】典型问题场景闭环：从问题聚类到主动挖数、训练与多层验证]（https://zhuanlan.zhihu.com/p/1975648416945157285）
- [【L4自动驾驶数据闭环实战·总结与展望】模型 × 数据：面向物理 AI 时代的数据基础设施]（https://zhuanlan.zhihu.com/p/1975918120725136170）
- [自动驾驶数据闭环涉及哪些技术栈？](https://www.zhihu.com/question/581571930/answer/1975546392517812248)
- [目前各家做的自动驾驶数据闭环平台真的闭环了吗](https://www.zhihu.com/question/552466858/answer/1973504909879030493)
- [最全自动驾驶数据集分享系列八 | 仿真数据集](https://zhuanlan.zhihu.com/p/565383336)
- [六大数据集全部SOTA！最新DriveMM：自动驾驶一体化多模态大模型](https://zhuanlan.zhihu.com/p/13640546831)

- [Pilot Roadmap - 面向L3+自动驾驶的功能安全架构方案](https://zhuanlan.zhihu.com/p/79624602)
- [介绍一篇自动驾驶的论文：AutoLay，非模态道路布局估计的数据集和基准](https://zhuanlan.zhihu.com/p/404652486)
- [六大数据集全部SOTA！最新DriveMM：自动驾驶一体化多模态大模型（美团&中山大学）](https://zhuanlan.zhihu.com/p/13640546831)
- [自动驾驶轨迹预测的大型基础模型：综述(上)](https://zhuanlan.zhihu.com/p/1957053326504993801)
- [自动驾驶轨迹预测算法：NeurIPS挑战赛冠军方案](https://zhuanlan.zhihu.com/p/348757935)
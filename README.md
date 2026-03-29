# U-Net Interference Localization

基于 U-Net 的单干扰源角度定位项目，面向 `8×8` 单元相控阵场景。

本项目建立在一个自研启发式定位算法之上。该启发式方法无需监督学习，在单干扰源定位任务中已经具备较好的效果；U-Net 模块用于在现有定位链路上进一步提升定位精度与稳定性。

## 项目亮点

- 应用场景：`8×8` 单元相控阵、单个干扰源角度定位
- 方法结构：自研启发式定位方法 + U-Net 增强模块
- 输入特征：SINR 图、精英邻居分数图、底部邻居分数图
- 输出结果：干扰源位置概率热力图，峰值点对应最终角度估计
- 任务形式：将角度定位问题重写为二维热力图预测任务

## 当前结果

在当前实验设置下：

- 自研启发式定位方法可作为无需监督学习的基础定位方案
- 引入 U-Net 后，单个干扰源定位任务中误差小于 `7.5°` 的准确度可提升到 `98%+`

说明：

- 上述结果基于当前仿真流程与实验配置
- 结果会受到样本分布、网格分辨率、损失函数和训练数据规模影响
- 主分支默认不包含大体积训练权重文件

## 方法概览

项目中的基础定位流程会综合搜索结果中的多种信息进行分析，重点包括：

- `SINR`
- 精英邻居分数
- 底部邻居分数

U-Net 模块使用同样的三类核心特征，但不再依赖固定的人工组合方式，而是直接从数据中学习更复杂的空间模式与非线性关系，从而进一步提升定位准确度。

### 输入

模型输入为形状 `(3, H, W)` 的三通道特征图：

1. `SINR map`
2. `Elite-neighbor score map`
3. `Bottom-neighbor score map`

### 输出

模型输出为形状 `(1, H, W)` 的概率热力图：

- 热力图峰值位置对应预测的干扰源角度
- 监督标签采用二维高斯分布，而不是单点标签

### 为什么选择 U-Net

- 能够保留空间位置信息，适合“角度网格 -> 热力图”任务
- Skip Connection 对小目标定位更友好
- 在中小规模数据集上较容易训练出稳定结果

## 仓库内容

```text
unet_jammer_localization/
├── data_generation.py      # 数据生成与特征构建
├── dataset.py              # PyTorch 数据集与 DataLoader
├── unet_model.py           # U-Net / U-Net Small 模型定义
├── train.py                # 训练入口
├── inference.py            # 推理与可视化
├── example_usage.py        # 从生成数据到推理的完整示例
├── QUICKSTART.md           # 快速上手说明
├── PROJECT_SUMMARY.md      # 项目总结
├── TROUBLESHOOTING.md      # 常见问题排查
├── FIXES_APPLIED.md        # 修复记录
└── weights说明.md          # Release 权重说明
```

## 依赖环境

- Python 3.7+
- PyTorch 1.8+
- NumPy / SciPy / Pandas / Matplotlib
- TensorBoard（可选，建议安装）

安装方式：

```bash
pip install -r requirements.txt
```

如需 GPU 版本 PyTorch，可根据 CUDA 版本从 [PyTorch 官网](https://pytorch.org/) 安装对应包。

## 运行说明

当前公开仓库中的训练与推理流程依赖配套仿真代码，尤其是父目录中的 `accuracy6_Bottom3.py`。

以下模块直接依赖该文件：

- `data_generation.py`
- `inference.py`
- `example_usage.py`

因此，仓库公开的是 U-Net 定位模块及其与现有仿真流程的集成代码；完整运行和复现需要配套仿真环境。

## 快速开始

### 1. 生成数据

```bash
python data_generation.py
```

默认示例配置会：

- 构建 `64 × 64` 角度网格
- 生成三通道输入特征
- 为每个样本生成二维高斯标签

脚本中的默认示例参数包括：

- `theta_range=(-60, 60)`
- `phi_range=(-90, 90)`
- `jammer_theta_range=(27, 52)`
- `jammer_phi_range=(-90, 90)`
- `gaussian_sigma=1.5`

生成后的数据默认写入父目录下的 `unet_dataset/`。

### 2. 训练模型

```bash
python train.py
```

当前默认训练配置位于 [train.py](train.py) 中，核心参数包括：

- 数据目录：`../unet_dataset`
- 模型：`unet`
- 批大小：`8`
- 训练轮数：`100`
- 学习率：`1e-3`
- 损失函数：`combined`，即 Dice Loss + Focal Loss

训练输出默认保存在：

- `./checkpoints`
- `./runs`

训练结束后通常会得到：

- `best_model.pth`
- `latest_checkpoint.pth`
- `training_history.png`

### 3. 推理预测

```bash
python inference.py
```

推理阶段会：

1. 根据仿真搜索结果生成与训练一致的三通道特征图
2. 使用训练好的 U-Net 输出概率热力图
3. 读取热力图峰值并映射回角度坐标

核心预测接口是 [inference.py](inference.py) 中的 `JammerLocalizationPredictor`。

### 4. 运行完整示例

```bash
python example_usage.py
```

脚本内容包括：

- 小规模数据生成
- 样本可视化
- 模型结构测试
- 快速训练
- 预测示例

## 训练设计

### 标签设计

- 使用二维高斯热力图作为监督信号
- 比单点标签更平滑
- 对网格离散误差更鲁棒

### 损失函数

默认采用 `CombinedLoss`：

- Dice Loss：优化预测区域与真实区域的重叠
- Focal Loss：缓解类别不平衡，强化难样本学习

这套组合尤其适合“小目标 + 大背景”的定位问题。

### 模型版本

当前提供两个模型：

- `UNet`
- `UNetSmall`

其中标准 `UNet` 是默认版本。

## 仓库中未包含的内容

为避免仓库体积过大以及 GitHub 大文件限制，以下内容默认不纳入主分支版本库：

- 训练得到的 `.pth` 权重文件
- TensorBoard 日志
- 测试预测图片
- `__pycache__`
- 动态缓存文件

需要公开分发模型文件时，更适合使用 GitHub Releases 或 Git LFS。

## 相关文档

- [QUICKSTART.md](QUICKSTART.md)
- [PROJECT_SUMMARY.md](PROJECT_SUMMARY.md)
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- [FIXES_APPLIED.md](FIXES_APPLIED.md)
- [weights说明.md](weights说明.md)

## License

仓库当前未附带单独的 `LICENSE` 文件。代码复用与分发前应补充相应许可证信息。

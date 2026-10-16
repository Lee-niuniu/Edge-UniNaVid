# uninavid_mini
# Edge-UniNaVid: 面向 Orin NX 的 20Hz 实时视觉语言跟拍与导航端侧部署方案

本项目基于开源视觉语言导航模型 **Uni-NaVid**（EVA-CLIP + Vicuna-7B），针对真实场景下的目标跟踪（Tracking）任务进行了**知识蒸馏轻量化**与 **TensorRT-LLM 端侧极致推理优化**。最终将 14GB 的原始模型压缩至 **1.4GB**，在算力受限的 **NVIDIA Jetson Orin NX（100 TOPS / 16GB）** 平台上实现了 **20Hz** 的实时动作输出频率，超越了车载级计算平台 Thor（2070 TOPS）上的原 8Hz 表现。

---

## 🚀 核心优化路线（OOM $\rightarrow$ 1Hz $\rightarrow$ 8Hz $\rightarrow$ 20Hz）

### 1. 模型量化与内存优化（解决 OOM $\rightarrow$ 1Hz）
* **低比特量化压缩**：在云端对 14GB 原始大模型进行 SmoothQuant（INT8，7GB）与 AWQ（INT4，4GB）量化，在大幅缩减模型体积的同时将精度损失控制在 1% 以内。
* **引擎构建峰值内存优化**：针对 `trtllm-build` 构建引擎时内存峰值超过 14GB 导致 Orin NX 统一内存崩溃的问题，通过挂载 4GB Swap 交换空间（以硬盘换内存）顺利完成端侧引擎编译。
* **C++ 零拷贝加载**：解决 Python `ModelRunner` 中 `f.read()` 将引擎文件读为常驻内存字节对象，导致“读取引擎 + 反序列化 + 运行时开销”三重叠加的 OOM 问题；改用 **C++ 运行时配合 `mmap`** 内存映射机制实现高效加载。

### 2. 算子编译与多模态注入优化（1Hz $\rightarrow$ 8Hz）
* **SM87 定制算子编译**：针对 TensorRT-LLM 在端侧缺少对应 INT4 GPU GEMM 矩阵算子的问题，基于官方 CUTLASS 模板，指定 Orin NX 对应的 CUDA 架构参数 **SM87** 重新编译定制 GPU Kernel。
* **视觉特征虚拟 Token 注入**：针对编译后的推理 Engine 不直接接受视觉特征 Embedding 输入的限制，利用 `prompt_table` 作为预留表，将视觉 Embedding 序列通过虚拟 Token ID 动态注入推理引擎。

### 3. 知识蒸馏与网络裁剪（8Hz $\rightarrow$ 20Hz）
* **结构裁剪与双重蒸馏**：对原始 32 层 Transformer 教师网络（14GB）进行网络层裁剪，构建轻量级 Student 网络（4GB）。训练时结合**遥操作专家数据**与**教师网络泛化输出**进行双重指导蒸馏，兼顾特定跟踪任务的准确性与通用泛化能力。
* **极致端侧加速**：将蒸馏后的学生网络结合 INT4 量化与端侧 Engine 编译，最终生成仅 **1.4GB** 的超轻量推理引擎，配合图像预处理与动作推理多线程解耦及 ROS 底盘控制适配，在 Orin NX 小车上实现近 **20Hz** 的高频实时跟拍控制。

---

## 📊 端侧部署性能对比

| 硬件平台 / 优化阶段 | 算力与内存规格 | 模型 / 引擎大小 | 精度损失 | 推理帧率 | 运行状态 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **NVIDIA Thor** (FP16 原始模型) | 2070 TOPS / 128GB | 14 GB | 0% | ~8 Hz | 车载级大算力平台基线 |
| **Jetson Orin NX** (FP16 原始模型) | 100 TOPS / 16GB | 14 GB | - | OOM | 内存溢出无法运行 |
| **Jetson Orin NX** (INT8 + C++ `mmap`) | 100 TOPS / 16GB (+4GB Swap) | 7 GB | ~1% | ~1 Hz | 解决编译与加载 OOM |
| **Jetson Orin NX** (INT4 + `SM87` + `prompt_table`) | 100 TOPS / 16GB | 4 GB | ~1% | ~8 Hz | 完成算子与多模态输入适配 |
| **Jetson Orin NX** (**蒸馏裁剪 + INT4 Engine**) | 100 TOPS / 16GB | **1.4 GB** | 极低 | **~20 Hz** | **实时高频输出，超越 Thor 基线** |

---

## 🛠️ 部署流程简述

1. **蒸馏与量化导出**：在云端对 32 层教师网络进行裁剪蒸馏得到 4GB 学生网络，并导出 INT4 AWQ 量化 Checkpoint。
2. **环境与 Swap 配置**：在 Orin NX（16GB）上配置基础环境并挂载 4GB Swap 分区。
3. **编译算子与引擎**：指定 CUDA 架构 `SM87` 编译 CUTLASS GEMM 算子，使用 `trtllm-build` 将量化权重编译为 1.4GB 的 TensorRT-LLM 推理 Engine。
4. **启动真机推理**：运行基于 C++ `mmap` 的推理节点与多线程 ROS 运动解算节点，通过 `prompt_table` 注入实时视频流特征，输出 20Hz 离散控制动作驱动底盘运动。

---

## 🙏 致谢
本项目基于 [Uni-NaVid](https://github.com/jzhzhang/Uni-NaVid) 与 [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) 开展端侧优化与实机部署

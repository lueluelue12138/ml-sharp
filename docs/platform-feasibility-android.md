# 平台与安卓可行性报告

## 1. 项目现状（基于代码阅读）
- 推理核心：`src/sharp/cli/predict.py` 直接使用 PyTorch 2.8 (`torch`, `torchvision`) 进行单张图片推理，输出 3D Gaussian 参数并存为 `.ply`。没有自定义算子或训练逻辑。
- 设备选择：CLI 默认优先 `cuda`，其次 `mps`，否则回退 `cpu`。推理可在 CPU 完成，但速度会明显下降。
- 渲染：`--render` 依赖 `gsplat`（CUDA），只在支持 CUDA 的 GPU 上启用；不影响生成 `.ply`。
- 依赖：纯 Python 依赖（`plyfile`, `numpy`, `scipy`, `timm` 等）+ 可选 CUDA 组件随 PyTorch 预编译轮子提供。

## 2. 桌面端/服务器端可移植性
- **Linux x86\_64（有/无 CUDA）**：PyTorch 官方提供 CPU 与 CUDA 轮子，当前代码可直接运行；无 CUDA 时只能生成 `.ply`，渲染需关闭 `--render`。
- **Windows x86\_64（有/无 CUDA）**：同样有官方轮子，命令行可工作；如需渲染需安装匹配的 CUDA 版 PyTorch/`gsplat`。
- **macOS Intel**：仅 CPU 路径可用（无 MPS），性能较慢但功能可用。
- **Apple Silicon**：已有 MPS 支持，已在默认设备选择中覆盖。
- **结论**：推理部分与 Apple 平台无强绑定，主要瓶颈在硬件算力和是否需要 CUDA 渲染。

## 3. Web 形态可行性
- 纯 WebAssembly/WebGPU 直接跑模型不可行或成本极高（PyTorch/`timm` 体积大、算子支持不足，显存与编译链复杂）。
- 可行方案：
  - **前端上传 + 后端推理**：前端仅做文件上传与结果下载/预览，后端复用当前 Python 代码（GPU/CPU），最小改动。
  - **前端预览**：得到 `.ply` 后，用 WebGL/WebGPU（如 three.js + PLY loader）渲染点云/高斯；推理仍放在服务端。
- 结论：Web 前端可以做可视化与任务编排，但推理应放在服务器/桌面节点。

## 4. 安卓端离线方案（导入图片 → 生成/预览 `.ply`）
目标：不依赖云端，尽量本地推理并生成/预览 `.ply`。

### 4.1 可行性依据
- 模型推理只用标准 PyTorch 运算，可导出为 **TorchScript** 或 **ExecuTorch**（适配移动端 Vulkan/NNAPI）。
- 预处理/后处理（resize、内参构造、`unproject_gaussians`、`save_ply`）均由基础张量运算与文件写入构成，可在移动端复刻。
- 渲染不依赖推理，可用安卓侧点云/高斯渲染器替代 `gsplat`。

### 4.2 推荐实现路径
1. **模型导出**：在桌面端用示例输入将 `RGBGaussianPredictor` 导出为 TorchScript（或 ExecuTorch）并固化权重，放入 APP 资源或首次启动下载到本地缓存。
2. **推理封装**：使用 PyTorch Mobile/ExecuTorch C++/JNI 接口在安卓端执行推理；输入是 RGB 图像 + 焦距，输出高斯参数张量。
3. **后处理与存储**：在 JNI 层实现简化版 `.ply` 写入（结构与 `save_ply` 保持一致），避免依赖 Python 解释器。
4. **预览渲染**：
   - 简单方案：仅导出 `.ply`，用现成点云/网格查看器（Sceneform/Filament 自定义点渲染）展示。
   - 进阶方案：移植轻量级高斯/点云渲染器到 Vulkan/OpenGL ES，在 C++ 层把高斯转换为可渲染顶点/纹理。
5. **性能取舍**：CPU-only 可能需要数十秒级；Vulkan/NNAPI 后端可明显加速，但需针对目标机型做基准测试与模型裁剪（如降分辨率、量化）。

### 4.3 替代/更简单路径
- **混合模式**：安卓端负责采集/预览，推理委托给本地局域网或云端服务，结果 `.ply` 下发后本地渲染。
- **仅生成不渲染**：在资源受限设备上只生成 `.ply`（或仅在服务器生成），安卓端只做查看。

## 5. 推荐的最小改造方案
1. 先在桌面（Linux/Windows）验证 TorchScript/ExecuTorch 导出与推理等价性。
2. 在安卓用 PyTorch Mobile/ExecuTorch 跑一次端到端（不含渲染），确认时延与内存占用。
3. 选择渲染路径（简单点云 vs 自研高斯渲染器），完成 `.ply` 预览。
4. 如需 Web 入口，构建“前端上传 + 后端推理 + 前端 PLY 预览”的模式。

## 6. 结论
- 生成 `.ply` 的推理链路与平台无强耦合，桌面/服务器端可直接运行；渲染仅受 CUDA 约束。
- 安卓离线版在技术上可行，但需移动端推理框架（TorchScript/ExecuTorch）、JNI 后处理与独立渲染器，性能取决于 SoC GPU/NNAPI 能力。
- 纯 Web 端不适合直接本地推理；前端可承担可视化，推理应放在后端或本地原生组件。

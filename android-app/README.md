# Android 离线版应用技术方案（高通旗舰，GPU 优先）

## 1. 目标与范围
- 离线运行：不依赖云端，端上完成「导入图片 → 推理生成 3D Gaussian → 输出 `.ply` → 本地高质量预览」。
- 主要适配：高通旗舰 SoC（Adreno 7xx 系列），优先利用 GPU/NN 加速。
- UI 风格：遵循 Apple 简洁圆角、浅色留白、层次分明的设计语言，可用 Compose + 自定义 Cupertino 风格组件。
- 首版功能：单图导入、推理、保存 `.ply`、可视化预览（点云/高斯 splat）。

## 2. 技术选型
### 2.1 推理框架
- **首选：ExecuTorch (PyTorch Mobile 下一代)**  
  - 优点：支持 Vulkan/NNAPI 后端，体积更小，可静态链接；适合离线推理与 AOT 模型。
  - 模型准备：在桌面用 `RGBGaussianPredictor` 导出 ExecuTorch 格式（或 TorchScript 作为备选）。
- **备选：PyTorch Mobile (Lite Interpreter + Vulkan)**  
  - 简化迁移成本；若 ExecuTorch 适配阻塞，可先用 Vulkan 后端跑通。

### 2.2 渲染与预览
- **首选：Filament + 自定义点/高斯渲染器（OpenGL ES 3.1+ / Vulkan）**  
  - 高质量 PBR 管线，可自定义材质；用 instancing 或 shader 近似高斯 splatting（对 Adreno 友好）。
- **备选：基于 OpenGL ES 的轻量点云渲染**  
  - 先用点精灵/屏幕空间圆片实现快速预览，后续迭代高斯核。
- 数据管线：在 JNI 层将高斯/点数据转换为 GPU buffer（positions、colors、scale/opacity）。

### 2.3 后处理与存储
- JNI/C++ 实现与 `sharp.utils.gaussians.save_ply` 对齐的 `.ply` 导出，避免 Python 依赖。
- 预处理（resize、内参矩阵、disparity_factor）与 `predict_image` 对齐，放在 C++/Kotlin 层。

### 2.4 UI 框架
- Jetpack Compose + Material3 基础，定制 Cupertino 风格（大圆角卡片、毛玻璃/阴影、浅色分隔、San Francisco 字体或替代）。
- 页面分层：输入页（选择/拍照）→ 推理状态页 → 结果页（`.ply` 保存、预览、分享）。

## 3. 开发与构建环境
- IDE：Android Studio Iguana+（最新稳定），Gradle 8.7+。
- SDK/NDK：Android SDK 34，NDK r26d，CMake 3.22+。
- ABI：`arm64-v8a`（主支持），可选 `x86_64` 仅供模拟器调试（无性能保证）。
- 设备要求：Adreno 7xx、内存 ≥ 8 GB，系统 Android 13+，开启 Vulkan 支持。

### 环境搭建步骤
1) 安装 Android Studio，勾选 SDK 34、NDK r26d、CMake。  
2) 拉取本仓库，切到 `android-app/` 子目录作为应用根。  
3) 下载并放置模型（ExecuTorch/TorchScript）到 `android-app/app/src/main/assets/models/sharp_execu.pt`（文件名可自定义）。  
4) 在 `gradle.properties` 启用 `android.defaults.buildfeatures.buildconfig=true`，配置 `minSdk=26`、`targetSdk=34`。  
5) 在 `app/build.gradle` 引入：  
   - ExecuTorch 或 PyTorch Mobile AAR（CPU + Vulkan）；  
   - Filament/GLTF 依赖（`com.google.android.filament:filament-android` 等）；  
   - Compose BOM、Accompanist（模糊、Insets）。  
6) 打开开发者选项，启用 USB 调试；使用真机（旗舰 SoC）做性能测试。

## 4. 模块分层与数据流
1) **输入层（Kotlin/Compose）**：图库/相机选图 → content URI。  
2) **预处理（JNI/C++）**：加载 JPEG/PNG → 转 RGB tensor（float, CHW, 0-1）→ resize(1536x1536) → 构造内参、`disparity_factor`。  
3) **推理（ExecuTorch/PyTorch Mobile）**：调用 `RGBGaussianPredictor` 导出的模型，输出 NDC 高斯参数。  
4) **后处理（JNI/C++）**：`unproject_gaussians` + `.ply` 写入（兼容桌面输出）。  
5) **渲染（Filament/OpenGL ES）**：将高斯/点数据上传 GPU，使用 instancing + shader 做屏幕空间 splat/点精灵，支持旋转/缩放/曝光调节。  
6) **存储与分享**：`.ply` 写入 `Documents/Sharp`，暴露分享 Intent。

## 5. TODO 列表（首版必须项）
- [ ] 导出并校验 ExecuTorch/TorchScript 模型（桌面对齐 `predict_image` 输出）。  
- [ ] 初始化 Android 项目骨架（Compose、NDK、Filament 依赖）。  
- [ ] JNI 预处理：图像解码、resize、内参/`disparity_factor` 计算。  
- [ ] JNI 推理封装：加载模型、Vulkan/NNAPI 选择、同步/超时控制。  
- [ ] JNI 后处理：`unproject_gaussians`、`.ply` 写入与桌面一致的字段顺序。  
- [ ] 渲染器 V1：OpenGL ES/Filament 点精灵渲染，支持轨迹球交互。  
- [ ] UI 流程：导入 → 推理进度 → 结果页（保存/预览/分享），Apple 风格皮肤。  
- [ ] 性能基线：旗舰机单帧推理耗时、内存占用，渲染帧率。  
- [ ] 稳定性：超时/错误提示，存储权限（scoped storage），模型缓存校验。

## 6. 迭代方向（可选）
- 高斯渲染优化：基于 Vulkan 的 tile-based splatting、批处理排序。  
- 模型体积与时延优化：量化 (Q8/Q4)、低分辨率快速模式。  
- 多图批处理/队列、相册历史管理。  
- Web/桌面协同：ADB 内网推理或本地局域网节点作为 fallback。

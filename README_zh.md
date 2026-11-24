# ComfyUI-MiniMaxH3-PixelHead（`v0.2.0` 实验预览版）

用于 **MiniMax-H3** 的 ComfyUI 免 Video VAE 3D 视频像素直出节点（`45.7 MiB` BF16，对应官方 `2.68 GB` ViT3D Video VAE）。

[English](README.md) | [简体中文](README_zh.md)

[![状态: 实验预览版](https://img.shields.io/badge/%E7%8A%B6%E6%80%81-%E5%AE%9E%E9%AA%8C%E9%A2%84%E8%A7%88%E7%89%88%20(%E4%B8%8D%E6%8E%A8%E8%8D%90%E4%BD%BF%E7%94%A8)-orange)](#%E5%BD%93%E5%89%8D%E5%AD%98%E5%9C%A8%E7%9A%84%E9%97%AE%E9%A2%98)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Hugging Face](https://img.shields.io/badge/Hugging%20Face-pocis%2FMiniMax--H3--Video--PixelHead-yellow)](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead)
[![Comfy Registry](https://img.shields.io/badge/Comfy%20Registry-comfyui--minimaxh3--pixelhead-blueviolet)](https://registry.comfy.org/publishers/wither033/nodes/comfyui-minimaxh3-pixelhead)

> [!WARNING]
> **当前版本效果还不理想，不推荐在日常出片中使用，请等待后续优化版本。**
>
> 这版 `v0.2.0` 仅作为阶段性实验记录和低显存构图草稿工具先行放出：
> - 整体色调、动作走向、`(1, 4, 4, 4, 4)` 非均匀帧数还原（`17k + 5` 帧）以及扁平色块文字（`36.73 dB`）是正常的；
> - 但在**复杂自然纹理（人物五官细节、油画笔触、植被颗粒）和快速前景运动遮挡（`25.91 dB`，`LPIPS = 0.29～0.37`）上，画面有明显的涂抹感和柔焦模糊**，清晰度与官方 `2.68 GB` 36 层 ViT3D VAE 差距较大；
> - 正式渲染视频请继续使用官方 Video VAE，后续我们会针对感知损失和 `16px` 潜变量直注做新一版训练。

## 实测数据（`384×256×22帧` · NVIDIA L4）

以官方 `minimax_h3_video_vae_int8_convrot.safetensors` 为参照，在 20 段留出的 22 帧视频（`T_lat = 7`，跨越两个 `(1, 4, 4, 4, 4)` 周期，下图展示第 `0`、`11`、`17` 帧）上测试：

| 视频解码器 | 体积 | 单段中位延迟 | 相对速度 | 总体均值 PSNR | 自然纹理子集 PSNR | LPIPS-VGG (↓) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| 官方 `minimax_h3_video_vae` (36层 ViT3D) | `2680.8 MiB` | `251.30 ms` | `1.0×` | *(基准)* | *(基准)* | `0.0000` |
| `MiniMaxH3VideoPixelHead` (`v0.2.0`) | `45.7 MiB` (BF16) / `91.4 MiB` (FP32) | `29.59 ms` | `8.5×` | `31.03 dB` | `25.91 – 30.13 dB` | `0.2338` (自然画面 `0.29–0.37`) |

分场景表现：

| 测试子集 | PSNR (↑) | LPIPS-VGG (↓) | 实际情况 |
| :--- | :---: | :---: | :--- |
| `kinetic_cel_typo`（纯色图形与滚动文字） | `36.73 dB` | `0.0547` | 纯色边缘正常，但这部分分数把整体均值拉高到了 `31.03 dB`。 |
| `camera_pan_zoom`（油画/实拍推拉摇移） | `30.13 dB` | `0.2923` | 大轮廓和色调一致，但花瓣肌理（下图第 2 组 `25.7 dB`）和五官偏糊。 |
| `static_lockoff`（固定镜头纹理） | `29.17 dB` | `0.3059` | 帧间稳定无闪烁，但炭笔颗粒等细纹发软。 |
| `parallax_occlusion`（前景运动遮挡） | `25.91 dB` | `0.3677` | 运动交界处模糊明显，是当前最差的一类。 |

![关键帧对比 (f=0, 11, 17)：上排为官方 ViT3D VAE，下排为 v0.2.0 预览版](assets/video_keyframes_comparison.png)

## 当前存在的问题

排查 `comfy/ldm/minimax/vae.py` 与本轮训练配置后，导致细节发糊的主要原因有四点：

1. **单 Token 解压倍率比图像模型大 16 倍**：在 `Qwen-Image 2.1` 和 `Krea-2` 中，1 个 DiT Token 只对应 `16×16 = 256` 个像素；而 MiniMax-H3 的 `16×` 空间 VAE 叠加 `(1, 2, 2)` Patch 后，1 个 Token 需要还原 `4帧 × 32×32 = 4,096` 个像素。官方用的是 36 层 `2048` 维全局注意力 ViT3D，而本版仅用了 `23.96M` 参数（`width = 768`）的轻量卷积主干。
2. **高末步噪声（$\sigma_{\text{last}} = 0.6316$）下缺少感知损失**：在 `video_shift = 12.0` 下，8 步采样的末步起点噪声高达 `0.6316`。本轮只用了像素空间的 `L1 + MSE + 相邻差分` 训练，未加入 LPIPS 感知损失或频域约束，网络遇到不确定的高频纹理时会倾向于输出平滑均值（即肉眼看到的磨皮感）。
3. **潜变量注入层级偏粗**：`x0_hat_vid`（`[B, 24, T_lat, H_lat, W_lat]`）本身已经在 `16px` 网格上，本版在入口处将其 `2×2` 折叠进了 `32px` 层，损失了一部分空间定位精度，后续应改为在 `16px` 层直接注入。
4. **训练数据尺寸与步数偏少**：本轮仅在 `280` 段 `384×256` 片段上跑了 `1,800` Rollouts，其中还有 `1/4` 是纯色文字合成片段。

## 同系列仓库

| 适配模型 | 当前状态 | ComfyUI 节点仓库 | Hugging Face 权重 | 说明 |
| :--- | :---: | :--- | :--- | :--- |
| **Qwen-Image 2.1** | 可正常使用 | [Wither033/ComfyUI-QwenImage21-PixelHead](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead) | [pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead) | `v0.3` 小波版 (`79.3 MiB`, `37.5 ms`, `33.81 dB`, `LPIPS 0.0736`) + `v0.1` (`48.8 MiB`)，4 通道 RGBA |
| **Krea-2** | 可正常使用 | [Wither033/ComfyUI-Krea2-PixelHead](https://github.com/Wither033/ComfyUI-Krea2-PixelHead) | [pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead) | `52.6 MiB` BF16，`6144d` 降维桥 RGB，`59.55 ms` (`24.7×`)，均值 `33.33 dB` |
| **MiniMax-H3** *(本仓库)* | 实验预览 | [Wither033/ComfyUI-MiniMaxH3-PixelHead](https://github.com/Wither033/ComfyUI-MiniMaxH3-PixelHead) | [pocis/MiniMax-H3-Video-PixelHead](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead) | 暂不推荐正式使用；`45.7 MiB` BF16，提速 `8.5×`，高频纹理偏糊 |
| **训练代码与方法整理** | 工具与文档 | [Wither033/FastFrames-PixelHead-Trainer](https://github.com/Wither033/FastFrames-PixelHead-Trainer) | [HF 合集页](https://huggingface.co/collections/pocis/fastframes-zero-vae-direct-pixelheads-comfyui-6b031df8e9212f8bae294154) | 通用 `pixelhead_trainer` 库、各底座挂载点说明与 Colab L4 训练脚本 |

## 安装说明（仅供测试或低显存看草稿）

1. 安装节点：`git clone https://github.com/Wither033/ComfyUI-MiniMaxH3-PixelHead.git` 到 `ComfyUI/custom_nodes/`（或 `comfy node install comfyui-minimaxh3-pixelhead`）。
2. 从 [pocis/MiniMax-H3-Video-PixelHead](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead) 下载 `minimax_h3_video_pixel_head_bf16.safetensors`（`45.7 MiB`）放入 `ComfyUI/models/pixel_heads/`。
3. 示例工作流在 [`example_workflows/`](example_workflows/) 目录中：将 `MiniMax-H3 Video Pixel Head Tap` 接在 `MiniMaxH3SigmaShift` 之后，采样输出连到 `MiniMax-H3 Video Pixel Decode (Zero Video VAE)`。

## 许可与致谢

- **开源协议**：MIT License（独立编写的 PyTorch 实现）。
- **参考来源**：架构思路来自 [*FastFrames* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226) 与 `sorryhyun/anima-pixel-decoder`。

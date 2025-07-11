# 超小参数推理模型复现日志 (Project Log: Reproducing Ultra-Small Inference Models)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status: Training](https://img.shields.io/badge/Status-Training-blue.svg)](https://github.com/YOUR_USERNAME/YOUR_REPO/issues)

## 项目简介

本项目旨在完整记录我个人从零开始，复现一个高效、超小参数规模推理模型的技术历程。众所周知，将多项前沿工作整合并打造出工业级水准的模型是一项极具挑战性的任务。这份文档不仅是我的开发日志，更是对我在模型架构、分词器、训练策略等方面踩过的坑、获得的经验以及最终技术选型的深度总结与思考，希望能为同样走在这条路上的开发者提供一些参考。

## 目录

- [核心理念与技术选型](#核心理念与技术选型)
  - [1. 架构选择：拥抱主流，而非闭门造车](#1-架构选择拥抱主流而非闭门造车)
  - [2. 分词器选型：遵循规模定律](#2-分词器选型遵循规模定律)
  - [3. MoE 架构：资源有限下的性能权衡](#3-moe-架构资源有限下的性能权衡)
  - [4. 语料与训练策略：模型的燃料](#4-语料与训练策略模型的燃料)
- [项目进展日志](#项目进展日志)
- [当前挑战与思考](#当前挑战与思考)
- [未来计划](#未来计划)
- [如何贡献](#如何贡献)
- [许可证](#许可证)

## 核心理念与技术选型

早在 OpenAI 公开其 O1 模型时，我就已开始探索小型推理模型的构建。期间经历了多次失败，最终决定摒弃所有历史包袱，从零开始这一征途。以下是我在此过程中沉淀下的核心思考。

### 1. 架构选择：拥抱主流，而非闭门造车

在项目初期，我曾尝试构建一个全新的、自定义的 GPT 架构。然而，纵观当前众多成功的开源模型，一个清晰的共识是：**模型的最终效果更多取决于语料的质量和训练的深度，而非架构的标新立异。**

我强烈不建议任何尝试构建 LLM 的个人或小团队走“自研架构”这条路。原因如下：

*   **生态兼容性是生命线**：一个无法被 `transformers` 的 `AutoModelForCausalLM` 加载，或不兼容主流分词器标准的模型，无异于在自己的小圈子里自娱自乐，极大地限制了其应用和发展。
*   **工程成本巨大**：为了让自定义架构兼容主流的训练框架（如 DeepSpeed, FSDP）、强化学习框架（如 TRL），你需要付出远超模型研发本身的巨大工程努力。
*   **行业标准的重要性**：模型的实用性远比其结构上的“独创性”重要。放弃兼容性，就是放弃了被更广泛社区验证和使用的可能性。

**结论**：选择一个经过验证的主流架构（如 Llama, Mistral, Qwen, DeepSeek 等）作为基座，能让您将精力聚焦在数据、训练和对齐等更有价值的环节。

### 2. 分词器选型：遵循规模定律

分词器是模型与世界交互的窗口，其重要性不言而喻。

*   **早期尝试与失败**：我曾预训练过一个约 16k 词元、覆盖中英双语的 BPE 分词器，但其性能远未达到预期。
*   **大词表的优势**：消融实验证明，**分词器的词表大小同样存在“规模拓展定律” (Scaling Law)**。虽然像 Qwen (150k+) 或 DeepSeek (100k+) 这样的大型分词器对于纯中英任务看似有些“大材小用”，但它们是巨头们利用海量 GPU 资源在极其广泛和多样化的语料上训练得出的最优解。
*   **选择大词表的收益**：
    1.  **提升训练效率**：更大的词表意味着可以用更少的 Token 来表示相同的文本，直接减少了训练所需的总 Token 数量。
    2.  **优化推理与强化学习**：在推理或 RL 阶段，模型可以用更精炼的 Token 序列来表达复杂的思想，而不是耗费大量 Token 去拼凑浅层的语义。

**结论**：直接采用或微调一个由大公司发布的高质量、大词表分词器，是性价比最高的选择。

### 3. MoE 架构：资源有限下的性能权衡

在模型架构层面，我最终倾向于采用 **MoE (Mixture of Experts)** 架构。

我大约在 23 年末就开始了对 MoE 的探索。选择 MoE 的初衷非常明确：**在有限的计算资源下，最大化模型的性能**。MoE 架构能以一个相对较小的激活参数量，实现媲美参数规模大得多的密集型模型的性能。

然而，尽管我个人选择了 MoE，但我依然给其他实践者一个忠告：

> **如果你希望从零开始构建一个推理模型，我更推荐你从参数密集的传统架构开始。** 它的实现更简单、社区支持更成熟，是构建和调试推理能力的更稳固的起点。

### 4. 语料与训练策略：模型的燃料

*   **语料选择**: 目前计划采用 **Ultra-FineWeb** 数据集，它整合了高质量的中文 Fine-Web 和英文 Fine-Web 语料。我的下一个模型将基于此语料进行训练。
*   **训练策略**: 单纯的余弦学习率衰减可能不是最优解，后续计划尝试**梯形学习率调度 (Trapezoidal Learning Rate Schedule)**。
*   **数据需求**: 目前仍在积极寻找高质量的**中文数学和代码数据集**，这是提升模型逻辑推理和代码能力的关键。

---

## 项目进展日志

### **当前实验：架构探索 - 融合 Gemma-3n 设计 (2025-06-24)**

第四次重启预训练，本次实验的核心是探索更前沿的模型架构。灵感来源于对 Gemma-3n 的逆向工程分析项目 **[reverse-engineering-gemma-3n](https://github.com/antimatter15/reverse-engineering-gemma-3n)**，架构设计上更偏向 Gemini。

*   **架构改进**:
    *   **逐层嵌入 (Per-Layer Embedding)**: 引入 `per_layer_embed` 技术，替代传统的单一输入嵌入层。
    *   **增强残差连接 (Enhanced Residual)**: 采用增强型残差层设计，以改善梯度流和信息传播。
    *   **技术融合**: 新架构整合了 **MTP (Multi-head Token Parallelism)** 多令牌预测技术和 **专家负载均衡损失 (Expert Load Balancing Loss)**，以协同提升训练效率和 MoE 性能。
*   **初步观察**:
    *   **损失显著下降**: 在初步测试中，新架构的训练损失相较于之前版本有明显降低。
    *   **待解问题**: 目前尚需通过消融实验来明确性能增益的来源：是源于参数量增加带来的直接收益，还是新架构设计本身带来的效率提升。

---

### **历史日志**

**2025-06-14**
*   **核心事件**: 多次重启预训练，对训练稳定性和泛化能力进行深度反思。
*   **关键教训**:
    1.  **实现细节的重要性**: 第一次重启因 `MTP` 模块的实现存在缺陷而失败。
    2.  **Batch Size 的关键作用**: 第二次重启仅使用 `batch_size=1` 进行训练，导致模型不收敛。实践证明，足够大的 `batch_size` 对于模型学习数据的通用表征（通识）至关重要，并直接影响生成文本的多样性。
    3.  **对基础设施的思考**: 认识到强化学习（RL）的瓶颈不仅在于算法本身，更深层次地受到 GPU 资源和系统工程（如模型并行、张量并行、专家并行）优化水平的制约。

**2025-06-09**
*   **核心事件**: 复现 DeepSeek-V3 架构，重点引入 `MTP` 和 `aux_loss`。
*   **关键教训**:
    *   **`bfloat16` 方案**: `torch.float8_e4m3fn` 与 FlashAttention 存在兼容性问题，最终采用 `FlashAttention + bfloat16` 混合精度方案。
    *   **梯度尖峰**: 训练日志在约 60k 步时出现梯度尖峰，虽被优化器成功抑制，但暴露了训练稳定性的挑战，梯度裁剪是需要精调的关键参数。

**2025-04-01**
*   **核心事件**: 训练 `qwen3moe` 模型，并发现之前 DeepSeek 模型训练失败的根本原因。
*   **关键教训**:
    1.  **上下文长度的重要性**: 预训练语料的实际长度远小于设定的上下文长度，导致模型在处理短文本时处于“冗余训练”状态，性能畸形。
    2.  **分词器词表大小的影响**: Qwen 的大词表 (150k+) 显著增加了强化学习阶段的显存压力。

**2025-03-21**
*   **核心事件**: 转向使用官方的 DeepSeek-V2 架构进行训练。
*   **关键教训**: 在没有足够资源和精力的情况下，直接使用经过验证的官方实现是更务实的选择。

**2025-01-27**
*   **核心事件**: 尝试自行实现 DeepSeek-V3 架构，但训练后模型无法转换为 `gguf` 格式，因其架构未在 `llama.cpp` 中注册。
*   **关键教训**:
    1.  **工程兼容性至上**: 修改核心架构需要巨大的工程量来实现与下游生态的兼容。
    2.  **对“乱码”的初步反思**: 小模型输出乱码，根本原因是训练不充分，而非分词器问题。

## 当前挑战与思考

#### 关于强化学习的瓶颈
在对模型进行对齐时，强化学习（RL）阶段遇到了瓶颈：
1.  **使用 KL 散度约束**: 模型能很好地遵循预设格式，但探索能力严重受限，难以产生更高质量的解。
2.  **不使用 KL 散度约束**: 模型在“学习格式”和“学习推理”之间挣扎，收敛极其困难。

这或许是一种语言混合或模型寻找自身表达方式的自然过程，正如人在学习外语时会不自觉地混合母语。

#### 关于模型输出“乱码”的分析
我初步判断：**指令微调可能带来了过度的“格式惯性”**。当模型在新的、更复杂的推理语料上训练时，它试图挣脱原有指令微调带来的强约束，寻找一种更适合表达复杂逻辑的新形式。这个探索过程就外显为暂时的“乱码”或格式崩溃。

## 未来计划

- [ ] 完成当前基于 Gemma-3n 改进架构的预训练实验。
- [ ] 进行消融实验，分析 `per_layer_embed`、增强残差等架构改进与参数量增加对模型性能的各自贡献。
- [x] 整合并开源包含 MTP 和 `aux_loss` 的训练脚本和配置文件。
- [ ] 发布预训练完成后的模型权重及详细的性能评测报告。
- [ ] 探索在当前模型基础上进行指令微调和人类偏好对齐（如 DPO）的有效方法。
- [ ] 撰写更详细的技术博客，分享 MTP、`aux_loss`、梯形学习率调度等模块的实现心得。

## 如何贡献

本项目目前主要为个人探索日志。但非常欢迎您通过 **Issues** 提出问题、分享见解或提供宝贵建议。如果您对项目的某些部分有独到的想法或改进，也欢迎创建 **Pull Request**。

## 许可证

本项目采用 **MIT 许可证**。

Copyright (c) 2024 [Your Name/Organization]

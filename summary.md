# ACE 集成摘要

本文档概述 ACE (Agentic Context Engineering) 的核心理念、系统组成、运行流程以及将其嵌入其他项目时的实施步骤和应用思路。

## 核心概念
- **面向上下文的自改进框架**：将策略、公式、模板等知识沉淀到可增量更新的“Playbook”，避免多次重写导致的上下文塌缩。【F:README.md†L13-L48】
- **三代理协同**：Generator 生成解题轨迹，Reflector 评估并提取要点，Curator 将洞察合并成结构化增量更新，必要时使用 BulletpointAnalyzer 去重与压缩。【F:ace/ace.py†L22-L110】【F:README.md†L42-L48】
- **多模式运行**：支持 offline/online/eval_only 三种模式，覆盖训练、自适应与纯评估流程。【F:ace/ace.py†L166-L200】【F:README.md†L103-L128】

## 体系结构
- **核心类**：`ACE` 作为总控，负责初始化三代理、管理 Playbook、日志与路径，并暴露统一的 `run()` 接口。【F:ace/ace.py†L22-L200】
- **关键配置**：`num_epochs`、`max_num_rounds`、`curator_frequency`、`eval_steps`、`playbook_token_budget` 等参数可通过 `config` 传入，默认值见 `ACE._extract_config_params`。【F:ace/ace.py†L111-L135】
- **依赖与环境**：安装 `requirements.txt`，并在 `.env` 配置 API 密钥；支持 `sambanova`、`together`、`openai` 等提供商。【F:README.md†L54-L128】

## 实施步骤
1. **安装与准备**：克隆仓库、安装依赖、复制并填写 `.env`。【F:README.md†L54-L64】
2. **初始化 ACE**：选择 API Provider 和模型（默认 DeepSeek-V3.1），设置 `max_tokens` 等资源限制，必要时启用 `use_bulletpoint_analyzer` 去重。【F:README.md†L66-L129】【F:ace/ace.py†L33-L110】
3. **配置运行参数**：构造 `config` 字典，指定训练轮次、评估频率、保存目录、Playbook 令牌预算、JSON 模式等。【F:README.md†L84-L100】【F:ace/ace.py†L111-L135】
4. **准备数据处理器**：实现 `DataProcessor.process_task_data`、`answer_is_correct`、`evaluate_accuracy` 三个方法，将原始数据映射到 `context/question/target` 并提供评测逻辑（可参考 finance 任务）。【F:EXTENDING_ACE.md†L23-L115】
5. **调用 `run()`**：
   - offline：传入 `train_samples`、`val_samples`，可选 `test_samples`，周期性评估与保存 Playbook。
   - online：仅需 `test_samples`，边推理边更新 Playbook。
   - eval_only：纯评估已存在或空的 Playbook。
   【F:README.md†L103-L164】【F:ace/ace.py†L166-L200】
6. **产出管理**：ACE 会在 `save_dir` 下创建带时间戳的运行目录，保存中间 Playbook、最佳 Playbook 与详细 LLM 日志，便于复现与迁移。【F:ace/ace.py†L137-L165】

## 典型应用场景
- **领域自适应推理**：在金融、代码生成或多模态助手中，通过 offline 训练获得高质量 Playbook，再以 online 模式持续迭代。【F:README.md†L130-L194】
- **知识库型助手**：用 Curator 的增量更新把用户交互沉淀成可重用的策略与模板，减少重复提示成本。【F:README.md†L13-L48】
- **实验与对比研究**：利用统一配置与日志输出，方便在不同模型或任务间比较增量学习效果与效率指标。【F:README.md†L25-L40】【F:ace/ace.py†L137-L165】

## 在其他项目中的集成提示
- 将本文件随代码复制到目标项目，作为接入手册；保持 `ace/` 目录结构或以子模块形式引入。
- 确保目标项目的任务数据有对应的 `DataProcessor`，并在运行脚本中按上述步骤初始化 ACE 和配置。
- 若需要多模型混用，可在 `initialize_clients` 中扩展新 Provider；保持三代理模型兼容即可。【F:ace/ace.py†L33-L110】

## 将 ACE 用作通用 LLM 中间件
- **目标**：让任何调用基础 LLM 的应用先经过 ACE 的三代理推理、反思与 Playbook 复用，以提升稳定性、可解释性和可持续改进能力。
- **可带来的提升**：
  - 统一的 ACE 模板与 Playbook 使提示工程标准化，减少临场 prompt 拼接导致的波动。【F:README.md†L13-L48】
  - Curator 持续沉淀高价值策略，BulletpointAnalyzer 压缩去重，随着交互积累自动演进提示质量。【F:ace/ace.py†L57-L110】
  - 结构化日志与版本化 Playbook 便于审计、回溯与复现，降低直接 API 调用的不可控风险。【F:ace/ace.py†L137-L165】
- **最小接入路线（在线模式）**：
  1. 在应用的 LLM 调用入口实现 `DataProcessor`，将请求映射为 `context/question`，并实现 `answer_is_correct`（无标注时可返回 `True` 跳过打分）。【F:EXTENDING_ACE.md†L23-L115】
  2. 初始化 `ACE(mode="online", use_bulletpoint_analyzer=True, save_dir="ace_runs")`，设置模型 provider、`max_tokens`、`temperature` 等限制。【F:ace/ace.py†L33-L135】【F:README.md†L66-L129】
  3. 每次需要调用 LLM 时，将请求包装为 `test_samples=[sample]` 调用 `ace.run(test_samples=test_samples)`，返回生成答案并更新 Playbook；在后续请求中复用最新 Playbook。
  4. 需要低延迟时，可预加载 Playbook、调整 `curator_frequency` 为按轮次更新，或减少推理轮次 `max_num_rounds` 以换取速度。【F:ace/ace.py†L111-L200】
  5. 若希望快速获得高质量基线，可先离线用少量标注样本跑几轮 `mode="offline"` 生成初始 Playbook，再切回在线模式持续迭代。
- **能否满足“中间件”诉求？** 可以。ACE 在在线模式下即插即用，无需额外训练即可包装 LLM 调用，并且通过 Playbook 持续积累策略，兼顾质量提升与治理需求。


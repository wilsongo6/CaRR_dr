# CaRR 训练复现指南（从零开始）

本文档用于在当前仓库中从零复现 `CaRR / C-GRPO` 训练与评测流程，重点回答：
- 用的是什么模型
- 用的是什么数据
- 启动脚本在哪
- 训练脚本和评测脚本在哪

> 说明：本流程以仓库当前脚本为准，默认基于 `slime` 框架、`ray + sglang` 分布式运行方式。

---

## 1. 你会用到哪些脚本（总览）

### 环境与服务启动脚本
- 环境模板生成：`scripts/setup/prepare_env.sh`
- Tool Server 启动：`scripts/setup/start_tool_server.sh`
- 训练 RM 启动（DeepSeek）：`scripts/setup/start_rm_deepseek.sh`
- 评测 RM 启动（GPT-5）：`scripts/setup/evaluation_start_rm_gpt5.sh`
- 评测数据解密：`scripts/setup/decrypt_eval_data.sh`

### 模型转换脚本（HF -> torch_dist）
- 4B：`scripts/setup/convert_4b.sh`
- 30B：`scripts/setup/convert_30b.sh`

### 训练脚本
- 4B C-GRPO：`scripts/training/training_run_4b-C-GRPO-rubric0.3.sh`
- 30B C-GRPO：`scripts/training/training_run_30b-C-GRPO-rubric0.3.sh`
- 30B 另一入口（同类流程）：`scripts/training/training_run_30b.sh`

### 评测脚本
- 4B 评测：`scripts/eval/evaluation_run_4b.sh`
- 30B 评测：`scripts/eval/evaluation_run_30b.sh`

---

## 2. 模型与数据（你要准备什么）

## 2.1 模型

当前仓库训练脚本默认使用：
- 4B SFT 初始模型：`slime/init_ckpts/DeepDive-4B-SFT`
- 30B SFT 初始模型：`slime/init_ckpts/DeepDive-30B-SFT`

对应实验背景模型是：
- Qwen3-4B-Thinking-2507
- Qwen3-30B-A3B-Thinking-2507

可参考仓库主页给出的公开资源：
- 模型与数据集合：`https://huggingface.co/datasets/THU-KEG/CaRR-DeepDive`
- CaRR/C-GRPO 资源集合：`https://huggingface.co/collections/THU-KEG/carr-and-c-grpo`

你需要把模型按脚本默认路径放好，或者通过环境变量覆盖：
- `HF_DIR`
- `HF_MODEL_PATH`
- `REF_MODEL_PATH`

## 2.2 训练数据

训练脚本默认读取：
- `slime/outputs/data/deepdive-rl-2k-browser-oss-rubric.jsonl`

该文件当前仓库不直接提供，需要你自行下载/准备后放到该路径，或者设置：
- `PROMPT_DATA=/your/path/xxx.jsonl`

## 2.3 评测数据

仓库内提供的是加密评测数据：
- `data/eval/*.jsonl.enc`

需要先解密到：
- `data/eval_decrypted/*.jsonl`

解密脚本：
- `scripts/setup/decrypt_eval_data.sh`

---

## 3. 运行前提与环境假设

这些脚本默认你在多机多卡训练环境中运行，且具备：
- Docker / CUDA / 驱动环境可用
- `ray`、`python3`、`ssh` 可用
- `slime` 代码在仓库内（当前已包含）
- 可访问外部 API（DeepSeek/OpenAI/Serper/Jina）

训练脚本还默认依赖以下集群环境变量（通常由平台注入）：
- `MLP_WORKER_0_HOST`
- `MLP_WORKER_0_PORT`
- `MLP_SOCKET_IFNAME`
- `MLP_WORKER_NUM`

并默认读取主机文件：
- `/root/mpi_rack_hostfile`

如果你不是该类集群环境，需要先按自己的集群形态改脚本再跑。

---

## 4. 从零开始复现（推荐顺序）

## Step 0) 准备代码和容器

推荐镜像（来自 `slime/docker/version.txt`）：

```bash
docker pull slimerl/slime:nightly-dev-20260311a
```

进入仓库根目录（本文命令都在仓库根目录执行）。

## Step 1) 生成 `.env`

```bash
bash scripts/setup/prepare_env.sh
```

然后编辑 `.env`，至少补全：
- 路径：`CARR_ROOT`, `SLIME_ROOT`
- Tool Server：`SERP_API_KEY`, `JINA_API_KEY`, `TOOL_SERVER_PORT`
- 训练 RM：`DEEPSEEK_API_KEY`, `DEEPSEEK_BASE_URL`, `RM_TRAIN_PORT`
- 评测 RM：`OPENAI_API_KEY`, `OPENAI_BASE_URL`, `RM_EVAL_PORT`
- 数据解密：`EVAL_DATA_PASSWORD`
- 可选代理：`HTTP_PROXY`, `HTTPS_PROXY`

模板文件在：
- `.env.example`

## Step 2) 启动 Tool Server

```bash
bash scripts/setup/start_tool_server.sh
```

健康检查：

```bash
curl -sS http://127.0.0.1:${TOOL_SERVER_PORT:-7230}/ \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"session_id":"health","name":"start_session","arguments":{},"remote_env_info":null}'
```

## Step 3) 启动训练 Reward Model 服务

```bash
bash scripts/setup/start_rm_deepseek.sh
```

该服务默认监听 `${RM_TRAIN_PORT:-8888}`。

## Step 4) 启动评测 Reward Model 服务

```bash
bash scripts/setup/evaluation_start_rm_gpt5.sh
```

该服务默认监听 `${RM_EVAL_PORT:-6759}`。

## Step 5) 解密评测数据

```bash
bash scripts/setup/decrypt_eval_data.sh
```

解密后你会在 `data/eval_decrypted/` 下看到可读 jsonl 文件。

## Step 6) 转换初始模型（HF -> torch_dist）

### 4B
```bash
bash scripts/setup/convert_4b.sh
```

默认输入：
- `slime/init_ckpts/DeepDive-4B-SFT`

默认输出：
- `slime/outputs/converted_ckpts/DeepDive-4B-SFT-torch_dist`

### 30B
```bash
bash scripts/setup/convert_30b.sh
```

默认输入：
- `slime/init_ckpts/DeepDive-30B-SFT`

默认输出：
- `slime/outputs/converted_ckpts/DeepDive-30B-SFT-torch_dist`

## Step 7) 启动训练

### 4B C-GRPO 训练
```bash
bash scripts/training/training_run_4b-C-GRPO-rubric0.3.sh
```

### 30B C-GRPO 训练
```bash
bash scripts/training/training_run_30b-C-GRPO-rubric0.3.sh
```

这两个脚本都会：
- 清理旧 `ray/sglang` 进程
- 拉起 ray head + workers
- 提交 `slime/train.py` 任务
- 使用 `configs/source_config.json` 对接 tool server + RM

## Step 8) 训练后评测

### 4B 评测
```bash
bash scripts/eval/evaluation_run_4b.sh
```

### 30B 评测
```bash
bash scripts/eval/evaluation_run_30b.sh
```

---

## 5. 默认关键配置（建议先理解）

## 5.1 Reward 和工具环境路由

配置文件：
- `slime/configs/source_config.json`

默认含义：
- 训练源 `deepsearch_with_browser` 走 `http://localhost:8888/evaluate`
- 评测源 `deepsearch_with_browser_eval` 走 `http://localhost:6759/evaluate`
- 远程工具环境走 `http://localhost:7230`

## 5.2 C-GRPO 相关参数（脚本内已给定）

训练脚本默认开启了典型 C-GRPO 组合，例如：
- `--advantage-estimator grpo`
- `--rubric-reward-ratio 0.3`
- `--normalize-rubric-reward`
- `--rubric-only-positive`

## 5.3 日志和输出目录

默认日志：
- 4B 训练日志：`slime/test_4b.log`
- 30B 训练日志：`slime/test_30b.log`
- 4B 评测日志：`slime/test_eval_4b.log`
- 30B 评测日志：`slime/test_eval_30b.log`

默认 checkpoint 输出：
- `slime/outputs/<EXP_TAG>/`

默认评测结果：
- `slime/outputs/<EXP_TAG>/eval_results/`

---

## 6. 一条龙最小命令清单

```bash
# 1) 环境
bash scripts/setup/prepare_env.sh

# 2) 三个服务（建议开三个终端）
bash scripts/setup/start_tool_server.sh
bash scripts/setup/start_rm_deepseek.sh
bash scripts/setup/evaluation_start_rm_gpt5.sh

# 3) 解密评测数据
bash scripts/setup/decrypt_eval_data.sh

# 4) 模型转换（二选一或都转）
bash scripts/setup/convert_4b.sh
bash scripts/setup/convert_30b.sh

# 5) 训练（二选一）
bash scripts/training/training_run_4b-C-GRPO-rubric0.3.sh
bash scripts/training/training_run_30b-C-GRPO-rubric0.3.sh

# 6) 评测（二选一）
bash scripts/eval/evaluation_run_4b.sh
bash scripts/eval/evaluation_run_30b.sh
```

---

## 7. 常见问题

1. 找不到训练数据 `deepdive-rl-2k-browser-oss-rubric.jsonl`
- 这是外部准备数据，不在当前仓库。
- 解决：放到 `slime/outputs/data/` 或设置 `PROMPT_DATA`。

2. `ray` worker 拉不起来
- 脚本默认通过 `ssh root@<worker_ip>` 拉 worker，依赖免密和 `/root/mpi_rack_hostfile`。
- 解决：检查 SSH、hostfile、网络接口变量。

3. 评测数据无法解密
- 需要 `.env` 中正确设置 `EVAL_DATA_PASSWORD`。

4. RM 或 tool server 连接失败
- 检查端口：`7230 / 8888 / 6759`
- 检查 `slime/configs/source_config.json` 与实际端口是否一致。

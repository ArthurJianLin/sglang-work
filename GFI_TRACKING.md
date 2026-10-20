# SGLang Good-First-Issue 跟踪（patch 开发工作页）

> 个人工作文档，不参与上游 PR。换环境 clone 后从本文件开始。
> 生成时间：2026-10-20。仓库基线：main = `49766a74f9`（Gitee 镜像快照，上游最新为 `306ccf6f`，开发前先同步）。

## 一、社区 Patch 合入流程标准（摘自 docs/docs/developer_guide/contribution_guide.mdx）

### 1. 分支与提交

- 新贡献者**无权直接 push 官方仓库**：fork 到自己账号 → clone fork → 开发
- **禁止直接 commit 到 main**，永远开新分支：`feature/my-new-feature` 或 `fix/xxx`
- 提交前必须过格式检查：

```bash
pip3 install pre-commit
pre-commit install
pre-commit run --all-files   # 失败就重跑到全绿
```

### 2. PR 标题格式（CI 强制校验，格式错了直接挂）

```text
[Type] scope: title
[Type] [Hardware] scope: title
```

- **Type**：`Feat` / `Fix` / `Refactor` / `Perf` / `Doc` / `Test` / `CI` / `Chore`
- **scope 必填**：模块或模型名，子模块用 `/`（如 `mem_cache`、`diffusion/scheduler`）
- **Hardware 可选**：`[NVIDIA]` `[AMD]` `[Intel]` `[Huawei]` `[Ascend]`
- 示例：`[Fix] mem_cache: Fix radix cache eviction bug`

### 3. 单元测试要求（关键合入门槛）

- 改了 `python/sglang/srt/` 下的文件 → 检查 `test/registered/unit/` 有无对应测试，**保护具体行为就必须加回归覆盖**
- 映射关系：`srt/mem_cache/radix_cache.py` → `unit/mem_cache/test_radix_cache_unit.py`
- CI 注册测试必须：`CustomTestCase`（非裸 unittest.TestCase）+ `register_*_ci(...)` + `__main__` 入口
- 本地跑法：

```bash
python3 test/registered/unit/mem_cache/test_radix_cache_unit.py   # 最接近 CI
pytest test/registered/unit/mem_cache/ -v                          # 目录级
```

### 4. CI 触发机制（新人须知）

- CI 资源有限，**新人无权触发 CI**——PR 需被加 `run-ci` label（有权限者操作）
- **PR 作者可用**：`/rerun-failed-ci`（自己的 PR 上重跑失败项）
- 冷却期默认 120 分钟，别频繁重跑
- review 合并流程：Merge Oncall + Codeowner approve（见 .github/MAINTAINER.md）

### 5. 代码风格红线（review 常打回点）

- 重复代码 >5 行 → 抽共享函数
- **最小化 CPU-GPU 同步**：`tensor.item()` / `tensor.cpu()` 尽量不用，向量化
- 模型 forward 路径上的运行时检查 → `__init__` 里缓存成布尔
- 函数 <100 行；文件 >2000 行要拆；新数据容器用 `msgspec.Struct`（不用 dataclass/attrs）
- **禁止 pickle** 反序列化不可信数据（`pickle.loads`/`recv_pyobj`）
- 测试文件 <500 秒；单文件复用 server 启动
- 新硬件/新功能：优先新文件（如 `allocator_ascend.py`），别大改现有代码

### 6. 新手官方推荐材料

- [Mini-SGLang](https://github.com/sgl-project/mini-sglang)（结构速览）
- [Code Walk-through](https://github.com/zhaochenyang20/Awesome-ML-SYS-Tutorial/tree/main/sglang/code-walk-through)
- 有问题进 Slack（slack.sglang.io）

## 二、5 个候选 GFI（按推荐顺序）

> 状态核实时间：2026-10-20。开工前先到 issue 页确认"无人认领 + 未被修复"，礼貌留言 "I'd like to work on this"。

### 🥇 #1 首选：#6357 — Prometheus metric `avg_request_queue_latency` 未采集（Bug）

- **链接**：https://github.com/sgl-project/sglang/issues/6357
- **难度**：⭐⭐（低）| **类型**：bug 修复 | **状态**：open，6 评论，2026-09-13 更新
- **做什么**：`avg_request_queue_latency` 指标在 Prometheus 暴露端缺失/不更新，定位并补上采集链路
- **为什么适合**：bug 修复范围明确、验收清晰（指标出现即修好）；涉及 scheduler → metrics 传递链，正好摸清调度观测面
- **入手路径**：`python/sglang/srt/managers/metrics.py` + scheduler 里 queue 时间戳记录点
- **PR 标题建议**：`[Fix] metrics: Collect avg_request_queue_latency in scheduler`

### 🥇 #2 并行首选：#5979 — Add KV cache usage prometheus metrics（Feature）

- **链接**：https://github.com/sgl-project/sglang/issues/5979
- **难度**：⭐⭐（低-中）| **类型**：feature | **状态**：open，**11 评论**，2026-08-13 更新（偏冷，认领前必须确认）
- **做什么**：对标 vLLM，给 SGLang 加 KV cache 利用率 Prometheus 指标
- **为什么适合**：KV cache 是主线 C2 核心知识；有 vLLM 参照实现可抄思路；纯 Python 观测层风险低
- **⚠️ 风险**：评论多且停更两个月，很可能别人已在做——**先留言确认再动工**
- **入手路径**：`python/sglang/srt/mem_cache/` 的使用量统计 → `managers/metrics.py` 暴露 gauge
- **PR 标题建议**：`[Feat] metrics: Expose KV cache usage prometheus metrics`

### 🥈 #3：#19612 — Unified JIT / Precompilation Cache Directory（Feature）

- **链接**：https://github.com/sgl-project/sglang/issues/19612
- **难度**：⭐⭐⭐（中）| **类型**：feature | **状态**：open，5 评论，2026-10-13 更新（活跃）
- **做什么**：统一分散的 JIT/预编译缓存目录到单一可配置位置
- **为什么适合**：**文件系统/缓存路径管理 = 存储内核背景对口**；近期还在更新说明维护者关注
- **入手路径**：搜索各 JIT 编译路径的 cache dir 硬编码 → 抽统一配置项 + 环境变量
- **PR 标题建议**：`[Feat] runtime: Unify JIT and precompilation cache directory`

### 🥈 #4：#18891 — server-disk checksum for update weights from disk（Diffusion）

- **链接**：https://github.com/sgl-project/sglang/issues/18891
- **难度**：⭐⭐⭐（中）| **类型**：feature | **状态**：open，3 评论，2026-03-03 更新（冷）
- **做什么**：Diffusion 模型从磁盘更新权重时，对 VAE/text encoder 做 checksum 比对
- **为什么适合**：**磁盘校验/完整性 = 存储背景对口**
- **⚠️ 风险**：停更半年，需先确认 issue 是否仍有效
- **入手路径**：`python/sglang/srt/model_loader/` 权重加载路径 + hash 校验
- **PR 标题建议**：`[Feat] loader: Add disk checksum verification for weight updates`

### 🥉 #5 长线挑战：#1715 — Cascade attention kernels（Kernel）

- **链接**：https://github.com/sgl-project/sglang/issues/1715
- **难度**：⭐⭐⭐⭐⭐（高）| **类型**：kernel 集成 | **状态**：open，8 评论，2026-07-24 更新
- **做什么**：集成 FlashInfer 的 cascade attention kernel（长上下文 KV 超 L2 缓存场景）
- **为什么放最后**：high priority + 真内核工程，适合**第 3+ 个 PR** 再碰（前两个练手攒信任）
- **参考**：https://flashinfer.ai/2024/02/02/cascade-inference.html

## 三、换环境开发流程（本仓库专用）

```bash
# 1. clone（任意环境）
git clone https://github.com/ArthurJianLin/sglang.git
cd sglang

# 2. 同步上游（github.com 时通时断，多试；或等 Gitee 镜像更新）
git remote add upstream https://github.com/sgl-project/sglang.git
git fetch upstream && git merge upstream/main   # 注意：merge 保留本地 GFI_TRACKING.md，不要 reset

# 3. 开工（例：#6357）
git checkout -b fix/metrics-queue-latency
# ... 改代码 + 加单测 ...
pre-commit run --all-files
python3 test/registered/unit/metrics/test_xxx.py

# 4. 推回自己仓库（多环境同步）
git push github fix/metrics-queue-latency

# 5. 从 GitHub 网页向 sgl-project/sglang 提 PR（标题按 [Type] scope: title）
```

## 四、基线与已知偏差

- 本仓库 main 来自 **Gitee 镜像**（github.com 不可达时的国内通道），拉取时 HEAD=`49766a74`
- 真实上游 2026-10-20 HEAD=`306ccf6f`——**做 patch 前必须先同步到上游最新**，否则基准落后会冲突
- 166 个 tags 已同步；远端校验：commit SHA / tree SHA 与本地一致

> AI生成

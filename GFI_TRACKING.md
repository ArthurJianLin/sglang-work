# SGLang Good-First-Issue 跟踪（patch 开发工作页）

> 个人工作文档，不参与上游 PR。换环境 clone 后从本文件开始。
> 生成时间：2026-10-20。仓库基线：main = `309a6802fa`（含本文件；基于 Gitee 镜像 `49766a74`）。
> **认领核实更新：2026-10-20 晚**——原首选 #1/#2/#3 均有活跃开放 PR，勿重复开工。

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

> 初次筛选：2026-10-20。**认领复核不通过——见第二节末「现状」**。

### 🥇 #1 ~~首选~~：#6357 — `avg_request_queue_latency` 未采集

- **链接**：https://github.com/sgl-project/sglang/issues/6357
- **本地缺口**：`SchedulerStats` / `metrics_collector.py` 仍无此 gauge；只有 `queue_time_seconds` histogram
- **⛔ 勿新开 PR**：已有活跃 PR [#38228](https://github.com/sgl-project/sglang/pull/38228)（Guts1005，10-13 更新，含单测；等 maintainer review / `run-ci`）；另 [#36737](https://github.com/sgl-project/sglang/pull/36737) 仍 open
- **可选协助**：帮 #38228 过 CI / 跟进 review（勿再写第三份实现）

### 🥇 #2 ~~并行首选~~：#5979 — KV cache usage prometheus metrics

- **链接**：https://github.com/sgl-project/sglang/issues/5979
- **本地已有**：`sglang:full_token_usage` / `kv_used_tokens` 等；缺的多半是 vLLM 别名 `gpu_cache_usage_perc`
- **⛔ 勿新开**：[#34714](https://github.com/sgl-project/sglang/pull/34714) open（10-03）

### 🥈 #3 ~~JIT cache~~：#19612 — Unified JIT / Precompilation Cache Directory

- **链接**：https://github.com/sgl-project/sglang/issues/19612
- **⛔ 多人做过**：[#44068](https://github.com/sgl-project/sglang/pull/44068) closed；[#33424](https://github.com/sgl-project/sglang/pull/33424) / [#35621](https://github.com/sgl-project/sglang/pull/35621) 仍 open

### 🥈 #4 ★ 当前最可做切片：#18891 — disk checksum（Diffusion）

- **链接**：https://github.com/sgl-project/sglang/issues/18891
- **历史**：JasonZhao47 认领后放弃（03-03）；assignee 仍是 maintainer `zhaochenyang20`
- **VAE 半边已被占**：[#41407](https://github.com/sgl-project/sglang/pull/41407)（PansaLegrand，draft）明确写 *Related to #18891 (VAE portion)*，并声明 **text-encoder 命名映射 / fused weights / sharded refit 是 follow-up**
- **✅ 可认领切片**：text-encoder 的 server↔disk checksum 比对（issue 原意的另一半）
- **入手路径**：
  - `python/sglang/multimodal_gen/runtime/post_training/weights_updater.py`
  - `tensor_update_checker.py` / `get_weights_checksum`
  - `test/single_test_file/test_update_weights_from_disk.py`
  - 参考 #41407 的 VAE parity 测法，扩到 text_encoder
- **PR 标题建议**：`[Feat] diffusion: Verify text-encoder checkpoint parity after disk updates`
- **开工前必做**：在 #18891 留言认领 text-encoder 切片，并 @ PansaLegrand 避免与 #41407 撞车

### 🥉 #5 长线：#1715 — Cascade attention kernels

- 仍高难度，维持「第 3+ 个 PR 再碰」

### 现状结论（2026-10-20 晚）

| 原排序 | Issue | 结论 |
|--------|-------|------|
| #1 | 6357 | 有完整实现 PR，旁观/协助即可 |
| #2 | 5979 | 有 PR + 本地已有大量 KV usage gauges |
| #3 | 19612 | 多 PR 缠斗 |
| #4 | 18891 | VAE 有 draft PR；**text-encoder 仍空**（次选） |
| #5 | 1715 | 暂缓 |

其他热门 GFI（#6531 / #18427 / #28808 / #8540 / #19092 的 spam PR 等）多数有人占或已关闭。

### ★ 重筛结果：当前最合适、真正没人干的 GFI

> 扫了全部 46 个 open GFI + 关联 open/closed PR。多数要么有活跃 PR，要么 10 月中有人刷号认领后被 maintainer 关掉。

#### 🥇 首选：**#19092 — Parallelize Diffusion module loading**

- **链接**：https://github.com/sgl-project/sglang/issues/19092
- **难度**：⭐⭐⭐（中）| **类型**：perf / loading | **assignee**：仅 maintainer 挂名
- **为什么现在空**：Asthenia0412 的 [#44127](https://github.com/sgl-project/sglang/pull/44127) 因「8 小时开 16 个无关 PR」被 hnyls2002 关掉；**当前无 open PR**；本地 `composed_pipeline_base.py` 仍是串行 `for spec in tqdm(component_load_specs)`（约 L621）
- **做什么**：`text_encoder` + `transformer` 并行加载（ThreadPool / 现有 LLM 并行加载策略），冷启动从 ~63s load 压到 ~max(两者)
- **为什么适合**：磁盘/权重加载路径 = 存储背景对口；范围清晰；有 #19087 的 profiling 数字当基线
- **入手**：`python/sglang/multimodal_gen/runtime/pipelines_core/composed_pipeline_base.py` → `load_modules`
- **PR 标题建议**：`[Perf] diffusion: Parallelize heavy component loading`
- **开工前**：issue 留言认领 + 说明不重复 Asthenia 的 spam PR

#### 🥈 次选：**#18891 — text-encoder disk checksum**（VAE 半边已有 #41407）

- 见上文 #4；与 #41407 协调 text-encoder 切片即可

#### 🥉 备选（更难 / 需先核实）

| Issue | 说明 |
|-------|------|
| #7892 TBO in draft models | 无 open PR（WIP #32529 已关）；代码仍 `not model_runner.is_draft_worker`——真需求，但是硬核 speculative |
| #10585 / #21085 / #12562 | Asthenia 认领后 PR 被清；暂无 open PR，但难度/GPU 依赖更高 |

#### ❌ 已排除（看起来空、实际有人）

6357→#38228；5979→#34714；3244→#36116；8174→建议关闭+#40476；6531→#35782；19087→#35056；8072→Pansa 认领中；28808→多人 PR 等。

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

# PR #19092 发布草稿（未发布，待校验）

> 状态：**DRAFT — 未发布**。本文件供多环境校验与审核，确认无误后再发布到 GitHub。
> 分支：`perf/diffusion-parallel-module-loading`（HEAD = `06e44d9661`）
> 校验记录：本地 stub 环境单测 9/9 通过；pre-commit（4 个改动文件）全绿。

---

## 一、Issue #19092 认领留言（草稿）

> Hi, I'd like to take this one. I have a patch ready on branch `perf/diffusion-parallel-module-loading`: it overlaps DiT/text-encoder weight loads with a small thread pool (platform device pinned in workers) while keeping VAE/tokenizer on the serial path. I'll open a single PR for this issue.

备注：
- 单 PR 提交，避免重蹈 Asthenia0412 的 spam 印象（其 #44127 因 8 小时开 16 个无关 PR 被关）。
- 在 issue 留言后再创建 PR。

---

## 二、PR 草稿

### 标题

```
[Perf] diffusion: Parallelize heavy component loading
```

### head / base

- head: `ArthurJianLin/sglang:perf/diffusion-parallel-module-loading`
- base: `sgl-project/sglang:main`

### 正文（草稿）

```markdown
## Summary

Overlap the DiT and text-encoder weight loads with a small thread pool so
diffusion cold-start is no longer strictly serial. VAE/tokenizer loads stay
on the serial path, and worker threads pin the platform device to avoid
cross-device weight placement.

## Changes

- `python/sglang/multimodal_gen/runtime/managers/memory_managers/component_loading_order.py`
  - New `partition_component_load_specs()`: groups load specs by component
    type and ranks DiT/text-encoder (weight-heavy) first.
- `python/sglang/multimodal_gen/runtime/pipelines_core/composed_pipeline_base.py`
  - Parallelize heavy loads in `load_component_specs()` with a small thread
    pool; VAE/tokenizer stay serial; worker threads pin platform device.
- `python/sglang/multimodal_gen/test/unit/test_component_loading_order.py`
  - 7 unit tests for ordering/partitioning logic (weight-heavy first,
    larger variants first, serial partition for single heavy module, etc.).
- `python/sglang/multimodal_gen/test/unit/test_parallel_component_loading.py`
  - 2 unit tests for parallel-vs-serial behavior of `load_component_specs()`.

## Tests

- Local stub-environment unit tests: 9/9 passed (no CUDA needed).
- `pre-commit run --files` on all 4 changed files: all hooks green.
- CI not triggered yet (needs `run-ci` label from a maintainer).
```

### 发布后需跟进

1. 若 CI 触发，盯 `multimodal-gen-test-1-gpu` / `multimodal-gen-test-2-gpu` 结果；
2. 若 maintainer 未加 `run-ci`，在 PR 评论礼貌询问；
3. 若收到 review 意见，逐条响应并更新分支。

---

## 三、校验清单（发布前逐项打勾）

- [ ] 其他环境（如 Cursor 侧）代码校验通过
- [ ] issue #19092 确认无人正在实现（Asthenia 的 PR 已关，无活跃 PR）
- [ ] 单测 9/9 通过（本地 stub 环境）
- [ ] pre-commit 4 文件全绿（含 check-chinese-characters）
- [ ] PR 标题符合 `[Type] scope: title` 规范
- [ ] commit 仅含 4 个目标文件，无无关改动
- [ ] 未在代码/测试中留下本地路径或调试痕迹

> AI生成

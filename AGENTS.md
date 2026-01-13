# Repository Guidelines

## 项目结构与模块组织

- `slime/`：核心库代码（训练、rollout、数据/参数工具等）。
- `slime_plugins/`：插件与可选扩展点。
- `tests/`：测试与 CI 相关脚本（含 `tests/ci/` 工具）。
- `tools/`：模型转换、检查与杂项工具脚本。
- `scripts/`：训练/模型配置脚本与示例启动脚本。
- `docs/`：Sphinx 文档（`docs/en/`、`docs/zh/`）。
- `docker/`、`examples/`、`imgs/`：镜像、示例与图片资源。

## 构建、测试与开发命令

- 环境（目标 Python 3.10）：`python -m venv .venv && source .venv/bin/activate && pip install -U pip`
- 安装（开发模式）：`pip install -e .`
- 常见入口：`python train.py --help`、`python train_async.py --help`（更多示例见 `examples/` 与 `docs/`）
- 代码风格（提交前必跑）：`pre-commit install`，以及 `pre-commit run --all-files --show-diff-on-failure`
- 单元测试：`pytest`（配置在 `pyproject.toml`，默认收集 `tests/`）
- 端到端/分布式用例：`python tests/test_*.py`（通常需要 GPU、模型与数据集）
- 文档构建（可选）：`cd docs && pip install -r requirements.txt && bash ./build.sh en`（或 `zh`），预览：`bash ./serve.sh en`

## 编码风格与命名约定

- 格式化：Black（`line-length = 119`）；导入排序：isort（`profile=black`）。
- 静态检查：ruff（通过 pre-commit 执行，包含自动修复）。
- 需要手动执行时：`ruff check .`、`black .`、`isort .`（以 pre-commit 结果为准）。
- 命名：Python 模块/变量使用 `snake_case`；测试文件使用 `tests/**/test_*.py`。

## 测试指南

- 优先补充可在 CPU 上运行的 pytest 测试；重型训练/集成场景使用 `tests/` 顶层脚本并明确依赖。
- pytest markers 已预置（`unit`/`integration`/`system` 等），可用 `pytest -m unit` 进行快速筛选。
- GPU CI 通常由 PR label 触发（例如 `run-ci-short`、`run-ci-fsdp`、`run-ci-megatron`）。

## Commit 与 Pull Request 规范

- Commit 信息以简短英文为主，常见格式：`feat(scope): ...`、`fix: ...`、`perf: ...`、`docs: ...`，或标签如 `[refactor]`；可附带 `(#1234)` 关联 PR/Issue。
- PR 需要：清晰的变更动机与影响范围、可复现的验证命令（如 `pytest` 或具体 `python tests/...`）、以及用户可见行为变更对应的文档/示例同步更新。

## 建议的本地开发流程

1. `pip install -e .` 安装依赖并确保 `import slime` 可用。
2. 改动尽量限制在相关模块（例如 `slime/` 或 `slime_plugins/`）并同步更新最小可验证用例。
3. 运行 `pre-commit run --all-files` 与目标测试（`pytest` 或指定 `python tests/...`）再提交。

## 安全与配置提示

- 不要提交密钥/凭据/私有数据；仓库已通过 pre-commit hook 做基础泄漏扫描。
- 不要在 PR 中提交超大文件；pre-commit 会阻止新增超过约 1000KB 的文件进入提交历史。
- 与训练/追踪相关的环境变量（如 `WANDB_API_KEY`、代理变量）请通过本地环境配置注入，不要写入代码或示例中的默认值。

# OpenKB + PyMuPDF PageIndex

基于 [VectifyAI/OpenKB](https://github.com/VectifyAI/OpenKB) 0.4.4，
依赖 [Leooone/PageIndex@pymupdf-toc](https://github.com/Leooone/PageIndex/tree/pymupdf-toc)，
为带嵌入式书签的 PDF 提供零 LLM 目录提取。

## 与上游的区别

| 文件 | 改动 |
|---|---|
| `pyproject.toml` | pageindex 依赖改为 `Leooone/PageIndex@pymupdf-toc` |
| `openkb/agent/compiler.py` | 统一日志系统（进度 + LLM 请求/响应 + 内容预览） |

其余代码与上游完全一致。

## 日志

所有日志统一在知识库根目录的 `logs/` 下：

| 文件 | 来源 | 内容 |
|---|---|---|
| `pageindex_progress.log` | PageIndex | TOC 分支选择、跳过步骤 |
| `pageindex_llm.log` | PageIndex | LLM 请求/响应 |
| `openkb_progress.log` | OpenKB | compiler 每步进度 |
| `openkb_llm.log` | OpenKB | compiler LLM 调用详情 |

## 安装

```bash
uv pip install git+https://github.com/Leooone/OpenKB.git@pymupdf-toc
```

## 效果

以 USB4 规范（839 页 / 1062 条目录）为例：

| 阶段 | 上游（LLM 调用） | 本仓库 |
|---|---|---|
| TOC 检测 | ~20 次 | 0 |
| TOC 提取 | ~5 次 | 0 |
| 页码验证 | ~1062 次 | 0 |
| 标题验证 | ~1062 次 | 0 |
| summary 生成 | 可配置 | 可配置 |

## 注意事项

- 只有带嵌入式书签的 PDF（USB4、DisplayPort、DSC 等标准文档）才能零 LLM TOC
- 扫描件或无书签 PDF 走原有 LLM 路径，不受影响
- `.env` 文件（API key）永远在本地，不进入仓库

## 限速配置

MiniMax Token Plan 等按量付费 API 即使并发=1 也可能触发 HTTP 429。
通过环境变量控制：

```bash
# .env
PAGEINDEX_MAX_CONCURRENCY=1   # 并发上限（默认 5）
PAGEINDEX_RPM_LIMIT=20        # 每分钟请求上限（默认 0=不限）
```

| 参数 | 作用 | 建议值 |
|---|---|---|
| `PAGEINDEX_MAX_CONCURRENCY` | 同时飞行中的 LLM 调用数 | 1–5 |
| `PAGEINDEX_RPM_LIMIT` | 滑动窗口限速（60s 窗口） | 20（MiniMax Token Plan） |

限速逻辑来自 [PageIndex PR #343](https://github.com/VectifyAI/PageIndex/pull/343)：
- `SlidingWindowRateLimiter`：滑动窗口限速，强制最小间隔
- `_extract_retry_delay()`：429 时解析 Retry-After 头，否则指数退避+抖动

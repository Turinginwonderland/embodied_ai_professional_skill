# Embodied AI Research Plugin

> 具身智能研究助手（团队共享版）——素材摄取、知识库裁决、按需报告。

## 它做什么

让研究/分析师同事能用自然语言：
- 把 PPT、研报、论文、政策喂进知识库（结构化提取 + 8 字段 schema）
- 与已有条目交叉对比，自动判别证实/互补/冲突（双轨并存、不二选一删除）
- 按需生成行业深度报告（仅当用户明确要求时触发）
- 联网核对最新信息（手动触发，按"最新最权威优先"裁决）

## 包含什么

3 个 skill + firecrawl MCP（联网检索能力）+ 2 个 slash 命令（可选）：

- `input-knowledge-evolution` —— 摄取/条目化/裁决
- `synthesis-output-generation` —— 取数/大纲/报告
- `firecrawl` —— 联网检索（被 §6 演进机制依赖）

Slash 命令（`commands/` 下）：
- `/ingest <file>` —— 摄取单个素材文件
- `/report <topic>` —— 生成指定主题的深度报告

## 安装

```bash
# Claude Code 内
/plugin install embodied-ai-research
```

安装后还需：
1. **设置 API key**：export `FIRECRAWL_API_KEY=fc-...`（每个使用者自己注册，不要互相借用）
2. **准备知识库目录**（二选一）：
   - 默认：在自己项目根建 `docs/outputs/knowledge_base/`
   - 自定义：设环境变量 `EMBODIED_KB_DIR=<绝对路径>`

## 知识库不是 plugin 的一部分

**重要**：本 plugin 只装"能力"（skill + MCP + commands），不装"数据"（你的 100+ 条知识库条目）。
每个团队成员各自维护自己的知识库——因为大家的课题方向不同、积累不同，不应共用同一份结论库。

## 路径约定

两个业务 SKILL.md 顶部 §0 写明了路径查找优先级（环境变量 > 默认路径 > plugin fallback）。同事无需改 skill 代码。

## 安全

- API key 走环境变量，plugin 不内嵌任何 key
- 详见 `phaseA.md` / `phaseB.md` 与各 `kb_delta_*.md` 裁决日志
- 如发现 key 泄漏：立刻去 firecrawl 吊销，并在 git 历史中替换

## 阶段二（默认搁置）

`newphase2.md` 描述了原文层向量检索落地计划。触发条件：raw_materials/ 累计 > 10 份原文素材，或单任务需 3+ 原文同参考。条件未满足前不建库、不装 chromadb。

## 反馈与裁决日志

本项目的所有冲突裁决（如 VLA vs 世界模型战略地位、形态路线、落地节奏）都记录在 `docs/outputs/knowledge_base/kb_delta_*.md`，可作为"双轨并存、互补裁决"工作流的参考案例。

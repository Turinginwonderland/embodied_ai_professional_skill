---
description: 摄取单个素材文件（PPT/研报/论文/政策）到知识库。触发 input-knowledge-evolution skill 的完整工作流：结构化提取→新旧对比→裁决→沉淀。
argument-hint: <文件路径>
---

# /ingest 命令

调用 `input-knowledge-evolution` skill 的完整工作流，处理用户指定的素材文件。

## 执行步骤

1. **读取素材**：用 Read 工具读取 `$ARGUMENTS` 指定的文件
2. **结构化提取**：按 8 字段 schema 提取事实条目（id/内容/类型/来源/时效/状态/裁决标签）
3. **新旧对比**：与 `docs/outputs/knowledge_base/kb_*.md` 已有条目交叉验证
   - 证实 → 追加关联条目
   - 互补 → 新增条目
   - 冲突 → 双轨并存，标"冲突-待裁决"或联网裁决
4. **写回知识库**：更新对应 `kb_<来源标识>.md` 与 `kb_delta_<日期>.md`
5. **报告**：列出本轮新增/更新/裁决的条目 ID 清单

## 约束

- 必须挂来源、标类型、给条目 ID（参考 SKILL.md §2 schema）
- 无充足客观证据前不把猜测当定论
- 市场数据若无统计机构/口径/年份，标"待补出处"

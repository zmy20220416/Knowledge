---
name: add-knowledge
description: 将用户明确提供的新知识整理并写入个人知识库，避免重复笔记
---

# Add Knowledge

负责将用户明确要求保存的内容加入 Knowledge。

## 工作流程

1. 先理解用户明确要求保存的内容。
2. 搜索 Knowledge 中是否存在相关笔记。
3. 如果已有相关笔记：
   - 优先更新已有笔记。
   - 不创建重复文件。
4. 如果不存在：
   - 判断所属分类。
   - 创建新的 Markdown 文件。
5. 保留重要原始信息。
6. 命令和代码必须保持可复制。
7. 不得擅自加入用户没有提供的个人经历。
8. 不得主动整理过去聊天内容，除非用户明确要求。
9. 修改完成后检查文件内容。

## 分类

Linux -> linux/
开发 -> development/
AI -> ai/
工具 -> tools/
项目 -> projects/
阅读 -> reading/
无法判断 -> inbox/

## 文件命名

使用：

- 小写
- kebab-case
- 简洁明确

例如：

linux/ubuntu-package-management.md
development/react-hooks.md
tools/ghostty.md

## 笔记结构

# 标题

## 摘要

## 核心内容

## 示例

## 注意事项

## 相关知识

## 外部来源

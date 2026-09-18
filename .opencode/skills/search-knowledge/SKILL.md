---
name: search-knowledge
description: 在个人 Knowledge 中搜索已有知识并基于相关笔记回答问题
---

# Search Knowledge

负责从个人知识库中查找和综合已有知识。

## 工作流程

1. 搜索整个 Knowledge。
2. 找到与问题最相关的 Markdown 文件。
3. 阅读相关内容。
4. 综合多个笔记。
5. 优先使用用户自己的知识库内容。
6. 如果知识库没有答案，明确说明。
7. 如果用户要求，可以进一步使用外部资料。

## 回答时区分

### Knowledge

来自个人知识库的内容。

### 推断

根据已有知识进行的合理推断。

### 外部资料

来自知识库之外的资料。

不要把推断伪装成 Knowledge 中已经存在的事实。

## 搜索范围

优先搜索：

inbox/
linux/
development/
ai/
tools/
projects/
reading/
daily/

## 重要原则

不要因为没有找到精确关键词就判断“知识库没有”。

应该使用：

- 同义词
- 相关技术名称
- 文件名
- 目录
- 上下文

进行多角度搜索。

---
title: "Markdown 写作示例"
date: 2026-10-07T09:00:00+08:00
draft: true
author: "xiangxin"
description: "一份本地写作参考，展示标题、引用、代码、表格和数学公式。"
summary: "博客写作参考：Markdown 基础、代码块、数学公式与图片。此文默认作为草稿保留。"
tags: ["写作"]
ShowToc: true
TocOpen: true
---

这是一篇**写作参考草稿**，默认不会发布到正式网站。使用 `hugo server --buildDrafts` 可以在本地预览它。

## 标题与段落

文章标题写在文件顶部的 `title` 字段中。正文从二级标题开始，用 `##` 划分主要章节、`###` 划分小节。

Markdown 支持 **加粗**、*强调* 和 `行内代码`。段落之间留一行空行，就能自然分段。

## 链接与引用

为引用的资料提供有意义的链接文字，例如 [Hugo 官方文档](https://gohugo.io/documentation/)。

> 在总结一个观点之前，先确认它试图解决的问题。

引用他人的内容时，请注明作者和来源。上面的句子只是这份写作示例中的原创示意。

## 列表与表格

写实验记录时，可以按顺序说明：

1. 要解决的问题。
2. 使用的方法与实验条件。
3. 观察到的结果。
4. 局限与下一步。

| 内容 | 建议记录什么 |
| --- | --- |
| 环境 | 软件版本与硬件条件 |
| 方法 | 关键参数与对照条件 |
| 结果 | 原始数据、观察与误差 |

## 代码块

在三个反引号后写语言名称，可以启用语法高亮。

```python
def mean(values: list[float]) -> float:
    """Return the arithmetic mean of a non-empty list."""
    if not values:
        raise ValueError("values must not be empty")
    return sum(values) / len(values)


print(mean([1.0, 2.0, 3.0]))
```

代码最好保持简短，并在正文交代输入、输出与适用条件。上面的示例计算一组数的算术平均值，对空列表抛出异常。

## 数学公式

公式会自动渲染，无需在文章顶部添加开关。行内公式使用一个美元符号包围，例如 $x_i$；独立公式使用两个美元符号包围：

$$
\bar{x} = \frac{1}{n}\sum_{i=1}^{n} x_i, \qquad n > 0
$$

这里的 $n$ 是样本数量，$x_i$ 是第 $i$ 个数，$\bar{x}$ 是它们的算术平均值。

Hugo 在构建时将公式转换为 MathML，浏览器即可直接显示。正文中如需输入普通美元符号，在 Markdown 源文件中写作 `\\$`（两个反斜杠加美元符号），避免被识别为公式边界；代码块中的美元符号保持原样。

## 图片

推荐使用文章目录组织图片：

```text
content/posts/my-note/
├── index.md
└── experiment.png
```

在 `index.md` 中写：

```markdown
![实验结果图：说明横轴、纵轴和主要发现](experiment.png)
```

提供简洁的替代文字，让图片的含义在无法显示图片时也能被理解。

## 发布前

- 确认标题、日期、摘要和标签。
- 检查链接、代码、公式和图片。
- 为参考资料补充来源。
- 将 `draft: true` 改为 `draft: false` 后再提交。

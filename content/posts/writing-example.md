---
title: "Markdown Writing Guide"
date: 2026-10-07T09:00:00+08:00
draft: true
author: "xiangxin"
description: "A local writing reference with examples of headings, quotations, code, tables, and math."
summary: "A guide to writing posts with Markdown, code blocks, math, and images. This post is kept as a draft by default."
tags: ["Writing"]
ShowToc: true
TocOpen: true
---

This is a **draft writing guide** and is not published to the live site by default. Run `hugo server --buildDrafts` to preview it locally.

## Headings and Paragraphs

Set the post title in the `title` field at the top of the file. Start headings in the body at level two: use `##` for main sections and `###` for subsections.

Markdown supports **bold text**, *emphasis*, and `inline code`. Leave a blank line between paragraphs.

## Links and Quotations

Use descriptive link text for your sources, such as the [Hugo documentation](https://gohugo.io/documentation/).

> Before summarizing an idea, identify the problem it is trying to solve.

When quoting someone else's work, name the author and link to the source. The sentence above is an original example written for this guide.

## Lists and Tables

When documenting an experiment, work through these points:

1. The problem you want to solve.
2. Your method and experimental conditions.
3. The results you observed.
4. Limitations and next steps.

| Topic | What to Record |
| --- | --- |
| Environment | Software versions and hardware specifications |
| Method | Key parameters and control conditions |
| Results | Raw data, observations, and errors |

## Code Blocks

Add the language name after the opening triple backticks to enable syntax highlighting.

```python
def mean(values: list[float]) -> float:
    """Return the arithmetic mean of a non-empty list."""
    if not values:
        raise ValueError("values must not be empty")
    return sum(values) / len(values)


print(mean([1.0, 2.0, 3.0]))
```

Keep code examples concise, and explain their inputs, outputs, and assumptions in the surrounding text. The example above calculates the arithmetic mean of a list of numbers and raises an exception for an empty list.

## Math

Math is rendered automatically; no per-post setting is needed. Wrap inline math in single dollar signs, as in $x_i$, and display equations in double dollar signs:

$$
\bar{x} = \frac{1}{n}\sum_{i=1}^{n} x_i, \qquad n > 0
$$

Here, $n$ is the number of samples, $x_i$ is the $i$-th value, and $\bar{x}$ is their arithmetic mean.

Hugo converts equations to MathML at build time so browsers can display them directly. To write a literal dollar sign in body text, use `\\$` in the Markdown source (two backslashes followed by a dollar sign) so it isn't treated as a math delimiter. Dollar signs in code blocks can stay as they are.

## Images

Keep images together with their post in a page bundle:

```text
content/posts/my-note/
├── index.md
└── experiment.png
```

In `index.md`, write:

```markdown
![Experiment results: describe the axes and the main finding](experiment.png)
```

Write concise alt text so readers can understand the image even when it cannot be displayed.

## Before Publishing

- Check the title, date, summary, and tags.
- Verify links, code, equations, and images.
- Add sources for referenced material.
- Change `draft: true` to `draft: false` before committing.

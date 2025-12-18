---
title: "Test Post: HANA SQL Window Functions (Chirpy Setup Check)"
date: 2025-12-17 09:00:00 -0500
categories: [test]
tags: [test]
toc: true
comments: true
pin: false
image:
  path: /assets/img/social-preview.png
  alt: "Abstract database / SQL illustration"
---

This is a quick test post to validate the blog setup (rendering, code blocks, images, TOC, and comments).

## What this post tests

- Markdown rendering (headings, lists, links)
- Code blocks + syntax highlighting
- Images and relative paths
- Callouts/notes
- Giscus comments (if enabled)

## Quick example: window functions

Here’s a simple example using `ROW_NUMBER()` to rank rows within a partition.

```sql
SELECT
  SalesOrg,
  CustomerId,
  SalesAmount,
  ROW_NUMBER() OVER (
    PARTITION BY SalesOrg
    ORDER BY SalesAmount DESC
  ) AS rn
FROM Sales;
```

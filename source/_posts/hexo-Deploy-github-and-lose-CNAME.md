---
title: 'hexo: Deploy github and lose CNAME'
date: 2025-07-04 22:59:59
tags:
  - hexo
  - github
---

# Issue

I deployed my hexo blog to github pages, but I lost the custom domain in github pages.

I tried to re-add the custom domain in the repository settings and I found this operation will add CNAME file under the repository root directory.

# Solution

So I copy this file(or create a new one and only put custom domain into this file) to the `source` directory of my hexo blog, and then commit and push it to the repository.

Everything works fine!

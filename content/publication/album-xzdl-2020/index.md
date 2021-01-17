---
# Documentation: https://wowchemy.com/docs/managing-content/

title: '移动应用中相册排序优化方法'
subtitle: ''
summary: ''
authors:
- 赵化
- 金小俊
tags: []
categories: []
date: '2020-02-01'
lastmod: 2020-12-29T01:18:33Z
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ''
  focal_point: ''
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
publishDate: '2020-12-29T01:18:33.339938Z'
publication_types:
- '2'
abstract: '相册是当前移动应用中的重要功能模块，如何快速的加载和展示相册图片对于用户体验的提升具有显著意义。本文提出了一种基于分批加载和取尾排序的相册枚举及排序优化方法。当用户相册中图片较多时，枚举每隔一定数量的图片后即抛出排序显示，由于排序耗时大于枚举耗时，在每批排序完成后取最后一批枚举的图片再行排序，即取尾排序。实践表明采用本文的优化方法后，相册排序显示效率提升明显，具有极大的应用价值。'
publication: "**写真地理**"
---

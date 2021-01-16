---
# Documentation: https://wowchemy.com/docs/managing-content/

title: '基于机器视觉的除草机器人杂草识别'
subtitle: ''
summary: ''
authors:
- 金小俊
- 陈勇
- 侯学贵
- 郭伟斌
tags: []
categories: []
date: '2012-04-01'
lastmod: 2020-12-29T01:18:33Z
featured: false
draft: false

links:
- name: 核心期刊

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
abstract: '根据杂草颜色特征，提出了新的图像分割算法，在RGB空间直接将杂草从土壤背景中分割出来。首先顺序搜索图像中每一个像素点，如果当前像素RGB值中G>R且G>B，则将该像素值置1（杂草），否则为0（背景），从而完成图像分割。然后采用8邻域消除孤立点，并确定杂草区域位置。利用Visual C++开发了除草机器人杂草识别软件，设计了除草机器人结构模型。试验表明，该分割算法实时性好，可有效识别出杂草，并能够适应户外自然光变化。除草机器人机械臂能够准确定位，完成除草动作。'
publication: "*山东科技大学学报(自然科学版)*"
doi: 10.16452/j.cnki.sdkjzk.2012.02.016
---

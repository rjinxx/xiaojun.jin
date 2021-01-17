---
# Documentation: https://wowchemy.com/docs/managing-content/

title: '基于Bayes与SVM的玉米彩色图像分割新算法'
subtitle: ''
summary: ''
authors:
- 程玉柱
- 陈勇
- 车军
- 金小俊
tags: []
categories: []
date: '2012-07-01'
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
abstract: '提出了基于Bayes（贝叶斯）与SVM（支持向量机）的玉米彩色图像分割新算法。统计原始RGB图像中的玉米和土壤背景的均值向量和协方差矩阵，利用正态分布的Bayes分类器计算每个像素的目标和背景的判别函数值，用训练好的SVM对判别函数值进行分类，实现彩色图像分割。Matlab试验结果表明，该方法能够实现高光强下彩色图像分割，平均错分率为9.1%，平均漏分率为12.0%，平均相似度为80.8%。'
publication: "`江苏农业科学`"
doi: 10.15889/j.issn.1002-1302.2012.07.058
---

---
# Documentation: https://wowchemy.com/docs/managing-content/

title: 'A deep learning-based method for classification, detection, and localization of weeds in turfgrass'
subtitle: ''
summary: ''
authors:
- Xiaojun Jin
- Muthukumar Bagavathiannan
- Patrick E. McCullough
- Yong Chen
- Jialin Yu
tags: []
categories: []
date: '2022-07-28'
lastmod: 2020-12-29T01:18:33Z
featured: true
draft: false

links:
- name: SCI
  url: ''
- name: JCR Q1
  url: ''
- name: IF 4.462
  url: ''

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
abstract: 'Precision spraying of synthetic herbicides can reduce herbicide input. Previous research demonstrated the effectiveness of using image classification neural networks for detecting weeds growing in turfgrass, but did not attempt to discriminate weed species and locate the weeds on the input images. The objectives of this research were to: (i) investigate the feasibility of training deep learning models using grid cells (subimages) to detect the location of weeds on the image by identifying whether or not the grid cells contain weeds; and (ii) evaluate DenseNet, EfficientNetV2, ResNet, RegNet and VGGNet to detect and discriminate multiple weed species growing in turfgrass (multi-classifier) and detect and discriminate weeds (regardless of weed species) and turfgrass (two-classifier). The VGGNet multi-classifier exhibited an F1 score of 0.950 when used to detect common dandelion and achieved high F1 scores of ≥0.983 to detect and discriminate the subimages containing dallisgrass, purple nutsedge and white clover growing in bermudagrass turf. DenseNet, EfficientNetV2 and RegNet multi-classifiers exhibited high F1 scores of ≥0.984 for detecting dallisgrass and purple nutsedge. Among the evaluated neural networks, EfficientNetV2 two-classifier exhibited the highest F1 scores (≥0.981) for exclusively detecting and discriminating subimages containing weeds and turfgrass. The proposed method can accurately identify the grid cells containing weeds and thus precisely locate the weeds on the input images. Overall, we conclude that the proposed method can be used in the machine vision subsystem of smart sprayers to locate weeds and make the decision for precision spraying herbicides onto the individual map cells. © 2022 Society of Chemical Industry.'

# Summary. An optional shortened abstract.
summary: The grid cells were marked as spraying areas if the inference result indicated that they contained weeds. Only those nozzles corresponding to those cells infested with weeds were turned on, thus realizing a smart sensing and spraying system.

publication: "Pest Management Science"
doi: 10.1002/ps.7102
---

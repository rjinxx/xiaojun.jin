---
# Documentation: https://wowchemy.com/docs/managing-content/

title: 'Intra-row weed recognition using plant spacing information in stereo images'
subtitle: ''
summary: ''
authors:
- Yong Chen
- admin
- Lie Tang
- Jun Che
- Yanxia Sun
- Jun Chen
tags: []
categories: []
date: '2013-07-15'
lastmod: 2020-12-29T01:18:33Z
featured: false
draft: false

links:
- name: EI

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
- '1'
abstract: 'Crop/weed recognition is a crucial step for selective herbicide application. A machine vision based sensing system was developed to detect intra-row weeds when crops were at their early growth stages. The proposed methods used color feature to extract vegetation from the background, whilst height and plant spacing information analysis techniques were applied to discriminate between crops and weeds. Firstly the identification of the weeds that were lower than crops was done by a height-based segmentation method using a stereo vision system. During the stereo matching process, correspondence search was performed on edged stereo images and disparity calculation was applied only to the edge pixels. This strategy could largely reduce the correspondence search range, thereby enhanced the weed recognition speed and accuracy. Afterwards, the higher weeds were distinguished from the crops by utilizing plant spacing characters. The histogram of plant pixels and their peak position were calculated from each pixel row of the segmented disparity image. Then plant centers were located and each weed region was further extracted based on the interplant distance in a row.'
publication: "`2013 ASABE Annual International Meeting`"
doi: 10.13031/aim.20131592292
---

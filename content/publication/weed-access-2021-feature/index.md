---
# Documentation: https://wowchemy.com/docs/managing-content/

title: 'Weed Identification Using Deep Learning and Image Processing in Vegetable Plantation'
subtitle: ''
summary: ''
authors:
- admin
- Jun Che
- Yong Chen
tags: []
categories: []
date: '2021-01-06'
lastmod: 2020-12-29T01:18:33Z
featured: true
draft: false

links:
- name: SCI
  url: ''
- name: EI
  url: ''
- name: SCI 2区
  url: ''
- name: IF 3.745
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
abstract: 'Weed identification in vegetable plantation is more challenging than crop weed identification due to their random plant spacing. So far, little work has been found on identifying weeds in vegetable plantation. Traditional methods of crop weed identification used to be mainly focused on identifying weed directly; however, there is a large variation in weed species. This paper proposes a new method in a contrary way, which combines deep learning and image processing technology. Firstly, a trained CenterNet model was used to detect vegetables and draw bounding boxes around them. Afterwards, the remaining green objects falling out of bounding boxes were considered as weeds. In this way, the model focuses on identifying only the vegetables and thus avoid handling various weed species. Furthermore, this strategy can largely reduce the size of training image dataset as well as the complexity of weed detection, thereby enhancing the weed identification performance and accuracy. To extract weeds from the background, a color index-based segmentation was performed utilizing image processing. The employed color index was determined and evaluated through Genetic Algorithms (GAs) according to Bayesian classification error. During the field test, the trained CenterNet model achieved a precision of 95.6%, a recall of 95.0%, and a F1 score of 0.953, respectively. The proposed index -19R + 24G -2B ≥ 862 yields high segmentation quality with a much lower computational cost compared to the wildly used ExG index. These experiment results demonstrate the feasibility of using the proposed method for the ground-based weed identification in vegetable plantation.'

# Summary. An optional shortened abstract.
summary: Traditional methods of crop weed identification used to be mainly focused on identifying weed directly; however, there is a large variation in weed species. This paper proposes a new method in a contrary way, which combines deep learning and image processing technology.

publication: "IEEE Access"
doi: 10.1109/ACCESS.2021.3050296
---

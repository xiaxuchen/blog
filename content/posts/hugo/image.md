---
title: "Image"
date: 2024-02-09T15:40:30+08:00
draft: true
---

# Hugo 中使用图片

在 Hugo 中，content 目录下的图片是无法被引用的，我们如果需要引用图片，需要在根目录创建 static 目录，并创建下级文件夹，如 images，然后将图片放入 images 文件夹中。

## 误区

我们会认为在 assets/images 文件夹下面可以存放文件，但是事实是其只能作为文章封面被引用。

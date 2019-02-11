---
title: Pdf.js
date: 2017-12-06 10:49:28
tags:
  - 前端
---

## 下载

[Pre-built](https://mozilla.github.io/pdf.js/getting_started/)

## 集成

将下载好的包放你项目的里,nginx配置
```
  location / {
    # 没找到对应html返回index.html
    try_files $uri /index.html =404;
  }
```

## 访问

根据你的路径访问web/viewer.html?file=${yourpdf}

---
title: "[memo] HugoのSummaryが長くなるのを短くなるように設定する"
description: ""
date: 2018-03-13T23:46:56+09:00
categories:
  - 小ネタ
tags:
  - hugo
draft: false
---

デフォルトだと日本語文章でHugoの単語分割が正しく動かないので `hasCJKLanguage = true` をconfig.tomlで設定すればOK

https://gohugo.io/content-management/summaries/#summary-splitting-options

---
title: "[memo] 普段記事書くときに使うhugoの操作"
description: "[memo] 普段記事書くときに使うhugoの操作"
date: 2018-03-11T17:27:37+09:00
categories:
  - short-memo
tags:
  - hugo
draft: false
---

## ビルド

```
$ hugo
```

## 新ページの作成

```
$ hugo new path_to/page_name.md
```

これで `content/path_to/page_name.md` が生成される

## ローカル上でサーバ立ち上げてプレビュー

```
$ hugo server
```

## デプロイ

このブログはGitHub+Netlifyで運用している。
GitHubのmasterにpushすればNetlifyでビルドされ、公開される。

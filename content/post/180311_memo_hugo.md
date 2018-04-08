---
title: "[memo] 普段記事書くときに使うhugoの操作"
description: "[memo] 普段記事書くときに使うhugoの操作"
date: 2018-03-11T17:27:37+09:00
categories:
  - 小ネタ
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

(2018/3/21追記)なお、 `hugo server -D` とするとドラフト状態の記事も表示される。

## デプロイ

このブログはGitHub+Netlifyで運用している。
GitHubのmasterにpushすればNetlifyでビルドされ、公開される。

## 運用上の注意

(2018/3/21追記)テーマを変更した際には、netlify側のビルドコマンド中のテーマ指定オプションも忘れずに変更すること

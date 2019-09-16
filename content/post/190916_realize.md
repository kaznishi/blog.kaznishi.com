---
title: "Goのサーバ開発の心強い味方, realize を導入する"
description: "realizeの導入について簡単に書きました。"
date: 2019-09-16T22:36:00+09:00
categories:
  - 小ネタ
tags:
  - Go
draft: false
---

Goでサーバ開発をしていると動作確認で go run のかけ直しをすることが多々あると思う。
勝手にリロードされてくれたら…。

それを解決するツールが存在する。 realizeである。  
https://github.com/oxequa/realize

導入には `go get github.com/oxequa/realize` を行う。

また、プロジェクトルートに下記のような設定ファイル(`.realize.yaml`)を設置する。

```
settings:
  legacy:
    force: false
    interval: 0s
schema:
- name: realize_sample
  path: .
  commands:
    build:
      status: true
      method: go build -o ./bin/realize_sample_server ./cmd/realize_sample_server/main.go
    run:
      status: true
      method: ./bin/realize_sample_server
  watcher:
    extensions:
    - go
    paths:
    - /
    ignored_paths:
    - .git
    - .realize
```

サンプルプロジェクトを作成したので参照してほしい。  
https://github.com/kaznishi/realize_sample  
このサンプルプロジェクトをcloneして `realize start` すれば、サーバ(`realize_sample_server`)が立ち上がる。

ソースコードの変更の保存が検知行われる度にbuildが走ってその成果物のサーバプログラムが自動で起動される。  
httpサーバのアプリケーションだけではなくgRPCのサーバでも大丈夫(なはず。確かめてないけど)。

設定ファイルの項目の細かい中身は[realize GitHubの記述](https://github.com/oxequa/realize#config-sample)を参照されたし。

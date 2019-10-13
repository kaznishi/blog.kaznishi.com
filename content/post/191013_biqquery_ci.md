---
title: "CircleCIでBigQueryのテストを継続的に回す"
description: "CircleCIでBigQueryのテストが回るように組んでみました。"
date: 2019-10-13T16:30:00+09:00
categories:
  - 小ネタ
tags:
  - Go
  - BigQuery
draft: false
---

[前回の記事](https://blog.kaznishi.com/post/190923_bigquery_test_with_testdata/)でBigQueryのテストを記述して実際にBigQueryにデータを流し込んでテストを回すコードを書いてみましたが、今回はCircleCIでそのテストを実行する環境を構築してみたいと思います。

CircleCIで以下の工程を実行するとします。

1. (Goソースコードのコンパイル)
2. BigQueryデータセットの作成
3. テスト実行
4. BigQueryデータセットの削除

1.についてはテストを回す前に前提としてコンパイルが通ることのチェックを行うために入れています。実際の現場ではコンパイルが通ることの他、各種Lintが通ることのチェックを行ったりもするでしょうが、今回は割愛します。

2.ではGitのコミットハッシュをデータセット名に組み込んだデータセットを作成します。このようにするのは、複数人で開発をしている状況でCIが並列に走ったときに別々のデータセットを使ってテストをすることを保証するためです。

3.では実際にテストを実行します。1.と2.の後に実行されるようにworkflowを設定します。

4.はテストで使用したデータセットを削除します。具体的には、2日前に作成されたテストデータセットをまとめて削除するworkflowを組んでおき、schedule実行させます。テスト実行の直後にデータセット削除しないのは、テスト実行のworkflowが途中でコケたときのハンドリングが面倒だったからです。

では、これから具体的な `.circle/config.yml` ファイルを示していきます。
ソースコードは[GitHubリポジトリ](https://github.com/kaznishi/bq_circle_ci_example)にあげてあります。

## 1. Goソースコードのコンパイル

```

executors:
  golang:
    docker:
      - image: golang:1.13-stretch
        environment:
          GOOGLE_APPLICATION_CREDENTIALS: /etc/google/application_default_credentials.json
          GO111MODULE: "on"

jobs:
  build:
    executor: golang
    working_directory: /go/src/github.com/kaznishi/bq_circle_ci_example
    steps:
      - checkout
      - run:
          name: check compile
          command: make build

```

executorsで実行環境 `golang` を定義し、 `build` ジョブとしてソースコードをcheckoutしてmake buildするだけの単純な構成です。  
なお、環境変数 `GOOGLE_APPLICATION_CREDENTIALS` は後述する `test` ジョブを実行する際にBigQueryアクセスするのに必要なので入れてあります。

## 2. BigQueryデータセットの作成

```
executors:
  ...
  gcloud:
    docker:
      - image: google/cloud-sdk:latest
        environment:
          GOOGLE_APPLICATION_CREDENTIALS: /etc/google/application_default_credentials.json

...

commands:
  setup_credential:
    steps:
      - run:
          name: Setup Credential
          command: |
            mkdir -p /etc/google/
            echo "${CI_GOOGLE_SERVICE_KEY_BASE64}" | base64 --decode --ignore-garbage > "${GOOGLE_APPLICATION_CREDENTIALS}"
  setup_bq_envvar:
    steps:
      - run:
          name: set env BQ_DATASET
          command: echo "export BQ_DATASET=${BQ_DATASET_PREFIX}_`date '+%Y%m%d'`_${CIRCLE_SHA1:0:7}" >> $BASH_ENV

...

jobs:
  ...
  bq_make_dataset:
    executor: gcloud
    steps:
      - checkout
      - setup_credential
      - setup_bq_envvar
      - run:
          name: Authorization
          command: gcloud auth activate-service-account --key-file ${GOOGLE_APPLICATION_CREDENTIALS}
      - run:
          name: make dataset
          command: make bq/setup

```

BigQueryデータセットを作成する `bq_make_dataset` ジョブです。  
実行環境として executor `gcloud` を定義しました。このexecutorには imageとして `google/cloud-sdk` を使用します。  
command `setup_credential` では、環境変数 `CI_GOOGLE_SERVICE_KEY_BASE64` から、cloud-sdkの認証にCredentialを読み込みます。  
環境変数 `CI_GOOGLE_SERVICE_KEY_BASE64` には、事前に作成したGCPのService Accountのjson認証キーをBASE64エンコードした文字列を登録してあります(※1)。  
GCPのService Accountには `BigQuery データオーナー` と `BigQuery ジョブユーザー` のロールを付与してあります。  
command `setup_bq_envvar` では、環境変数 `BQ_DATASET` にテスト用のデータセットの名前を設定します。データセットの名前のルールは `${BQ_DATASET_PREFIX}_yyyymmdd形式の日付_gitのコミットハッシュ` としました。日付を含めておくことで後日、日付からデータセットを探してまとめて削除することが可能になります。また、Gitのコミットハッシュを入れておくことで複数人の開発において別々にCIを実行することができます。  
環境変数 `BQ_DATASET_PREFIX` にはこのテストに使用するということが判別できる任意の自由な文字列を登録しておきます。  
job `bq_make_dataset` のステップとしては、`setup_credential` `setup_bq_envvar` を実行した後、`gcloud auth` で認証を行ってからデータセット作成を実行しています。

なお、 `make bq/setup` の中身は下記のようになっています。

Makefile
```
.PHONY: bq/setup
bq/setup: ## create BigQuery Dataset And Tables
	./scripts/bq_clean
	./scripts/bq_setup
```

./scripts/bq_clean
```
#!/bin/bash -eu

bq rm -r -f -d $GCP_PROJECT:$BQ_DATASET

```

./scripts/bq_setup
```
#!/bin/bash -eu

bq --location=US mk --dataset $GCP_PROJECT:$BQ_DATASET

bq mk \
--table \
$GCP_PROJECT:$BQ_DATASET.students \
id:INTEGER,name:STRING,group:STRING

bq mk \
--table \
$GCP_PROJECT:$BQ_DATASET.scores \
student_id:INTEGER,date:DATE,score:INTEGER
```

## 3. テスト実行

```
jobs:
  ...
  test:
    executor: golang
    working_directory: /go/src/github.com/kaznishi/bq_circle_ci_example
    steps:
      - checkout
      - setup_credential
      - setup_bq_envvar
      - run:
          name: test
          command: make test

...

workflows:
  version: 2
  default_flow:
    jobs:
      - build
      - bq_make_dataset
      - test:
          requires:
            - build
            - bq_make_dataset


```

job `test` を定義しました。また、workflow `default_flow` の中で `build` および `bq_make_dataset` が実行されたあとに `test` が実行されるように組みました。

ここまで設定すると、GitHubの方で改修がpushされると、その度に専用のテストデータセットが作られ、それを使用したテストが実行されるようになります。

## 4. BigQueryデータセットの削除

テストで作成されたデータセットはどんどん溜まっていくので、2日前の日付で作成されたデータセットを削除するという定期ジョブを毎日回すようにします。
削除対象を1日前にしていないのは、テストが回っている間に日付をまたぐとテストで使用しているデータセットが消えてしまうのを避けるためです。

```
jobs:
  ...
  ## for scheduled job
  bq_cleanup_2daysago_datasets:
    executor: gcloud
    steps:
      - checkout
      - setup_credential
      - run:
          name: Authorization
          command: gcloud auth activate-service-account --key-file ${GOOGLE_APPLICATION_CREDENTIALS}
      - run:
          name: clear 2 days ago datasets
          command: bq ls --project_id=$GCP_PROJECT --datasets | grep ${BQ_DATASET_PREFIX}_`date -d '2 days ago' '+%Y%m%d'` | xargs ./scripts/bq_delete_datasets || true

workflows:
  ...
  cleanup_test_dataset:
    triggers:
      - schedule:
          cron: "0 0 * * *"
          filters:
            branches:
              only: master
    jobs:
      - bq_cleanup_2daysago_datasets

```

なお、jobの中で呼び出している `./scripts/bq_delete_datasets` の中身は下記のようになっています。

```
#!/bin/bash -eu

for each_dataset in $@
do
    echo "remove $each_dataset..."
    bq rm -r -f -d $GCP_PROJECT:$each_dataset
done
```

これで2日前の日付が入った名前に含まれるデータセットが毎日午前0時(UTC)に削除されるようになります。

---

※1 一言で書いてしまいましたが、詳しく書いた方がいいような気がしたので、また記事化しようと思います。

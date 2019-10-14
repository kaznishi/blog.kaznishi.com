---
title: "Elasticsearchの勉強を始めました"
description: "環境準備と、公式リファレンスのGetting Startedのクエリを動かすところまで。"
date: 2019-10-14T16:00:00+09:00
categories:
  - 勉強の記録
tags:
  - Elasticsearch
draft: false
---

表題の通り、Elasticsearchの勉強を始めました。環境準備と公式リファレンスのGetting startedのクエリを動かすところまでやったので記録として残しておきます。

## 環境準備

公式のDockerイメージを使ってさくっと準備しました。  
Elasticsearchのバージョンは現時点で最新の7.4.0を使用します。  
下記のdocker-compose.ymlを書いて `docker-compose up` すれば準備完了。

```
version: '3'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.4.0
    environment:
      - discovery.type=single-node
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - 9200:9200

volumes:
  esdata:
```

ElasticsearchではHTTPのAPIエンドポイントが提供されており、JSONを投げつけて命令することが可能です。
HTTPのエンドポイントを叩くクライアントとして、今回はcurlを使用しました。
また、レスポンスもJSONで返ってくるので、見やすくするために[jq](https://stedolan.github.io/jq/)も使用します。

## insert

`Elasticsearchのホスト:ポート/インデックス名/_doc/ID` に対してPUTメソッドでドキュメントの内容のJSONを投げるとinsert(存在する場合は上書き処理)の命令になります。
(ちなみに、DBでテーブルとレコードにあたるものがElasticsearchではそれぞれ"インデックス"と"ドキュメント"になります。)

```
curl -X PUT -H "Content-Type: application/json" localhost:9200/test_index/_doc/1 \
-d @- <<EOF | jq .
{
    "name": "hogehoge",
    "score": 85
}
EOF
```

## get

`Elasticsearchのホスト:ポート/インデックス名/_doc/ID` に対してGETメソッドでリクエストすると、ドキュメントを取得することができます。

```
curl localhost:9200/test_index/_doc/1 | jq .
```

## delete document

`Elasticsearchのホスト:ポート/インデックス名/_doc/ID` に対してDELETEメソッドでリクエストすると、ドキュメントを削除することができます。

```
curl -X DELETE -H "Content-Type: application/json" localhost:9200/test_index/_doc/1 | jq .
```

## delete index

`Elasticsearchのホスト:ポート/インデックス名` に対してDELETEメソッドでリクエストすると、指定されたインデックスを削除することができます。

```
curl -X DELETE -H "Content-Type: application/json" localhost:9200/test_index | jq .
```

## bulk insert

ここから[公式リファレンス](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/getting-started-index.html#getting-started-batch-processing)をなぞっていきます。
次にElasticsearchのメイン機能ともいえるsearchのクエリを投げていくのですが、その前にテスト用のデータを投入しておきます。
データ投入には[Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/docs-bulk.html)を使用します。
投入するデータとして、[Elasticsearchリポジトリに用意されているaccounts.json](https://github.com/elastic/elasticsearch/blob/v7.4.0/docs/src/test/resources/accounts.json)をそのまま使わせてもらいました。

```
curl -H "Content-Type: application/json" -X POST "localhost:9200/test_accounts/_bulk?pretty&refresh" \
--data-binary "@accounts.json" | jq .
```

リクエストパラメータの `pretty` と `refresh` が謎だったので調べました。

`pretty` はレスポンスのJSONの結果を整形するパラメータです。(これがあったらjqいらないようですね…。)

`refresh` は処理の結果を検索に反映するタイミングを指定するためのものです。何も指定しなければ結果を即座に検索に反映する挙動をします。パフォーマンスに影響がないことをよく確認のうえ指定するように、とのことです。個人のローカルでElasticsearchを動かしている分には気にせずに使ってよいと思いますが、本番環境や他人と共有している環境などでは注意が必要でしょう。詳しくは[こちら](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/docs-refresh.html)。

## search

すべてのドキュメントを対象に、ソートをかけたものを返します。
なお、返ってくるのは先頭から10件となります。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "query": {
        "match_all": {}
    },
    "sort": [
        { "account_number": "asc" }
    ]
}
EOF
```

fromとsizeを指定することで、何件目から何件分を表示するのかをコントロールすることができます。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "query": {
        "match_all": {}
    },
    "sort": [
        { "account_number": "asc" }
    ],
    "from": 10,
    "size": 10
}
EOF
```

"address"フィールドに"mill"もしくは"lane"を含むドキュメントを検索します。`match`を使用します。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "query": {
        "match": {
            "address": "mill lane"
        }
    }
}
EOF
```

"mill lane"という固まりで検索したい場合には`match_phrase`を使用します。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "query": {
        "match_phrase": {
            "address": "mill lane"
        }
    }
}
EOF
```

"age"が"40"かつ、"state"が"ID"ではないもの、という組み合わせ条件を表現するには、`bool`と`must`,`must_not`を使用します。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "query": {
        "bool": {
            "must": [
                {"match": {"age": "40"}}
            ],
            "must_not": [
                {"match": {"state": "ID"}}
            ]
        }
    }
}
EOF
```

下記は、`filter`句を使って"balance"が20000以上30000以下のドキュメントを絞り込み検索します。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "query": {
        "bool": {
            "must": { "match_all": {} },
            "filter": {
                "range": {
                    "balance": {
                        "gte": 20000,
                        "lte": 30000
                    }
                }
            }

        }
    }
}
EOF
```

次に示すのはAggregationのsearchです。
"state"ごとにいくつのaccountsが存在しているのかを集計するクエリは以下です。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "size": 0,
    "aggs": {
        "group_by_state": {
            "terms": {
                "field": "state.keyword"
            }
        }
    }
}
EOF
```

グループごとに"balance"の平均の情報を加えて表示したい場合は下記です。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "size": 0,
    "aggs": {
        "group_by_state": {
            "terms": {
                "field": "state.keyword"
            },
            "aggs": {
                "average_balance": {
                    "avg": {
                        "field": "balance"
                    }
                }
            }
        }
    }
}
EOF
```

並び順はグループごとのドキュメント数降順がデフォルトですが、"balance"の平均の順番通りに結果を並べ替えしたい場合は下記のように書きます。

```
curl -H "Content-Type: application/json" localhost:9200/_search \
-d @- <<EOF | jq .
{
    "size": 0,
    "aggs": {
        "group_by_state": {
            "terms": {
                "field": "state.keyword",
                "order": {
                    "average_balance": "desc"
                }
            },
            "aggs": {
                "average_balance": {
                    "avg": {
                        "field": "balance"
                    }
                }
            }
        }
    }
}
EOF
```

---

ここまででGetting startedは終わりですが、Elasticsearchのクエリ表現はElasticの通り柔軟で、かなり奥が深そうです。
まだ表面しかなぞれていないので、さらに勉強を進めたいと思います。

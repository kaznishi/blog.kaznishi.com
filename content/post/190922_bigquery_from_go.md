---
title: "GoでBigQueryのデータを読み出す"
description: "GoでBigQueryのデータを読み出す方法について簡単に書きました。"
date: 2019-09-22T23:13:00+09:00
categories:
  - 小ネタ
tags:
  - Go
  - BigQuery
draft: false
---


今回はGoでBigQueryからデータを読み出す方法について紹介する。

サンプルとして、このようなデータが入ったBigQueryのテーブルがあるとする。

テーブル名: `test`

 フィールド名 | タイプ | モード 
---|---|---
 string_column | STRING | NULLABLE 
 integer_column | INTEGER | NULLABLE 
 date_column | DATE | NULLABLE 
 timestamp_column | TIMESTAMP | NULLABLE 

データ

 string_column | integer_column | date_column | timestamp_column 
---|---|---|---
hoge1 | 1 | 2019-09-01 | 2019-09-01 00:00:00 UTC 
hoge2 | 2 | 2019-09-02 | 2019-09-02 00:00:00 UTC 
hoge3 | 3 | 2019-09-03 | 2019-09-03 00:00:00 UTC 

Goからデータを読み出すのに`cloud.google.com/go/bigquery`パッケージを使う。

BigQueryに対してクエリを投げるには、まず下記のようにclientを準備しておく。

```
ctx := context.Background()

client, err := pkgbigquery.NewClient(ctx, projectID)
if err != nil {
    // error handling
}
```

そしてクエリを記述する。

```
q := "SELECT " +
    "string_column as StringColumn,  " +
    "integer_column as IntegerColumn,  " +
    "date_column as DateColumn, " +
    "timestamp_column as TimestampColumn " +
    "FROM `" + projectID + "." + dataset + ".test` "
```

実際にクエリを投げる`Read`メソッドを実行する。[Docの通り](https://godoc.org/cloud.google.com/go/bigquery#example-Query-Read)である。

```
it, err := client.Query(q).Read(ctx)
if err != nil {
    return nil, err
}
```

得られたイテレータからデータを取得する。これも[Docの通り](https://godoc.org/cloud.google.com/go/bigquery#example-RowIterator-Next-Struct)。  

```
var hoges []*hoge
for {
    var h hoge
    err := it.Next(&h)
    if err == iterator.Done {
        break
    }
    if err != nil {
        return nil, err
    }
    hoges = append(hoges, &h)
}
```

BigQueryデータを構造体へマッピングするのは`it.Next(&h)`の部分が行っている。  
ちなみに`hoge`は下記のような構造体である。

```
type hoge struct {
	StringColumn    string
	IntegerColumn   int64
	DateColumn      civil.Date
	TimestampColumn time.Time
}
```

ただし、注意点として、[**構造体のフィールド名とBigQueryのデータのカラム名が同じでないといけない**](https://github.com/googleapis/google-cloud-go/blob/v0.46.3/bigquery/iterator.go#L80)。  
今回のサンプルのテーブルではカラム名が`string_column`などのようにスネークケースであるため、クエリ中で `string_column as StringColumn` として記述することでBigQuery側のカラム名をGoの構造体のフィールド名に合わせている。

また、BigQueryのデータ型とGoの構造体のフィールドのデータ型との対応については[ライブラリ中のコメントに記述がある](https://github.com/googleapis/google-cloud-go/blob/v0.46.3/bigquery/iterator.go#L83,L94)。下記はその引用である。

```
STRING      string
BOOL        bool
INTEGER     int, int8, int16, int32, int64, uint8, uint16, uint32
FLOAT       float32, float64
BYTES       []byte
TIMESTAMP   time.Time
DATE        civil.Date
TIME        civil.Time
DATETIME    civil.DateTime
```



以上、説明してきたものをすべてまとめて、テーブルのレコードをすべて取得して中身をPrintlnで出力するコードは、下記になる。

```
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	pkgbigquery "cloud.google.com/go/bigquery"
	"cloud.google.com/go/civil"
	"google.golang.org/api/iterator"
)

var projectID string
var dataset string

func init() {
	projectID = os.Getenv("GCP_PROJECT")
	dataset = os.Getenv("BQ_DATASET")
}

func main() {
	ctx := context.Background()

	client, err := pkgbigquery.NewClient(ctx, projectID)
	if err != nil {
		log.Fatal(err)
	}

	hoges, err := queryfunc(ctx, client)
	if err != nil {
		log.Fatal(err)
	}

	for _, v := range hoges {
		fmt.Println(*v)
	}
}

type hoge struct {
	StringColumn    string
	IntegerColumn   int64
	DateColumn      civil.Date
	TimestampColumn time.Time
}

func queryfunc(ctx context.Context, client *pkgbigquery.Client) ([]*hoge, error) {
	q := "SELECT " +
		"string_column as StringColumn,  " +
		"integer_column as IntegerColumn,  " +
		"date_column as DateColumn, " +
		"timestamp_column as TimestampColumn " +
		"FROM `" + projectID + "." + dataset + ".test` "

	it, err := client.Query(q).Read(ctx)
	if err != nil {
		return nil, err
	}

	var hoges []*hoge
	for {
		var h hoge
		err := it.Next(&h)
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, err
		}
		hoges = append(hoges, &h)
	}

	return hoges, nil
}
```

---

### まとめ

- GoからBigQueryのデータを読み出すのは簡単
- `cloud.google.com/go/bigquery`パッケージを使う
- 構造体へのマッピングもDocの例に従えばできる
- 構造体のデータ型とBigQueryのデータ型の対応はソースコード中に記載がある
- 構造体のフィールド名がBigQuery側と合っていないとマッピングされないので注意

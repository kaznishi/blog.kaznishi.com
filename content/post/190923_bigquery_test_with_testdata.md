---
title: "テスト時にGoでBigQueryにテストデータを書き出す"
description: "Goでfixtureテストを行う方法を書きました。"
date: 2019-09-23T15:46:00+09:00
categories:
  - 小ネタ
tags:
  - Go
  - BigQuery
draft: false
---

前回の記事ではGoでBigQueryのデータを読み出す処理の書き方を述べた。
今回は、Goで記述した読み出し処理のテストコードの記述について書く。

今回のサンプルとして用意したのは下記のような定義の2つのテーブルである。

テーブル: students

フィールド名 | タイプ | モード 
---|---|---
id | INTEGER | NULLABLE 
name | STRING | NULLABLE 
group | STRING | NULLABLE 

テーブル: scores

フィールド名 | タイプ | モード 
---|---|---
student_id | INTEGER | NULLABLE 
date | DATE | NULLABLE 
score | INTEGER | NULLABLE 

studentsのマスタテーブルと、試験の成績が記録されたscoresテーブルである。
今回実装することにしたのは、生徒のgroup毎の成績平均をテスト日を引数として取得する関数 `getGroupAvgScoresByDate` であるとしよう。

`getGroupAvgScoresByDate` および、その結果の構造体 `groupAvgScore` は下記のような実装になる。
(clientの準備、projectID、datasetの初期化については[前回記事](https://blog.kaznishi.com/post/190922_bigquery_from_go/)と同様にinit,mainで行われているとする)

```
type groupAvgScore struct {
	Group    string
	AvgScore float64
}

func getGroupAvgScoresByDate(ctx context.Context, client *bigquery.Client, date civil.Date) ([]*groupAvgScore, error) {
	q := "SELECT " +
		"st.group,  " +
		"avg(sc.score) as AvgScore  " +
		"FROM `" + projectID + "." + dataset + ".scores` as sc " +
		"INNER JOIN `" + projectID + "." + dataset + ".students` as st ON(st.id = sc.student_id) " +
		"WHERE sc.date = '" + date.String() + "' " +
		"GROUP BY st.group "

	it, err := client.Query(q).Read(ctx)
	if err != nil {
		return nil, err
	}

	var scores []*groupAvgScore
	for {
		var s groupAvgScore
		err := it.Next(&s)
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, err
		}
		scores = append(scores, &s)
	}

	return scores, nil
}
```

さて、実装できたはいいが、この実装が正しいことをどうやって担保しようかと考えたときに、やはりテストコードを書いておきたいところである。

ただし、BigQueryにはエミュレータなどは存在していない。BigQueryを使った処理に関してテストを書こうとすると、モックなどを使ってお茶を濁すか、実際にBigQueryに繋いでみるといった感じになる。今回は実際にBigQueryに繋いだテストコードを書いていく。

BigQueryにクエリを投げる前に、あらかじめBigQueryにテスト用データを投入しておかなければならない。 `cloud.google.com/go/bigquery` では指定形式で書かれたファイルのデータをBigQueryにロードするメソッドが提供されており、それを使用する。(※1)
ファイルはtestdataディレクトリを作成してその下に配置するとよい。ファイルの中身は下記である。

`testdata/get_group_avg_scores_by_date/students.json`
```
{"id": 1, "name": "John", "group": "GroupA"}
{"id": 2, "name": "Dominic", "group": "GroupA"}
{"id": 3, "name": "Steve", "group": "GroupA"}
{"id": 4, "name": "Chris", "group": "GroupB"}
{"id": 5, "name": "Ken", "group": "GroupB"}
```

`testdata/get_group_avg_scores_by_date/scores.json`
```
{"student_id": 1, "date": "2019-09-01", "score": 50}
{"student_id": 2, "date": "2019-09-01", "score": 40}
{"student_id": 3, "date": "2019-09-01", "score": 30}
{"student_id": 4, "date": "2019-09-01", "score": 20}
{"student_id": 5, "date": "2019-09-01", "score": 10}
{"student_id": 1, "date": "2019-09-11", "score": 60}
{"student_id": 2, "date": "2019-09-11", "score": 64}
{"student_id": 3, "date": "2019-09-11", "score": 66}
{"student_id": 4, "date": "2019-09-11", "score": 62}
{"student_id": 5, "date": "2019-09-11", "score": 69}
```

ファイルの指定形式としては、今回はJSONを選択した(他の対応形式については[ライブラリソースコード中のコメント](https://github.com/googleapis/google-cloud-go/blob/v0.46.3/bigquery/file.go#L52)参照)。ちなみに、1レコードを1つのJSONで表したものが改行コードで区切られて複数格納されている形式でないといけない。

上記のようにファイルを用意したうえで、テストコード中でデータロードできるようなユーティリティ関数を作成しておく。

```
const (
	// StudentsTableName is "students"
	StudentsTableName = "students"
	// ScoresTableName is "scores"
	ScoresTableName = "scores"
)

func loadTestDataFromJSON(ctx context.Context, client *bigquery.Client, tableName, jsonFilePath string) error {
	jsonFile, err := os.Open(jsonFilePath)
	if err != nil {
		return err
	}
	source := bigquery.NewReaderSource(jsonFile)
	source.SourceFormat = bigquery.JSON
	loader := client.Dataset(dataset).Table(tableName).LoaderFrom(source)
	loader.WriteDisposition = bigquery.WriteTruncate

	job, err := loader.Run(ctx)
	if err != nil {
		return err
	}
	status, err := job.Wait(ctx)
	if err != nil {
		return err
	}
	if err := status.Err(); err != nil {
		fmt.Println(status.Errors)
		return err
	}

	return nil
}
```

`loader.WriteDisposition = bigquery.WriteTruncate` としてあるのがポイントで、このように指定しておくと、それまで残っていたデータは削除したうえでデータロードしてくれる。テストコードで使うにはこのうえなく便利なモードである。

あとは `getGroupAvgScoresByDate` のテストコードを書いていけばよい。

```
func TestGetGroupAvgScoresByDate(t *testing.T) {
	ctx := context.Background()
	client, err := bigquery.NewClient(ctx, projectID)
	if err != nil {
		t.Fatal(err)
	}
	if err := loadTestDataFromJSON(ctx, client, StudentsTableName, "./testdata/get_group_avg_scores_by_date/students.json"); err != nil {
		t.Fatal(err)
	}
	if err := loadTestDataFromJSON(ctx, client, ScoresTableName, "./testdata/get_group_avg_scores_by_date/scores.json"); err != nil {
		t.Fatal(err)
	}

	testCases := map[string]struct {
		haveDate   civil.Date
		wantScores []*groupAvgScore
	}{
		"case1_normal_20190901": {
			haveDate: civil.Date{Year: 2019, Month: 9, Day: 1},
			wantScores: []*groupAvgScore{
				&groupAvgScore{
					Group:    "GroupA",
					AvgScore: float64(40.0),
				},
				&groupAvgScore{
					Group:    "GroupB",
					AvgScore: float64(15.0),
				},
			},
		},
		"case2_no_result_20190902": {
			haveDate:   civil.Date{Year: 2019, Month: 9, Day: 2},
			wantScores: []*groupAvgScore{},
		},
	}

	for k, v := range testCases {
		t.Run(k, func(t *testing.T) {
			gotScores, err := getGroupAvgScoresByDate(ctx, client, v.haveDate)
			if err != nil {
				t.Fatal(err)
			}

			if len(gotScores) != len(v.wantScores) {
				t.Fatal("count of gotScores and wantScores is different.")
			}

			sort.Slice(gotScores, func(i, j int) bool {
				return gotScores[i].Group < gotScores[j].Group
			})
			sort.Slice(v.wantScores, func(i, j int) bool {
				return v.wantScores[i].Group < v.wantScores[j].Group
			})

			for i, want := range v.wantScores {
				if diff := cmp.Diff(*want, *gotScores[i]); diff != "" {
					t.Fatalf("comparing mismatch. want: %v, got: %v, diff: %s", want, gotScores[i], diff)
				}
			}

		})
	}
}
```

これでテストができるようになった。
ただし注意したいのが、データロードやクエリ処理はそれなりに時間がかかる処理ということだ。テストケース毎にデータの中身を変えてロードし直していたりすると、テストにかかる時間がどんどん膨れ上がっていく。

### まとめ

- testdataディレクトリ以下に、BigQueryに流し込める形式のJSONファイルを用意しておく
- JSONデータを流し込む際には `WriteTruncate` モードを指定する
- JSONデータを流し込んでから検証対象のメソッドを実行するようにテストケースを実装する
- JSONデータを流し込む処理に時間がかかるのが欠点

---

※1 ちなみに[Inserter.Put](https://godoc.org/cloud.google.com/go/bigquery#Inserter.Put)でもBigQueryへのデータ投入を行うことができるが、これはStreaming Insertを行うメソッドであり、Steaming InsertでInsertしたレコードは一定時間削除できないという性質や、テーブルのメタデータ変更に対する伝播に時間がかかるという面で、テストコード実行に悪影響を及ぼすので、Inserter.Putは使わないこととした。

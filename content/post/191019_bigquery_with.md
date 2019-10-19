---
title: "WITH句で可読性の高いBigQueryのクエリを書く"
description: "WITH句の存在を知ってとても感動したので紹介します。"
date: 2019-10-19T19:00:00+09:00
categories:
  - 小ネタ
tags:
  - BigQuery
  - GCP
draft: false
---

最近仕事でBigQueryを使って運用上必要なデータを抽出をするために複雑なクエリを書くことがあります。様々なテーブルをJOINしつつサブクエリによってネストしたクエリです。

そのようなクエリは後から読んだときに何をやっているのかよく分からないといったことになりがちですが、WITH句によって可読性を上げられることを知りました。
数年前からサポートされているようなので全然新しくないトピックではあるのですが、WITH句によるクエリの書きやすさ、可読性の向上に感動したので記事にしておこうと思います。

WITH句
https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#with_clause

簡単に言ってしまうと、その場でのみ使えるビューを定義してその後のクエリ内で呼び出せる機能です。
以下は公式リファレンスに載っているWITH句を使ったクエリのサンプルです。

```
WITH subQ1 AS (SELECT SchoolID FROM Roster),
     subQ2 AS (SELECT OpponentID FROM PlayerStats)
SELECT * FROM subQ1
UNION ALL
SELECT * FROM subQ2;
```

上記の例はシンプルなのでありがたみが伝わりづらいかもしれません。ですが、例えば下記のような例はいかがでしょう。

---

架空のユーザ行動履歴データベースが存在するとします。
ユーザ行動はwalk,run,sleep,study,eatの5種類です。
ユーザ行動種類毎に行動ログテーブルが保持されているとします。

table: walk_logs

フィールド名 | タイプ | 内容 
---|---|---
id | INTEGER | (テーブルのPRIMARY KEY) 
user_id | INTEGER | ユーザID
from_x | FLOAT64 | 出発地点のx座標 
from_y | FLOAT64 | 出発地点のy座標 
to_x | FLOAT64 | 到着地点のx座標 
to_y | FLOAT64 | 到着地点のy座標 
consumed_calorie | FLOAT64 | 消費カロリー 
start_at | TIMESTAMP | 開始日時 
end_at | TIMESTAMP | 終了日時 

table: run_logs

フィールド名 | タイプ | 内容 
---|---|---
id | INTEGER | (テーブルのPRIMARY KEY) 
user_id | INTEGER | ユーザID
from_x | FLOAT64 | 出発地点のx座標 
from_y | FLOAT64 | 出発地点のy座標 
to_x | FLOAT64 | 到着地点のx座標 
to_y | FLOAT64 | 到着地点のy座標 
consumed_calorie | FLOAT64 | 消費カロリー 
start_at | TIMESTAMP | 開始日時 
end_at | TIMESTAMP | 終了日時 

table: sleep_logs

フィールド名 | タイプ | 内容 
---|---|---
id | INTEGER | (テーブルのPRIMARY KEY) 
user_id | INTEGER | ユーザID
start_at | TIMESTAMP | 開始日時 
end_at | TIMESTAMP | 終了日時 

table: study_logs

フィールド名 | タイプ | 内容 
---|---|---
id | INTEGER | (テーブルのPRIMARY KEY) 
user_id | INTEGER | ユーザID
subject | STRING | 勉強した科目 
start_at | TIMESTAMP | 開始日時 
end_at | TIMESTAMP | 終了日時 


table: eat_logs

フィールド名 | タイプ | 内容 
---|---|---
id | INTEGER | (テーブルのPRIMARY KEY) 
user_id | INTEGER | ユーザID
menu_id | INTEGER | 食事のメニューID
intake_calorie | FLOAT64 | 摂取カロリー 
start_at | TIMESTAMP | 開始日時 
end_at | TIMESTAMP | 終了日時 

また、ユーザの管理マスタテーブルusersが下記のように存在しているとします。

table: users

フィールド名 | タイプ | 内容 
---|---|---
user_id | INTEGER | ユーザID(PK)
name | STRING | 名前

このような構成のBigQueryデータセットを運用しているなかで、「ある日のユーザ毎の移動距離と摂取/消費カロリーのリストを抽出してほしい」という依頼が来たとします。
さて、どのようなクエリを組めばよいでしょうか。

- 移動距離についてはwalk_logsとrun_logsから集計
- 摂取/消費カロリーについてはwalk_logsとrun_logsとeat_logsから集計
- それらをusersテーブルにJOIN

まあ、このように考えるでしょう。それではまずはWITH句を使わずにクエリを書いてみましょう。
ちなみにデータセット名は `user_action_logs` としておきます。

```
SELECT
  u.user_id,
  d.sum_distance,
  c.sum_calorie
FROM user_action_logs.users as u
LEFT JOIN (
  -- ユーザ毎の移動距離
  SELECT
    user_id,
    SUM(SQRT(POW(to_x-from_x, 2) + POW(to_y-from_y, 2))) as sum_distance
  FROM
    -- walk_logsとrun_logsを結合して地点のカラムのみを抽出
    (
      SELECT user_id,from_x,from_y,to_x,to_y FROM user_action_logs.walk_logs WHERE DATE(end_at) = "2019-10-19"
      UNION ALL
      SELECT user_id,from_x,from_y,to_x,to_y FROM user_action_logs.run_logs WHERE DATE(end_at) = "2019-10-19"
    )
  GROUP BY
    user_id
) as d ON (u.user_id = d.user_id)
LEFT JOIN (
  -- ユーザ毎の摂取/消費カロリー
  SELECT
    user_id,
    SUM(calorie) as sum_calorie
  FROM
    -- walk_logsとrun_logsとeat_logsを結合してカロリーのカラムのみを抽出
    (
      SELECT user_id, -consumed_calorie as calorie FROM user_action_logs.walk_logs WHERE DATE(end_at) = "2019-10-19"
      UNION ALL
      SELECT user_id, -consumed_calorie as calorie FROM user_action_logs.run_logs WHERE DATE(end_at) = "2019-10-19"
      UNION ALL
      SELECT user_id, intake_calorie as calorie FROM user_action_logs.eat_logs WHERE DATE(end_at) = "2019-10-19"
    )
  GROUP BY
    user_id
) as c ON (u.user_id = c.user_id)
```

書き終わりました。
書く際の順番としてネストの深い場所から書き始めました。
では、これをWITH句を使って書いてみるとどうなるでしょうか。


```
WITH
-- walk_logsとrun_logsを結合して地点のカラムのみを抽出
points as (
  SELECT user_id,from_x,from_y,to_x,to_y FROM user_action_logs.walk_logs WHERE DATE(end_at) = "2019-10-19"
  UNION ALL
  SELECT user_id,from_x,from_y,to_x,to_y FROM user_action_logs.run_logs WHERE DATE(end_at) = "2019-10-19"
),
-- ユーザ毎の移動距離
sum_distance_by_user as (
  SELECT
    user_id,
    SUM(SQRT(POW(to_x-from_x, 2) + POW(to_y-from_y, 2))) as sum_distance
  FROM points
  GROUP BY user_id
),
-- walk_logsとrun_logsとeat_logsを結合してカロリーのカラムのみを抽出
calories as (
  SELECT user_id, -consumed_calorie as calorie FROM user_action_logs.walk_logs WHERE DATE(end_at) = "2019-10-19"
  UNION ALL
  SELECT user_id, -consumed_calorie as calorie FROM user_action_logs.run_logs WHERE DATE(end_at) = "2019-10-19"
  UNION ALL
  SELECT user_id, intake_calorie as calorie FROM user_action_logs.eat_logs WHERE DATE(end_at) = "2019-10-19"
),
-- ユーザ毎の摂取/消費カロリー
sum_calorie_by_user as (
  SELECT
    user_id,
    SUM(calorie) as sum_calorie
  FROM calories
  GROUP BY user_id
)
-- ユーザ毎の移動距離と摂取/消費カロリー
SELECT
  u.user_id,
  d.sum_distance,
  c.sum_calorie
FROM user_action_logs.users as u
LEFT JOIN sum_distance_by_user as d ON (u.user_id = d.user_id)
LEFT JOIN sum_calorie_by_user as c ON (u.user_id = c.user_id)
```

行数的にはそんなに変わっていないかもしれません(むしろ増えている)が、クエリを組み立てる際の思考の順番通りにクエリを読み書きできるようになっているのが分かるでしょうか。
ネストが深くなりすぎることもありません。
また、「ユーザ毎の移動距離と摂取/消費カロリー」とコメントをつけた場所から下の、最終結果抽出のためのクエリの記述がとてもシンプルになっているのが分かります。

非常に整理がしやすくなっているのではないでしょうか(感じ方には個人差はあるでしょうが)。
私はこれからもBigQueryを記述することがあればWITH句を積極的に活用していきたいと思います。

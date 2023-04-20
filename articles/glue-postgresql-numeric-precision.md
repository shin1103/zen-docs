---
title: "GlueジョブでPostgresqlのNumericの精度調整に失敗した話"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['postgresql', 'glue', 'aws']
published: true
---
# 概要
PostgreSql to PostgreSqlのAWS GlueジョブでデータソースのNumericの精度調整に失敗して最終的にPostgreSqlの[trim_scale関数](https://www.postgresql.jp/document/13/html/functions-math.html)で対応した。

# 事象
データソース・データディスティネーションともにPostgresqlのGlue ETLジョブを作成したところ小数点の精度が変わってしまった。
Glueが型がNumericであると、自動で精度を設定してくれる様子。

* データソースの値

| my_value(Numeric) | your_value(Numeric) |
| ----------------- | ------------------- |
| 1234              | 2022                |
| 1111.11           | 2023                |

* データディスティネーションの値

| my_value(Numeric)       | your_value(Numeric)     |
| ----------------------- | ----------------------- |
| 1234.000000000000000000 | 2022.000000000000000000 |
| 1111.110000000000000000 | 2023.000000000000000000 |

# 検討内容
Glueが原因なのでGlue側で何とかしようと検討を実施。

## 検討内容１（PythonのDecimal型操作）
まずは0を除去することを検討した。PostgresqlのNumeric型はGlue上ではDecimal型として扱われるので、PythonのDecimal型で0を除去できる方法を調査。
以下のサイトからPython公式のページに削除する方法が記載されていることを発見。対象のコードをGlue側に記載
https://takeg.hatenadiary.jp/entry/2019/11/29/220817
https://docs.python.org/ja/3/library/decimal.html#decimal-faq

```python
def remove_exponent(rec):
    def _remove_exponent(d):
        return d.quantize(Decimal(1)) if d == d.to_integral() else d.normalize()

    rec['my_value'] = _remove_exponent(rec['my_value'])
    rec['your_value'] = _remove_exponent(rec['your_value'])

datasink1_mapped = datasink1.map(f = remove_exponent)
```

1234の値をそのまま反映してほしかったのだが、1111.11の値に引きずられて1234.00と小数点第2位まで精度が設定されてしまった。すべてが整数のyour_valueは小数点第2位までとならなかったことからGlue側で何か制御をしておりうまく対応できなかった。

* 対応後の値

| my_value(Numeric) | your_value(Numeric) |
| ----------------- | ------------------- |
| 1234.00           | 2022                |
| 1111.11           | 2023                |

## 検討内容２（DataFrameの操作）
Decimal型ではうまくいかなかったのでGlueで操作するDataFrameで対応できないか検討した。
DataFrameの編集では以下のサイトを参考にした。
https://yomon.hatenablog.com/entry/2019/04/23/decimalonglue

この方法ではDataFrameの型で精度を設定することができるのだが、小数点の精度を設定するscaleをすべて一緒にしないといけないので上記と同じ結果となってしまった。

# 解決方法
上記の通りGlueというかSparkでは解決する方法が見つからなかったので、PostgreSqlで解決する方法がないか探したところ、[trim_scale関数](https://www.postgresql.jp/document/13/html/functions-math.html)を発見し対処ができた。

以下がドキュメントの記述だが引数がnumericとなっていることから安心して使用できる。
> trim_scale ( numeric ) → numeric
後方のゼロを削除することにより値の桁数（小数点以下の10進桁数）を減じる
trim_scale(8.4100) → 8.41

```sql
UPDATE my_table SET my_value = trim_scale(my_value);
UPDATE my_table SET your_value = trim_scale(your_value);
```

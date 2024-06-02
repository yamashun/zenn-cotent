---
title: "GASによるGoogle Sheetsのグラフ調整でドキュメントに書かれていないオプションを見つける方法"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas", "googlesheets"]
published: false
---

## はじめに
Google Sheetsでデータから大量にグラフを生成して画像に変換して保存したいということがありGASで対応しました。
その際に行ったグラフのスタイル調整についてのTipsについて紹介します。

## TL;DR
- GASによるグラフのスタイル調整は、公式ドキュメントに書かれているオプションだけでは細かいところに手が届かない場合がある。
- ドキュメントに記載がないオプションを使ってスタイル調整ができ、その設定はグラフをWeb公開してDevToolで確認して見つけることができる

:::message alert
ドキュメントに書かれている方法ではないため今後使えなくなる可能性があります
:::

## サンプルデータでグラフを生成
以下のような適当なデータから縦棒グラフを作成して説明していきます。

![](/images/gas-chart-decorate-tips/chart_data.png =200x)


```js
function generateBarChart() {
  const spreadsheet = SpreadsheetApp.openById('xxxxxxxxxxxxxx') // スプレッドシートIDは実際のIDに置き換える
  const sheet = spreadsheet.getSheetByName('シート1')

  const data = sheet.getDataRange().getValues()
  const chart = sheet.newChart()
          .setChartType(Charts.ChartType.COLUMN)
          .addRange(sheet.getRange(2, 1, 12, 2))
          .setPosition(1, 4, 0, 0)
          .build()
  sheet.insertChart(chart)
}
```

のようなコードで画像のようなグラフが生成されます。

![](/images/gas-chart-decorate-tips/bar_chart.png =400x)



## やりたいこと
このグラフに対してデータラベルを表示したいとします。


## 公式ドキュメントの確認

[Class EmbeddedColumnChartBuilder](https://developers.google.com/apps-script/reference/spreadsheet/embedded-column-chart-builder?hl=ja)のページに縦棒グラフで使えるメソッドが書かれており、その中の `setOption` を使うことで詳細な設定ができるとあります。
さらに `setOption` で使用可能なオプションのページに[縦棒グラフのオプション](https://developers.google.com/apps-script/chart-configuration-options?hl=ja#column-config-options)一覧が書かれています。

例えばグラフの縦横の大きさを調整したい場合は、`width`や`height`を使って調整できることが書かれています。

```js
sheet.newChart()
     .setChartType(Charts.ChartType.COLUMN)
     .setOption('width', 2600)
     .setOption('height', 1300)
```

オプションの中に`series`というオプションがあり、これを使用することでデータラベルを表示できそうに見えますが、例の通り設定してもラベルは表示されませんでした。

```js
sheet.newChart()
     .setChartType(Charts.ChartType.COLUMN)
     .setOption('series', { // ドキュメントの例の通りに設定しても特にグラフに変化はない
       0: {
         annotations: {
           textStyle: { fontSize: 12, color: 'red' }
         }
       }
     })
```

## Webで表示したグラフからオプションを見つける方法
一通りオプションを確認してもデータラベルの設定が見つけられず困っていましたが、stack overflowに同じようなことで悩んでいる人がいました。

[How to set "Use column A as labels" in chart embedded in spreadsheet?](https://stackoverflow.com/questions/13594839/how-to-set-use-column-a-as-labels-in-chart-embedded-in-spreadsheet?noredirect=1&lq=1)

その回答の中に、グラフを公開してそのグラフのJavaScriptデータ構造を確認することで設定を見つけられると書かれています。

どういうことか試してみます。

### グラフを公開して設定を確認
①グラフを設定したいスタイルに手動で変更

![](/images/gas-chart-decorate-tips/chart_with_data_label.png =500x)

②グラフを公開

![](/images/gas-chart-decorate-tips/publish_chart.png =400x)

③公開したグラフの設定内容をDevtoolで確認

![](/images/gas-chart-decorate-tips/chartjson.png =400x)

図のchartJsonの中にスタイルに関する設定が入っています。

chartJsonの文字列はエンコードされており、読みづらいのでデコードして`options`を抜き出すと以下のようになっていました。

```json
"options":{"series":{"0":{"dataLabel":"value","hasAnnotations":true}},"useFirstColumnAsDomain":true,"width":600,"height":371}
```

`dataLabel`が設定したいオプションにあたりそうなので、GASのコードに追加してグラフを再作成します。

```js
.setOption('series', {
  "0":{ "dataLabel": "value","hasAnnotations": true }
})
```


![](/images/gas-chart-decorate-tips/chart_with_data_label_by_gas.png =500x)

手動で設定した場合と同じようにデータラベルが表示されました🎉


## まとめ
GASでGoogle Sheets上のデータからグラフを作成する際のスタイル調整について、ドキュメントに記載されていないオプションの設定を見つける方法を紹介しました。

今回はデータラベルの例で説明しましたが、他のオプションについても同様の流れで設定を見つけることができます。
一度グラフを公開してjson構造からリバースエンジニアリングするような流れなので、少し面倒ですがGAS上でのスタイル調整に困った場合はぜひ試してみてください。

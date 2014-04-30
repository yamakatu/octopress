---
layout: post
title: "ElasticsearchでCSVからインデクシングする"
date: 2014-04-30 16:28:37 +0900
comments: false
categories: es-kibana
---
<!-- more -->
<br/>

インデクシングしたいデータがCSVだった場合、CSVからjsonへの変換は意外と面倒だったりしませんか。

そんな場合、CSV River Pluginを使ってCSVから直接インデクシングがオススメです。

<a href="https://github.com/AgileWorksOrg/elasticsearch-river-csv">CSV River Plugin</a>

###インストール

(ver 2.0.1の場合。ESとのバージョン対応は上記Githubで要確認)

```bash
bin/plugin -install river-csv -url https://github.com/AgileWorksOrg/elasticsearch-river-csv/releases/download/2.0.1/elasticsearch-river-csv-2.0.1.zip
```

###登録

こんな感じ。他のオプションは上記ページで要確認

```bash
curl -XPUT localhost:9200/river/my_csv_river/meta -d ’ {
  "type" : "csv",
  "csv_file" : {
    "first_line_is_header":"true",
    "folder" : "/tmp",
    "filename_pattern" : ".*\\.csv$"
  },
  "index" : {
    "index" : "index_name",
    "type" : "type+name",
    "bulk_size" : 1000,
    "bulk_threshold" : 10
  }
}’
```

###罠

しかし罠がありました。
<br/>

GithubのREADME.mdには一言も書いてませんが、処理されたCSVファイルは以下の命名規則でRenameされます。
<br/>

（hoge.csvの場合）

- 読み込み中：hoge.csv.processing 
- 読み込み済：hoge.csv.processing.imported
<br/>

登録のjsonを見れば雰囲気でわかりますが、folder以下のfilename_patternで指定したファイルが対象になります。

そのため、

```sh
“filename_pattern” : “.*”
```

みたいな正規表現で設定すると、読み込み済みのCSVファイルが、次回polling時に読み込み対象になります。

pollingはデフォルト60minなので（変更可）、1時間ごとにすべてのファイルが再インデクシングされます。

（しかも倍々でw）

デフォルト値が上記正規表現なので、通常であればこのままにしとくのが吉です。
<br/>

変更する場合、登録する正規表現は

.csv$

で終わるようにしましょう。
<br/>


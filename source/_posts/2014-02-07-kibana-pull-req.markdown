---
layout: post
title: "【Kibana】Histgramで欠損値に0が自動補完されるのがアレだったので、OFFにする機能を実装してプルリクしてマージしてもらったお話"
date: 2014-02-09 15:40:59 +0900
comments: false
categories: es-kibana
---

<!-- more -->

#■前提
Kibana <= kibana-3.0.0 milestone4
<br/>

#■何が起きたか
去年の12月ぐらいに、「Elasticsearch + Kibana って、ログの可視化に超便利ジャーン！」という感じでKibanaを利用していたところ、Histgramの描画でちょっと微妙な現象に遭遇。
<br/>
例えば、
<br/>
<table border=1><tr align="center"><td width="200">Time</td><td width="100">value</td></tr><tr align="center"><td>2014/11/08T09:00:00</td><td>10</td></tr><tr align="center"><td>2014/11/08T09:30:00</td><td>30</td></tr><tr align="center"><td>2014/11/08T10:00:00</td><td>20</td></tr></table>
<br/>
な感じのデータでHistgramを描画した時、自分はこう描画して欲しかった。
<br/>
{% img /images/kibana-ok.png '' '' %}
<br/>
が、描画されるのは以下のようなグラフ。
<br/>
{% img /images/kibana-ng.png '' '' %}
<br/>

#■どういうことなのか

この二つのグラフはご覧の通り、Intervalの値が異なる。前者のグラフは30分、後者のグラフは10分。

んで、KibanaはIntervalごとに値を表示しようとする。

しかし、10分ごとの場合、9:10、9:20などには値が欠損している。

###この時Kibanaは勝手に0を補完する。
<br/>

いやー気が利くねー（白目
<br/>

#■どう思った？
確かに0を補完して描画して欲しい場合もあるだろうから、間違ってはいない。

しかし、今回の自分のように、余計なことすんじゃねーよ、という場合も普通にある。

なので、選べるようにした方がよくねー？
<br/>

あと前者のグラフのように、データが一定間隔でかつ欠損がなく、Intervalの値をその間隔に設定できる場合、この事象は発生しない。

だけど、こういう場合であろうとなかろうと、発生しないでほしい。。。
<br/>

#■それでどうしたのか

Githubでソース読んでたら、実は既にグラフ描画の仕方が選択できるように実装されてた。<a href="https://github.com/elasticsearch/kibana/blob/master/src/app/panels/histogram/timeSeries.js#L103">ここらへん</a>

(ちなみにこの描画方式はコード上ではstrategyと呼ばれているので、以下strategy）

なんだけど、今回のような「値のない時間に0を補完しない」というstrategyは当時はなかった。
<br/>

というわけで、実装して、issue作って、プルリク送っときました。<a href="https://github.com/elasticsearch/kibana/pull/742">#742</a>

そしたら忘れた頃にマージしてもらえました。

<a href="https://github.com/elasticsearch/kibana/blob/master/src/app/panels/histogram/timeSeries.js#L103">上記コード</a>にある_getNoZeroFlotPairsがソレです。

この実装ではIntervalがどんな値だろうが、欠損値に0を補完しなくなります。
<br/>

次のリリースでは乗っかると思われます。

ただ、以前コードを見た時は、そもそもstrategyを選択するUIがなかったので、画面からはまだ使えないかもしれん。。。
<br/>

#■すぐ使いたければどうすればいいのか

リリースまで待てない人は上記ソースを参考にして改修すると良いと思います。
<br/>

もしくは、とりあえず今すぐちょっと試したい方なんかは、乱暴だけど、

app/panels/histogram/timeSeries.js

の

ts.ZeroFilled.prototype._getMinFlotPairs

を以下のように改修するのが一番手っ取り早いです。

```js
  ts.ZeroFilled.prototype._getMinFlotPairs = function (result, time, i, times) {
    if(this._data[time]){
      result.push([ time, this._data[time]]);
    }
    return result;
  };
```

\#人によっては改修後にブラウザのキャッシュクリアが必要かもしれん

ちなみにコードをいじる場合は、公式サイトからDLしたkibanaだと改修が大変なので、<a href="https://github.com/elasticsearch/kibana/releases">Github内のリリース</a>から落としたコードを利用すると幸せになれると思います。

<br/>

#■まとめ
- 今のKibanaのHistgramは欠損値に勝手に0を入れる
- それをOFFにするオプションを実装して、プルリク送って、マージされた
- 自分でがんばって書き換えればすぐに使えるよ


---
layout: post
title: "ClouderaManager の管理コンソールをとりあえず HTTPS 化する方法（EC2 を使うときは特に使おうね）"
date: 2014-02-17 02:06:25 +0900
comments: false
categories: hadoop
---
<!-- more -->

#■環境
ClouderaManager 4.8.1 インストール済み（他のバージョンでもそんなに対して変わらないはず）
<br/>

#■何があったか
Hadoop Advent Calendar 2013で<a href="http://qiita.com/yamakatu/items/495246f98044398ad3d3">こんな記事</a>書いたら、<a href="http://twitter.com/shiumachi">@shiumachi</a>さんに

###そのやり方だとブラウザと管理コンソール間が暗号化されてないから、AWSの鍵をアップロードする際に盗聴されちゃうじゃん。m9ぷぎゃー（かなり意訳
<br/>
と、ご指摘を頂きました。

薄々気づいてたんだけど面倒かったので手を抜いた。今では反省している。
<br/>

#■もうちょっとkwsk

EC2でClouderaManagerを使うと、指定したインスタンスタイプと台数でHadoopクラスタを自動で作成してくれます。すげー便利。

この時ClouderaManagerはAWSのAPIを当然たたくんだけど、この際にはAWSの鍵が必要になります。

そして、この鍵はブラウザでClouderaManagerの管理コンソールからPostしてあげる必要があります。

この時、ClouderaManagerの管理コンソールはデフォだとhttpsに対応しておらず、httpでアクセスすることになるので、ブラウザとClouderaManagerが稼働するサーバ間の経路は暗号化されません。

よって、AWSの鍵が盗聴されちゃうだろ。m9ぷぎゃー。となりました。
<br/>

#■どうする？
ClouderaManagerの管理コンソールをhttps化する方法を書いときます。

今回のようなEC2を使う場面でなくても有用だと思います。たぶん。

設定方法は本家の<a href="http://www.cloudera.com/content/cloudera-content/cloudera-docs/CM4Ent/latest/Cloudera-Manager-Administration-Guide/cmag_config_tls_security.html">ココ</a>に書いてあるんだけど、英語がアレな人もいらっしゃると思いますので、意味があると信じている。

本家サイトにはいくつか方法が記載されていますが、今回はとりあえずhttps化ということで、一番手軽な方法(<a href="http://www.cloudera.com/content/cloudera-content/cloudera-docs/CM4Ent/latest/Cloudera-Manager-Administration-Guide/cmag_config_tls_encr.html?scroll=topic_2__title_4_unique_8">コレ</a>)にしてます。
<br/>

#■手順
###手順1 鍵作成&配置

まず鍵を作成します。（※注 この鍵はHTTPSで利用する鍵で、AWSの鍵ではありません）

この鍵の作成にはJavaのkeytoolを使います。
<br/>

そのため、鍵を作成するマシンにはJavaがインストールされている必要がありますが、ClouderaManagerがインストールされていれば、自ずとJavaがインストールされています。

作成した鍵はいずれにせよClouderaManaerが動作するサーバに設置する必要があるので、ClouderaManagerがインストールされたサーバで鍵を作成するのが一番手っ取り早いです。

よって、<b>今回はClouderaManagerが稼働するサーバ上で鍵を作成します。</b>
<br/>

以下のコマンドで作成します。名前やら組織はめんどいので入力してませんが、適宜入力しましょう。

```
ubuntu@ip-10-133-198-199:~$ keytool -validity 180 -keystore cm.key -alias jetty -genkeypair -keyalg RSA
Enter keystore password:
Re-enter new password:
What is your first and last name?
  [Unknown]:
What is the name of your organizational unit?
  [Unknown]:
What is the name of your organization?
  [Unknown]:
What is the name of your City or Locality?
  [Unknown]:
What is the name of your State or Province?
  [Unknown]:
What is the two-letter country code for this unit?
  [Unknown]:
Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
  [no]:  yes

Enter key password for <jetty>
	(RETURN if same as keystore password):
Re-enter new password:
```

このままだとClouderaManagerのサーバプロセスがアクセスできないので、権限変えつつ、移動します。

```
ubuntu@ip-10-133-198-199:~$ sudo mv cm.key /etc/cloudera-scm-server/
ubuntu@ip-10-133-198-199:~$ sudo chown cloudera-scm:cloudera-scm /etc/cloudera-scm-server/cm.key
ubuntu@ip-10-133-198-199:~$ sudo chmod 600 /etc/cloudera-scm-server/cm.key
```

mv先は上記ディレクトリでなくても、サーバプロセスの起動ユーザ（今回はcloudera-scm）がアクセスできれば問題ないと思います。今回は解りやすそうなところに置いてみました。
<br/>

###手順2 ClouderaManager設定

ClouderaManagerの新規インストール時の場合を例にとります。

クラスタ構築済みの場合でも設定箇所は変わらないので、適宜読み飛ばしてください。
<br/>

まず、ブラウザからClouderaManagerにアクセスし、ログインします。（初期状態だとポートは7180、IDとパスワードはadmin/admin）
<br/>

ログイン後、以下のように右上のadminからパスワード変更を選択

{% img /images/cm-1.png '' '' %}
<br/>
<br/>

パスワード変更のフォームが出るが無視して、上部メニューの「管理」から「設定」を選択

{% img /images/cm-2.png '' '' %}
<br/>
<br/>

左メニューからセキュリティを選択

{% img /images/cm-3.png '' '' %}
<br/>
<br/>

以下を設定する

「管理コンソールで TLS 暗号化を使用する」にチェック

「TLS Keystore ファイルのパス」に鍵の配置先を指定。上記の例であれば/etc/cloudera-scm-server/cm.key

「Keystore パスワード」に作成時に指定したパスワードを入力

{% img /images/cm-4.png '' '' %}
<br/>

「変更の保存」ボタンを押す
<br/>

###手順3 ClouderaManager再起動
以下みたいな感じでClouderaManagerを再起動します。

```
ubuntu@ip-10-133-198-199:~$ sudo /etc/init.d/cloudera-scm-server restart
```

httpsでアクセスして確認します。

###httpsの場合のポートは7183です。
<br/>

ただ再起動後すぐにアクセスしても多分無理。ちょっと待つ。

アクセスできればおk。

しらばく待ってもアクセスできなければ、iptablesとかEC2ならセキュリティグループの設定あたりを疑うのが吉かも。

#■まとめ
・ClouderaManagerはデフォではhttpsが利用できない

・この状況だと何かとまずい（EC2の場合は特に）

・こんな手順でさくっといけるよ


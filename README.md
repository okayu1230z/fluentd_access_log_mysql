# docker base nginx access log management

dockerベースで動作するNginxのアクセスログをFluentdで拾ってMysqlにログを保存するサンプル

- Nginx: webサーバー。アクセスログをローカルに吐き出しておく
- Fluentd: データの収集管理ツール。Logstashと比較してWeighted Load Balancingなどができるなどの利点がある
- Mysql: RDBMS。Fluentdが接続に来てログを保存する

## Nginx

webサーバとはHTTPリクエストを送ったときに何かしらのレスポンスを返すプログラムのこと

nginxとよく比較されるApacheとの違いは[ApacheとNginxについて比較](https://qiita.com/kamihork/items/49e2a363da7d840a4149)の記事が簡潔で詳しい

C10K問題というClientが10Kを超えるとハードウェア・ネットワーク性能に関わらず同時クライアント接続数がある一定を超えたときにサーバがパンクしてしまう問題を解決する目的で設計されている

nginxはシングルスレッドモデルのイベント駆動アーキテクチャなので、マルチプロセスのプロセス稼働アーキテクチャのApacheに比較して同時接続数に関しては10-100倍くらいになる

## Fluentd

Fluentdはデータ収集に用いられるApacheライセンスのOSS

HTTP, fileのtail, dstatなどで集めたデータをファイルに吐き出したりDBに保存できたりする

よく議論されるLogstashとの比較に関しては[このスライド](https://www.slideshare.net/td-nttcom/fluentd-vs-logstash-for-openstack-log-management)がよくまとまっているので一度は目を通しておきたい

Logstashと置き換えれるので、ELK stackにおきかえてElasticsearch、Kibanaと連携してEFKが構成できる

td-agentという名前でFluentdや色々のプラグインが同梱されているソフトウェアがRedHat(rpm)、Debian(deb)で配布されていて、導入が楽にできる

- ログ転送: Active-Stanby、Load Balancingなどの冗長化構成も取れる
- バッファリング機能: バッファに溜め込んだログの再送制御を行い、ログ欠損率を低く抑える工夫がある
- フィルタ機能: 正規表現によるフィルタリングが行える
- 豊富なプラグイン: 様々なケースで導入を検討できる

### 設定ファイル書き方

カスタムしないのであれば

[公式ドキュメント](https://docs.fluentd.org/configuration/config-file)から抜粋して重要なところだけ

`<system>...</system>`:  Set system-wide configuration

ログレベルやスレッド数制御などfluentd全体のパラメータを設定する

`<source>...</source>`: where all the data come from

入力(input)処理に関する設定を行う

`<filter>...</filter>`: Event processing pipeline

データの加工など(filter)に関する設定を行う

`<match>...</match>`: Tell fluentd what to do!

どこに出力(output)処理に関する設定を行う

`<label></label>`: Group filter and output

ラベルをつけてグループ分け

`@include`: Reuse your config

設定を読み込める

```
<source>
  @type tail
  path /var/log/nginx/access.log
  pos_file /tmp/access.log.pos
  tag nginx.access
  <parse>
    @type nginx
  </parse>
</source>

<match nginx.access>
  @type mysql_bulk
  host mysql
  database mydb
  username root
  password mypassword
  column_names remote, host, user, method, path, code, size, referer, agent, http_x_forwarded_for, time
  key_names remote, host, user, method, path, code, size, referer, agent, http_x_forwarded_for, ${time}
  table logs
  <buffer time>
    @type file
    path /fluentd/log/buf_access
    timekey 1
    timekey_wait 0
    timekey_zone Asia/Tokyo
  </buffer>
</match>
```


## Mysql

Fluentdから接続に来てmatchディレクティブの設定アクセスログを保存

[MysqlのDocker Hub](https://hub.docker.com/_/mysql)を見ると`/docker-entrypoint-initdb.d`以下に置いた`.sh, .sql and .sql.gz`の拡張子のファイルがコンテナ起動時にalphabeticalな順番で実行されるって書いてある

これがなかなか便利な機能で、初期データの流し込みまで完了できてしまう

## 展望

Nginxのアクセスログはデフォルト設定で出力しているが、webサーバで取れる全てのデータを吐き出しているわけではない（例えばPOSTのパラメータは出力されていない）

しかし、残されたログからアプリケーションに起こったことを知るためにデバッグを行うことを考えるとユーザのログは全て取っておくことが望ましい

ログに新しいパラメータを追加すると出力されるログ、つまりそれを受け取るFlunetdのinputファイルが変わるわけなので、fluentdのconfigも改変が必要になってくる


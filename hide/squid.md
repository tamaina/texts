MisskeyサーバーのIPアドレスを完全に秘匿するためには、Summaly Proxyだけでは不十分です。  
「連合のinboxに投稿を送信する」「連合のファイルを取ってくる」など、Misskeyサーバーから外に向かって通信をする際には、アクセス先のサーバーにIPアドレスがわかってしまいます。  
ActivityPubの仕様上、インスタンス運営者でなくとも[簡単にIPアドレスを入手](https://gist.github.com/cutiful/4f36da3ed37b24f9a7106064393f5e7f)できてしまいます。

アクセスしたサーバーに対してIPアドレスを教えないために、Misskeyにはそういった外向き通信にプロキシを用いる設定があります。

nginxやCloudflareといったリバースプロキシとは全く別物と思ってください。  
Cloudflareのプロキシを使っていなど、外からのアクセスをサーバーのグローバルIPアドレスで直接受けている場合、外向き通信をプロキシしても意味はないでしょう。

# サーバーのIPを秘匿したい理由
そもそもなぜサーバーのIPを秘匿したいのか。  
悪いことをしようとしているわけではありません。

最大の理由は、DDoSのターゲットにされてサーバーがダウンしないようにするためです。  
かつてのMisskey公式インスタンス(misskey.xyz)は、DDoS攻撃が多すぎて**VPSを止められてしまい**、misskey.ioに移転することになりました。

IPアドレスがわからなければ、SSHやnginxの脆弱性でデータが盗まれたり乗っ取られたりするといったことを防ぐこともできます。

# プロキシの動きについて
まず、**Misskeyとは別のグローバルIPを持ったサーバー**にプロキシ用のソフトをインストールします。

Misskeyに設定を加えて、外向き通信がプロキシを経由するようにします。

こうすると、通信相手にはプロキシから通信が来たように見え、MisskeyサーバーのIPアドレスはわかりません。

# プロキシを使うかどうか？
プロキシについては、一人で使うインスタンスであれば設定する必要性は薄いかと思います。  
DDoSを食らってもサーバーを再起動すれば大抵大丈夫ですし……。  
ただ、「自分しか使っていないし攻撃されないから大丈夫」ということはありません。  
Minecraftの略奪隊のように、どのサーバーにも無差別にDDoS攻撃は来ます。厄介なんです。

DDoS対策の手法はプロキシだけではないのですが、プロキシは最も単純かつ有効な対策手段だと思います。

# 環境
この記事に書かれていることは、環境の違い等の理由から、実際にあなたが必要な操作とは異なる可能性があります。あらかじめご了承ください。

実証環境（Squidサーバー・Misskeyサーバー両方とも）

- Oracle Cloud Infrastructure
- Ubuntu 20.04を利用しました。

Squidサーバーには、MisskeyとグローバルIPアドレスが違うサーバーを使ってください。

# Squidのインストール
Linuxのプロキシサーバーは、Squidが主流です。  
Misskeyとは別の適当なサーバーでSquidを動作させましょう。

aptデフォルトは…相変わらず古いですね…。  
記事執筆時点での最新バージョンは4.16。[セキュリティリスク](https://github.com/squid-cache/squid/security/advisories)が全て修正された4.15は欲しいです。

![](https://firebasestorage.googleapis.com/v0/b/hideaki-97c59.appspot.com/o/images%2FPFOKUISFS1RaFz4ghSnc2GS6l5z2%2FmObcKBlXf.png?alt=media)

## ビルド
ppaもどれを選んだらいいかちょっとわからなかったので、自前でビルドしましょう。  
せっかくなので、最新のv5をビルドします。

squidのビルド・実行については下の記事を参考にしました。  
[https://github.com/squid-cache/squid/blob/v5/INSTALL](https://github.com/squid-cache/squid/blob/v5/INSTALL)  
[https://mindchasers.com/dev/squid](https://mindchasers.com/dev/squid)  

ビルドとインストールに必要なコマンドを示します。

```
sudo apt install build-essential automake libtool
git clone https://github.com/squid-cache/squid.git
cd squid
git checkout v5
./bootstrap.sh
mkdir build
cd build
../configure --prefix=/opt/squid --with-default-user=squid --enable-stacktraces --without-mit-krb5 --without-heimdal-krb5 --with-logdir=/opt/squid/log/squid --with-pidfile=/opt/squid/run/squid.pid
make all
sudo make install
ls /opt/squid
```

（ちなみに、SquidでSSL通信に手を加えたいわけではないので、ssl-bumpは必要ありません。）

## セットアップ
3128ポートを解放します。

IPアドレスは接続を許可するIPアドレスに置き換えてください。  
（例: 10.0.0.0/24）

```
sudo iptables -I INPUT -s IPアドレス -p tcp --dport 3128 -j ACCEPT
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

クラウドサービスを使っている場合、ローカル通信であっても**イングレスルールで3128ポートの通信を許可する必要がある**場合があります（私はこれで3日潰しました。）。

## Squidユーザー作成
squidユーザーを作り、/opt/squidをsquidユーザーの所有にします。

```
sudo useradd -m -U -s /bin/bash squid
mkdir -p /opt/squid/log
sudo chown -R squid:squid /opt/squid

sudo su - squid
```

## 実行
設定が終わったのでSquidを起動します。

```
/opt/squid/sbin/squid
```

psで実行状況を確認してみます。

```
ps -e | grep squid
```

ログはファイルに保存されるので、とりあえず次のように確認します。

```
tail -n 30 -f /opt/squid/log/squid/cache.log
```

# Misskeyにプロキシを設定
## プロキシが動作しているか確認
プロキシ経由でhttps接続できるか確認します。

```
export https_proxy=http://SquidサーバーのIPアドレス:3128
curl http://checkip.amazonaws.com
curl https://checkip.amazonaws.com
```

http接続とhttps接続でIPアドレスが違うこと、httpsでSquidサーバーのグローバルIPが表示されることを確認します。

ネットワーク設定が間違っているとタイムアウトになります。

## Misskeyに設定
**Misskeyのdefault.ymlへ、proxy設定を下記のように追加してください。**

```
proxy: http://SquidサーバーのIPアドレス:3128
```

Misskeyを再起動します。  
systemdの場合…

```
sudo systemctl restart misskey
```

# アクセスログを見る
Squidサーバーでアクセスログを確認してみます。

```
tail -n 30 -f /opt/squid/log/squid/access.log
```

ノートを投稿すると、リモートインスタンスのinboxにアクセスしているのが分かるかと思います。

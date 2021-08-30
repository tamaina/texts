Summaly Proxyだけでは、MisskeyサーバーのIPアドレスを完全に秘匿するには不十分です。  
「連合のinboxに投稿をプッシュ送信する」「連合のファイルを取ってくる」ということをすると相手のサーバーにIPアドレスが分かります。  
特に、ActivityPubエンドポイントを使えば[簡単にIPアドレスを入手できてしまう](https://gist.github.com/cutiful/4f36da3ed37b24f9a7106064393f5e7f)ようです。

アクセスしたサーバーに本当のIPアドレスを教えないために、Misskeyにはリクエストにプロキシを用いる機能があります。

# サーバーのIPを秘匿したいワケ
そもそもなぜサーバーのIPを秘匿したいのか。  
身元を隠して悪いことをしようとしているわけではありません。

最大の理由は、DoS/DDoSのターゲットにされてサーバーがダウンしないようにするためです。  
misskey.xyz（misskey.ioの前身）が使用不能になった原因もDDoSです。

さらに、何かしらのソフトウェアに脆弱性があっても、それが悪用されてデータが盗まれたりボットに乗っ取られたりすることを防げます。

プロキシについては、おひとり様インスタンス等なら、設定する必要性は薄いかと思います。  
DDoSを食らってもサーバーを再起動すれば大抵大丈夫ですし……。

また、サーバーのIPを隠匿する方法は、Misskeyのプロキシ機能だけでなく、NATやVPNを使う方法もあります。

# 環境
SquidサーバーもMisskeyサーバーも、Ubuntu 20.04を利用します。  
実行にはOracle Cloud Infrastructureを使用しました。  
この記事に書かれていることは、環境の違い等の理由から、実際にあなたが必要な操作とは異なる可能性があります。あらかじめご了承ください。

# Squidのインストール
Linuxのプロキシサーバーは、Squidが主流のようです。  
Misskeyとは別の適当なサーバーでSquidを動作させましょう。

aptデフォルトは…相変わらず古いですね…。

![](https://firebasestorage.googleapis.com/v0/b/hideaki-97c59.appspot.com/o/images%2FPFOKUISFS1RaFz4ghSnc2GS6l5z2%2FmObcKBlXf.png?alt=media)

## ビルド
aptに頼れないし、ppaもちょっと…？な感じがあるので、自前でビルドしましょう。  
最新のv5をビルドします。

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

（ちなみに、SquidでSSL通信に何か手を加えたいわけではないので、ssl-bumpについては今回は必要ありません。）

## セットアップ
コンピュータの3128ポートは解放しておきましょう。

"IPアドレス"は接続を許可するIPアドレスに置き換えてください。  
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
**Misskeyのdefault.ymlへ、下記のように追加してください。**

```
proxy: http://SquidサーバーのIPアドレス:3128
```

systemctlならMisskeyを再起動します。

```
sudo systemctl restart misskey
```

Dockerについては割愛します……。

## アクセスログを確認してみる
実際にSquidサーバーでアクセスログを確認してみましょう。

```
tail -n 30 -f /opt/squid/log/squid/access.log
```

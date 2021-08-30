MisskeyサーバーのIPアドレスを完全に秘匿するには、Summaly Proxyだけでは実は不十分です。  
「連合のinboxに投稿をプッシュ送信する」「連合のファイルを取ってくる」ということをすると相手のサーバーにIPアドレスが分かります。  
特に、ActivityPubエンドポイントを使えば[簡単にIPアドレスを入手できてしまう](https://gist.github.com/cutiful/4f36da3ed37b24f9a7106064393f5e7f)ようです。

相手のサーバーにIPアドレスを教えないために、Misskeyにはリクエストにプロキシを用いる機能があります。

そもそもなぜサーバーのIPを秘匿したいのか。身元を隠して悪用したいわけではありません。  
最大の理由は、DoS/DDoSのターゲットになってサーバーがダウンするのを防ぎたいからです。  
また、何かしらのソフトウェアに脆弱性があっても、それが悪用されてデータが盗まれたりボットに乗っ取られたりすることを防げます。

プロキシについては、おひとり様インスタンス等なら、設定する必要性は薄いかと思います。  
DDoSを食らってもサーバーを再起動すれば大抵大丈夫ですし……。

また、サーバーのIPを隠匿する方法は、Misskeyのプロキシ機能だけでなく、NATやVPNを使う方法もあります。

# 環境
適当な記事のURLを提示すればいいかなとも思いましたが、思いの外調べることが多く、結局ここに一連の流れを纏めることとしました。

SquidサーバーもMisskeyサーバーも、Ubuntu 20.04を利用します。  
実行にはOracle Cloud Infrastructureを使用しました。  
この記事に書かれていることは、環境の違い等の理由であなたが実際に必要な操作とは異なる点がありますので、あらかじめご了承ください。

# Squidを使う
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
[https://www7390uo.sakura.ne.jp/wordpress/archives/777](https://www7390uo.sakura.ne.jp/wordpress/archives/777)  
[https://qiita.com/subaru44k/items/9adf2142fcc677108a13](https://qiita.com/subaru44k/items/9adf2142fcc677108a13)

実行すべきコマンドをメモしておきます。

```
sudo apt install build-essential automake libtool libssl-dev
git clone https://github.com/squid-cache/squid.git
cd squid
git checkout v5
./bootstrap.sh
mkdir build
cd build
../configure --prefix=/opt/squid --with-default-user=squid --with-openssl -enable-ssl --enable-ssl-crtd --enable-stacktraces --without-mit-krb5 --without-heimdal-krb5 --with-logdir=/opt/squid/log/squid --with-pidfile=/opt/squid/run/squid.pid
make all
sudo make install
ls /opt/squid
```

## セットアップ
コンピュータの3128ポートは解放しておきましょう。  
クラウドサービスを使っている場合、ローカル通信であってもネットワークのセキュリティ設定で3128ポートの通信を許可する必要がある場合が多いです。

```
sudo iptables -I INPUT -p tcp --dport 3128 -j ACCEPT
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

squidユーザーを作り、/opt/squidをsqidユーザーの所有にします。

```
sudo useradd -m -U -s /bin/bash squid
mkdir -p /opt/squid/log
sudo chown -R squid:squid /opt/squid

sudo su - squid
```

証明書を作成します。

```
openssl req -new -newkey rsa:2048 -sha256 -nodes -x509 -extensions v3_ca -keyout myCA.pem -out myCA.pem
```

myCA.pemの内容は保存しておいてください。あとで使います（使うときにcatするので構いません）。

```
cat myCA.pem
```

squid.confを編集します。

```
nano /opt/squid/etc/squid.conf
```

まず、`http_port 3128`の行を次のように変更してください。

```
http_port 3128 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=16MB cert=/home/squid/myCA.pem
```

また、末尾に次の内容を付け足してください。

```
sslcrtd_program /opt/squid/libexec/security_file_certgen -s /opt/squid/var/cache/squid/ssl_db -M 16MB
sslcrtd_children 20
sslproxy_cert_error deny all
ssl_bump stare all
```

「SSL証明書データベースディレクトリ」を作るために以下のコマンドを実行します。

```
/opt/squid/libexec/security_file_certgen -c -s /opt/squid/var/cache/squid/ssl_db -M 16MB
```

## 実行
設定が終わったのでSquidを起動します。

```
/opt/squid/sbin/squid
```

ログはファイルに保存されるので、とりあえず次のように確認します。

```
tail -n 30 -f /opt/squid/log/squid/cache.log
```

## プロキシさせたいサーバーに証明書インストール
ここからはプロキシを使わせたいサーバーのターミナルを開いて作業します。

証明書をインストールします。
ca-certificatesは/usr/share配下にもありますが、Ubuntu 20.04ではupdate-ca-certificatesは/usr/local/shareを参照するようです。

```
sudo mkdir /usr/local/share/ca-certificates/squid
```

myCA.pemの`-----BEGIN CERTIFICATE-----`から`-----END CERTIFICATE-----`をコピーしてください。

```
sudo nano /usr/local/share/ca-certificates/squid/squid.crt
```

update-ca-certificatesします。

```
sudo update-ca-certificates
```

1 addedと表示されたらOKです。

## プロキシが動作しているか確認
https接続できるか確認します。

```
export https_proxy=http://SquidサーバーのIPアドレス:3128
curl https://google.com
```

301 MovedのHTMLが表示されたら成功です！

![](https://firebasestorage.googleapis.com/v0/b/hideaki-97c59.appspot.com/o/images%2FPFOKUISFS1RaFz4ghSnc2GS6l5z2%2Fn4_Fwz92k.png?alt=media)

証明書のインストールにエラーがある場合は次のように表示されます。

![](https://firebasestorage.googleapis.com/v0/b/hideaki-97c59.appspot.com/o/images%2FPFOKUISFS1RaFz4ghSnc2GS6l5z2%2FbgxNEU-rD.png?alt=media)

また、ネットワーク設定が間違っているとタイムアウトなどになります。

## 

実際にSquidサーバーでアクセスログを確認してみましょう。

```
tail -n 30 -f /opt/squid/log/squid/access.log
```

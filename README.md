# 環境構築手順書

## はじめに

今回のToDoリストが動く環境の構築手順を記述していきたいと思います。

## バージョン

| vagrant | php    | Nginx  | MySQL | Laravel | centOS |
| ------- | ------ | ------ | ----- | ------- | ------ |
| 2.2.16  | 7.3.29 | 1.21.2 | 5.7   | 6.0.0   | 7      |

## インストール

はじめに環境構築に必要なソフトウェアのインストールを行っていきます。

手順としては以下の通りです。



1. vagrantのインストール
2. OS(今回はCentOSの7)のインストール
3. PHPのインストール
4. Nginxのイントール
5. MySQLのインストール




### 1. vagrantのインストール

まず，vagrantのインストールをする前に，**Virtual Box**のインストールを行います。

#### Virtual Boxのインストール

[Virtual Box](https://www.virtualbox.org/wiki/Download_Old_Builds_6_0)

選択するバージョンはvagrantが対応しているver6.0.14を選択してください。

それぞれのOSのhostsを選択しインストールした後`$ virtualbox`のコマンドを実行しウィンドウが表示されれば正常にインストールされています。

```bash
$ virtualbox
^C(command C)
```

次にvagrantのインストールに移ります。

#### vagrantのインストール

```bash
$ brew cask install vagrant
```

エラーの場合は

```bash
$ brew install --cask vagrant
```

以上のコマンドでインストール完了です。

### 2. CentOSのインストール

次に，VirtualBoxにCentOSのダウンロードを行います。

```shell
$ vagrant box add centos/7
```

実行後に選択肢が表示されます。

```shell
1) hyperv
2)libvirt
3)virtualbox
4)vmare_desktop

Enter your choice: 3
```

`Successfully`と出たら完了です。

#### 作業ディレクトリの用意

どこでも良いのでホストOS側に**vagrant**の作業用のディレクトリを用意してください。

コマンドラインで作ったディレクトリに移動し，下記のコマンドを実行します。

```shell
$ vagrant init centos/7
```

#### vagrantfileの編集

下記の２点をコメントインします。

```none
config.vm.network "forwarded_port", gust: 80, host:8080

config.vm.network "private_network", ip: "192.168.33.10"
```

更に下記の点を，編集した後，これもコメントインしていきます。

```none
# config.vm.synced_folder "../data", "/vagrant_data"
↓
config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
```

#### vagrantプラグインのインストール

```shell
$ vagrant plugin install vagrant-vbguest
```

この拡張機能を入れることでGuest AddditionsのバージョンをVirtualBoxのバージョンに合わせてくれます。

以下のコマンドでインストールされたか確認できます。

```shell
$ vagrant plugin list
```

#### vagrantを通してのゲストOSの起動とログイン

コマンドラインで，vagrantfileのある作業ディレクトリに移動し下記のコマンド実行をします。

```shell
$ vagrant up
```

起動が完了したら`ssh`でゲストOSへログインしていきます。

```shell
$ vagrant ssh
```

ログインが完了したら下記のコマンドを実行し，開発に必要なツールをまとめてインストールします。

```shell
$ sudo yum -y gtoupinstall "development tools"
```

### 3. PHPのインストール

`yum`コマンドを使用した場合古いバージョンがインストールされてしまいますので，外部ツールからPHPのインストールをしていきます。

```shell
$ sudo yum -y install epel-release wget
$ sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
$ sudo rpm -Uvh remi-release-7.rpm
$ sudo yum -y install --enablerepo=remi-php72 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
$ php -v
```

`php -v`でバージョンが表示されれば完了です。

#### composerのインストール

phpのパッケージ管理ツールである`composer`をインストールしていきます。

```shell
$ php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

$ php composer-setup.php
$ php -r "unlink('composer-setup.php');"

$ sudo mv composer.phar /usr/local/bin/composer
composer -v
```

### 4.Nginxインストール

#### インストールの準備

下記のコマンドでインストールの準備をしていきます。

```shell
$ sudo vi /etc/yum.repos.d/nginx.repo
```

このファイルに以下を書き込んでいきます。

```none
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```

#### インストール

```shell
$ sudo yum install -y nginx
$ nginx -v
```

#### Laravelを動かすための設定

LaravelをNginxで動かすための設定を施していきます。

##### Nginxのコンフィグファイルの編集

下記のファイルを編集

```shell
$ sudo vi /etc/nginx/conf.d/default.conf
```

```none
server {
  listen       80;
  server_name  192.168.33.19; # 今回は192.168.33.1で設定いたします。
  # 下記の2項目を追記
  root /vagrant/laravel_app/public; 
  index  index.html index.htm index.php; 

  #charset koi8-r;
  #access_log  /var/log/nginx/host.access.log  main;

  location / {
  		# 下記2項目をコメントアウト。
      #root   /usr/share/nginx/html; # コメントアウト
      #index  index.html index.htm;  # コメントアウト
      
      # 更に下記を追記
      try_files $uri $uri/ /index.php$is_args$args;  # 追記
  }

# 省略

  # 必要な部分だけ，コメントインしていきます。
  # 「root」以外の「location { }」 までのコメントを解除。

  location ~ \.php$ {
  #    root           html;
      fastcgi_pass   127.0.0.1:9000;
      fastcgi_index  index.php;
      # 下記の$fastcgi_script_name以前を /$document_root/に変更
      fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name;  
      include        fastcgi_params;
  }

```



##### php-fpmの設定

```shell
$ sudo vi /etc/php-fpm.d/www.conf
```

```none
# 24行目付近でuserとgroupの項目を探し，nginxに編集

user = nginx
group = nginx
```



#### Nginxとphp-fpmの起動

以上の設定を行ったあと，php-fpmの起動と，Nginxの再起動を行っていきたいと思います。

```shell
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```



#### Nginxに対する権限の付与

現在の状態のままだと，一ユーザーであるNginxにはファイルの読み込み権限や実行権限がほとんど存在せず，Laravelが起動したところで全く何もできない状態になっています。

なので，`chmod`コマンドで権限を付与していきたいと思います。

```shell
$ cd /vagrant/laravel_app
$ sudo chmod -R 777 storage
```

#### ファイアウォールの正常化

このままではファイアウォールに阻まれてしまうので，使っているポートを経由した通信ができるようにコマンドを実行します。

```shell
$ sudo systemctl start firewalld.service
$ sudo firewall-cmd --add-service=http --zone=public --permanent

$ sudo firewall-cmd --reload
```

ファイアウォールを無効にしても***Forbidden***が表示されてしまう場合は，SELinuxを調整してみてください。

### 5. データベースのインストール

今回使用するのは，MySQL<small>(ver 5.7)</small>です。

```shell
$ sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
$ sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
$ sudo yum install -y mysql-community-server
$ mysql --version
```

バージョンが確認できたら起動します。

```shell
$ sudo systemctl start mysqld
$ mysql -u root -p
```

パスワードを変更する必要があるので，下記のコマンドを実行します。

```shell
$ sudo cat /var/log/mysqld.log | grep 'temporary password'
2017-01-01T00:00:00.000000Z 1 [Note] A temporary password is generated for root@localhost: hogehoge
```

`hogehoge`の部分が現在のパスワードです。これで入ってパスワードを変更していきます。

```shell
$ mysql -u root -p
$ Enter passward:
```

**「大文字小文字の英数字 + 記号かつ8文字以上」**が設定できます。

```mysql
set password = "newPassword";
```

これでパスワードがリセットできました。

#### データベースの作成

```mysql
create database laravel_app;
```

#### Laravelへ設定

`Laravel_app`ディレクトリ配下に存在する`.env`ファイルを以下に編集します。

```none
DB_PASSWORD=設定したパスワード
```

その後，Laravel_appに移動して`php artisan migrate`を実行します。

```shell
$ php artisan migrate
```

マイグレーションができた状態でアクセスし，データベース必要な動作が問題なく動いていればLaravelとデータベースが接続されていることが確認できます。



### 終わり

以上で環境構築が終了しているはずです。すべての設定が完了したので，自身のファイルをマウントディレクトリに配置すればLaravelを動かせるかと思います。





## 環境構築の雑感，注意点

何らかの理由でVagrant配下がマウントされないという問題が発生いたしました。いくつか原因はあると思うのですが，私の場合は以下の記事を参考にさせていただきました。

[# Vagrant + VirtualBOx で 最新のCentOS7 vbox(centos/7 2004.01)でマウントできない問題](https://qiita.com/mao172/items/f1af5bedd0e9536169ae)

他の問題はタイポだったりがほとんどだったので，今回の環境構築の中で私がぶつかった理不尽な問題としては，上記に限ります。



と思ったんですが，そういえば途中までSELinuxが有効になっていて一生`403 Forbidden`になっていたこともありました。

完全に失念していましたが，下記の記事を参考にして思い出しました。

[設定もファイルのパーミッションも所有者も問題ないのに 403 Forbidden になってしまうときの対処法](https://zenn.dev/noraworld/articles/how-to-disable-selinux)


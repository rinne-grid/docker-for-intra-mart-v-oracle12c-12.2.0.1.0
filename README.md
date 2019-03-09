

## docker for intra-mart  version Oracle

* intra-martの検証環境をDockerコンテナ上に作成し、環境構築にかかる時間を削減

### Dockerで作成するintra-martのシステム構成

#### Docker上の環境


|ホスト名|コンテナイメージ|目的|
|--------|------------------------------|-----------|
|ap|CentOS 7.5.1804(※1）(library/centos)|resin-proの実行及びwarデプロイ|
|db|Oracle Database 12.2.0.1.0(ビルドで作成)|intra-martに関するデータの保存|
|adminer|Adminer(library/adminer)|dbのデータ参照用のアプリ|


（※1 厳密には、NTTデータイントラマート社はintra-martが動作するLinux環境として、Red Hat Enterprise Linux 6.x、7.xのみを動作保証しているため、
本Docker関連ファイルを利用する場合は、あくまでも動作検証用の環境に留めておくことをおすすめします。本記事の内容によって発生した障害等について、一切責任を負いません）



### 必要なPC環境

* 下記のいずれか

|OS|バージョン|Docker|
|--|----------|------|
|Windows7以降|64bit|Docker Toolbox|
|Windows10|64bit|Docker for Windows|
|macOS|-|Docker for Mac|


### 事象別のコマンドリファレンス

|事象|コマンド|
|----|--------|
|Dockerコンテナを開始したい|docker-compose up|
|Dockerコンテナをビルドしたい|docker-compose build --no-cache|
|Dockerコンテナを終了させたい|docker-compose down|
|DBデータやストレージを削除して、<br>新しくテナント環境セットアップから始めたい（永続化しているコンテナのデータが全部消えるので要注意）|docker-compose up <br>docker-compose down -v<br>docker-compose up|
|warファイルをアンデプロイしたい|docker-compose exec ap /ap-server/bin/resinctl undeploy imart|
|warファイルをデプロイしたい|docker-compose exec ap /ap-server/bin/resinctl deploy /war/imart.war|
|Oracleイメージを単独で作成したい場合|docker build -t oracle12c_local:latest ./db/build --build-arg DB_EDITION=SE2|

### コンテナごとの接続情報

* APサーバー（CentOS）

|コンテナ名|ホスト名|ポート番号(ホスト)|ポート番号(コンテナ)|
|----------|--------|----------|------|
|im_ap_n|ap|8888|8080|

* Oracle Database

|コンテナ名|ホスト名|DB名|ユーザ名|パスワード|ポート番号(ホスト)|ポート番号(コンテナ)|
|----------|--------|----|--------|----------|----------|----|
|im_db_n|db|IMART|IMART|IMART|15210|1521|

* Adminer

|コンテナ名|ホスト名|ポート番号(ホスト)|ポート番号(ゲスト)|
|----------|--------|----------|----|
|im_ap_n|adminer|8889|8080|





### 利用手順

本Dockerプロジェクトを利用して、intra-martの環境を構築する手順を記述します。
（この手順は、Windows10 + Docker Toolboxを前提にしています。必要に応じて適宜読み替えをお願いします。）

#### [1] Docker Toolboxのインストール

* 下記のURLより、Docker Toolboxをダウンロードし、インストールします

https://docs.docker.com/toolbox/toolbox_install_windows/


#### [2] Dockerのメモリ、ストレージを増やす

Oracle Databaseをインストールする際、空きディスク容量は15GB以上必要とのことです。

したがって、下記のコマンドで新たにdocker-machineを作ります。
今回は、ディスクサイズが50GB、メモリは8GBにしました。
必要に応じて、各数値は変更してください。

```sh
$ docker-machine create -d virtualbox --virtualbox-disk-size 50000 --virtualbox-memory "8192" default
```


* 既存のdocker-machineのメモリを増やす方法については下記のURLの記事に詳しく記載されています  
https://qiita.com/niisan-tokyo/items/2d7d21aeb4e25f7a7bbe


* 既存のdocker-machineのストレージサイズを増やしたい場合でも、下記の記事のように
一度作り直したほうがシンプルでわかりやすいです  
https://qiita.com/sumyapp/items/ff5529eb1681c7bde3d0


* Docker for Windowsの場合は、下記のURLが参考になります  
https://qiita.com/fkooo/items/d2fddef9091b906675ca

  > タスクトレイのDockerアイコン右クリック->Settingsを開く

* Docker for Macの場合の場合、下記のURLが参考になります  
https://qiita.com/mks1412/items/9356187a3bcb20b64e82
  > Preferences → Advanced にあるMemoryで使用量を調節

  


#### [2] Gitのインストール

* 下記のURLより、Git for Windowsをダウンロードし、インストールします

  https://gitforwindows.org/



#### [3] Dockerプロジェクトのダウンロードと初期設定

* 任意のフォルダで、以下のコマンドを実行し、dockerプロジェクトをダウンロードします

```sh
> git clone https://github.com/rinne-grid/docker-for-intra-mart-v-oracle12c-12.2.0.1.0 im
> cd im
```

* warファイルの配置用フォルダを作成します

```sh
> mkdir .\ap\war
```


#### [4] Oracle Database 12.2.0.1.0をダウンロードします

* 下記のURLにアクセスし、(12.2.0.1.0) - Standard Edition 2 and Enterprise Editionを見つけ、Linux x86-64をダウンロードします

  https://www.oracle.com/technetwork/jp/database/enterprise-edition/downloads/index.html

* ダウンロードしたファイル(linuxx64-12201_database.zip)をim/db/buildフォルダに配置します


#### [5] Jugglingでwarファイルを作成

プロジェクト名をimartにして、必要なモジュールを選択し、設定を行います。
今回のDocker環境をそのまま利用するためには、下記ファイルの設定を変更する必要があります

* storage-config.xmlを設定する
* resin-web.xmlを設定する
* 出力するwarファイル名をimart.warとする

（Dockerプロジェクトのap/.envファイルを変更することで、別の値を指定することも可能です）

##### storage-config.xmlの設定

imart/config/storage-config.xml の 19行目付近を以下のとおりに変更します

```xml
<root-path-name>/im-data/storage</root-path-name>
```

##### resin-web.xmlの設定

imart/resin-web.xml 内容を下記のとおりにします

```xml
<web-app xmlns="http://caucho.com/ns/resin" xmlns:resin="urn:java:com.caucho.resin">
    <character-encoding>UTF-8</character-encoding>

    <log-handler name="" class="jp.co.intra_mart.common.platform.log.handler.JDKLoggingOverIntramartLoggerHandler"/>
    <logger name="debug.com.sun.portal" level="warning" />

	<!-- im_service(im_asynchronous) -->
	<resource jndi-name="jca/work" type="jp.co.intra_mart.system.asynchronous.impl.executor.work.resin.ResinResourceAdapter" />
	<jsp>
		<recycle-tags>false</recycle-tags>
	</jsp>
	<database jndi-name="jdbc/default">
		<driver>
		   <type>oracle.jdbc.driver.OracleDriver</type>
		   <url>jdbc:oracle:thin:@db:1521/RNGD</url>
			<user>IMART</user>
			<password>IMART</password>
		</driver>
		<max-connections>20</max-connections>
		<prepared-statement-cache-size>0</prepared-statement-cache-size>
	</database>
	<session-config>
	    <reuse-session-id>false</reuse-session-id>
		<session-timeout>30</session-timeout>
	</session-config>

	<mime-mapping extension=".json" mime-type="application/json"/>
</web-app>
```


##### warファイルの出力

imart.warという名称でwarファイルを出力したら、
プロジェクトのim/ap/warフォルダの中に、warファイルをコピーします



#### [6] intra-martのサイトからLinuxのresin-proをダウンロード

* intr-martのサイトにアクセスし、プロダクトファイルダウンロードボタンを押下します。

  https://www.intra-mart.jp/download/library/

ライセンスキーを入力すると、ダウンロード可能なファイル一覧が表示されます。

なお、intra-martサイトにも書いているとおり、.tar.gzがLinux用のresin-proになります。

  https://www.intra-mart.jp/download/product/iap/setup/iap_setup_guide/texts/install/linux/resin_linux.html

最新のResin<b>resin-pro-4.0.xx.tar.gz</b>を入手します。



#### [7] Dockerプロジェクトのフォルダにresin-proをコピー

* 上記のresin-pro.4.0.xxを展開し「resin-pro.4.0.xx」の名称をresin-proに変更します
* resin-proフォルダをim/apフォルダにコピーします


#### [8] JDBCドライバー(ojdbc8.jar)をダウンロードし、im/ap/resin-pro/libにコピー

* 下記のURLにアクセスし、ojdbc8.jarを見つけ、ダウンロードします

  https://www.oracle.com/technetwork/database/features/jdbc/jdbc-ucp-122-3110062.html

* ojdbc8.jarをim/ap/resin-pro/libにコピーします

#### [9] プロジェクトのフォルダ構成の確認

* フォルダを確認し、以下の構成と同じになっていることを確認します
* ポイント
  * im/ap/resin-proフォルダがあり、直下にautomakeやlibフォルダ等が存在する
  * im/ap/resin-pro/libフォルダ内にodbc8.jarファイルが存在する
  * im/ap/warフォルダがあり、imart.warファイルが存在する
  * im/db/buildフォルダ内にlinuxx64_12201_database.zipが存在する

```
im
├─ap
│  │
│  ├─resin-pro
│  │  ├─lib
│  │  │   ├─ojdbc8.jar
│  │  │   │
│  │  │   ├─activation.jar など
│  │  │          
│  │  ├─automake  など
│  │
│  ├─war
│  │  └─imart.war
│  │
│  └─Dockerfile
│
├─db
│  ├─build
│  │  ├─linuxx64_12201_database.zip
│  │  │
│  │  ├─checkDBStatus.sh など
│  │
│  ├─scripts_setup
│  │  └─01_create_imart.sh
│  │   
│  └─scripts_startup
│
├─.env
├─.gitignore
├─docker-compose.yml
└─README.md

```

#### [10] 必要に応じて、設定ファイルを変更する

* im/ap/resin-pro/conf/resin.properties の 82行目付近 - jvm_args

-Xmx, -Xmsの値が、初期状態だと8192m(8GB)が設定されているため、自分のPCのメモリ状況に合わせて変更します

```ini
jvm_args : -Dfile.encoding=UTF-8 -Djava.io.tmpdir=tmp -Xmx4096m -Xms4096m -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=30 -XX:NewSize=512m -XX:MaxNewSize=512m -XX:+CMSClassUnloadingEnabled -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+HeapDumpOnOutOfMemoryError -Xloggc:log/gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=10M
```

* HTTPプロキシの設定 im/.env

社内ネットワーク等で、プロキシサーバーを経由する必要がある場合、.envのHTTP_PROXY、HTTPS_PROXYに値を設定します

```env
HTTP_PROXY=http://user:password@server:port/
HTTPS_PROXY=http://user:password@server:port/
```

#### [11] Docker Machineを起動

* Docker Machineを起動していない場合、下記コマンドで起動します
  * Docker Toolbeltという青いアイコンのショートカットを起動すると、自動でDocker Machineを起動してくれます

![docker-toolbelt](http://www.rinsymbol.sakura.ne.jp/github_images/docker/docker-toolbelt.PNG)

```sh
> docker-machine start
```


#### [12] Dockerコンテナの起動

* プロジェクトフォルダに移動します

```sh
> cd any_folder\im
```

* docker-composeを利用し、コンテナを起動します

```sh
> $ docker-compose up -d ; docker-compose logs -f
```

イメージ作成の際にアップデート等の処理が走るので、15分程度～30分程度かかります。

terminalにいろいろと出力されていきます。
以下の表示が出たら終了と判断して良いと思います。Ctrl+C等でを強制終了します。

```
db_1       | #########################
db_1       | DATABASE IS READY TO USE!
db_1       | #########################
db_1       | The following output is now a tail of the alert.log:
db_1       | EXTENT MANAGEMENT LOCAL AUTOALLOCATE
db_1       | SEGMENT SPACE MANAGEMENT AUTO
db_1       | 2019-03-07T14:11:29.343682+00:00
db_1       | RNGD(3):Completed: CREATE TABLESPACE IMART
db_1       | DATAFILE '/opt/oracle/oradata/ORCL/XXXX/IMART.dbf' SIZE 7000M REUSE
db_1       | LOGGING
db_1       | ONLINE
db_1       | BLOCKSIZE 8K
db_1       | EXTENT MANAGEMENT LOCAL AUTOALLOCATE
```


ホストマシン側からOracle Databaseに接続したい場合は下記の指定を行います。

(設定を変更していない場合)

|ユーザID|パスワード|SID|
|----------|--------|----------|
|IMART|IMART|192.168.99.100:15210/RNGD|

（Docker for WindowsやDocker for Macの場合は、上記IPアドレスではなく  localhost:15210  もしくは自分で設定しているホスト名やIPアドレスを指定します。）


* resinのindexページに接続します

http://192.168.99.100:8888

![docker-toolbelt](http://www.rinsymbol.sakura.ne.jp/github_images/docker/resin-top.PNG)

（Docker for WindowsやDocker for Macの場合は、上記IPアドレスではなく  http://localhost:8888  もしくは自分で設定しているホスト名やIPアドレスにアクセスします。）

* resinのページが開けることが確認できたら、warファイルをデプロイします

```sh
> docker-compose exec ap /ap-server/bin/resinctl deploy /war/imart.war
```


コンテナ内のresin-proの場所は、/ap-serverです。
im/.envファイル内の変数を変更することで、お好きな場所を指定できます。

* デプロイのコマンドが終了して、数分経ったらintra-martのセットアップページにアクセスします
  * 503 Service Temporarily Unavailableが発生する場合は、もう少しだけ待ってあげてください。

http://192.168.99.100:8888/imart/system/login



* 無事にテナント設定画面が表示されるので、テナント環境セットアップを実行します

![tenant1](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant.PNG)


* テナントIDはimartを指定します

![tenant2](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant2.PNG)
* リソース参照名は一覧に表示されたものを選択します

![tenant3](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant3.PNG)

* テナント登録を行い、しばらく待ちます

![tenant4](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant4.PNG)

* テナント環境セットアップが適切に動作しているかについて、ログを確認したい場合以下のコマンドを実行します

```
$ docker-compose exec ap bash
$ less -f ./log/jvm-app-0.log
```


* データベースやストレージ情報はDocker Volumeに保存しているため、データは永続化されています
  * 一度、docker-compose downで終了し、もう一度docker-compose upを試して、システムログイン画面にアクセスすると、ダッシュボードが表示されることがわかります

![dashboard](http://www.rinsymbol.sakura.ne.jp/github_images/docker/dashboard.PNG)


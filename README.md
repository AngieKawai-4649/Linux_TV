# Linux (主にLinuxMint ubuntu系)でTVを視聴／録画する為のインストールガイド)  
- 視聴アプリ vlc or smplayer  
- TV配信webサーバー  mirakurun  
- 録画アプリ EPGStation  
- B25復号化  libarib25.so  
- 対象チューナーデバイス
  - 慶安 fsusb2n(2期)  
  - さんぱくん外出  
  - PT2/PT3 (ドライバーはlinux defaultのdvb 録画アプリ recdvb)  

## 【予めインストールする必須パッケージ】  
  **[ビルド関連]**
  - g++
  - pkg-config

  **[各アプリ取得]**
  - git

  **[EPGStation]**
  - ffmpeg
  - mariadb-server
  - python

  **[mirakurun EPGStation が動作する基盤ソフトウエア]**
  - nodejs
  - npm

  **[ICカードリーダー関連]**
  - libccid
  - libpcsclite-dev
  - libpcsclite1 (libpcsc-perl)
  - pcsc-tools
  - pcscd

  **[視聴アプリ]**
  - vlc
  - smplayer

  **[推奨: synapticパッケージマネージャ]**
  - synaptic

**環境構築ディレクトリ**  
  /opt/TV_app  
  とする  

## 【ライブラリパスの確認及び設定】
  libarib25 libsoftcas はシステムライブラリパス /usr/local/libに配置するようにmakefileが作成されているのでそのままで問題無いが  
  ライブラリを独自のディレクトリで管理する場合は以下の設定をする  
    システム全体に反映させるライブラリパス、またはローカル環境で使用するライブラリパスのどちらか使用する方を決めて設定する

    1.システムライブラリパス
    1.1 システムパスの確認
        独自のディレクトリにライブラリパスが通っているか調べる
        $ ldconfig -v | grep 独自のディレクトリ
    1.2 システムパスの設定
        独自のライブラリパスをシステムパスに追加
        /etc/ld.so.conf.d/original_path.conf (ファイル名は任意)を作成
        $ echo オリジナルパス(フルパス) | sudo tee /etc/ld.so.conf.d/original_path.conf > /dev/null
        または
        $ sudo sh -c "echo オリジナルパス(フルパス) > /etc/ld.so.conf.d/original_path.conf"

    1.3 共有ライブラリ配置後に/etc/ld.so.cacheを作成
        $ sudo ldconfig (これはlibarib25 libsoftcas make install で/usr/local/libにコピー後実行している)
        リンカーはキャッシュファイル/etc/ld.so.cacheを参照し、リンクするのでライブラリ配置後、必ずldconfigを実行する
        内容確認
        $ ldconfig -p | grep -e libarib25 -e libsoftcas

    2.ローカル環境のライブラリパス
        ビルドする前に環境変数 LD_LIBRARY_PATH にパスを設定する
        このパスはシステムライブラリパスより優先される
        例
        $ export LD_LIBRARY_PATH=/opt/lib/arib25:/opt/lib/softcas:$LD_LIBRARY_PATH
        上記ディレクトリに libarib25.so libsoftcas.so を置く
        ビルド後に
        $ ldd ロードモジュール でリンクされたライブラリを確認する
        /usr/local/libに同様のライブラリが存在しても上記ディレクトリのライブラリがリンクされる

## 【ライブラリのリンク】
    Makefile の LDFLAGS または LIBS に -larib25 等と記載してあるが、これは  
    -l:ライブラリファイルを指定 この場合はarib25の前にlibを付け、libarib25... というライブラリファイルをLD_LIBRARY_PATH、  
       ライブラリキャッシュの順に探しundefined symbolを解決しに行く
    因みに
    -L:ライブラリパスを指定
    例：-L/opt/work/lib

    ライブラリファイル、パスを指定する際、その順番が重要である（undefined symbolを探す順番となる)
    上記例では
    LDFLAGS=`pkg-config libpcsclite --libs` -larib25  (pkg-config libpcsclite --libsの結果は-lpcsclite)
    と記載するとundefined referenceでリンクエラーとなる
    libarib25.soからlibpcsclite.so内の関数を呼び出している為である
    呼び出しているライブラリ -> 呼び出されているライブラリの順で記載しなければならない
    正しくは以下とする
    LDFLAGS=-larib25 `pkg-config libpcsclite --libs`

## 【柔らかいヤツ】
  割愛

## 【arib25】
  b25復号化する為のライブラリおよびロードモジュール  
  tunerアプリでb25解除する場合にセットアップする  
  mirakurun側(arib-b25-stream-test)で解除する場合は  
  arib-b25-stream-testにb25デコーダー機能が組み込まれているので導入する必要は無い

    1.ソースファイル一式を取得
      cd /opt/TV_app/
      $ git clone https://github.com/AngieKawai-4649/libarib25.git
      または
      $ curl -OL https://github.com/AngieKawai-4649/libarib25/releases/download/master/master_src.tar.gz
      を解凍
    2.ビルド
    2.1 共有ライブラリのビルド
      $ cd /opt/TV_app/libarib25/src
      $ make -f Make_lib
      $ sudo make -f Make_lib install
    2.2 ロードモジュールのビルド
      $ cd /opt/TV_app/libarib25/src
      $ make -f Make_exe [オプション]
      オプションについては Make_exeのコメント欄を参照
      $ sudo make -f Make_exe install
    3.確認
      $ ldd b25                              リンクされた共有ライブラリを確認
      $ ldd libarib25.so                     リンクされた共有ライブラリを確認
      $ readelf -s libarib25.so | grep FILE  ライブラリを構成しているソースファイルを確認
      $ ldconfig -p | grep libarib25         ldキャッシュにlibarib25.soが組み込まれていることを確認

## 【チューナーデバイス環境構築】

**[慶安 fsusb2n(2期)]**  
**[さんぱくん外出]**

    1.videoグループの作成
      確認
      $ cat /etc/group | grep video
      無ければ作成
      $ sudo groupadd video
    2.使用ユーザーをvideoグループに追加
      確認
      $ groups
      無ければ追加
      $ sudo gpasswd -a ユーザー名 video
    3.udevのルールを設定
      確認
      $ ls -l /lib/udev/rules.d /etc/udev/rules.d
            ここに無い数字から始まるファイルを/etc/udev/rules.d に作成する
      例:/etc/udev/rules.d/93-tuner.rules
    4.USBデバイスの固定
      USB機器を追加、取り外しを行うとチューナーデバイスのUSBパスが変わってしまうので
       チューナーを接続しているUSBポートの物理的な場所に対してチューナーデバイス名称を付ける
      確認
      $ lsusb    各チューナーデバイスのUSBパスを調べる
      $ udevadm info -n /dev/bus デバイスパス -q path
      例 $ udevadm info -n /dev/bus/usb/001/005 -q path
      出力値例
     /devices/pci0000:00/0000:00:08.1/0000:05:00.3/usb1/1-4/1-4.1 が物理的な場所である
     rulesファイルでDEVPATHに指定する
     SYMLINKで指定した名称で /devにデバイスのシンボリックリンクが作成されるのでmirakurun tuners.ymlでそのデバイス名称を使用する
     ATTRを調べる例 $ udevadm info -a -p `udevadm info -q path -n /dev/bus/usb/001/005`

**[93-tuner.rulesファイルの内容例]**  
\# FSUSB2N  
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTRS{idVendor}=="0511", ATTRS{idProduct}=="0029", DEVPATH=="/devices/pci0000:00/0000:00:08.1/0000:04:00.4/usb3/3-2/3-2.1", MODE="0664", GROUP="video", SYMLINK+="FSUSB2N_1"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTRS{idVendor}=="0511", ATTRS{idProduct}=="0029", DEVPATH=="/devices/pci0000:00/0000:00:08.1/0000:04:00.4/usb3/3-2/3-2.2", MODE="0664", GROUP="video", SYMLINK+="FSUSB2N_2"
\# さんぱくん  
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTRS{idVendor}=="0511", ATTRS{idProduct}=="0045", DEVPATH=="/devices/pci0000:00/0000:00:08.1/0000:04:00.4/usb3/3-2/3-2.3", MODE="0664", GROUP="video", SYMLINK+="SANPAKUN_1"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTRS{idVendor}=="0511", ATTRS{idProduct}=="0045", DEVPATH=="/devices/pci0000:00/0000:00:08.1/0000:04:00.4/usb3/3-2/3-2.4", MODE="0664", GROUP="video", SYMLINK+="SANPAKUN_2"

    5.ファイル保存後、ルールを反映
      $ sudo udevadm control --reload
    6.リブート後、デバイスを確認する
      $ ls /dev

**[PT2 PT3 (dvb)]**

PT2,PT3はデフォルトでdvbドライバーがインストールされるのでそれを使用する
デフォルトのデバイス割当は以下となっている
- PT2  BS/CS:adapter0 2  地上波:adapter1 3
- PT3  BS/CS:adapter0 1  地上波:adapter2 3

**[デバイスの固定]**

1PCにPT2とPT3を取り付けているとチューナーデバイスがかち合ってしまいブートする度に割当が異なるのでデバイスを固定する  
実際の割当状況を /var/log/kern.log で確認できる  
- earth_pt1
- earth_pt3

デバイス場所

    $ ls /dev/dvb  
      adapter0  adapter1  adapter2  adapter3  adapter4  adapter5  adapter6  adapter7 ...

固定方法

    /etc/modprobe.d/options-dvb.conf を作成し以下を記述する
    衛星0 地上0 衛星1 地上1の順に指定  
    options earth_pt1 adapter_nr=0,1,2,3
    options earth_pt3 adapter_nr=4,5,6,7
    この場合は PT2 0 2 がBS/CS 1 3 が地上波
    PT3 4 6 がBS/CS 5 7 が地上波
    となる
    リブートして確認する

## 【tunerアプリビルド】
実行形式ファイルの共有ライブラリリンク状況は以下で確認できる  
例  $ ldd recfsusb2n

**[recfsusb2n]**

    1. ソースコードの取得
      $ cd /opt/TV_app
      $ git clone https://github.com/AngieKawai-4649/recfsusb2n.git
    ２．ビルド
      $ cd recfsusb/src
      $ make [オプション]
      オプションについてはMakefileのコメントを参照

**[recsanpakun]**

    1. ソースコードの取得
      $ cd /opt/TV_app
      $ git clone https://github.com/AngieKawai-4649/recsanpakun.git

    ２．ビルド
      $ cd recsanpakun/src
      $ make [オプション]
      オプションについてはMakefileのコメントを参照

   **[recdvb]**

    1. ソースコードの取得
      $ cd /opt/TV_app
      $ git clone https://github.com/AngieKawai-4649/recdvb.git
    ２．ビルド
      $ cd recdvb/src
      $ make [オプション]
      オプションについてはMakefileのコメントを参照

## 【nodejs】
mirakurun EPGStation をインストール前にnodejsをセットアップする
- nodejs: サーバーサイドjavascript
- npm:    Node Package Manager Nodejsのパッケージを管理するツール
- n:      nodejs version管理ツール
- pm2:    nodejs 上で稼働する各アプリケーションの動作を管理するツール

**[セットアップ]**

    1.nodejs npm をインストール
      $ sudo apt install -y nodejs npm
       または synaptic パッケージマネージャーでインストール
    2.version確認
      $ node -v
      $ npm -v
    3.version管理する場合 n を導入する
      $ sudo npm install -g n
    3.1 nodejs version 10 にする場合
      $ sudo n 10
    3.2 nodejs安定版にversion up する場合
      $ sudo n stable
    3.3 nodejs最新版にする場合
      $ sudo n latest
    3.4 nodejsのバージョンについて
        mirakurun@3.9.0-rc.4
        nodejs version v20でインストールエラーとなるのでv18以下にする
    4.npmのversion up (任意)
      $ sudo npm update -g npm
    5.pm2のインストール
      $ sudo npm install -g pm2

## 【mirakurun】
    1.pm2 起動
      $ sudo pm2 startup
    2.mirakurun インストール
      $ sudo npm install mirakurun -g --unsafe --production
      $ sudo npm install arib-b25-stream-test -g --unsafe
      注：これは録画アプリでB25解除を行う場合は必要無い
    3.config 設定
    3.1 mirakurun停止
      $ sudo mirakurun stop (またはsudo pm2 stop 起動番号 or mirakurun-server )
    3.2 設定ファイル編集
      # cd /usr/local/etc/mirakurun/
      tuners.yml  :使用するチューナーデバイス、アプリを設定する
      channels.yml:最新のチャンネル情報に設定する
      server.yml  :各自の環境に合わせて編集する
    4.ログファイルの出力先を変えたい場合にシンボリックリンクを設定する
      例:
      $ mkdir -p /opt/TV_app/mirakurun/db/mirakurun
      $ mkdir /opt/TV_app/mirakurun/log
      $ mkdir /opt/TV_app/mirakurun/run
      $ cd /usr/local/var
      $ sudo rm -r db log run
      $ sudo ln -s /opt/TV_app/mirakurun/db db
      $ sudo ln -s /opt/TV_app/mirakurun/log log
      $ sudo ln -s /opt/TV_app/mirakurun/run run
    5.mirakurun コマンド
        # 起動
            $ sudo mirakurun start
            (またはsudo pm2 start 起動番号 or mirakurun-server )
        # 停止
            $ sudo mirakurun stop
            (またはsudo pm2 stop 起動番号 or mirakurun-server )
        # 再起動
            $ sudo mirakurun restart
            (またはsudo pm2 restart 起動番号 or mirakurun-server )
        # 確認
            sudo mirakurun status
            (またはsudo pm2 status 起動番号 or mirakurun-server )
        注:pm2をrootで起動すると失敗するのでsudoで実行する
           $HOME/.pm2/ に設定ファイルがある
         [PM2][ERROR] script not found : /usr/lib/node_modules/mirakurun/mirakurun-server
         script not found : /usr/lib/node_modules/mirakurun/mirakurun-server
    6.環境変数設定及び反映
      mirakurunから起動するtunerアプリに環境変数を渡す場合
      /etc/environment に環境変数を設定する (root起動の場合)

      SOFTCASPATH,BSCSCHPATHを追加する例
        
        export SOFTCASPATH=/opt/TV_app/config
        export BSCSCHPATH=/opt/TV_app/config

        リブート後、以下を実行し環境変数を反映する (0 mirakurun 1 EPGStation の場合)
        $ sudo pm2 restart mirakurun-server EPGStation --update-env
        $ sudo pm2 save

    7.apiガイド
       localhost:40772/swagger-ui/?url=/api/docs

## 【mariadb】
EPGStationで使用する為のセットアップを行う

    1.インストール
      $ sudo apt install mariadb-server
      または synaptic パッケージマネージャーを使用しインストール
      バージョン確認
      $ mysql --version
    2.設定
    2.1 文字コード設定(utf8mb4)
    2.1.1 文字コード確認
          データースにルートでログオンする
          # mysql -u root -p  (パスワードは入れなくて良い)
          MariaDB [(none)]> show variables like "chara%";
          ログオフ
          MariaDB [(none)]> exit or quit or Ctrl+d
    2.1.2 文字コード(utf8mb4)設定
          DBサービス停止
            # systemctl stop mariadb
          設定ファイル修正
          [DBサーバ設定ファイル]
            /etc/mysql/mariadb.conf.d/50-server.cnf
          [設定値]
          [カテゴリ:mysqld]
            character-set-server  = utf8mb4
            collation-server      = utf8mb4_general_ci
          ※expire_logs_daysも合わせて修正
          [カテゴリ:mysqld]
            expire_logs_days = 1
          [DBクライアント設定ファイル]
            /etc/mysql/mariadb.conf.d/50-client.cnf
          [設定値]
          [カテゴリ:[client-mariadb]
            default-character-set = utf8mb4
          確認
          mariadbサービスを起動 (systemctl start mariadb) DBにログオンし文字コードを確認する
    2.2 データベース作成
        PGStationで使用するデータベースをepgstation_dbという名称で作成する
        [構文]  (SQLには大文字小文字の区別は無い)
              CREATE DATABASE データベース名 [CHARACTER SET 文字コード][COLLATE 照合順序];
        [作成]
              MariaDB [(none)]> create database epgstation_db;
        [確認]
              MariaDB [(none)]> show databases;
        [削除]
              MariaDB [(none)]> drop database epgstation_db;
        [DBへ接続]
             MariaDB [(none)]> use epgstation_db;
        [切り替え]
             MariaDB [(none)]> CONNECT epgstation_db;

## 【EPGstation】
    1.EPGstation ファイルの取得
      $ cd /opt/TV_app
      $ git clone https://github.com/l3tnun/EPGStation.git

    2.導入 ( ver2 )
       動作環境
      Node.js 14.6.0 以上 ( sudo n 14.6.0 )
      Mirakurun 3.2.0 以上
      ffmpeg
      Python 2.7, v3.5, v3.6, v3.7 or v3.8 node-gyp にて必要

      $ cd /opt/TV_app/EPGStation
      $ npm run all-install
      $ npm run build

    3.設定ファイルリネーム
      $ cd /opt/TV_app/EPGStation/config
      $ mv config.yml.template config.yml
      $ mv enc-enhance.js.template enc-enhance.js
      $ mv enc.js.template enc.js
      $ mv epgUpdaterLogConfig.sample.yml epgUpdaterLogConfig.yml
      $ mv operatorLogConfig.sample.yml operatorLogConfig.yml
      $ mv serviceLogConfig.sample.yml serviceLogConfig.yml

    4.config.yml ファイル編集

    4.1  mariadb設定の該当箇所を書き換える
      
      dbtype: mysql
      mysql:
        host: localhost
        port: 3306
        user: epgstation
        password: epgstation
        database: epgstation_db
        connectTimeout: 20000
        connectionLimit: 10

    4.2 ffmpegパス変更
      ffmpeg: /usr/bin/ffmpeg
      ffprobe: /usr/bin/ffprobe

    4.3 Drop Log
      Drop log を出力する場合
      isEnabledDropCheck: true を追加する

     上記以外に設定する場合は逆引きマニュアルを参照
     https://github.com/l3tnun/EPGStation/blob/master/doc/conf-manual.md

    5.起動/停止
      EPGStation ディレクトリから
    5.1 起動
      $ npm start
    5.2 停止
      $ npm stop

    6.自動起動化
      EPGStation ディレクトリから
      $ sudo pm2 startup (既に起動されている場合は必要無い)
      $ sudo pm2 start dist/index.js --name "EPGStation"
      $ sudo pm2 save

    7.アップデート方法
      $ sudo pm2 stop EPGStation
      $ cd /opt/TV_app/EPGStation
      $ git pull
      $ sudo npm update
      $ sudo npm update -D
      $ sudo npm run build
      $ sudo pm2 start epgstation

    8.EPGStationブラウザ表示
      http://localhost:8888/

## 【mirakurun & EPGStation log tempfs(ramdisk)設定】
予め /tmp をtempfs化しておく

    1. /usr/bin に以下の内容のshell script　を作成
      # vi /usr/bin/mirakurun_epgstation_cache.sh

**[ファイルの内容]**

    #! /bin/sh
    export TV_CACHE_HOME=/tmp/.cache

    # EPGStation log tempfs 設定
    if [ ! -d "${TV_CACHE_HOME}/EPGStation/logs/Operator" ]; then
      mkdir -p ${TV_CACHE_HOME}/EPGStation/logs/Operator
    fi
    if [ ! -d "${TV_CACHE_HOME}/EPGStation/logs/EPGUpdater" ]; then
      mkdir -p ${TV_CACHE_HOME}/EPGStation/logs/EPGUpdater
    fi
    if [ ! -d "${TV_CACHE_HOME}/EPGStation/logs/Service" ]; then
      mkdir -p ${TV_CACHE_HOME}/EPGStation/logs/Service
    fi
    if [ ! -L "/opt/TV_app/EPGStation/logs" ]; then
      ln -s ${TV_CACHE_HOME}/EPGStation/logs /opt/TV_app/EPGStation/logs
    fi

    # Mirakurun log tempfs 設定
    if [ ! -d "${TV_CACHE_HOME}/mirakurun/log" ]; then
      mkdir -p ${TV_CACHE_HOME}/mirakurun/log
    fi
    if [ ! -L "/usr/local/var/log" ]; then
      ln -s ${TV_CACHE_HOME}/mirakurun/log /usr/local/var/log
    fi


  ファイル保存後実行ビットを立てる  
  \# chmod +x /usr/bin/mirakurun_epgstation_cache.sh

    2. systemctl serviceファイルの作成
      # vi /etc/systemd/system/tv-cache.service

**[ファイルの内容]**

    [Unit]
    Description=mirakurun and EPGStation log output tempfs
    Before=network.target

    [Service]
    ExecStart=/usr/bin/mirakurun_epgstation_cache.sh
    Type=oneshot
    RemainAfterExit=yes

    [Install]
    WantedBy = multi-user.target


  ファイル保存後  
  \# systemctl enable tv-cache.service

    3.既存ディレクトリの削除
      # rm -r EPGStation/logs
      EPGStation log dir
      # rm -r /usr/local/var/log
      mirakurun log dir

    4. reboot
      rebootし、tempfsにディレクトリが作成されシンボリックリングされていることを確認

    5. systemd 起動順序を確認
      $ systemd-analyze plot > systemd.svg
      systemd.svg をブラウザにドラッグ＆ドロップする

## 【vlc or smplayer】
  プレイリスト(拡張子m3u8)を作成し、mirakurun httpストリーム配信URLを指定することで  
  vlc または smplayer を使用しチャンネルを切り替えてTVを視聴するようなことができる  
  playlist_GRBSCS.m3u8を参照

    1. IPアドレス
       mirakurunと同一PCで視聴: localhost
       別PCまたはスマホでネットワーク越しに視聴: mirakurunをインストールしているPCのIPアドレス

    2.プレイリストフォーマット
       チャンネルを指定
         http://mirakurunが稼働しているPCのIPアドレス:ポート番号/api/channels/GR or BS or CS/チャンネル/stream
       サービスIDを指定
         http://mirakurunが稼働しているPCのIPアドレス:ポート番号/api/channels/GR or BS or CS/チャンネル/services/sid/stream

        例：
        #EXTINF:-1,NHK総合１・東京
        http://localhost:40772/api/channels/GR/27/services/1024/stream
        #EXTINF:-1,東京MX1
        http://localhost:40772/api/channels/GR/16/services/23608/stream
        #EXTINF:-1,NHK BS1
        http://localhost:40772/api/channels/BS/BS15_0/stream
        #EXTINF:-1,BS10 スターチャンネル
        http://localhost:40772/api/channels/BS/BS15_1/stream
        #EXTINF:-1,BS10
        http://localhost:40772/api/channels/BS/BS15_2/stream
        #EXTINF:-1,放送大学231
        http://localhost:40772/api/channels/BS/BS13_2/services/231/stream
        #EXTINF:-1,Dlife
        http://localhost:40772/api/channels/CS/CS22/services/312/stream

**注意：**
チャンネル名称 (BS03_1 CS22等)はtunerアプリのチャンネル定義ファイル(bscs_ch.conf)と  
mirakurunのチャンネル定義ファイル(channels.yml)で一致させること  

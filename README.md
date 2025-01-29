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
  mirakurun側(arib-b25-stream-test)で解除する場合は必要無いがtunerアプリのコンパイル時に  
  ヘッダファイルが必要となる可能性があるので導入する

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



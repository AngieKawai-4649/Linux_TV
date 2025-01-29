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
    -l:ライブラリファイルを指定 この場合はarib25の前にlibを付け、libarib25... というライブラリファイルをライブラリキャッシュから探し出し  
       undefined symbolを解決しに行く
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

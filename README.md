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




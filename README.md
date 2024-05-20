## はじめに
現在わたしのシステムでは、snapアプリケーションが日本語入力を受け付けない、ibusとfcitxが混在しているなど、日本語入力周りの小さな問題が山積しています。
そこで今回、ibusやfcitxの問題をスッキリ整理し、日本語入力周りの用語などの確認も兼ねてドキュメントとしてまとめようと思います。

## 環境
```bash
inxi -S
System:
  Host: terms Kernel: 6.5.0-35-generic x86_64 bits: 64 Desktop: GNOME 42.9
    Distro: Ubuntu 22.04.4 LTS (Jammy Jellyfish)
```

## 確認ポイントと修正箇所
1. iBusとfcitxの競合について
   1. それぞれの特徴をおさらい
   2. 現状のシステムにおけるログの確認
   3. どちらをシステムに残すかを決定
   4. 日本語入力における前提知識を整理
2. Snap, Flatpakそれぞれの日本語入力の仕様

## 1. iBusとfcitxの競合について
### 1. それぞれの特徴をおさらい
#### iBus (Intelligent Input Bus)
**ibus (Intelligent Input Bus)**
- **マルチプラットフォームサポート**: Linux, UNIX系システムで広く使用
- **拡張性**: 様々な入力メソッドエンジンをサポートし、プラグインを通じて機能を拡張可能
- **ユーザーインターフェース**: GNOMEやその他のデスクトップ環境との統合が進んでおり、ユーザーフレンドリーな設定GUIを提供

**fcitx (Flexible Input Method Framework)**
- **高いカスタマイズ性**: ユーザーが詳細な設定を調整可能。
- **ユーザーインターフェース**: カスタマイズが可能で多様なスキンやテーマを利用可能
- **プラグイン**: クリップボードなど、標準プラグインの機能が充実
- **マクロ機能**: 使用していなかったが、特定のキーシーケンスに対してプリセットされたテキストを挿入するマクロが組めるらしい

#### 2. 現状のシステムにおけるログの確認
##### iBus
```bash
$ journalctl -b | grep ibus
 5月 10 06:14:03 **user** /usr/libexec/gdm-x-session[1707]: dbus-daemon[1707]: [session uid=128 pid=1707] Activating service name='org.gtk.vfs.Daemon' requested by ':1.18' (uid=128 pid=2025 comm="ibus-daemon --panel disable --xim " label="unconfined")
 5月 10 06:14:03 **user** /usr/libexec/gdm-x-session[1707]: dbus-daemon[1707]: [session uid=128 pid=1707] Activating service name='org.freedesktop.portal.IBus' requested by ':1.18' (uid=128 pid=2025 comm="ibus-daemon --panel disable --xim " label="unconfined")
 5月 10 06:14:16 **user** dbus-daemon[2243]: [session uid=1000 pid=2243] Activating service name='org.freedesktop.portal.IBus' requested by ':1.62' (uid=1000 pid=2776 comm="/usr/bin/ibus-daemon --panel disable --xim " label="unconfined")
 5月 10 06:22:45 **user** rsync_backup.sh[6514]: .config/ibus/bus/
 5月 10 06:22:45 **user** rsync_backup.sh[6514]: .config/ibus/bus/449a14417c3e47e59cd1625b7ebca6a6-unix-1
 5月 10 06:23:16 **user** rsync_backup.sh[6514]: ドキュメント/Input_Method_fcitx_ibus/README.md
```
   - `ibus-daemon`が複数回セッションによってアクティブ化されている。
##### fcitx
```bash
$ journalctl -b | grep fcitx
 5月 10 06:14:14 **user** /usr/libexec/gdm-x-session[2453]: dbus-update-activation-environment: setting GTK_IM_MODULE=fcitx
 5月 10 06:14:14 **user** /usr/libexec/gdm-x-session[2453]: dbus-update-activation-environment: setting XMODIFIERS=@im=fcitx
 5月 10 06:14:14 **user** /usr/libexec/gdm-x-session[2453]: dbus-update-activation-environment: setting QT_IM_MODULE=fcitx
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-fullwidth-char.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-xkbdbus.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-keyboard.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-dbus.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-xim.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-kimpanel-ui.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-unicode.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-punc.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-ipcportal.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-xkb.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-quickphrase.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-autoeng.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-lua.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-notificationitem.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (WARN-2909 fcitx-config.c:172) missing value: Name
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-imselector.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-vk.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-x11.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-spell.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-chttrans.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-clipboard.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-ipc.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-freedesktop-notify.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-pinyin-enhance.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-classic-ui.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-pinyin.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-mozc.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-remote-module.conf
 5月 10 06:14:16 **user** fcitx.desktop[2909]: (INFO-2909 addon.c:151) アドオンの設定ファイルを読み込み: fcitx-table.conf
 5月 10 06:14:17 **user** fcitx.desktop[2909]: (ERROR-2909 ime.c:432) fcitx-keyboard-tr-otk already exists
 5月 10 06:14:17 **user** fcitx.desktop[2909]: (ERROR-2909 ime.c:432) fcitx-keyboard-us already exists
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/cached_layout
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/profile
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/clipboard/history.dat
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/conf/fcitx-classic-ui.config
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/conf/fcitx-notify.config
 5月 10 06:22:39 **user** rsync_backup.sh[6514]: .config/fcitx/dbus/449a14417c3e47e59cd1625b7ebca6a6-1
 5月 10 06:23:16 **user** rsync_backup.sh[6514]: ドキュメント/Input_Method_fcitx_ibus/README.md
 5月 10 06:23:27 **user** fcitx.desktop[2909]: (WARN-2909 x11selection.c:309) Selection is too long.
 5月 10 06:29:30 **user** fcitx.desktop[2909]: (WARN-2909 x11selection.c:309) Selection is too long.
```
1. **環境変数の設定**:
   - `GTK_IM_MODULE`, `XMODIFIERS`, `QT_IM_MODULE` が `fcitx` に設定されている。
   - これはFcitxをインストールしたときの設定。
2. **アドオンの読み込み**:
   - アドオン設定ファイルの中に、`missing value: Name`という警告が出ている。
   - 特定のアドオン設定ファイルに不足している値があるらしい。このアドオン設定を見直す必要があります。
3. **重複エントリーのエラー**:
   - `fcitx-keyboard-tr-otk` および `fcitx-keyboard-us` が既に存在しているというエラー。
   - 同じキーボードレイアウトが複数回設定されているか、不正に登録されている可能性。
4. **セレクションが長すぎる警告**:
   - `Selection is too long` ：クリップボードやテキスト選択機能に関連する問題？。

### 3. どちらをシステムに残すかを決定
システム安定化も目的の一つなので、デフォルトのiBusを残し、fcitxを削除




























## 参考文献
- [WaylandとGNOME環境でfcitx5 + mozcがサードパーティアプリで機能しない問題を解決する方法](https://zenn.dev/moxak/articles/b1e7792be705ed)
- [KDE PlasmaでMozcなどIM使うなら・・ 環境変数を書いちゃダメ。自動起動に入れてもダメ。 というおはなし。](https://zenn.dev/armcore/articles/87398d56d5b64b)
- [IBus_archwiki](https://wiki.archlinux.jp/index.php/IBus#.E3.82.A4.E3.83.B3.E3.82.B9.E3.83.88.E3.83.BC.E3.83.AB)
- [Fcitx_archwiki](https://wiki.archlinux.jp/index.php/Fcitx#.E8.A8.AD.E5.AE.9A.E3.83.84.E3.83.BC.E3.83.AB)
- [Blenderで日本語入力を実装するには](https://zenn.dev/hzuika/articles/2021-07-31-blender_ime)
- [Flatpak vs. Snap. 違いと特性](https://zenn.dev/ykesamaru/articles/a9586cc52a376e)
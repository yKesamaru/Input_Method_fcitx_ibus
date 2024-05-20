## はじめに
現在わたしのシステムでは、snapアプリケーションが日本語入力を受け付けない、IBusとFcitxが混在しているなど、日本語入力周りの小さな問題が山積しています。
そこで今回、IBusやFcitxの問題をスッキリ整理し、日本語入力周りの用語などの確認も兼ねてドキュメントとしてまとめようと思います。

## 環境
```bash
inxi -S
System:
  Host: terms Kernel: 6.5.0-35-generic x86_64 bits: 64 Desktop: GNOME 42.9
    Distro: Ubuntu 22.04.4 LTS (Jammy Jellyfish)
```

## 確認ポイントと修正箇所
1. IBusとFcitxの競合について
   1. それぞれの特徴をおさらい
   2. 現状のシステムにおけるログの確認
   3. どちらをシステムに残すかを決定
   4. 日本語入力における前提知識を整理
2. Snap, Flatpakそれぞれの日本語入力の仕様

## 前提知識
1. ユーザーセッション開始時に読み込まれる日本語入力関連の設定ファイル
2. Snap, Flatpakにおける日本語入力関連
3. その他
についてそれぞれ整理します
### 1. ユーザーセッション開始時に読み込まれる日本語入力関連の設定ファイル
#### `.bashrc`
- エイリアスの設定
- 端末の表示カスタマイズ
- 環境変数の設定
- 仮想端末を開く際には必ず読み込まれる
- 仮想端末から呼び出されたGUIアプリケーションであればここに書かれた環境変数を引き継ぐが、**デスクトップ環境から直接起動されたアプリケーションには影響しないことに注意。**
#### `.bash_profile`
- Bashシェルがログインシェルとして起動する際、読み込まれる設定ファイル。
- ログインセッションの初期化（パスの設定、環境変数のエクスポートなど）
- `.bashrc`を呼び出す
- ログインセッション、非ログインセッションの両者で共通の設定を行う
#### `.xprofile`
- Xセッションの起動時に読み込まれる設定ファイル
- guiアプリケーションの環境変数などを設定する
- `GTK_IM_MODULE`や`QT_IM_MODULE`などの入力メソッドの環境変数を設定するのに使われることがある

#### `~/.config/environment.d/***.conf`
- 新しい形式の環境設定ファイル
- `systemd`か管理するユーザーセッションの一部としてロードされ、環境変数などを設定することが可能。
- `systemd`はこのファイル形式を使用して、ログイン時の環境変数を設定する
- この方式は`systemd`を使用するような新し目のバージョンで普及しつつある（Ubuntu 22.04では採用していない）
- 特定のシェルやログイン方法に依存しない汎用的な方法
  `.bashrc`, `.bash_profile`, `.xprofile`はBashだし、`.xprofile`はXセッションだし、という感じ。

### 2. Snap, Flatpakにおける日本語入力関連
#### Snapにおける日本語入力の背景
- Snapアプリケーションは基本的にホストシステムの環境変数や設定を直接引き継ぐことができない。ゆえに、`GTK_IM_MODULE`, `QT_IM_MODULE`, `XMODIFIERS`などの環境変数や設定を反映させることができない。そのため、これらの設定は明示的にSnap内で設定する必要がある。
- 開発者が日本語入力をサポートするために必要なライブラリをパッケージに含めている必要がある。
- ユーザーとしては、Snapアプリケーションが期待する入力メソッドと整合性を持つようにシステムを調整する必要が求められる。
##### Snapアプリケーションの日本語入力の設定
###### 前提
開発者がSnapcraft.yamlファイルに必要な環境変数（`GTK_IM_MODULE`, `QT_IM_MODULE`, `XMODIFIERS`など）を定義していること
###### プラグの利用
プラグとは、Snapアプリケーションにおいてシステムリソースへのアクセスを制御するための接続点。
日本語入力を行う場合、`desktop`および`desktop-legacy`プラグをアプリケーションに追加し、これを`Snapcraft.yamlファイル`で指定している必要がある。
逆に言えば、開発者がこれらの設定をSnapアプリケーションに適用していれば、ユーザーがインストール後に特別な設定を行わずとも日本語入力を使用できる。
- プラグ(Plug)
  - Snapアプリケーションがシステムリソースなどに接続するための、「要求側」のエンドポイントのこと。
  - Snapアプリケーションは、各々プラグを通じてシステムの機能やデータへのアクセスを要求する
  - 例：ネットワークアクセス、オーディオ出力、プリンターへのアクセスなど。
- スロット(Slot)
  - システムや他のSnapアプリケーションが提供する「提供側」のエンドポイント。
  - システム自体やいろいろなSnapアプリケーションがこの機能を通じて、異なるアプリケーション間で機能を共有することがある
通常多くのSnapアプリケーションはインストール時に基本的なプラグとスロットの接続を自動的に行う。
これはSnapcraft.yamlに定義されている`auto-connect`プロパティによるもの。
ユーザーによる手動接続には`snap connect`コマンドを利用する。
Snap Store, Software CenterなどでSnapアプリケーション個々の権限管理を行うこともできる。
また、一部のSnapアプリケーションでは特定のリソースへのアクセスを要求する際、動的にダイアログを表示するものもある。これはSnapの標準的な挙動ではなく、開発者が独自に実装する場合のみ。

###### Snapアプリケーションの接続情報の確認方法
```bash
snap connections [snap-name]
```
指定したSnapアプリケーションのすべてのプラグとスロットの接続状況を表示する
Snap Storeで記述されている場合もある
```bash
$ snap connections kdenlive
Interface                                 Plug                                  Slot                                                          Notes
audio-playback                            kdenlive:audio-playback               :audio-playback                                               -
audio-record                              kdenlive:audio-record                 :audio-record                                                 -
content[icon-themes]                      kdenlive:icon-themes                  gtk-common-themes:icon-themes                                 -
content[kf5-5-111-qt-5-15-11-core22-all]  kdenlive:kf5-5-111-qt-5-15-11-core22  kf5-5-111-qt-5-15-11-core22:kf5-5-111-qt-5-15-11-core22-slot  -
content[sound-themes]                     kdenlive:sound-themes                 gtk-common-themes:sound-themes                                -
dbus                                      -                                     kdenlive:session-dbus-interface                               -
desktop                                   kdenlive:desktop                      :desktop                                                      -
desktop-legacy                            kdenlive:desktop-legacy               :desktop-legacy                                               -
home                                      kdenlive:home                         :home                                                         -
network                                   kdenlive:network                      :network                                                      -
network-bind                              kdenlive:network-bind                 :network-bind                                                 -
opengl                                    kdenlive:opengl                       :opengl                                                       -
removable-media                           kdenlive:removable-media              :removable-media                                              -
system-observe                            kdenlive:system-observe               :system-observe                                               -
wayland                                   kdenlive:wayland                      :wayland                                                      -
x11                                       kdenlive:x11                          :x11                                                          -
$ 
```

#### Flatpakにおける日本語入力の背景
- ユーザーはFlatpakアプリケーションのポータルを通じて、システムの入力メソッドにアクセスする許可設定を行う必要がある。
- Flatpakアプリケーションは基本的に`freedesktop.org`ランタイムや`KDE`, `GNOME`ランタイムを使用している。これらランタイムは一般的に入力メソッドライブラリを含んでいる
##### Flatpakのポータルとは
Flatpakアプリケーションがサンドボックス外のシステムリソースに安全にアクセスするためのしくみ。
ポータルを使用することによって、Flatpakアプリケーションはセキュリティを担保したまま以下の機能を得ることができる
- ファイルアクセス
- プリンターなどのデバイスへのアクセス
- 入力メソッドフレームワークへのアクセス

###### Flatpakにおけるアプリケーションの権限の確認と管理
```bash
flatpak override --user --device=all [application-id]
```

### 3. その他
#### `xdg-desktop-portal`について。
##### `xdg`
XDGは「X Desktop Group」の略で、後に「freedesktop.org」と改名.
`xdg-desktop-portal` は、サンドボックス化されたアプリケーション（例えばFlatpakやSnapパッケージ）が、セキュリティ制約を保ちつつホストシステムのリソースやサービス（ファイルアクセス、カメラ、位置情報サービスなど）にアクセスできるようにするためのバックエンドサービス。
これらはデスクトップ環境に依存しない実装となっているが、たとえば`xdg-desktop-portal-gnome`や`xdg-desktop-portal-kde`などは、それぞれのデスクトップ環境にあわせたUIも同時に提供する。
たとえば自分がGNOME環境を使っていてKDEアプリケーションを使う場合、xdg-desktop-portal-gnomeにより、ダイアログなどのUIをGNOMEに馴染んだ表示にしてくれる。
ただし、アプリケーションがxdg-desktop-portalのAPIを通じてシステムリソースへアクセスするように作られている場合に限り、すべてのKDEアプリケーションがGNOME環境で完全に統一されたUIを提供するわけではない。
##### GNOME環境で、`xdg-desktop-portal-kde`は必要か？
必要ない。
通常、xdg-desktop-portal-gnome や xdg-desktop-portal-kde は、それぞれGNOMEとKDEのデスクトップ環境向けに特化した実装を提供する

## 1. IBusとFcitxの競合について
### 1. それぞれの特徴をおさらい
#### IBus (Intelligent Input Bus)
**IBus (Intelligent Input Bus)**
- **マルチプラットフォームサポート**: Linux, UNIX系システムで広く使用
- **拡張性**: 様々な入力メソッドエンジンをサポートし、プラグインを通じて機能を拡張可能
- **ユーザーインターフェース**: GNOMEやその他のデスクトップ環境との統合が進んでおり、ユーザーフレンドリーな設定GUIを提供

**Fcitx (Flexible Input Method Framework)**
- **高いカスタマイズ性**: ユーザーが詳細な設定を調整可能。
- **ユーザーインターフェース**: カスタマイズが可能で多様なスキンやテーマを利用可能
- **プラグイン**: クリップボードなど、標準プラグインの機能が充実
- **マクロ機能**: 使用していなかったが、特定のキーシーケンスに対してプリセットされたテキストを挿入するマクロが組めるらしい

#### 2. 現状のシステムにおけるログなどの確認
`systemd`のステータスを確認する
```bash
systemctl --user status ibus.service
systemctl --user status fcitx.service
```
デーモン起動してない場合は以下。
```bash
ps aux | grep ibus
ps aux | grep fcitx
```
実際のシステムの様子
```bash
$ systemctl --user status ibus.service
Unit ibus.service could not be found.
$ systemctl --user status fcitx
Unit fcitx.service could not be found.
$ ps aux | grep ibus
**user**       2813  0.0  0.0   2892  1536 ?        Ss   07:08   0:00 sh -c /usr/bin/ibus-daemon --panel disable $([ "$XDG_SESSION_TYPE" = "x11" ] && echo "--xim")
**user**       2816  0.0  0.0 316488 11900 ?        Sl   07:08   0:00 /usr/bin/ibus-daemon --panel disable --xim
**user**       2879  0.0  0.0 164828  7424 ?        Sl   07:08   0:00 /usr/libexec/ibus-memconf
**user**       2884  0.0  0.0 274252 29996 ?        Sl   07:08   0:01 /usr/libexec/ibus-extension-gtk3
**user**       2893  0.0  0.0 195856 24752 ?        Sl   07:08   0:00 /usr/libexec/ibus-x11 --kill-daemon
**user**       2906  0.0  0.0 238844  7680 ?        Sl   07:08   0:01 /usr/libexec/ibus-portal
**user**       3295  0.0  0.0 164828  7424 ?        Sl   07:08   0:00 /usr/libexec/ibus-engine-simple
**user**       5837  0.0  0.0 313212 12160 ?        Sl   07:08   0:00 /usr/lib/ibus-mozc/ibus-engine-mozc --ibus
**user**      20548  0.0  0.0  10272  2432 pts/0    S+   14:10   0:00 grep --color=auto ibus
$ ps aux | grep fcitx
**user**       2952  0.0  0.1 437000 54968 ?        S    07:08   0:19 fcitx
**user**       3031  0.0  0.0   8832  3468 ?        Ss   07:08   0:03 /usr/bin/dbus-daemon --syslog --fork --print-pid 5 --print-address 7 --config-file /usr/share/fcitx/dbus/daemon.conf
**user**       3067  0.0  0.0   6580  1284 ?        SN   07:08   0:00 /usr/bin/fcitx-dbus-watcher unix:abstract=/tmp/dbus-4Ni6lT2a6h,guid=51d3f5057e0dc44275ba286a664a785a 3031
**user**      20571  0.0  0.0  10272  2432 pts/0    S+   14:12   0:00 grep --color=auto fcitx

```

- **IBus デーモンが実行中**：`ibus-daemon`が `--panel disable --xim` オプションを付けて実行中。これは、X Input Method (XIM) サポートを有効にして、パネル (GUIの一部) を無効にする設定。
- **複数のIBus関連プロセスがアクティブ**：`ibus-memconf`, `ibus-extension-gtk3`, `ibus-x11`, `ibus-portal`, `ibus-engine-simple` および `ibus-engine-mozc` など、複数のIBus関連プロセスがシステム上で稼働

- **Fcitx メインプロセスが実行中**：`Fcitx` プロセス→Fcitxのメイン入力メソッドフレームワーク
- **FcitxのD-Bus設定**：`/usr/bin/dbus-daemon` がFcitx専用のD-Busデーモンとして実行されてる。この設定は、Fcitxが独自のD-Bus接続を通じて他のアプリケーションと通信できるようにするためのもの。

`im-config -a`
![](assets/2024-05-20-15-02-56.png)
> Current configuration for the input method:
>  * Default mode defined in /etc/default/im-config: 'auto'
>  * Active configuration: 'missing' (normally missing)
>  * Normal automatic choice: 'ibus' (normally ibus or fcitx5)
>  * Override rule: 'zh_CN,fcitx5:zh_TW,fcitx5:zh_HK,fcitx5:zh_SG,fcitx5'
>  * Current override choice: '' (Locale='ja_JP')
>  * Current automatic choice: 'ibus'
>  * Number of valid choices: 11 (normally 1)
>  * Desktop environment: 'ubuntu:GNOME'
> The configuration set by im-config is activated by re-starting the system.
> default/auto/cjkv/missingが有効な場合は、自動設定を有効にするために明示的に選択を行う必要はありません。
>   使用可能なインプットメソッド: IBus Fcitx Fcitx5 uim hime gcin maliit scim thai xim kinput2 xsunpinyin
> これらすべてが必要でない場合は、必ず一つだけのインプットメソッドツールをインストールするようにしましょう。

```bash
$ im-config -l
 ibus fcitx xim
```
`synaptic`で確認したところ、`xim`はインストールされていないんだが…🤔
```bash
im-config -o
/usr/bin/im-config: 88: .: cannot open /usr/share/im-config/data/??_.conf: No such file
```
現状、`.xinputrc`ファイルは存在しない。
`im-config`コマンドを用いてインプットメソッドを指定した場合のみ、`.xinputrc`ファイルが作成され、そこに記述される。

##### IBus
```bash
$ journalctl -b | grep ibus
 5月 10 06:14:03 **user** /usr/libexec/gdm-x-session[1707]: dbus-daemon[1707]: [session uid=128 pid=1707] Activating service name='org.gtk.vfs.Daemon' requested by ':1.18' (uid=128 pid=2025 comm="ibus-daemon --panel disable --xim " label="unconfined")
 5月 10 06:14:03 **user** /usr/libexec/gdm-x-session[1707]: dbus-daemon[1707]: [session uid=128 pid=1707] Activating service name='org.freedesktop.portal.ibus' requested by ':1.18' (uid=128 pid=2025 comm="ibus-daemon --panel disable --xim " label="unconfined")
 5月 10 06:14:16 **user** dbus-daemon[2243]: [session uid=1000 pid=2243] Activating service name='org.freedesktop.portal.ibus' requested by ':1.62' (uid=1000 pid=2776 comm="/usr/bin/ibus-daemon --panel disable --xim " label="unconfined")
 5月 10 06:22:45 **user** rsync_backup.sh[6514]: .config/ibus/bus/
 5月 10 06:22:45 **user** rsync_backup.sh[6514]: .config/ibus/bus/449a14417c3e47e59cd1625b7ebca6a6-unix-1
 5月 10 06:23:16 **user** rsync_backup.sh[6514]: ドキュメント/Input_Method_fcitx_ibus/README.md
```
   - `ibus-daemon`が複数回セッションによってアクティブ化されている。
##### Fcitx
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
   - `GTK_IM_MODULE`, `XMODIFIERS`, `QT_IM_MODULE` が `Fcitx` に設定されている。
   - これはFcitxをインストールしたときの設定。
2. **アドオンの読み込み**:
   - アドオン設定ファイルの中に、`missing value: Name`という警告が出ている。
   - 特定のアドオン設定ファイルに不足している値があるらしい。このアドオン設定を見直す必要があります。
3. **重複エントリーのエラー**:
   - `fcitx-keyboard-tr-otk` および `fcitx-keyboard-us` が既に存在しているというエラー。
   - 同じキーボードレイアウトが複数回設定されているか、不正に登録されている可能性。
4. **セレクションが長すぎる警告**:
   - `Selection is too long` ：クリップボードやテキスト選択機能に関連する問題？。
##### システムトレイの様子
![](assets/2024-05-20-14-22-36.png)
①はIBus、②はFcitxが表示されている。
### 3. どちらをシステムに残すかを決定
結論：システム安定化も目的の一つなので、デフォルトのIBusを残し、Fcitxを削除
#### Fcitxの削除
```bash
# fcitxを含むパッケージを確認
apt list --installed | grep fcitx
fcitx-bin/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-config-common/jammy,jammy,now 0.4.10-3 all [インストール済み、自動]
fcitx-config-gtk/jammy,now 0.4.10-3 amd64 [インストール済み]
fcitx-data/jammy,jammy,now 1:4.2.9.8-5 all [インストール済み、自動]
fcitx-frontend-all/jammy,jammy,now 1:4.2.9.8-5 all [インストール済み、自動]
fcitx-frontend-gtk2/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-frontend-gtk3/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-frontend-qt5/jammy,now 1.2.7-1.2build1 amd64 [インストール済み、自動]
fcitx-module-dbus/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-module-kimpanel/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-module-lua/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-module-quickphrase-editor5/jammy,now 1.2.7-1.2build1 amd64 [インストール済み、自動]
fcitx-module-x11/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-modules/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-mozc-data/jammy,jammy,now 2.26.4220.100+dfsg-5.2 all [インストール済み、自動]
fcitx-mozc/jammy,now 2.26.4220.100+dfsg-5.2 amd64 [インストール済み]
fcitx-pinyin/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-table-emoji/jammy,jammy,now 0.2.4-2 all [インストール済み]
fcitx-table/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx-tools/jammy,now 1:4.2.9.8-5 amd64 [インストール済み]
fcitx-ui-classic/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
fcitx/jammy,jammy,now 1:4.2.9.8-5 all [インストール済み]
libfcitx-config4/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
libfcitx-core0/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
libfcitx-gclient1/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
libfcitx-qt5-1/jammy,now 1.2.7-1.2build1 amd64 [インストール済み、自動]
libfcitx-qt5-data/jammy,jammy,now 1.2.7-1.2build1 all [インストール済み、自動]
libfcitx-utils0/jammy,now 1:4.2.9.8-5 amd64 [インストール済み、自動]
```
- 事前にfcitx-mozcのユーザー辞書をエクスポートしておく
- `.bashrc`, `.xprofile`を書き換えておく
```diff
- export GTK_IM_MODULE=fcitx
- export QT_IM_MODULE=fcitx
- export XMODIFIERS=@im=fcitx
+ export GTK_IM_MODULE=ibus
+ export QT_IM_MODULE=ibus
+ export XMODIFIERS=@im=ibus
```
```bash
# Fcitxに関連するパッケージをremove
sudo apt-get remove fcitx*

以下のパッケージは「削除」されます:
  fcitx fcitx-bin fcitx-config-common fcitx-config-gtk fcitx-data fcitx-frontend-all fcitx-frontend-gtk2 fcitx-frontend-gtk3 fcitx-frontend-qt5
  fcitx-module-dbus fcitx-module-kimpanel fcitx-module-lua fcitx-module-quickphrase-editor5 fcitx-module-x11 fcitx-modules fcitx-mozc
  fcitx-mozc-data fcitx-pinyin fcitx-table fcitx-table-emoji fcitx-tools fcitx-ui-classic
アップグレード: 0 個、新規インストール: 0 個、削除: 22 個、保留: 3 個。
この操作後に 17.7 MB のディスク容量が解放されます。
続行しますか? [Y/n] 
## 削除処理出力

im-config -l
 ibus xim
# IBusをinput methodに指定
im-config -n ibus

# 再起動
```
![](assets/2024-05-20-15-46-48.png)
Fcitxのトレイアイコンが消えた。

## IBusの追加設定
`ibus-setup`にてIBus全般のセットアップを行う。
![](assets/2024-05-20-16-01-53.png)
### IBusの足りない機能
1. 入力メソッドの切り替えショートカットが1種類しかな
   1. 異なるキーにen⇔jaが切り替えられない
2. クリップボード機能が使えない
   1. Fcitxでは拡張機能としてクリップボード機能が存在した

1に関しては、GNOMEのキーボード・ショートカット割当機能にて実装が可能です。
![](assets/2024-05-20-16-24-34.png)
![](assets/2024-05-20-16-24-49.png)

- 直接入力に切り替えるスクリプト
```bash:直接入力に切り替えるスクリプトswitch_to_english.sh
#!/bin/bash
ibus engine xkb:us::eng
```
```bash:日本語へ切り替えるスクリプトswitch_to_japanese.sh
#!/bin/bash
ibus engine mozc-jp
```
![](assets/2024-05-20-16-34-02.png)
これにより、無変換キーで直接入力、変換キーで日本語入力を実現できます。
しかしながらシステムトレイの表示が切り替わりません。
![](assets/2024-05-20-16-40-33.png)
これについては解決法が見つかりませんでした。（あまり困りませんが）
また、入力モードは以下のようにしておく必要があります。
![](assets/2024-05-20-16-48-27.png)



























## 参考文献
- [WaylandとGNOME環境でfcitx5 + mozcがサードパーティアプリで機能しない問題を解決する方法](https://zenn.dev/moxak/articles/b1e7792be705ed)
- [KDE PlasmaでMozcなどIM使うなら・・ 環境変数を書いちゃダメ。自動起動に入れてもダメ。 というおはなし。](https://zenn.dev/armcore/articles/87398d56d5b64b)
- [IBus_archwiki](https://wiki.archlinux.jp/index.php/ibus#.E3.82.A4.E3.83.B3.E3.82.B9.E3.83.88.E3.83.BC.E3.83.AB)
- [Fcitx_archwiki](https://wiki.archlinux.jp/index.php/Fcitx#.E8.A8.AD.E5.AE.9A.E3.83.84.E3.83.BC.E3.83.AB)
- [IBus: wikipedia](https://ja.wikipedia.org/wiki/ibus)
- [Blenderで日本語入力を実装するには](https://zenn.dev/hzuika/articles/2021-07-31-blender_ime)
- [Flatpak vs. Snap. 違いと特性](https://zenn.dev/ykesamaru/articles/a9586cc52a376e)
---
title: "Raspberry Piでpulseaudioをsystem modeにする"
date: 2022-05-06T21:37:06+09:00
categories:
- Raspberry Pi
tags:
- 環境構築
keywords:
- pulseaudio
---
# 前置き
## 概要
Raspberry Pi Zeroを軽量なオーディオモジュールとするべく、軽量なディストリビューション"Diet Pi"に入れ替えた。
pulseaudioをユーザーログイン如何を問わず起動させておきたいため、インストール時のデフォルトであるユーザーモードを止め、システムモードで起動するように変更した。

## pulseaudioのモード
- ユーザーモード
   - aptでインストールするとデフォルトでこのモードになる。pulseaudioの使い方としては推奨されている模様。root以外のユーザーログイン時に、systemdにより自動起動される。
- システムモード
   - 全ユーザー共通のシステムとして起動するモード。起動すると本当にわかっているかとログで問われる。推奨されていない模様だが、ユーザーとしてログインせずに使う場合はこちら。
- デーモンモード(厳密にはdaemonizeオプションで切り替えられる)
   - コマンドラインから起動したときに、バックグラウンド実行するモード。systemdで使用する場合は、systemdにpulseaudioのプロセスを掴ませるため、OFFにしなければならない。
   - よくcrontabなどからシェルスクリプトなどで起動する事例が見られる。

## なぜシステムモードにしたいのか
- 電源を入れただけで立ち上がるオーディオミキサー的なモノが作りたい。
- ユーザーとしてcronなどで起動する方法もあるが、systemdで管理したほうが、再起動操作などの扱いが楽になると考えた。
- そもそもraspiにユーザーとしてログインして使うことを考えていない(ユーザーはrootのみ)

# システムモードで動作するまで
## aptでインストール
```bash
# apt update && apt install -y pulseaudio
```

## システムモードで起動するための設定
- /etc/pulse/system.paに対し、下記2点を変更
   - 下記の部分に、`auth-anonymous=1`を追記
```config
load-module module-native-protocol-unix auth-anonymous=1
```
   - 末尾に、bluetooth関連モジュール読込みを追記
```config
load-module module-bluetooth-policy
load-module module-bluetooth-discover
```

## systemdのユーザー設定無効化
- /etc/systemd/user配下にダミーのpulseaudio.serviceを配置
```bash
# ln -s /dev/null /etc/systemd/user/pulseaudio.service
# ln -s /dev/null /etc/systemd/user/pulseaudio.socket
```

## systemdのシステムサービス作成・有効化
- /lib/systemd/system/pulseaudio.serviceとして下記ファイルを作成
```systemd
[Unit]
Description=Sound Service
[Service]
ExecStart=/usr/bin/pulseaudio --system --log-target=journal --disallow-exit --disallow-module-loading
LockPersonality=yes
MemoryDenyWriteExecute=yes
NoNewPrivileges=yes
Restart=on-failure
RestrictNamespaces=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
Type=notify
UMask=0077

[Install]
WantedBy=multi-user.target
```
- 尚、オプションの意味はこれから勉強

## 今後
複数bluetooth入力デバイス及びUSB-DACの認識と、sink-sourceの接続を任意に切り替えられるインターフェースを作成予定(前に一度作った記憶が...)


## ペアリングできても接続できない場合
```
root@DietPi:~# systemctl status bluetooth
● bluetooth.service - Bluetooth service
     Loaded: loaded (/lib/systemd/system/bluetooth.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2022-05-19 00:48:59 JST; 1h 28min ago
       Docs: man:bluetoothd(8)
   Main PID: 385 (bluetoothd)
     Status: "Running"
      Tasks: 1 (limit: 991)
        CPU: 286ms
     CGroup: /system.slice/bluetooth.service
             └─385 /usr/libexec/bluetooth/bluetoothd --plugin=a2dp --compat --noplugin=sap

May 19 00:48:59 DietPi bluetoothd[385]: Ignoring (cli) deviceinfo
May 19 00:48:59 DietPi bluetoothd[385]: Ignoring (cli) midi
May 19 00:48:59 DietPi bluetoothd[385]: Ignoring (cli) battery
May 19 00:48:59 DietPi bluetoothd[385]: Ignoring (cli) sixaxis
May 19 00:48:59 DietPi bluetoothd[385]: Bluetooth management interface 1.21 initialized
May 19 00:49:16 DietPi bluetoothd[385]: Endpoint registered: sender=:1.2 path=/MediaEndpoint/A2DPSink/sbc
May 19 00:49:16 DietPi bluetoothd[385]: Endpoint registered: sender=:1.2 path=/MediaEndpoint/A2DPSource/sbc
May 19 00:49:16 DietPi bluetoothd[385]: Endpoint registered: sender=:1.2 path=/MediaEndpoint/A2DPSink/sbc
May 19 00:49:16 DietPi bluetoothd[385]: Endpoint registered: sender=:1.2 path=/MediaEndpoint/A2DPSource/sbc
May 19 00:49:16 DietPi bluetoothd[385]: Failed to set privacy: Rejected (0x0b)
```

https://github.com/RPi-Distro/pi-bluetooth/issues/8

- 下記コマンドを実行すればよいようだ
   - bluetoothdの立ち上がり前に、sys-subsystem-なんとかを立ち上げて、hciをdownにしておく必要があるらしい
   - bluetoothdがすでに立ち上がっている場合は、一度止めてから上記を実行
   - 起動時のcronとして組み込んでおくよう進められているが、他との兼ね合いでどの順で起動させるかは要検討
```
# systemctl start sys-subsystem-bluetooth-devices-hci0.device; hciconfig hci0 down; systemctl start bluetooth
```

## 補足
- A2DPで双方向やり取りできると思ったら、RPi→Windows方向はただのマイクとして認識してくれないようだ
   - Linuxのsinkデバイスとwindowsのマイクデバイスは別の位置づけということか
   - 一応win10 2004から使えるようになっているが、専用のstoreアプリ(Bluetoothオーディオレシーバー)経由での「再生」という扱いのようだ
      - https://ja.macspots.com/microsoft-announced-windows-10-may-2019-update

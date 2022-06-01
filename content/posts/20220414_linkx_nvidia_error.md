---
title: "LinuxアップデートでGPUが使えなくなった→linux-headers-amd64インストール"
date: 2022-04-14T12:43:17+09:00
categories:
- Linux
tags:
- トラブル
keywords:
- ドライバ
---
# 問題点
- Linuxカーネルの更新(5.10.0-12 -> 5.10.0-13)後、以下のエラーが出てGPUが使えなくなった
```
[FAILED] Failed to start NVIDIA Persistence Daemon.
```

# 結論
- linux-headers-amd64がインストールされていない為、カーネルドライバを更新できなかった
- linux-headers-amd64をインストールしたうえでcuda(or nvidia driver)をインストールしなおせば回復
```
# apt install linux-headers-amd64
# apt install cuda
```

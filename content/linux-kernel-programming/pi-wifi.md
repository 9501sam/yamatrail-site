+++
date = '2026-02-16T14:10:41+08:00'
draft = false
title = '如何在文字界面底下讓樹莓派連上 Wi-Fi'
weight = 5
+++
## 啟用 `wlan0`
```sh
sudo rfkill unblock wifi
# or unlock all
sudo rfkill unblock all
```

```sh
sudo ip link set wlan0 up
```

```sh
sudo iwlist wlan0 scan
```

## 使用 `nmcli`
```sh
user@raspberrypi:~ $ sudo nmcli dev wifi rescan
user@raspberrypi:~ $ sudo nmcli device wifi connect "xxx" password "yyyyyyyyyy"
```

剛開始 scan 的時候，可能會看到
```sh
user@raspberrypi:~ $ sudo nmcli dev wifi rescan
Error: Scanning not allowed while unavailable.
```
這是因為還沒有設定國家代碼，這時要先
```sh
sudo raspi-config
```
5 Localisation Options -> L4 WLAN Country -> 選擇 TW Taiwan

## 把 ip 固定住
```sh
# 同時設定 IP、Gateway、DNS 與 Method
sudo nmcli connection modify "12_516" \
    ipv4.addresses 192.168.100.102/24 \
    ipv4.gateway 192.168.100.1 \
    ipv4.dns "8.8.8.8,8.8.4.4" \
    ipv4.method manual

# 立即套用並重啟連線
sudo nmcli connection up "12_516"
```

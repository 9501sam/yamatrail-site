+++
date = '2026-05-01T20:10:13+08:00'
draft = false
title = 'Quick Start'
+++

> 這篇文章的目的在於讓我自己快速的進入到開發環境中。

## 進入到 vm 中
### 結論：使用以下指令
```sh
ub # `ub` for ubuntu vm
```

### 環境建立過程
這裡結合
1. power on the vm
```sh
VBoxManage startvm "ubuntu" --type headless
```

2. ssh to the vm 
```sh
# ubuntu on virtualbox
Host ubuntu
    HostName 192.168.56.103
    User user
    PasswordAuthentication no
    RequestTTY force
    ForwardX11 yes
```
```sh
ssh ubuntu
```

最後在 `~/.config/fish/config.fish` 做 alias 的設定
```sh
alias ub='VBoxManage startvm "ubuntu" --type headless; ssh ubuntu'
```

## Logs
清除 log
```sh
sudo dmesg -C
```

+++
date = '2025-09-21T12:33:40+08:00'
draft = false
title = '[這個網站的誕生: 01] 建立一個免費的 vm'
series = "這個網站的誕生"
weight = 1
+++
### 如何建立一個免費的 vm

根據 google cloud 的說明

https://cloud.google.com/free?hl=en

滿足以下條件的 vm 是可以不用收錢的

- 1 non-preemptible `e2-micro` VM instance per month in one of the following US regions:
    - Oregon: `us-west1`
    - Iowa: `us-central1`
    - South Carolina: `us-east1`
- 30 GB-months standard persistent disk
- 1 GB of outbound data transfer from North America to all region destinations (excluding China and Australia) per month

因此等等在建立 vm 時，按照上面的條件建立，並且要注意的是建立兩個以上的 vm 還是要收錢的，免費的只有第一個

### 建立 vm 步驟

首先你會先需要建立一個 google cloud 帳號

在 compute engin 中，點選上方的 “Create Instance”

接著根據先前的免費條件

1. “Machine Configuration”:
    1. 選擇 e2-micro
    2. Region 選擇 `us-central1` (或其他兩個免費區域)
2. “OS and storage”: 修改 boot disk type 為 standard persistent disk
3. Networking: 句選 Allow HTTP traffic 與 Allow HTTPS traffic，因為這是要作為網站的主機

目前可能還是會看到試算金額會花掉一些錢，這是正常的，如果有符合免費的條件，最後他的金額會被扣除

之後就按下 create, vm 就會建立起來了

### 如何連接到 vm

連接到 vm 有兩個主要的方是，

第一個比較輕鬆，只需要按下 **VM instances** 列表中的 ssh, 就會跳出一個 **SSH-in-browser** 的界面了，用起來也跟一般的 terminal 差不多

另一個方式是使用 gcloud CLI 這東西可以讓我們使用自己電腦上的 termianl 連線到 vm 中

### 安裝 gcloud CLI

https://cloud.google.com/sdk/docs/install

我使用的是 Debian/Ubuntu 的版本

```bash
sudo apt-get update
```

```bash
sudo apt-get install apt-transport-https ca-certificates gnupg curl
```

1. Import the Google Cloud public key. 
    
    ```bash
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
    ```
    
2. Add the gcloud CLI distribution URI as a package source. 
    
    ```bash
    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
    ```
    
3. Update and install the gcloud CLI: 
    
    ```bash
    sudo apt-get update && sudo apt-get install google-cloud-cli
    ```
    

### 使用 gcloud 連線到 vm

```bash
gcloud init
```

```bash
gcloud compute ssh [INSTANCE_NAME] --project [PROJECT_ID] --zone [ZONE]
```

```bash
gcloud compute ssh instance-20250920-081618 --zone us-central1-f --project sacred-temple-472606-s9  
```

因為 gcp 上的金鑰是由 gcloud 控制的，所以一般使用 ssh 指令的時候會發現連不上去

這時候只要稍微修改一下 `~/.ssh/config` 就好

```bash
# Google Cloud VM for yamatrail
Host <hostname>
  HostName <vm IP>
  User <user name>
  IdentityFile ~/.ssh/google_compute_engine
  IdentitiesOnly yes
```

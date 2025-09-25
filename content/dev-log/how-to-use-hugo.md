+++
date = '2025-09-23T19:35:47+08:00'
draft = false
title = '[這個網站的誕生: 03] 使用 Hugo 產生網站'
series = "這個網站的誕生"
weight = 3
+++
當我們設定好 vm 之後，先回到我們工作用的電腦，以我自己來說，這會是我的筆電。
我們的網站編輯與開發都會在這台筆電上，下一篇文章才會說明如何把網站內容部署到 vm 中，  
  
總之，這篇文章的內容都在個人的筆電（或 PC）上操作。
### 安裝套件

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx git -y
```

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.146.0/hugo_extended_0.146.0_linux-amd64.deb

sudo dpkg -i hugo_extended_0.146.0_linux-amd64.deb

hugo version
```

在筆電上先使用 hugo 建立一個 repo 出來，並且使用 git 做版控，然後推到自己的 github 上

```bash
hugo new site yamatrail-site
cd yamatrail-site
```

```bash
git init
git add .
git commit -m "Init commit"
```

使用這個指令可以新增一篇文章:
```bash
hugo new content posts/hello-world.md
```

```bash
vim content/posts/hello-world.md
```

### 設定 theme
這裡可先在 [Hugo Themes](https://themes.gohugo.io/) 看看，再決定自己要用的主題，我自己用的是 [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/)  
  
例如可以用類似下面這幾個命令新增可使用的 theme:
```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

```bash
git submodule add https://github.com/jimfrenette/hugo-starter.git themes/starter
```

```bash
git submodule add https://github.com/lukeorth/poison.git themes/poison
```

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```
使用下列的命令之後，才可以把 submodule 中的 source code 放到我們的專案中
```bash
git submodule update --init --recursive
```

`hugo.toml` 是網站的設定檔，每一個 theme 的設定檔都有一些些的不同，需要讀每個 theme 的 documents
```bash
vim hugo.toml
```

```bash
baseURL = 'https://yamatrail.com/'
languageCode = 'zh-tw'
title = 'My New Hugo Site'
theme = 'PaperMod'
```

在我的筆電上，可以使用下列指定檢查網站的內容
```bash
# 預設的方式是不會顯示草稿
hugo server
```
```bash
# -D 或 --buildDrafts 參數是告訴 Hugo 把草稿也顯示出來
hugo server -D
```
之後可經由 [http://localhost:1313/](http://localhost:1313/) 觀看網站的輸出結果，此外我們在編輯文章或是設定檔的同時，`hugo server` 會即時的更新網站內容，這是我個人覺得非常方便的一點。


+++
date = '2025-09-23T19:35:47+08:00'
draft = true
title = '[這個網站的誕生: 03] 使用 Hugo 產生網站'
series = "這個網站的誕生"
+++
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
這裡可先在 [Hugo Themes](https://themes.gohugo.io/) 看看，再決定自己要用的主題
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

```bash
git submodule update --init --recursive
```

```bash
vim hugo.toml
```

```bash
baseURL = 'https://yamatrail.com/'
languageCode = 'zh-tw'
title = 'My New Hugo Site'
theme = 'PaperMod'
```

```bash
# -D 或 --buildDrafts 參數是告訴 Hugo 把草稿也顯示出來
hugo server -D
```

## 建立網站部署流程

### 總流程概覽
我這裡的部署流程為：大多數的檔案編輯都在我的筆電上，檔案修改完之後推到 github 上，利用 github action 自動部署到 vm 當中。

1. **ssh key 的準備**：在**筆電**和 **VM** 之間建立一對專門用於自動化部署的 SSH Key，讓 GitHub Actions 才有權限可以登入 vm 中。
2. **設定 GitHub Repo**：將 SSH 私鑰和伺服器資訊，以加密的方式儲存在 github repo 中的 Secrets 中。
3. **建立 GitHub Actions Workflow**：撰寫 `.github/workflows/deploy.yaml
` 設定檔，這會告訴 github 該做些什麼把網站內容放到 vm 中。
4. **觸發與驗證**：將程式碼 `git push` 到 GitHub，觸發自動化流程，檢查流程是否正確。

### 階段一：準備工作 - 建立 SSH 信任關係

步驟 1：「筆電」上產生一對 SSH Key

在筆電上：

```bash
# -t ed25519: 使用更現代、更安全的加密演算法
# -C "..." : 註解，方便你辨識這對 Key 的用途
# -f ~/.ssh/hugo_deploy_key: 指定金鑰檔案的路徑與名稱，避免覆蓋你現有的 id_rsa 或其他金鑰
ssh-keygen -t ed25519 -C "GitHub Actions for yamatrail.com" -f ~/.ssh/hugo_deploy_key
```

執行之後，會在 `~/.ssh/` 中產生公鑰與私鑰

- `hugo_deploy_key`
- `hugo_deploy_key.pub`

步驟 2：在你的「GCP VM」上設定公鑰

在筆電上使用：

```bash
cat ~/.ssh/hugo_deploy_key.pub
```

把公鑰內容複製下來，貼到 vm 上的

```bash
# 使用 vim 編輯器打開檔案
vim ~/.ssh/authorized_keys
```

加在最後一行就可以了

### 階段二：設定 GitHub Repo - 存放你的秘密

在 github 的 Secret and variables > Actions > 

- SSH_PRIVATE_KEY
    
    ```bash
    cat ~/.ssh/hugo_deploy_key
    ```
    
- SERVER_HOST
- SERVER_USER
    - 填寫筆電的使用者，因為就我的情況來說，產生金鑰與登入 vm 的使用者是跟我的筆電使用者一樣的，github action 要魔你的是這個使用者登入 vm 的行為。
- TARGET_DIR
	- 告訴 github action 這個網站要放在哪裡。
    
    ```bash
    /var/www/yamatrail.com/
    ```
    

### 階段三：建立 GitHub Actions Workflow

```bash
mkdir -p .github/workflows
```

```bash
vim .github/workflows/deploy.yaml
```

在 vm 上要設定路徑的使用者跟我們存放 SSH key 的帳號一樣，這裡一樣使用筆電的使用者用 ssh 登入 vm，並且執行以下指令更改

```bash
# 使用我的筆電登入之後，執行
sudo chown -R $USER:$USER /var/www/yamatrail.com
```

另外，vm 上也要安裝 `rsync` 因為 github action 在操作時會需要用到這個指令

```bash
sudo apt update

# 安裝 rsync
sudo apt install rsync
```

### 階段四：觸發與驗證

```bash
git add .

git commit -m "feat: Add GitHub Actions workflow for automatic deployment"

git push origin main
```

沒意外的話，網站上的內容就會自動更新了。

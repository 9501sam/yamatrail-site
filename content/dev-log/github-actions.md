+++
date = '2025-09-25T12:16:16+08:00'
draft = false
title = '[這個網站的誕生: 04] Github Actions 建立自動部署'
series = "這個網站的誕生"
weight = 4
+++
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
# 這個在 vm 上執行
vim ~/.ssh/authorized_keys
```

加在最後一行就可以了

### 階段二：設定 GitHub Repo

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

```yaml
# 工作流程的名稱，會顯示在 GitHub Actions 頁面上
name: Deploy Hugo Site to GCP VM

# 觸發條件：當 main 分支收到 push 時執行
on:
  push:
    branches:
      - main

# 定義要執行的任務
jobs:
  build-and-deploy:
    # 執行任務的虛擬環境類型
    runs-on: ubuntu-latest

    # 任務包含的步驟
    steps:
      # 步驟一：拉取你的 GitHub Repo 原始碼
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: true  # 如果你的佈景主題 (theme) 是用 git submodule 方式引入的，這行很重要

      # 步驟二：安裝並設定 Hugo 環境
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest' # 使用最新版的 Hugo
          extended: true         # 如果你的佈景主題需要 Sass/SCSS 支援

      # 步驟三：執行 Hugo 指令來編譯網站
      # --minify 會壓縮 HTML/CSS/JS，讓網站載入更快
      - name: Build static site
        run: hugo --minify

      # 步驟四：將編譯好的網站成品 (public/ 目錄) 部署到你的 VM
      - name: Deploy to server
        uses: easingthemes/ssh-deploy@v5.0.0
        with:
          # 從 GitHub Secrets 讀取 SSH 私鑰
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          # 遠端伺服器 IP
          REMOTE_HOST: ${{ secrets.SERVER_HOST }}
          # 遠端伺服器使用者
          REMOTE_USER: ${{ secrets.SERVER_USER }}
          # 要上傳的來源目錄 (Hugo 編譯後的成品)
          SOURCE: "public/"
          # 要部署到遠端伺服器的目標目錄
          TARGET: ${{ secrets.TARGET_DIR }}
          # 使用 rsync 的參數，-avz 是標準用法，--delete 會刪除伺服器上多餘的舊檔案
          ARGS: "-avz --delete"
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
> github action 失敗時，如果發現是 vm 設定或是 github 設定上的問題 (並非 hugo 設定或文章內容有誤)，可以按 `github repo` > `actions` > `點選出錯的 Action` > `Re-run all jobs` 就可以不經由 `git push` 直接重跑一次我們所設定的 action

至此，我們的工作流程就會是：
1. 在筆電上使用 `hugo server`、瀏覽器連上 [http://localhost:1313/](http://localhost:1313/)
1. 編輯文章內容與設定檔，同時瀏覽器上可檢查內容與效果
1. 內容沒問題之後 `git commit` 以及 `git push`，之後我們的網站就會經由 github action 更新了
1. 最後檢查 [https://yamatrail.com/](https://yamatrail.com/) 的內容是否更新成功。

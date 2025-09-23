+++
date = '2025-09-21T13:03:12+08:00'
draft = false
title = '[這個網站的誕生: 02] 在 vm 上建立 nginx server'
series = "這個網站的誕生"
+++
### 取得 domain

架設網站需要先註冊一個 domain, 這裡我使用的是 Cloudflare，

註冊好一個 domain 之後，在 DNS > Records 的地方新增一個 Record

- Type: A
- name: yamatrail.com
- IPv4 Address: 填入 vm 的 IP (在 google cloud 上取得)

這會告訴他當其他人在瀏覽器中輸入 [yamatrail.com](http://yamatrail.com) 時，會導到我們先前在 google cloud 上所建立的 vm

### 在 Cloudflare 產生憑證

1. 登入您的 Cloudflare 儀表板，選擇 `yamatrail.com`。
2. 點擊左側選單的 **SSL/TLS**，然後選擇 **Origin Server** (源伺服器) 子分頁。
3. 點擊「**Create Certificate**」按鈕。
4. 保持預設選項（私鑰類型 RSA，憑證有效期 15 年），直接點擊最下方的「**Create**」。
5. **這是最關鍵的一步**：Cloudflare 會顯示兩個大框框的文字，分別是 **Origin Certificate** 和 **Private Key**。
    - **請立刻將這兩段文字完整複製出來**，並分別儲存到您**本地筆電**的兩個純文字檔案中：
        - 將 **Origin Certificate** 的內容存為 `yamatrail.com.pem`。
        - 將 **Private Key** 的內容存為 `yamatrail.com.key`。
    - **注意：Private Key 只會顯示這一次，關閉頁面後就再也看不到了！**

之後就可以在 vm 上安裝並設定 nginx 了，在這之前可以先檢查 vm 的設定是否有允許 HTTP 及 HTTPS 連線

### 安裝 nginx

```bash
sudo apt install nginx
```

### 設定 nginx

安裝好之後，nginx 有以下幾個比較重要的資料夾

- 設定檔： `/etc/nginx`
- 網站內容：`/var/www/`
- log 存放位置： `/var/log/nginx`

### 設定檔：`/etc/nginx`

這裡的 key 可以經由 cloudflare 產生

```bash
sudo mkdir -p /etc/nginx/ssl
```

```bash
sudo vim /etc/nginx/ssl/yamatrail.com.pem
```

```bash
sudo vim /etc/nginx/ssl/yamatrail.com.key
```

```bash
sudo vim /etc/nginx/sites-available/yamatrail.com
```

```bash
# 這個 server block 會抓取所有 HTTP (port 80) 的請求，並永久轉址到 HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name yamatrail.com www.yamatrail.com;
    return 301 https://$server_name$request_uri;
}

# 這是主要的 HTTPS server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yamatrail.com www.yamatrail.com;

    # SSL 憑證設定
    ssl_certificate /etc/nginx/ssl/yamatrail.com.pem;
    ssl_certificate_key /etc/nginx/ssl/yamatrail.com.key;

    # 網站根目錄和其他設定
    root /var/www/yamatrail.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

做好設定檔之後，使用 link 的方式放到 `sites-enabled` 中

```bash
sudo ln -s /etc/nginx/sites-available/yamatrail.com /etc/nginx/sites-enabled/
```

利用 link 的好處在於可以快速的切換網站設定

### 網站本體的放置

```bash
sudo mkdir -p /var/www/yamatrail.com
```

```bash
sudo vim /var/www/yamatrail.com/index.html
```

可以先在 index.html 中先寫一些簡單的東西

```bash
Hello!
```

### 重啟 nginx

```bash
sudo nginx -t # 檢查有沒有語法上的錯誤
```

```bash
sudo systemctl restart nginx
```

```bash
sudo systemctl status nginx
```

之後就可以經由網址 [yamatrail.com](https://yamatrail.com) 進入網站了

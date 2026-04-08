# Other-PosteMailServer

```
Poste.io 郵件伺服器 + Nginx 反向代理 Docker 環境
```

---

## 目錄

- [Other-PosteMailServer](#other-postemailserver)
  - [目錄](#目錄)
  - [專案說明](#專案說明)
  - [架構概覽](#架構概覽)
  - [連接埠說明](#連接埠說明)
  - [各模式說明](#各模式說明)
    - [模式一：SSL 證書從其他主機取得](#模式一ssl-證書從其他主機取得)
    - [模式二：SSL 證書在本機](#模式二ssl-證書在本機)
  - [執行流程](#執行流程)
    - [模式一流程](#模式一流程)
    - [模式二流程](#模式二流程)
  - [使用方法](#使用方法)
    - [前置作業](#前置作業)
    - [模式一：遠端複製 SSL 證書](#模式一遠端複製-ssl-證書)
    - [模式二：本機 SSL 證書](#模式二本機-ssl-證書)
    - [啟動服務](#啟動服務)
    - [管理介面](#管理介面)
  - [設定檔說明](#設定檔說明)
    - [.env](#env)
    - [Nginx 設定](#nginx-設定)
    - [Haraka 白名單](#haraka-白名單)
    - [docker-compose.yml](#docker-composeyml)
  - [建議注意事項](#建議注意事項)
    - [DNS 設定](#dns-設定)
    - [SSL 證書](#ssl-證書)
    - [防火牆](#防火牆)
    - [Poste.io 設定](#posteio-設定)
    - [Nginx 反向代理](#nginx-反向代理)
    - [一般建議](#一般建議)
  - [參考資料](#參考資料)

---

## 專案說明

以 Docker Compose 架設 [Poste.io](https://poste.io/) 郵件伺服器，搭配 Nginx 作為反向代理並處理 SSL 終止，提供完整的郵件收發（SMTP / IMAP / POP3）及 Web 管理介面。

---

## 架構概覽

```
Internet
    │
    ▼
Nginx (80 / 443)          ← SSL 終止、反向代理
    │
    ├──► admin.example.com → Poste 管理介面 / Webmail
    └──► mail.example.com  → Poste Webmail
    │
    ▼
Poste.io (poste container)
    │
    ├── SMTP   :25   ← 郵件接收（MX）
    ├── POP3   :110  ← 郵件客戶端收信
    ├── IMAP   :143  ← 郵件客戶端收信
    ├── IMAPS  :993  ← IMAP over SSL
    └── POP3S  :995  ← POP3 over SSL
```

---

## 連接埠說明

| 埠號 | 協定 | 用途 |
|------|------|------|
| 25   | SMTP | 郵件伺服器接收外部郵件（MX） |
| 110  | POP3 | 郵件客戶端收信（未加密） |
| 143  | IMAP | 郵件客戶端收信（未加密） |
| 465  | SMTPS | SMTP over SSL（防火牆需開放） |
| 587  | Submission | SMTP 投遞（防火牆需開放） |
| 993  | IMAPS | IMAP over SSL |
| 995  | POP3S | POP3 over SSL |
| 80   | HTTP | Nginx（轉導至 HTTPS） |
| 443  | HTTPS | Nginx（反向代理至 Poste） |

---

## 各模式說明

本專案依 SSL 證書的取得方式分為兩種部署模式：

### 模式一：SSL 證書從其他主機取得

適用於 SSL 證書（Let's Encrypt）已存在於另一台主機（如已有 Nginx 或 Certbot 的伺服器），透過 SCP 複製到本機再部署。

**流程重點：**
1. 從持有證書的主機用 `scp` 下載 `/etc/letsencrypt/live/<domain>/`
2. 將證書放至目標主機的 `/etc/letsencrypt/live/<domain>/`
3. 複製至 `conf/ssl/` 供容器掛載

### 模式二：SSL 證書在本機

適用於 SSL 證書（Let's Encrypt）已透過 Certbot 直接在本機申請，直接從 `/etc/letsencrypt/live/<domain>/` 複製即可。

**流程重點：**
1. 確認本機 `/etc/letsencrypt/live/<domain>/` 存在
2. 直接複製至 `conf/ssl/` 供容器掛載

---

## 執行流程

### 模式一流程

```
1. Clone 專案並進入目錄
   git clone ... poste_mail_server && cd poste_mail_server
        │
        ▼
2. 設定域名變數、複製設定檔範本
   cp .env.sample .env
   cp conf/admin.example.com.conf conf/nginx/admin.<domain>.conf
   cp conf/mail.example.com.conf conf/nginx/mail.<domain>.conf
   sed -i "s/example.com/<domain>/g" .env conf/nginx/admin.<domain>.conf conf/nginx/mail.<domain>.conf
        │
        ▼
3. 建立 SSL 目錄
   mkdir -p /etc/letsencrypt/live
        │
        ▼
4. 從持有證書的主機下載證書
   scp -r <ssl_hostname>:/etc/letsencrypt/live/<domain> .
        │
        ▼
5. 上傳證書至目標主機（若在中繼機操作）
   scp -r ./<domain> <target_hostname>:/etc/letsencrypt/live
        │
        ▼
6. 複製證書至容器掛載路徑
   cp -r /etc/letsencrypt/live/<domain> conf/ssl
        │
        ▼
7. 開放防火牆連接埠
   # ufw: ufw allow 25,587,465/tcp
   # firewalld: firewall-cmd --permanent --add-port=25/tcp --add-port=587/tcp --add-port=465/tcp && firewall-cmd --reload
        │
        ▼
8. 啟動服務
   docker compose up -d
        │
        ▼
9. 設定 DNS，瀏覽管理介面完成初始設定
   https://admin.<domain>/admin/box/
```

### 模式二流程

```
1. Clone 專案並進入目錄
   git clone ... poste_mail_server && cd poste_mail_server
        │
        ▼
2. 設定域名變數、複製設定檔範本
   cp .env.sample .env
   cp conf/admin.example.com.conf conf/nginx/admin.<domain>.conf
   cp conf/mail.example.com.conf conf/nginx/mail.<domain>.conf
   sed -i "s/example.com/<domain>/g" .env conf/nginx/admin.<domain>.conf conf/nginx/mail.<domain>.conf
        │
        ▼
3. 複製本機證書至容器掛載路徑
   cp -r /etc/letsencrypt/live/<domain> conf/ssl
        │
        ▼
4. 開放防火牆連接埠
   # ufw: ufw allow 25,587,465/tcp
   # firewalld: firewall-cmd --permanent --add-port=25/tcp --add-port=587/tcp --add-port=465/tcp && firewall-cmd --reload
        │
        ▼
5. 啟動服務
   docker compose up -d
        │
        ▼
6. 設定 DNS，瀏覽管理介面完成初始設定
   https://admin.<domain>/admin/box/
```

---

## 使用方法

### 前置作業

```bash
git clone https://github.com/open222333/Other-PosteMailServer.git poste_mail_server
cd poste_mail_server
```

**DNS 設定（需在域名服務商設定）：**

```
;; A Records
admin.example.com.   1   IN   A   xxx.xxx.xxx.xxx
mail.example.com.    1   IN   A   xxx.xxx.xxx.xxx

;; MX Records
example.com.         1   IN   MX  10 mail.example.com.
mail.example.com.    1   IN   MX  10 mail.example.com.
```

**防火牆開放：**

ufw：

```bash
ufw allow 25/tcp
ufw allow 587/tcp
ufw allow 465/tcp
```

firewalld：

```bash
firewall-cmd --permanent --add-port=25/tcp
firewall-cmd --permanent --add-port=587/tcp
firewall-cmd --permanent --add-port=465/tcp
firewall-cmd --reload
```

---

### 模式一：遠端複製 SSL 證書

```bash
domain_name="yourdomain"

# 複製設定檔範本並替換域名
cp .env.sample .env
cp conf/admin.example.com.conf conf/nginx/admin.$domain_name.conf
cp conf/mail.example.com.conf conf/nginx/mail.$domain_name.conf
sed -i "s/example.com/$domain_name/g" .env
sed -i "s/example.com/$domain_name/g" conf/nginx/admin.$domain_name.conf
sed -i "s/example.com/$domain_name/g" conf/nginx/mail.$domain_name.conf

# 建立 SSL 目錄
mkdir -p /etc/letsencrypt/live
```

```bash
# 從持有證書的主機下載（在本機或中繼機執行）
ssl_hostname="ssl_hostname"
target_hostname="target_hostname"

scp -r $ssl_hostname:/etc/letsencrypt/live/$domain_name .
scp -r ./$domain_name $target_hostname:/etc/letsencrypt/live
```

```bash
# 複製至容器掛載路徑
cp -r /etc/letsencrypt/live/$domain_name conf/ssl
```

---

### 模式二：本機 SSL 證書

```bash
domain_name="yourdomain"

# 複製設定檔範本並替換域名
cp .env.sample .env
cp conf/admin.example.com.conf conf/nginx/admin.$domain_name.conf
cp conf/mail.example.com.conf conf/nginx/mail.$domain_name.conf
sed -i "s/example.com/$domain_name/g" .env
sed -i "s/example.com/$domain_name/g" conf/nginx/admin.$domain_name.conf
sed -i "s/example.com/$domain_name/g" conf/nginx/mail.$domain_name.conf

# 複製本機證書至容器掛載路徑
cp -r /etc/letsencrypt/live/$domain_name conf/ssl
```

---

### 啟動服務

```bash
docker compose up -d

# 查看容器狀態
docker compose ps

# 查看 Poste 容器日誌
docker logs -f poste

# 停止服務
docker compose down
```

---

### 管理介面

服務啟動後，透過瀏覽器存取以下網址：

| 網址 | 用途 |
|------|------|
| `https://admin.example.com/admin/box/` | 管理後台（新增帳號、域名、DKIM 等） |
| `https://admin.example.com/webmail/` | Webmail 網頁郵件 |
| `https://admin.example.com/admin/api/doc` | REST API 文件 |

> 首次啟動需設定管理員帳號密碼，請立即登入後台完成初始化設定。

---

## 設定檔說明

### .env

從 `.env.sample` 複製，主要設定域名：

```env
DOMAIN_NAME=example.com
```

### Nginx 設定

`conf/admin.example.com.conf`（範本，實際使用需複製為 `conf/nginx/admin.<domain>.conf`）：

```nginx
server {
    listen 80;
    server_name admin.example.com;
    return 301 https://$host$request_uri;   # HTTP → HTTPS 強制轉導
}

server {
    listen 443 ssl;
    server_name admin.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://poste:80;          # 反向代理至 Poste 容器
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

`conf/mail.example.com.conf`（Webmail 路由至 `/mail`）：類似上方，`location /mail` 代理至 Poste。

### Haraka 白名單

`conf/poste/haraka/connect.rdns_access.whitelist`：Haraka `connect.rdns_access` plugin 的 IP 白名單，白名單內的 IP 不受 rdns 檢查限制。

```
# 每行一個 IP 或 CIDR，# 開頭為註解

# Docker 預設網段
172.17.0.0/16

# Docker Compose 自訂網段（視實際情況調整）
# 172.18.0.0/16
```

此檔案已透過 `docker-compose.yml` 掛載至容器內 `/data/haraka/config/connect.rdns_access.whitelist`，修改後**不需重啟容器**，Haraka 會自動重新讀取。

若不確定 Docker 網段，可執行：

```bash
docker network inspect bridge --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

### docker-compose.yml

```yaml
services:
  poste:
    image: analogic/poste.io
    environment:
      - DISABLE_CLAMAV=TRUE    # 停用 ClamAV 防毒（節省資源）
      - DISABLE_RSPAMD=TRUE    # 停用 Rspamd 垃圾郵件過濾（節省資源）
      - HTTPS=OFF              # HTTPS 由 Nginx 處理，Poste 僅用 HTTP
    volumes:
      - ./conf/ssl:/etc/letsencrypt/live   # SSL 證書掛載
      - ./data/poste:/data                 # 郵件資料持久化

  nginx:
    image: nginx:latest
    volumes:
      - ./conf/nginx:/etc/nginx/conf.d     # Nginx 虛擬主機設定
      - ./conf/ssl:/etc/letsencrypt/live   # SSL 證書共享
```

---

## 建議注意事項

### DNS 設定

- DNS 生效需時（通常 1 小時內，最長 48 小時），建議在部署前提早設定
- **MX 記錄**必須正確指向 `mail.<domain>`，否則外部郵件無法送達
- 建議同時設定 **SPF**、**DKIM**（Poste 管理後台可產生）、**DMARC** 記錄，提升郵件送達率並避免被判垃圾郵件

### SSL 證書

- `conf/ssl/<domain>/` 內需包含 `fullchain.pem` 與 `privkey.pem`，路徑需與 Nginx 設定一致
- Let's Encrypt 證書有效期 90 天，到期前需重新申請並更新 `conf/ssl/` 目錄，更新後重啟 Nginx：`docker compose restart nginx`
- 使用模式一（SCP）複製時，確認來源主機的 SSH 連線權限與目標路徑正確

### 防火牆

- 埠號 **25（SMTP）必須對外開放**，否則其他郵件伺服器無法將郵件送達
- 587（Submission）與 465（SMTPS）用於郵件客戶端發信，視需求開放
- 若主機有雲端防火牆（如 AWS Security Group、GCP Firewall），除 `ufw` / `firewalld` 外也需在雲端控制台開放對應埠號

### Poste.io 設定

- `DISABLE_CLAMAV=TRUE` 與 `DISABLE_RSPAMD=TRUE` 可節省記憶體，若需防毒與垃圾郵件過濾請移除這兩項環境變數（需額外 1-2GB RAM）
- 郵件資料存放於 `./data/poste/`，請定期備份此目錄
- 首次啟動後**立即設定管理員密碼**，避免未授權存取

### Nginx 反向代理

- `conf/nginx/` 目錄下需有對應域名的 `.conf` 檔案才會被 Nginx 載入，確認檔名包含實際域名（非 `example.com`）
- 若修改 Nginx 設定後需重新載入：`docker compose exec nginx nginx -s reload`
- Nginx 設定錯誤會導致容器啟動失敗，可先用 `docker compose exec nginx nginx -t` 驗證設定語法

### 一般建議

- 正式環境部署前，先在測試環境驗證 DNS、SSL、防火牆設定均正確
- `.env` 包含域名資訊，`.env.sample` 作為範本提交至版本控制，`.env` 本身視情況加入 `.gitignore`
- 使用 `docker logs -f poste` 觀察郵件接收/傳送日誌，排查送信問題

---

## 參考資料

- [Poste.io（郵件伺服器管理工具）筆記](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/MailServer(%E9%83%B5%E7%AE%B1%E4%BC%BA%E6%9C%8D%E5%99%A8)/Poste.io(%E9%83%B5%E4%BB%B6%E4%BC%BA%E6%9C%8D%E5%99%A8%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7).md)

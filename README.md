# Other-PosteMailServer

```
```

## 目錄

- [Other-PosteMailServer](#other-postemailserver)
  - [目錄](#目錄)
  - [參考資料](#參考資料)
- [用法](#用法)

## 參考資料

[Poste.io(郵件伺服器管理工具)](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/MailServer(%E9%83%B5%E7%AE%B1%E4%BC%BA%E6%9C%8D%E5%99%A8)/Poste.io(%E9%83%B5%E4%BB%B6%E4%BC%BA%E6%9C%8D%E5%99%A8%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7).md)

# 用法

```bash
git clone https://github.com/open222333/Other-PosteMailServer.git poste_mail_server
cd poste_mail_server
```

防火牆連接埠

```bash
ufw allow 25
ufw allow 587
ufw allow 465
```

DNS 指向

```
;; A Records
admin.example.com.	1	IN	A	xxx.xxx.xxx.xxx
mail.example.com.	1	IN	A	xxx.xxx.xxx.xxx

;; MX Records
example.com.	1	IN	MX	10 mail.example.com.
mail.example.com.	1	IN	MX	10 mail.example.com.
```

```bash
domain_name="yourdomain.com"
cp .env.sample .env
cp conf/admin.example.com.conf conf/nginx/admin.$domain_name.conf
cp conf/mail.example.com.conf conf/nginx/mail.$domain_name.conf
sed -i "s/example.com/$domain_name/g" .env
sed -i "s/example.com/$domain_name/g" conf/nginx/admin.$domain_name.conf
sed -i "s/example.com/$domain_name/g" conf/nginx/mail.$domain_name.conf
cp -r /etc/letsencrypt/live/$domain_name conf/ssl
```

```bash
docker-compose up -d
```

```
https://admin.example.com/admin/box/
https://admin.example.com/webmail/
https://admin.example.com/admin/api/doc
```
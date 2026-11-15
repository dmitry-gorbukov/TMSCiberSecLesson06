[На главную страницу](../README.md)

# Задание 5

----------
**Поднятие HTTPS-сервиса без ошибок сертификата**

**Условие:**
Необходимо развернуть локальный HTTPS-сервис. 

**Задача:**

-- Сгенерировать сертификат 
-- Поднять nginx 
-- Добавить сертификат в доверенные 
-- Выполнить запрос: 
curl https://localhost 
**Требование:** в выводе не должно быть ошибок 

**Результат:** 
команды/шаги настройки 
вывод curl

----------


## Шаг 0. Подготовка и установка пакетов

```bash
sudo apt update
sudo apt install -y nginx openssl ca-certificates curl
```

Проверяем, что nginx установлен и запущен:

```bash
┌──(dima㉿kali)-[~]
└─$ nginx -v
nginx version: nginx/1.30.4

┌──(dima㉿kali)-[~]
└─$ sudo systemctl status nginx --no-pager
○ nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: inactive (dead)
       Docs: man:nginx(8)

┌──(dima㉿kali)-[~]
└─$ 
```

Если сервис не запущен:

```bash
┌──(dima㉿kali)-[~]
└─$ sudo systemctl start nginx

┌──(dima㉿kali)-[~]
└─$ sudo systemctl enable nginx
Created symlink '/etc/systemd/system/multi-user.target.wants/nginx.service' → '/usr/lib/systemd/system/nginx.service'.

┌──(dima㉿kali)-[~]
└─$ 
```

> **Пояснение:** `openssl` нужен для генерации ключа/сертификата, `nginx` — сам веб-сервер, `ca-certificates` — для работы хранилища доверенных сертификатов, `curl` — для проверки.

---

## Шаг 1. Генерация самоподписанного сертификата

Создадим каталог для сертификатов и сгенерируем ключ + сертификат.

```bash
┌──(dima㉿kali)-[~]
└─$ sudo mkdir -p /etc/nginx/ssl

┌──(dima㉿kali)-[~]
└─$ cd /etc/nginx/ssl

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```
Приватный ключ (2048 бит RSA)
```bash
sudo openssl genrsa -out localhost.key 2048
```
Самоподписанный сертификат на 365 дней с SAN (Subject Alternative Name)
```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo openssl req -new -x509 -key localhost.key -out localhost.crt -days 365 -subj "/C=RU/ST=Local/L=Local/O=LocalDev/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

Права
```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo chmod 600 localhost.key

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo chmod 644 localhost.crt

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

Проверяем, что сертификат создан и содержит SAN:

```bash

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ openssl x509 -in localhost.crt -noout -subject -issuer -dates -ext subjectAltName
subject=C=RU, ST=Local, L=Local, O=LocalDev, CN=localhost
issuer=C=RU, ST=Local, L=Local, O=LocalDev, CN=localhost
notBefore=Nov 15 15:40:25 2026 GMT
notAfter=Nov 15 15:40:25 2027 GMT
X509v3 Subject Alternative Name: 
    DNS:localhost, IP Address:127.0.0.1

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```


> **Пояснения:** 
> - **SAN (`subjectAltName`)** обязателен: современные клиенты (Chrome, curl) игнорируют `CN` и проверяют именно SAN. Без `DNS:localhost` браузер/curl выдаст ошибку `certificate subject name 'localhost' does not match`.
> - Сертификат **самоподписанный** (`-x509`), issuer = subject. Поэтому его нужно **явно добавить в доверенные**, иначе `curl` скажет `self-signed certificate`.

---

## Шаг 2. Настройка nginx на HTTPS

Создадим отдельный конфиг (чтобы не ломать дефолтный):

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ cat /etc/nginx/sites-available/localhost-ssl 
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/localhost.crt;
    ssl_certificate_key /etc/nginx/ssl/localhost.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    root  /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}


┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

Активируем сайт и проверяем конфиг:

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo ln -sf /etc/nginx/sites-available/localhost-ssl /etc/nginx/sites-enabled/localhost-ssl


┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful


┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo systemctl reload nginx


┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo systemctl status nginx           
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Sun 2026-11-15 18:34:26 +03; 13min ago
 Invocation: dc0aca41dfd247e9ae2eb5b39f585292
       Docs: man:nginx(8)
    Process: 119918 ExecReload=/usr/sbin/nginx -g daemon on; master_process on; -s reload (code=exited, status=0/SUCCESS)
   Main PID: 113347 (nginx)
      Tasks: 5 (limit: 7007)
     Memory: 5M (peak: 10.4M)
        CPU: 123ms
     CGroup: /system.slice/nginx.service
             ├─113347 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─119921 "nginx: worker process"
             ├─119922 "nginx: worker process"
             ├─119923 "nginx: worker process"
             └─119924 "nginx: worker process"

Nov 15 18:34:26 kali systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Nov 15 18:34:26 kali systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
Nov 15 18:47:39 kali systemd[1]: Reloading nginx.service - A high performance web server and a reverse proxy server...
Nov 15 18:47:39 kali nginx[119918]: 2026/11/15 18:47:39 [notice] 119918#119918: signal process started
Nov 15 18:47:39 kali systemd[1]: Reloaded nginx.service - A high performance web server and a reverse proxy server.

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

> **Пояснения:** `nginx -t` проверяет синтаксис и доступность файлов сертификата. `reload` применяет конфиг без простоя.

Проверяем, что порт 443 слушается:

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo ss -tlnp | grep ':443'
LISTEN 0      511          0.0.0.0:443       0.0.0.0:*    users:(("nginx",pid=119924,fd=17),("nginx",pid=119923,fd=17),("nginx",pid=119922,fd=17),("nginx",pid=119921,fd=17),("nginx",pid=113347,fd=17))

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

---

## Шаг 3. Добавление сертификата в доверенные (системное хранилище)

Скопируем наш сертификат в системное хранилище CA и обновим его:

```bash
# Копируем .crt в хранилище (расширение .crt обязательно)
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo cp /etc/nginx/ssl/localhost.crt /usr/local/share/ca-certificates/localhost.crt


# Обновляем хранилище доверенных сертификатов
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt, it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
Processing triggers for ca-certificates-java (20260311)…
Adding debian:localhost.pem
done.
done.

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```



Проверяем, что сертификат попал в хранилище:

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ ls -l /etc/ssl/certs/ | grep localhost
lrwxrwxrwx 1 root root     13 Nov 15 18:51 0ab4f94a.0 -> localhost.pem
lrwxrwxrwx 1 root root     46 Nov 15 18:51 localhost.pem -> /usr/local/share/ca-certificates/localhost.crt


┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ openssl verify /etc/nginx/ssl/localhost.crt
/etc/nginx/ssl/localhost.crt: OK


┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```


> **Пояснения:** 
> - `update-ca-certificates` читает `.crt` из `/usr/local/share/ca-certificates/` и добавляет в объединённый бандл `/etc/ssl/certs/ca-certificates.crt`.
> - `curl` использует именно этот бандл. После обновления `curl` начнёт доверять нашему сертификату.
> - **Файл** должен быть в формате PEM и иметь расширение `.crt` (не `.pem`), иначе `update-ca-certificates` его проигнорирует.

---

## Шаг 4. Проверка `curl https://localhost`

### 4.1. Базовый запрос

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ cat /var/www/html/index.html             
<!DOCTYPE html>
<html>
<head><title>Secure Localhost</title></head>
<body>
<h1>HTTPS is working!</h1>
<p>Certificate verified successfully.</p>
</body>
</html>
```

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ curl https://localhost
<!DOCTYPE html>
<html>
<head><title>Secure Localhost</title></head>
<body>
<h1>HTTPS is working!</h1>
<p>Certificate verified successfully.</p>
</body>
</html>

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```


### 4.2. Проверка с подробным выводом TLS

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ curl -v https://localhost
* Host localhost:443 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:443...
* connect to ::1 port 443 from ::1 port 53808 failed: Connection refused
*   Trying 127.0.0.1:443...
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* SSL Trust Anchors:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
*   CApath: /etc/ssl/certs
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519MLKEM768 / RSASSA-PSS
* ALPN: server accepted http/1.1
* Server certificate:
*   subject: C=RU; ST=Local; L=Local; O=LocalDev; CN=localhost
*   start date: Nov 15 15:40:25 2026 GMT
*   expire date: Nov 15 15:40:25 2027 GMT
*   issuer: C=RU; ST=Local; L=Local; O=LocalDev; CN=localhost
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   subjectAltName: "localhost" matches cert's "localhost"
* OpenSSL verify result: 0
* SSL certificate verified via OpenSSL.
* Established connection to localhost (127.0.0.1 port 443) from 127.0.0.1 port 33452 
* using HTTP/1.x
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/8.22.0
> Accept: */*
> 
* Request completely sent off
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
< HTTP/1.1 200 OK
< Server: nginx
< Date: Sun, 15 Nov 2026 16:04:56 GMT
< Content-Type: text/html
< Content-Length: 160
< Last-Modified: Sun, 15 Nov 2026 16:02:25 GMT
< Connection: keep-alive
< ETag: "6af9d791-a0"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head><title>Secure Localhost</title></head>
<body>
<h1>HTTPS is working!</h1>
<p>Certificate verified successfully.</p>
</body>
</html>
* Connection #0 to host localhost:443 left intact

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

**Смотрим вывод `curl -v`:**

```
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519MLKEM768 / RSASSA-PSS
* Server certificate:
*   subject: C=RU; ST=Local; L=Local; O=LocalDev; CN=localhost
*   subjectAltName: "localhost" matches cert's "localhost"
* OpenSSL verify result: 0
* SSL certificate verified via OpenSSL.
< HTTP/1.1 200 OK
```

**Всё решено:**

- ✅ **TLSv1.3** с современным шифром `TLS_AES_256_GCM_SHA384`;
- ✅ **`OpenSSL verify result: 0`** — цепочка доверия сошлась;
- ✅ **`SSL certificate verified via OpenSSL`** — сертификат доверенный;
- ✅ **`subjectAltName: "localhost" matches cert's "localhost"`** — SAN совпадает с запрошенным хостом;
- ✅ **`HTTP/1.1 200 OK`** — страница отдаётся успешно;
- ✅ **Никаких** `SSL certificate problem` / `curl: (60)` — ошибок нет.



------



Ожидаемо: `200` (или `301`, если настроен редирект — у нас `200`).

### 4.3. Проверка через `openssl s_client`

```bash
┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ openssl s_client -connect localhost:443 -servername localhost -CAfile /etc/ssl/certs/ca-certificates.crt </dev/null 2>/dev/null | grep -E "Verify return code|subject="
subject=C=RU, ST=Local, L=Local, O=LocalDev, CN=localhost
Verify return code: 0 (ok)

┌──(dima㉿kali)-[/etc/nginx/ssl]
└─$ 
```

> **Пояснения:** `Verify return code: 0 (ok)` — сертификат успешно проверен по цепочке доверия. Если бы мы не добавили его в доверенные, было бы `Verify return code: 18 (self-signed certificate)`.

---


[На главную страницу](../README.md)
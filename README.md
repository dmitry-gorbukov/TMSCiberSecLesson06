# Лабораторная работа 06

1. [Задание 1.](doc/Zadanie1.md) 

**Ответ:**


|**Login**|**Password**|
| ------------- | ------------------------ |
| **user**| MILO2008|
| **admin**| hxtour|
| **web_admin**| t061584|
| **root** | washington1776|
| **db_user**| mmr431723|


1. [Задание 2.](doc/Zadanie2.md)

**Ответ:**


| # | **hash (SHA-512)** | **PIN** |
|---|---|---|
| 1 | 75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a| 5472|
| 2 | 7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1| 4653|
| 3 | 8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0| 11082|
| 4 | 7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f| 17456|
| 5  | 684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324| 497235|



3. [Задание 3.](doc/Zadanie3.md)


**Ответ:**


| login | salt | hash (SHA-256) | password |
|---|---|---|---|
| user | 1337 | `70ab6e15…d4f987` | `54cami9` |
| admin | 9117 | `6c39425f…13726b25` | `428248h` |
| web_admin | 2560 | `433a276d…c5e3828d` | `656800` |
| root | 6766 | `ef62c8bb…42750dee` | `rubenkus` |
| db_user | 3780 | `44ca6ade…b164fd5` | `cup+preds` |

Формат ответа к задаче (`login → password`):

```
user      → 54cami9
admin     → 428248h
web_admin → 656800
root      → rubenkus
db_user   → cup+preds
```

4. [Задание 4.](doc/Zadanie4.md)


**Ответ:**

Формат ответа к задаче (`hash → password`):

```
d1e576b71ccef5978d221fadf4f0e289 → superpassword
22d7fe8c185003c98f97e5d6ced420c7 → qwertyui
8a24367a1f46c141048752f2d5bbd14b → P@ssw0rd!
8344711654700fe530f0d2ac74599a48 → aezakmi
a67e565b11cd18f7a922b58f5476b569 → hesoyam
```

5. [Задание 5.](doc/Zadanie5.md)

**Результат:**

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

-  **TLSv1.3** с современным шифром `TLS_AES_256_GCM_SHA384`;
-  **`OpenSSL verify result: 0`** — цепочка доверия сошлась;
-  **`SSL certificate verified via OpenSSL`** — сертификат доверенный;
-  **`subjectAltName: "localhost" matches cert's "localhost"`** — SAN совпадает с запрошенным хостом;
-  **`HTTP/1.1 200 OK`** — страница отдаётся успешно;
-  **Никаких** `SSL certificate problem` / `curl: (60)` — ошибок нет.




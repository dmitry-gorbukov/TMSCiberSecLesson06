[На главную страницу](../README.md)

# Задание 3

# Анализ с учётом соли


---

## Шаг 0. Подготовка рабочего каталога и данных

```bash
┌──(dima㉿kali)-[~]
└─$ mkdir -p ~/salt && cd ~/salt

┌──(dima㉿kali)-[~/salt]
└─$ 
```

Исходный файл в формате `login:salt:hash`:

```bash
┌──(dima㉿kali)-[~/salt]
└─$ cat > hashes.txt <<'EOF'
user:1337:70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987
admin:9117:6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25
web_admin:2560:433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d
root:6766:ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee
db_user:3780:44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5
EOF

┌──(dima㉿kali)-[~/salt]
└─$ cat hashes.txt          
user:1337:70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987
admin:9117:6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25
web_admin:2560:433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d
root:6766:ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee
db_user:3780:44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5

┌──(dima㉿kali)-[~/salt]
└─$ 
```

---

## Шаг 1. Анализ структуры и определение типа

### 1.1. Разбор полей


```bash
┌──(dima㉿kali)-[~/salt]
└─$ awk -F: '{printf "login=%-10s salt=%-6s hash_len=%d\n", $1, $2, length($3)}' hashes.txt
login=user       salt=1337   hash_len=64
login=admin      salt=9117   hash_len=64
login=web_admin  salt=2560   hash_len=64
login=root       salt=6766   hash_len=64
login=db_user    salt=3780   hash_len=64

┌──(dima㉿kali)-[~/salt]
└─$ awk -F: '{print $3}' hashes.txt | grep -cE '^[0-9a-f]+$'
5

┌──(dima㉿kali)-[~/salt]
└─$ 
```

**Рассуждение:**
- Формат — `login:salt:hash`.
- `salt` — число из 4 цифр (`1337`, `9117`, `2560`, `6766`, `3780`). Это **числовая соль**, а не hex-строка и не base64.
- `hash` — **64 hex-символа** = 256 бит → семейство **SHA-256**.
- Значит, схема — **SHA-256(password + salt)** или **SHA-256(salt + password)**. Порядок конкатенации надо определить экспериментально (см. шаг 4).

### 1.2. Проверка

```bash
┌──(dima㉿kali)-[~/salt]
└─$ hashid -m "$(awk -F: 'NR==1{print $3}' hashes.txt)"            
Analyzing '70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987'
[+] Snefru-256 
[+] SHA-256 [Hashcat Mode: 1400]
[+] RIPEMD-256 
[+] Haval-256 
[+] GOST R 34.11-94 [Hashcat Mode: 6900]
[+] GOST CryptoPro S-Box 
[+] SHA3-256 [Hashcat Mode: 5000]
[+] Skein-256 
[+] Skein-512(256) 
```
```bash
┌──(dima㉿kali)-[~/salt]
└─$ hash-identifier "$(awk -F: 'NR==1{print $3}' hashes.txt)"
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------

Possible Hashs:
[+] SHA-256
[+] Haval-256

Least Possible Hashs:
[+] GOST R 34.11-94
[+] RipeMD-256
[+] SNEFRU-256
[+] SHA-256(HMAC)
[+] Haval-256(HMAC)
[+] RipeMD-256(HMAC)
[+] SNEFRU-256(HMAC)
[+] SHA-256(md5($pass))
[+] SHA-256(sha1($pass))
--------------------------------------------------
```
**Рабочая гипотеза:** SHA-256 с солью-числом. В `hashcat` «чистого» режима «SHA-256 с солью в отдельном поле» нет; используются режимы:
- **`-m 1410`** — `sha256($pass.$salt)`.

Проверим точные форматы:

```bash
┌──(dima㉿kali)-[~/salt]
└─$ hashcat --example-hashes -m 1410 | head -n 40
hashcat (v7.1.2) starting in hash-info mode

Hash Info:
==========

Hash mode #1410
  Name................: sha256($pass.$salt)
  Category............: Raw Hash salted and/or iterated
  Slow.Hash...........: No
  Deprecated..........: No
  Deprecated.Notice...: N/A
  Password.Type.......: plain
  Password.Len.Min....: 0
  Password.Len.Max....: 256
  Salt.Type...........: Generic
  Salt.Len.Min........: 0
  Salt.Len.Max........: 256
  Kernel.Type(s)......: pure, optimized
  Example.Hash.Format.: plain
  Example.Hash........: 5bb7456f43e3610363f68ad6de82b8b96f3fc9ad24e9d1f1f8d8bd89638db7c0:12480864321
  Example.Pass........: hashcat
  Benchmark.Mask......: ?a?a?a?a?a?a?a
  Autodetect.Enabled..: Yes
  Self.Test.Enabled...: Yes
  Potfile.Enabled.....: Yes
  Keep.Guessing.......: No
  Custom.Plugin.......: No
  Plaintext.Encoding..: ASCII, HEX


┌──(dima㉿kali)-[~/salt]
└─$ 
```

Обратите внимание на порядок в примере: `hash:salt` — hash первым, salt вторым. Это важно при подготовке входного файла.

### 1.3. Приведение к формату hashcat (`hash:salt`)

**формат hash:salt для режима 1410**

```bash
┌──(dima㉿kali)-[~/salt]
└─$ awk -F: '{print $3":"$2}' hashes.txt > hashes_hc.txt

┌──(dima㉿kali)-[~/salt]
└─$ cat hashes_hc.txt
70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987:1337
6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25:9117
433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d:2560
ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee:6766
44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5:3780

┌──(dima㉿kali)-[~/salt]
└─$ 
```

Для `john` формат `login:salt:hash` **не** стандартный; приведём к `hash$salt` (динамический формат) или используем `--mask`/`--wordlist` с явным указанием соли (см. шаг 3).

---

## Шаг 2. Hashcat, словарь rockyou с солью

### 2.1. Вариант: `sha256($pass.$salt)` → `-m 1410`


```bash
┌──(dima㉿kali)-[~/salt]
└─$ ls -al
total 16
drwxrwxr-x  2 dima dima 4096 Nov 15 15:33 .
drwx------ 24 dima dima 4096 Nov 15 15:06 ..
-rw-rw-r--  1 dima dima  350 Nov 15 15:18 hashes_hc.txt
-rw-rw-r--  1 dima dima  384 Nov 15 15:06 hashes.txt


┌──(dima㉿kali)-[~/salt]
└─$ hashcat -m 1410 -a 0 hashes_hc.txt ~/rockyou.txt --potfile-path ~/salt/hc.potfile -o ~/salt/analyzed_1410.txt
hashcat (v7.1.2) starting

............
Dictionary cache hit:
* Filename..: /home/dima/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1410 (sha256($pass.$salt))

.............
Speed.#01........:  3816.5 kH/s (0.56ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 5/5 (100.00%) Digests (total), 5/5 (100.00%) Digests (new), 5/5 (100.00%) Salts

............
Started: Sun Nov 15 15:34:14 2026
Stopped: Sun Nov 15 15:34:24 2026

┌──(dima㉿kali)-[~/salt]
└─$ cat ./analyzed_1410.txt 
433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d:2560:656800
ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee:6766:rubenkus
44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5:3780:cup+preds
70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987:1337:54cami9
6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25:9117:428248h

┌──(dima㉿kali)-[~/salt]
└─$ cat ./hc.potfile       
433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d:2560:656800
ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee:6766:rubenkus
44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5:3780:cup+preds
70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987:1337:54cami9
6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25:9117:428248h

┌──(dima㉿kali)-[~/salt]
└─$ 
```

Смотрим результат:

```bash
┌──(dima㉿kali)-[~/salt]
└─$ hashcat -m 1410 hashes_hc.txt --potfile-path ~/salt/hc.potfile --show
70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987:1337:54cami9
6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25:9117:428248h
433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d:2560:656800
ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee:6766:rubenkus
44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5:3780:cup+preds

┌──(dima㉿kali)-[~/salt]
└─$ 
```



## Шаг 3. John the Ripper

### 3.1. Динамический формат `dynamic_62` c `sha256($p.$s)`

`john` умеет SHA-256 с солью через динамические форматы. Формируем файл `hash$salt`:

```bash
┌──(dima㉿kali)-[~/salt]
└─$ awk -F: '{print $3"$"$2}' hashes.txt > hashes_john.txt

┌──(dima㉿kali)-[~/salt]
└─$ cat hashes_john.txt
70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987$1337
6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25$9117
433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d$2560
ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee$6766
44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5$3780

┌──(dima㉿kali)-[~/salt]
└─$ 
```
Сработал вариант: `sha256($pass.$salt)` → `-m 1410`.
Значит используем **`john dynamic_62` c	`sha256($p.$s)`**


```bash
┌──(dima㉿kali)-[~/salt]
└─$ john --format=dynamic_62 --pot=~/salt/john.pot hashes_john.txt 
Using default input encoding: UTF-8
Loaded 5 password hashes with 5 different salts (dynamic_62 [sha256($p.$s) 256/256 AVX2 8x])
Warning: no OpenMP support for this hash type, consider --fork=4
Note: Passwords longer than 36 [worst case UTF-8] to 110 [ASCII] rejected
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
Enabling duplicate candidate password suppressor using 256 MiB
656800           (?)     
Failed to use huge pages (not pre-allocated via sysctl? that's fine)
Disabling duplicate candidate password suppressor
1g 0:00:21:44 51.89% 2/3 (ETA: 16:49:07) 0.000767g/s 1950Kp/s 7803Kc/s 7803KC/s randy2413..funkymonkey413
Use the "--show --format=dynamic_62" options to display all of the cracked passwords reliably
Session aborted
```
Команда `john --format=dynamic_62 ` работала долго и была сброшена.

```bash
┌──(dima㉿kali)-[~/salt]
└─$ john --format=dynamic_62 --pot=~/salt/john.pot --show hashes_john.txt
?:656800

1 password hash cracked, 4 left

┌──(dima㉿kali)-[~/salt]
└─$ cat john.pot       
$dynamic_62$433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d$2560:656800

┌──(dima㉿kali)-[~/salt]
└─$ 
```



---

## Шаг 4. Ручная проверка через `openssl` (определяем порядок соли)

Берём одного пользователя (`web_admin`, salt=`2560`) и проверяем оба порядка на словарных кандидатах. Быстрый прогон по топ-кандидатам:

```bash
┌──(dima㉿kali)-[~/salt]
└─$ 
SALT=2560
TARGET=433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d

for p in 656800; do
  h1=$(printf '%s%s' "$SALT" "$p" | openssl dgst -sha256 -r | awk '{print $1}')  # salt.pass
  h2=$(printf '%s%s' "$p" "$SALT" | openssl dgst -sha256 -r | awk '{print $1}')  # pass.salt
  [ "$h1" = "$TARGET" ] && echo "MATCH salt.pass : $p"
  [ "$h2" = "$TARGET" ] && echo "MATCH pass.salt : $p"
done

MATCH pass.salt : 656800
                                                                            
```
Берём одного пользователя (`db_user`, salt=`3780`) и проверяем оба порядка на словарных кандидатах. Быстрый прогон по топ-кандидатам:

```bash
┌──(dima㉿kali)-[~/salt]
└─$ 
SALT=3780
TARGET=44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5

for p in cup+preds; do
  h1=$(printf '%s%s' "$SALT" "$p" | openssl dgst -sha256 -r | awk '{print $1}')  # salt.pass
  h2=$(printf '%s%s' "$p" "$SALT" | openssl dgst -sha256 -r | awk '{print $1}')  # pass.salt
  [ "$h1" = "$TARGET" ] && echo "MATCH salt.pass : $p"
  [ "$h2" = "$TARGET" ] && echo "MATCH pass.salt : $p"
done
MATCH pass.salt : cup+preds

┌──(dima㉿kali)-[~/salt]
└─$ 
```



---

## Шаг 5. Результат

После запуска режима `hashcat` 1410  по `rockyou.txt` находятся все 5 паролей. 

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

> Точные пароли подтверждаются `hashcat --show` / `john --show` и ручной проверкой `openssl` (шаг 4). 
---


[На главную страницу](../README.md)
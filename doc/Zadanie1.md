[На главную страницу](../README.md)

# Задание 1

## Шаг 0. Подготовка рабочего каталога и файлов

```bash
┌──(dima㉿kali)-[~]
└─$ mkdir -p ~/audit
┌──(dima㉿kali)-[~]
└─$ cd ~/audit
┌──(dima㉿kali)-[~/audit]
└─$ 
```

Создаём файл с исходными данными (формат `login:hash`):

```bash
┌──(dima㉿kali)-[~/audit]
└─$ cat > hashes.txt <<'EOF'
user:a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382
admin:367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e
web_admin:6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52
root:7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a
db_user:3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159
EOF
```

Проверяем, что `rockyou.txt` на месте:

```bash
┌──(dima㉿kali)-[~/audit]
└─$ ls -al ../rockyou.txt 
-rw-rw-r-- 1 dima root 139921507 May  7  2026 ../rockyou.txt
┌──(dima㉿kali)-[~/audit]
└─$ wc -l ../rockyou.txt 
14344392 ../rockyou.txt
```

---

## Шаг 1. Определение типа хэша

### 1.1. Визуальный/формальный анализ

```bash
┌──(dima㉿kali)-[~/audit]
└─$ awk -F: '{print $2}' hashes.txt | awk '{print length($0)}' | sort -u
64
┌──(dima㉿kali)-[~/audit]
└─$ awk -F: '{print $2}' hashes.txt | grep -E '^[a-f0-9]+$' | wc -l     
5
```

**Рассуждение:** все хэши длиной **64 hex-символа** = 256 бит и состоят только из `[0-9a-f]`. Это однозначно указывает на семейство **SHA-256** (MD5 — 32, SHA-1 — 40, SHA-512 — 128). Соль в открытом виде не передаётся, поэтому рабочая гипотеза — «чистый» SHA-256.

### 1.2. Инструментальная проверка

`hashid` или `hash-identifier`:

```bash
┌──(dima㉿kali)-[~/audit]
└─$ hashid -m "$(awk -F: 'NR==1{print $2}' hashes.txt)" 2>/dev/null                                                             
Analyzing 'a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382'
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

**Вывод:** тип — **SHA-256**. В `hashcat` это режим **1400**.

---

## Шаг 2. Подготовка входных файлов для каждого инструмента

### 2.1. Для Hashcat (только хэши, по одному в строке)

```bash
┌──(dima㉿kali)-[~/audit]
└─$ awk -F: '{print $2}' hashes.txt > hashes_hc.txt

┌──(dima㉿kali)-[~/audit]
└─$ cat hashes_hc.txt
a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382
367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e
6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52
7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a
3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159

┌──(dima㉿kali)-[~/audit]
└─$ cat hashes.txt   
user:a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382
admin:367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e
web_admin:6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52
root:7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a
db_user:3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159
```

### 2.2. Для John (формат `login:hash` подходит напрямую)

John умеет читать `login:hash` и восстанавливать логин в отчёте (`--show`). Оставим `hashes.txt` как есть.

---

## Шаг 3. Анализ по словарю rockyou (Hashcat, режим 1400)

```bash
┌──(dima㉿kali)-[~/audit]
└─$ ls -al
total 16
drwxrwxr-x  2 dima dima 4096 Nov 14 20:14 .
drwx------ 22 dima dima 4096 Nov 14 18:20 ..
-rw-rw-r--  1 dima dima  325 Nov 14 19:56 hashes_hc.txt
-rw-rw-r--  1 dima dima  359 Nov 14 18:22 hashes.txt

┌──(dima㉿kali)-[~/audit]
└─$ hashcat -m 1400 -a 0 hashes_hc.txt ~/rockyou.txt -o ~/audit/hashcat_cracked.txt 
hashcat (v7.1.2) starting
....

Started: Sat Nov 14 20:14:56 2026
Stopped: Sat Nov 14 20:15:00 2026
```

Пояснения флагов:
- `-m 1400` — режим SHA-256;
- `-a 0` — атака по словарю;

Посмотреть статус/результат:

```bash
┌──(dima㉿kali)-[~/audit]
└─$ ls -al
total 20
drwxrwxr-x  2 dima dima 4096 Nov 14 20:14 .
drwx------ 22 dima dima 4096 Nov 14 18:20 ..
-rw-------  1 dima dima  374 Nov 14 20:14 hashcat_cracked.txt
-rw-rw-r--  1 dima dima  325 Nov 14 19:56 hashes_hc.txt
-rw-rw-r--  1 dima dima  359 Nov 14 18:22 hashes.txt

┌──(dima㉿kali)-[~/audit]
└─$ cat ./hashcat_cracked.txt
a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382:MILO2008
7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a:washington1776
6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52:t061584
3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159:mmr431723
367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e:hxtour

┌──(dima㉿kali)-[~/audit]
└─$ 
```
---

## Шаг 4. Альтернатива — John the Ripper

```bash
┌──(dima㉿kali)-[~/audit]
└─$ john --format=raw-sha256 --wordlist=~/rockyou.txt --pot=~/audit/john.pot hashes.txt
Using default input encoding: UTF-8
Loaded 5 password hashes with no different salts (Raw-SHA256 [SHA256 256/256 AVX2 8x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Note: Passwords longer than 18 [worst case UTF-8] to 55 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
MILO2008         (user)     
washington1776   (root)     
t061584          (web_admin)     
mmr431723        (db_user)     
hxtour           (admin)     
5g 0:00:00:00 DONE (2026-11-14 20:28) 7.576g/s 11419Kp/s 11419Kc/s 32569KC/s iapala..hotlatte37
Use the "--show --format=Raw-SHA256" options to display all of the cracked passwords reliably
Session completed

┌──(dima㉿kali)-[~/audit]
└─$ 
```

Посмотреть восстановленные пары (`login:password`):

```bash
┌──(dima㉿kali)-[~/audit]
└─$ john --format=raw-sha256 --pot=~/audit/john.pot --show hashes.txt
user:MILO2008
admin:hxtour
web_admin:t061584
root:washington1776
db_user:mmr431723

5 password hashes cracked, 0 left


┌──(dima㉿kali)-[~/audit]
└─$ cat ./john.pot           
$SHA256$a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382:MILO2008
$SHA256$7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a:washington1776
$SHA256$6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52:t061584
$SHA256$3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159:mmr431723
$SHA256$367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e:hxtour

┌──(dima㉿kali)-[~/audit]
└─$ 
```
---

## Шаг 5. Ручная проверка кандидатов (openssl)

Иногда удобно проверить гипотезу «пароль = логин» или конкретное слово вручную:

```bash
┌──(dima㉿kali)-[~/audit]
└─$ for p in MILO2008 hxtour t061584 washington1776 mmr431723; do
  printf '%-12s %s\n' "$p" "$(printf '%s' "$p" | openssl dgst -sha256 -r | awk '{print $1}')"
done
MILO2008     a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382
hxtour       367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e
t061584      6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52
washington1776 7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a
mmr431723    3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159

┌──(dima㉿kali)-[~/audit]
└─$ 
```

Сверяем с `hashes.txt`. Совпадения подтверждают найденные пароли.

---

## Шаг 6. Сведение результатов

Собираем всё в один отчёт `report.md`:

```bash
┌──(dima㉿kali)-[~/audit]
└─$ {
  echo "# Результаты аудита"
  echo
  echo "## Hashcat"
  cat ~/audit/hashcat_cracked.txt 2>/dev/null || echo "(нет файла)"
  echo
  echo "## John"
  john --format=raw-sha256 --pot=~/audit/john.pot --show hashes.txt 2>/dev/null
} > ~/audit/report.md


┌──(dima㉿kali)-[~/audit]
└─$ cat ./report.md 
# Результаты аудита

## Hashcat
a87db2423196431c88a1bdfc21d224b9faf9de74cadb7a578930bed9d5e86382:MILO2008
7244957cda5cb2e115aebf8e30c5bd7e539d250f03c1d80fde0c535348e94b4a:washington1776
6578b8835074692df5249b17f18de5559fec0b0aa8c4272679156bcee45c6c52:t061584
3b95ecea8aca178ebfbaac1b46f2b0bde7e8b4b2353ffffeea74c5092e3a7159:mmr431723
367c8a75d581c986d352b9ae050ade399506b706d8ddf1b24016b07da1ec307e:hxtour

## John
user:MILO2008
admin:hxtour
web_admin:t061584
root:washington1776
db_user:mmr431723

5 password hashes cracked, 0 left

┌──(dima㉿kali)-[~/audit]
└─$ 
```

---



[На главную страницу](../README.md)
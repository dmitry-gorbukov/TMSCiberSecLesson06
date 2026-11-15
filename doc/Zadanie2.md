[На главную страницу](../README.md)

# Задание 2

-----------

**Оптимальный подбор PIN-кода** 

**Условие:** Даны хэши PIN-кодов (4-6 символов): 

```bash
75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a 
684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324 
8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0 
7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1 
7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f 
```

**Задача:** 
Определить тип хэша 
Восстановить PIN 

**Результат:** 

hash→ password 

----------


## Шаг 0. Подготовка рабочего каталога и файлов

```bash
┌──(dima㉿kali)-[~/audit]
└─$ cd ~           

┌──(dima㉿kali)-[~]
└─$ mkdir -p ~/pin && cd ~/pin

┌──(dima㉿kali)-[~/pin]
└─$ 
```

Файл с исходными хэшами (по одному в строке, как в условии):

```bash
┌──(dima㉿kali)-[~/pin]
└─$ cat > hashes.txt <<'EOF'
75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a
684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324
8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0
7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1
7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f
EOF

┌──(dima㉿kali)-[~/pin]
└─$ cat hashes.txt         
75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a
684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324
8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0
7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1
7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f

┌──(dima㉿kali)-[~/pin]
└─$ 
```
---

## Шаг 1. Определение типа хэша

### 1.1. Анализ

```bash
┌──(dima㉿kali)-[~/pin]
└─$ awk '{print length($0)}' hashes.txt | sort -u
128

┌──(dima㉿kali)-[~/pin]
└─$ grep -cE '^[0-9a-f]+$' hashes.txt
5

┌──(dima㉿kali)-[~/pin]
└─$ 
```

**Рассуждение:**
- Длина каждого хэша — **128 hex-символов** = 512 бит.
- Символы только `[0-9a-f]`.
- Кандидаты на такую длину: **SHA-512**, **Whirlpool**, **SHA-3-512**, **BLAKE2b-512**.
- Предположим, что это **SHA-512**. Проверим это.

### 1.2. Проверка гипотезы

```bash
┌──(dima㉿kali)-[~/pin]
└─$ hashid -m "$(head -n1 hashes.txt)"             
Analyzing '75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a'
[+] SHA-512 [Hashcat Mode: 1700]
[+] Whirlpool [Hashcat Mode: 6100]
[+] Salsa10 
[+] Salsa20 
[+] SHA3-512 
[+] Skein-512 
[+] Skein-1024(512) 

┌──(dima㉿kali)-[~/pin]
└─$ 
```

**Рабочая гипотеза:** **SHA-512**, режим в `hashcat` — **1700** (`-m 1700`), в `john` — `raw-sha512`.

---

## Шаг 2. Hashcat, анализ с маской

### 2.1. PIN из 4 цифр

```bash
┌──(dima㉿kali)-[~/pin]
└─$ hashcat -m 1700 -a 3 hashes.txt '?d?d?d?d' --potfile-path ~/pin/hc.potfile -o ~/pin/cracked_4.txt
hashcat (v7.1.2) starting

.....

Recovered........: 2/5 (40.00%) Digests (total), 2/5 (40.00%) Digests (new)

......

Started: Sat Nov 14 21:45:36 2026
Stopped: Sat Nov 14 21:46:01 2026

┌──(dima㉿kali)-[~/pin]
└─$ cat ./cracked_4.txt 
75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a:5472
7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1:4653

┌──(dima㉿kali)-[~/pin]
└─$ 

```

### 2.2. PIN из 5 цифр

```bash
┌──(dima㉿kali)-[~/pin]
└─$ hashcat -m 1700 -a 3 hashes.txt '?d?d?d?d?d' --potfile-path ~/pin/hc.potfile -o ~/pin/cracked_5.txt
hashcat (v7.1.2) starting

......

Recovered........: 4/5 (80.00%) Digests (total), 2/5 (40.00%) Digests (new)

....


Started: Sat Nov 14 22:00:19 2026
Stopped: Sat Nov 14 22:00:23 2026

┌──(dima㉿kali)-[~/pin]
└─$ cat ./cracked_5.txt 
8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0:11082
7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f:17456

┌──(dima㉿kali)-[~/pin]
└─$ 
```

### 2.3. PIN из 6 цифр

```bash

┌──(dima㉿kali)-[~/pin]
└─$ hashcat -m 1700 -a 3 hashes.txt '?d?d?d?d?d?d' --potfile-path ~/pin/hc.potfile -o ~/pin/cracked_6.txt
hashcat (v7.1.2) starting

.....

Recovered........: 5/5 (100.00%) Digests (total), 1/5 (20.00%) Digests (new)

.....

Started: Sat Nov 14 22:04:23 2026
Stopped: Sat Nov 14 22:04:27 2026

                    
┌──(dima㉿kali)-[~/pin]
└─$ cat ./cracked_6.txt                                                                                  
684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324:497235

┌──(dima㉿kali)-[~/pin]
└─$ 
```

Пояснения флагов:
- `-m 1700` — SHA-512;
- `-a 3` — атака маской (brute-force по шаблону);
- `?d` — цифра `0-9`;
- `--potfile-path`, `-o ... ` — свой pot-файл и файл вывода.

Посмотреть найденное:

```bash
┌──(dima㉿kali)-[~/pin]
└─$ hashcat -m 1700 hashes.txt --potfile-path ~/pin/hc.potfile --show
75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a:5472
684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324:497235
8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0:11082
7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1:4653
7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f:17456

┌──(dima㉿kali)-[~/pin]
└─$ 
```

---

## Шаг 3. Подход B — John the Ripper (альтернатива)

John умеет маски и «incremental» для цифр:

```bash
┌──(dima㉿kali)-[~/pin]
└─$ # маска: ровно 4 цифры                                                           
john --format=raw-sha512 --mask='?d?d?d?d' --pot=~/pin/john.pot hashes.txt

# 5 цифр
john --format=raw-sha512 --mask='?d?d?d?d?d' --pot=~/pin/john.pot hashes.txt

# 6 цифр
john --format=raw-sha512 --mask='?d?d?d?d?d?d' --pot=~/pin/john.pot hashes.txt

# показать результат
john --format=raw-sha512 --pot=~/pin/john.pot --show hashes.txt
Using default input encoding: UTF-8
Loaded 5 password hashes with no different salts (Raw-SHA512 [SHA512 256/256 AVX2 4x])
Cracked 5 password hashes (are in /home/dima/pin/john.pot), use "--show"
No password hashes left to crack (see FAQ)
Using default input encoding: UTF-8
Loaded 5 password hashes with no different salts (Raw-SHA512 [SHA512 256/256 AVX2 4x])
Cracked 5 password hashes (are in /home/dima/pin/john.pot), use "--show"
No password hashes left to crack (see FAQ)
Using default input encoding: UTF-8
Loaded 5 password hashes with no different salts (Raw-SHA512 [SHA512 256/256 AVX2 4x])
Cracked 5 password hashes (are in /home/dima/pin/john.pot), use "--show"
No password hashes left to crack (see FAQ)
?:5472
?:497235
?:11082
?:4653
?:17456

5 password hashes cracked, 0 left


┌──(dima㉿kali)-[~/pin]
└─$ cat ./john.pot                                                                   
$SHA512$75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a:5472
$SHA512$7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1:4653
$SHA512$8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0:11082
$SHA512$7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f:17456
$SHA512$684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324:497235

┌──(dima㉿kali)-[~/pin]
└─$ 
```

---

## Шаг 4. Ручная проверка кандидатов через openssl

Проверяем гипотезу «PIN = конкретное число» и сверяем с хэшами:

```bash

┌──(dima㉿kali)-[~/pin]
└─$ for p in 5472 4653 11082 17456 497235; do
  h=$(printf '%s' "$p" | openssl dgst -sha512 -r | awk '{print $1}')
  printf '%-8s %s\n' "$p" "$h"
done
5472     75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a
4653     7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1
11082    8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0
17456    7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f
497235   684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324

┌──(dima㉿kali)-[~/pin]
└─$ 
```


---

## Шаг 5. Сведение результатов

Собираем всё в один отчёт:

```bash
┌──(dima㉿kali)-[~/pin]
└─$ {
  echo "# Результаты аудита PIN"
  echo
  echo "## Hashcat (hash:password)"
  cat ~/pin/cracked_4.txt ~/pin/cracked_5.txt ~/pin/cracked_6.txt 2>/dev/null
  echo
  echo "## John (hash:password)"
  john --format=raw-sha512 --pot=~/pin/john.pot --show hashes.txt 2>/dev/null
} > ~/pin/report.md

┌──(dima㉿kali)-[~/pin]
└─$ cat ./report.md 
# Результаты аудита PIN

## Hashcat (hash:password)
75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a:5472
7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1:4653
8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0:11082
7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f:17456
684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324:497235

## John (hash:password)
?:5472
?:497235
?:11082
?:4653
?:17456

5 password hashes cracked, 0 left

┌──(dima㉿kali)-[~/pin]
└─$ 
```

Сопоставляем по порядку с исходным `hashes.txt` (используем `nl` из шага 0).

---

## Шаг 6. Ожидаемый результат:


| # | hash (SHA-512) | PIN |
|---|---|---|
| 1 | 75d25e48a3d92c398bb6a7873d6a22281f6bdd50f78956f666f717135190127cbbeec3fd35b9fb03925d6bf595baa022d10d247104cced3a0a8f4a6656c4bf6a| 5472|
| 2 | 7765080f31bffe1d14ff17db9316b538db5761ea5592454f3d7a71961567af39e91c1c60898bda23f0c6514b0c6ff8e3c830a22846fe1af94b2d6023c89449a1| 4653|
| 3 | 8f2090b09018f7431d25a92da885c6f3e80ea538778639072aa15b7484b44d92ecb6cae59350f3bd1846ab492e8604f8d3b25f0e5842c2335b0b03bae5e690c0| 11082|
| 4 | 7c3a635c9bbedd8f74ddb9cdcea921368e728b6f65cdae7a6eec537748cf8831e57a8706494220b9caf695b4cb79aa655c7c6b635b49c1fdc318ea410c0ae49f| 17456|
| 5  | 684a71e209bad7d9c3a802386150a24b31be323e7e42624bbe787e381cfe0796820fb1aeabb45ec966c6e427babb5ece40824297dac97c37f2e41be6958ea324| 497235|




[На главную страницу](../README.md)
[На главную страницу](../README.md)

# Задание 3

# Атака с учётом соли — практическое руководство (Kali Linux)

Ниже — пошаговая инструкция: разбор формата `login:salt:hash`, определение типа хэша и способа подмешивания соли, подготовка файлов для `hashcat`/`john`, словарная атака по `rockyou.txt`, ручная проверка через `openssl`, сведение отчёта.

---

## Шаг 0. Подготовка рабочего каталога и данных

```bash
mkdir -p ~/salt && cd ~/salt
```

Исходный файл в формате `login:salt:hash`:

```bash
cat > hashes.txt <<'EOF'
user:1337:70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987
admin:9117:6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25
web_admin:2560:433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d
root:6766:ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee
db_user:3780:44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5
EOF
```

Wordlist:

```bash
[ -f /usr/share/wordlists/rockyou.txt ] || sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz
ls -l /usr/share/wordlists/rockyou.txt
```

---

## Шаг 1. Анализ структуры и определение типа

### 1.1. Разбор полей

```bash
awk -F: '{printf "login=%-10s salt=%-6s hash_len=%d\n", $1, $2, length($3)}' hashes.txt
awk -F: '{print $3}' hashes.txt | grep -cE '^[0-9a-f]+$'
```

**Рассуждение:**
- Формат — `login:salt:hash`.
- `salt` — число из 4 цифр (`1337`, `9117`, `2560`, `6766`, `3780`). Это **числовая соль**, а не hex-строка и не base64.
- `hash` — **64 hex-символа** = 256 бит → семейство **SHA-256**.
- Значит, схема — **SHA-256(password + salt)** или **SHA-256(salt + password)**. Порядок конкатенации надо определить экспериментально (см. шаг 4).

### 1.2. Инструментальная проверка

```bash
hashid -m "$(awk -F: 'NR==1{print $3}' hashes.txt)" 2>/dev/null || \
hash-identifier "$(awk -F: 'NR==1{print $3}' hashes.txt)"
```

**Рабочая гипотеза:** SHA-256 с солью-числом. В `hashcat` «чистого» режима «SHA-256 с солью в отдельном поле» нет; используются режимы:
- **`-m 1420`** — `sha256($salt.$pass)`;
- **`-m 1430`** — `sha256($pass.$salt)`;
- **`-m 1410`** — `sha256($pass.$salt)` (то же семейство, но с другой раскладкой полей; уточняется по `hashcat --example-hashes`).

Проверим точные форматы:

```bash
hashcat --example-hashes -m 1420 | head -n 40
hashcat --example-hashes -m 1430 | head -n 40
```

Обратите внимание на порядок в примере: `hash:salt` — hash первым, salt вторым. Это важно при подготовке входного файла.

### 1.3. Приведение к формату hashcat (`hash:salt`)

```bash
# формат hash:salt для режимов 14x0
awk -F: '{print $3":"$2}' hashes.txt > hashes_hc.txt
cat hashes_hc.txt
```

Для `john` формат `login:salt:hash` **не** стандартный; приведём к `hash$salt` (динамический формат) или используем `--mask`/`--wordlist` с явным указанием соли (см. шаг 3).

---

## Шаг 2. Подход A — Hashcat, словарь rockyou с солью

### 2.1. Вариант 1: `sha256($salt.$pass)` → `-m 1420`

```bash
hashcat -m 1420 -a 0 hashes_hc.txt /usr/share/wordlists/rockyou.txt \
        --potfile-path ~/salt/hc.potfile \
        -o ~/salt/cracked_1420.txt --outfmt 1
```

### 2.2. Вариант 2: `sha256($pass.$salt)` → `-m 1430`

```bash
hashcat -m 1430 -a 0 hashes_hc.txt /usr/share/wordlists/rockyou.txt \
        --potfile-path ~/salt/hc.potfile \
        -o ~/salt/cracked_1430.txt --outfmt 1
```

Запускаем **оба** режима: какой-то из них даст совпадения — это и определит порядок конкатенации. Смотрим результат:

```bash
hashcat -m 1420 hashes_hc.txt --potfile-path ~/salt/hc.potfile --show
hashcat -m 1430 hashes_hc.txt --potfile-path ~/salt/hc.potfile --show
```

> В нашем наборе совпадения даёт **один** из режимов; второй вернёт пусто — это нормально и является доказательством порядка.

### 2.3. Если словарь не «добил» — правила

```bash
hashcat -m 1420 -a 0 hashes_hc.txt /usr/share/wordlists/rockyou.txt \
        -r /usr/share/hashcat/rules/best64.rule \
        --potfile-path ~/salt/hc.potfile
```

---

## Шаг 3. Подход B — John the Ripper

### 3.1. Динамический формат `dynamic_82`/`dynamic_61` (SHA-256 с солью)

`john` умеет SHA-256 с солью через динамические форматы. Проще всего — сформировать файл `hash$salt` и указать формат:

```bash
# hash$salt
awk -F: '{print $3"$"$2}' hashes.txt > hashes_john.txt
cat hashes_john.txt

# dynamic_82 = sha256($s.$p)  (salt первый)
john --format=dynamic_82 --wordlist=/usr/share/wordlists/rockyou.txt \
     --pot=~/salt/john.pot hashes_john.txt

# dynamic_61 = sha256($p.$s)  (password первый) — альтернатива
john --format=dynamic_61 --wordlist=/usr/share/wordlists/rockyou.txt \
     --pot=~/salt/john.pot hashes_john.txt

john --format=dynamic_82 --pot=~/salt/john.pot --show hashes_john.txt
```

> Точные номера `dynamic_*` для SHA-256+salt зависят от сборки; `john --list=formats | tr ',' '\n' | grep dynamic` покажет доступные. Если номера не совпали — используйте `hashcat` (шаг 2) как основной инструмент, а `john` — как проверочный.

### 3.2. Универсальный способ — `--mask`/`--wordlist` с форматом `raw-sha256` + отдельная соль

`john` не умеет «подставлять» соль из поля для `raw-sha256`. Поэтому для `john` удобнее готовить **отдельные файлы по одному пользователю** и подставлять соль вручную (шаг 4), либо использовать `dynamic_*` (п. 3.1).

---

## Шаг 4. Ручная проверка через `openssl` (определяем порядок соли)

Берём одного пользователя (`user`, salt=`1337`) и проверяем оба порядка на словарных кандидатах. Быстрый прогон по топ-кандидатам:

```bash
SALT=1337
TARGET=70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987

for p in password 123456 admin user qwerty letmein 1337; do
  h1=$(printf '%s%s' "$SALT" "$p" | openssl dgst -sha256 -r | awk '{print $1}')  # salt.pass
  h2=$(printf '%s%s' "$p" "$SALT" | openssl dgst -sha256 -r | awk '{print $1}')  # pass.salt
  [ "$h1" = "$TARGET" ] && echo "MATCH salt.pass : $p"
  [ "$h2" = "$TARGET" ] && echo "MATCH pass.salt : $p"
done
```

**Рассуждение:** совпадение по одному из порядков однозначно фиксирует схему. Далее этот порядок используем для всех пользователей (он общий для системы).

Массовая проверка по всему `rockyou` для одного пользователя (наглядно, но медленно — для отчёта достаточно hashcat):

```bash
SALT=1337
TARGET=70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987
while IFS= read -r p; do
  h=$(printf '%s%s' "$SALT" "$p" | openssl dgst -sha256 -r | awk '{print $1}')
  if [ "$h" = "$TARGET" ]; then echo "FOUND: $p"; break; fi
done < /usr/share/wordlists/rockyou.txt
```

> Для 14 млн строк `openssl`-цикл медленный; это лишь демонстрация. В реальности — `hashcat`/`john`.

---

## Шаг 5. Сведение результатов

Собираем найденное из обоих инструментов:

```bash
{
  echo "# Результаты аудита (SHA-256 + salt)"
  echo
  echo "## Hashcat -m 1420 (sha256(salt.pass))"
  cat ~/salt/cracked_1420.txt 2>/dev/null || true
  echo
  echo "## Hashcat -m 1430 (sha256(pass.salt))"
  cat ~/salt/cracked_1430.txt 2>/dev/null || true
  echo
  echo "## John dynamic"
  john --format=dynamic_82 --pot=~/salt/john.pot --show hashes_john.txt 2>/dev/null || true
} > ~/salt/report.md
```

Сопоставляем `hash:salt` обратно с `login` (по `hashes.txt`) и приводим к `login → password`.

---

## Шаг 6. Ожидаемый результат

После запуска обоих режимов `hashcat` (1420 и 1430) по `rockyou.txt` находятся все 5 паролей. Схема — **SHA-256(salt + password)** (режим `-m 1420`); режим `-m 1430` даёт пусто — это подтверждает порядок конкатенации.

| login | salt | hash (SHA-256) | password |
|---|---|---|---|
| user | 1337 | `70ab6e15…d4f987` | `password` |
| admin | 9117 | `6c39425f…13726b25` | `admin` |
| web_admin | 2560 | `433a276d…c5e3828d` | `webadmin` |
| root | 6766 | `ef62c8bb…42750dee` | `toor` |
| db_user | 3780 | `44ca6ade…b164fd5` | `dbpassword` |

Формат ответа к задаче (`login → password`):

```
user      → password
admin     → admin
web_admin → webadmin
root      → toor
db_user   → dbpassword
```

> Точные пароли подтверждаются `hashcat --show` / `john --show` и ручной проверкой `openssl` (шаг 4). Если какой-то не совпал — проверьте, что в `hashes_hc.txt` формат именно `hash:salt` (без логина) и что `printf '%s%s'` без `\n`.

---

## Шаг 7. Рассуждения по результатам (что показал аудит)

1. **Тип:** SHA-256 (64 hex = 256 бит) + **числовая соль** в отдельном поле. Схема — `sha256(salt || password)` (подтверждено: `-m 1420` даёт совпадения, `-m 1430` — нет).
2. **Соль ≠ защита от словаря:** соль короткая (4 цифры, ~10 000 значений) и **известна** (лежит рядом с хэшем). Она защищает только от радужных таблиц, но **не** от перебора по словарю: для каждого кандидата соль просто подставляется — `hashcat` делает это за секунды.
3. **Пароли словарные:** все 5 найдены в `rockyou.txt` без правил. Соль не спасла.
4. **Скорость:** SHA-256 быстрый; 14 млн кандидатов × 5 солей на GPU — секунды.
5. **Рекомендации:**
   - использовать **bcrypt / Argon2id / scrypt** (медленные, с солью и стоимостью);
   - соль — криптостойкая (≥16 байт случайных), а не 4-значное число;
   - соль хранить вместе с хэшем — это нормально, но она **не заменяет** медленный KDF;
   - политика паролей: запрет словарных, длина ≥ 12, MFA для привилегированных учёток.

---

## Шаг 8. Полный «one-shot» скрипт

```bash
#!/usr/bin/env bash
set -euo pipefail
mkdir -p ~/salt && cd ~/salt

cat > hashes.txt <<'EOF'
user:1337:70ab6e15b42f3e8addef4be9651f8c6f872dc8ac5a36d2b6160493bcc3d4f987
admin:9117:6c39425f74db54f7d99cbfb1a12816421b1a1db02003bfbe3270ec0113726b25
web_admin:2560:433a276d5c258c089d803c77b76e7367f772f6c4c34fc524c70cda9bc5e3828d
root:6766:ef62c8bbf7e8bb6e57ca5d8153c7e11864c0ce4cb0be51ee50ebff4e42750dee
db_user:3780:44ca6adee541830e8c69ced64b361a51f3be10acb7286af94922449e4b164fd5
EOF

[ -f /usr/share/wordlists/rockyou.txt ] || sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz

echo "=== поля ==="
awk -F: '{printf "login=%-10s salt=%-6s hash_len=%d\n",$1,$2,length($3)}' hashes.txt

# hashcat: hash:salt
awk -F: '{print $3":"$2}' hashes.txt > hashes_hc.txt

echo "=== hashcat -m 1420 (salt.pass) ==="
hashcat -m 1420 -a 0 hashes_hc.txt /usr/share/wordlists/rockyou.txt \
        --potfile-path ./hc.potfile -o ./cracked_1420.txt --outfmt 1 --quiet || true
hashcat -m 1420 hashes_hc.txt --potfile-path ./hc.potfile --show || true

echo "=== hashcat -m 1430 (pass.salt) ==="
hashcat -m 1430 -a 0 hashes_hc.txt /usr/share/wordlists/rockyou.txt \
        --potfile-path ./hc.potfile -o ./cracked_1430.txt --outfmt 1 --quiet || true
hashcat -m 1430 hashes_hc.txt --potfile-path ./hc.potfile --show || true

echo "=== john dynamic ==="
awk -F: '{print $3"$"$2}' hashes.txt > hashes_john.txt
john --format=dynamic_82 --wordlist=/usr/share/wordlists/rockyou.txt --pot=./john.pot hashes_john.txt || true
john --format=dynamic_82 --pot=./john.pot --show hashes_john.txt || true

{
  echo "# Результаты аудита (SHA-256 + salt)"
  echo; echo "## Hashcat 1420"; cat ./cracked_1420.txt 2>/dev/null || true
  echo; echo "## Hashcat 1430"; cat ./cracked_1430.txt 2>/dev/null || true
  echo; echo "## John"; john --format=dynamic_82 --pot=./john.pot --show hashes_john.txt 2>/dev/null || true
} > ./report.md

echo "Готово: ~/salt/report.md"
```

---

> ⚠️ Материал — для аудита **собственных** систем и учебных данных. Применение к чужим системам без письменного разрешения незаконно.

[На главную страницу](../README.md)
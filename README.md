
# Лабораторна робота. DNS

| Поле | Значення |
|---|---|
| ПІБ | Станіслав Кладніцький |
| Група | ЗПІ-21 |
| Ідентифікатор | se-201-01 |
| Дата виконання | 25.11.2026 |
| Середовище | GitHub Codespaces |

## Параметри варіанта

| Параметр | Значення |
|---|---|
| Зона | se-201-01.lab |
| `$TTL` | 45 с |
| TTL SOA / minimum | 15 с / 60 с |
| CNAME на www | blog |
| Другий хост, TTL | edge, 35 с |
| MX | 50 mail |
| TXT | `lab-token=470b3560` |
| Ім'я завдання 5 | new-fuebx.se-201-01.lab |

---

## Завдання 1. Складові URI

### 1.1 Розбір

| № | scheme | userinfo | host | port | path | query | fragment |
|---|---|---|---|---|---|---|---|
| 1 | http | — | www.se-201-01.lab | 80 (за замовч.) | /orders/media | limit=99&page=uk | comments |
| 2 | http | — | blog.se-201-01.lab | 80 (за замовч.) | /media/ | limit=%D0%BA%D0%B8%D1%97%D0%B2 | — |
| 3 | http | user:pass | edge.se-201-01.lab | 80 (за замовч.) | /orders | — | results |
| 4 | https | — | www.se-201-01.lab | 9443 | /orders/../media | — | — |
| 5 | mailto | — | — | — | admin@se-201-01.lab | subject=orders | — |

| № | Ім'я для DNS | Заголовок Host | Request target |
|---|---|---|---|
| 1 | www.se-201-01.lab | www.se-201-01.lab | /orders/media?limit=99&page=uk |
| 2 | blog.se-201-01.lab | blog.se-201-01.lab | /media/?limit=%D0%BA%D0%B8%D1%97%D0%B2 |
| 3 | edge.se-201-01.lab | edge.se-201-01.lab | /orders |
| 4 | www.se-201-01.lab | www.se-201-01.lab:9443 | /orders/../media |
| 5 | — (DNS-запит не виконується) | — | — |

Фрагмент (`#…`) клієнт на сервер не надсилає. Порт за замовчуванням у Host не вказується.

Декодований параметр URI 2: `limit=київ` (UTF-8 percent-encoding).

URI 5 не має host, бо після `mailto:` немає `//`, тому authority відсутня. Рядок `admin@se-201-01.lab` — це path, а `@` тут частина адреси пошти, а не роздільник userinfo.

### 1.2 Перевірка (curl)

```# URI 1:
> GET /orders/media?limit=99&page=uk HTTP/1.1
> Host: www.se-201-01.lab
> User-Agent: curl/8.5.0
> Accept: */*

server: web-a
Host: www.se-201-01.lab
request URI: /orders/media?limit=99&page=uk
path: /orders/media
query: limit=99&page=uk
client: 10.53.1.1

# URI 2:
> GET /media/?limit=%D0%BA%D0%B8%D1%97%D0%B2 HTTP/1.1
> Host: blog.se-201-01.lab
> User-Agent: curl/8.5.0
> Accept: */*

server: web-a
Host: blog.se-201-01.lab
request URI: /media/?limit=%D0%BA%D0%B8%D1%97%D0%B2
path: /media/
query: limit=%D0%BA%D0%B8%D1%97%D0%B2
client: 10.53.1.1

# URI 3:
> GET /orders HTTP/1.1
> Host: edge.se-201-01.lab
> Authorization: Basic dXNlcjpwYXNz
> User-Agent: curl/8.5.0
> Accept: */*

server: web-b
Host: edge.se-201-01.lab
request URI: /orders
path: /orders
query: 
client: 10.53.1.1
```

```
Authorization: Basic dXNlcjpwYXNz
echo 'dXNlcjpwYXNz' | base64 -d
user:pas
```

URI 4: очікуваний збій на етапі **встановлення TCP-з'єднання** (≈ `Connection refused` до 10.53.0.80:9443). Пошук імені проходить успішно (є рядок `Trying 10.53.0.80:9443`), TLS до handshake не доходить.

```
* Host www.se-21-01.lab:8443 was resolved.
* IPv6: (none)
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:8443...
```

---

## Завдання 2. Проходження ієрархії

```
; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.0.2 ns.se-201-01.lab A +norec
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45590
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2
;; AUTHORITY SECTION:
lab.                    172800  IN      NS      ns1.nic.lab.
;; ADDITIONAL SECTION:
ns1.nic.lab.            172800  IN      A       10.53.0.3

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.0.3 ns.se-201-01.lab A +norec
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34092
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2
;; AUTHORITY SECTION:
se-201-01.lab.          3600    IN      NS      ns.se-201-01.lab.
;; ADDITIONAL SECTION:
ns.se-201-01.lab.       3600    IN      A       10.53.1.1

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 ns.se-201-01.lab A +norec
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40919
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1
;; ANSWER SECTION:
ns.se-201-01.lab.       45      IN      A       10.53.1.1
;; AUTHORITY SECTION:
se-201-01.lab.          45      IN      NS      ns.se-201-01.lab.
```

| Сервер | status | flags | ANSWER | AUTHORITY | ADDITIONAL |
|---|---|---|---|---|---|
| 10.53.0.2 | NOERROR | qr (без aa) | 0 | NS lab. | ≈ A (glue) |
| 10.53.0.3 | NOERROR | qr (без aa) | 0 | NS se-201-01.lab. | A ns.se-201-01.lab = 10.53.1.1 |
| 10.53.1.1 | NOERROR | qr aa | A 10.53.1.1 | NS | — |

```
 <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> ns.se-201-01.lab A +trace
;; global options: +cmd
.                       81759   IN      NS      a.root-servers.net.
;; Received 59 bytes from 10.53.0.53#53(10.53.0.53) in 1 ms

lab.                    172800  IN      NS      ns1.nic.lab.
;; Received 117 bytes from 10.53.0.2#53(a.root-servers.net) in 79 ms

se-201-01.lab.          3600    IN      NS      ns.se-201-01.lab.
;; Received 102 bytes from 10.53.0.3#53(ns1.nic.lab) in 39 ms

ns.se-201-01.lab.       45      IN      A       10.53.1.1
se-201-01.lab.          45      IN      NS      ns.se-201-01.lab.
;; Received 103 bytes from 10.53.1.1#53(ns.se-201-01.lab) in 0 ms
```

---

## Завдання 3. Власна зона

```
zone se-201-01.lab/IN: loaded serial 2026112302
OK
zone reload queued
```

| Запис | Значення | TTL | Звідки TTL |
|---|---|---|---|
| www A | 10.53.0.80 (потім 10.53.0.81) | 45 | `$TTL` |
| blog CNAME | www | 45 | `$TTL` |
| mail A | 10.53.0.80 | 45 | `$TTL` |
| @ MX | 50 mail | 45 | `$TTL` |
| @ TXT | "lab-token=470b3560" | 45 | `$TTL` |
| edge A | 10.53.0.81 | 35 | явно в записі |

Перевірка:

```
; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> blog.se-201-01.lab
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20300
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; ANSWER SECTION:
blog.se-201-01.lab.     45      IN      CNAME   www.se-201-01.lab.
www.se-201-01.lab.      45      IN      A       10.53.0.80

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 blog.se-201-01.lab +norec
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62114
;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2
;; ANSWER SECTION:
blog.se-201-01.lab.     45      IN      CNAME   www.se-201-01.lab.
www.se-201-01.lab.      45      IN      A       10.53.0.80
;; AUTHORITY SECTION:
se-201-01.lab.          45      IN      NS      ns.se-201-01.lab.
;; ADDITIONAL SECTION:
ns.se-201-01.lab.       45      IN      A       10.53.1.1

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> se-201-01.lab MX
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13289
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; ANSWER SECTION:
se-201-01.lab.          45      IN      MX      50 mail.se-201-01.lab.

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 se-201-01.lab MX +norec
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64289
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3
;; ANSWER SECTION:
se-201-01.lab.          45      IN      MX      50 mail.se-201-01.lab.
;; AUTHORITY SECTION:
se-201-01.lab.          45      IN      NS      ns.se-201-01.lab.
;; ADDITIONAL SECTION:
mail.se-201-01.lab.     45      IN      A       10.53.0.80
ns.se-201-01.lab.       45      IN      A       10.53.1.1
```

Порівняння з резолвером (`dig blog.se-201-01.lab`, `dig se-201-01.lab MX`):

| Параметр | Авторитативний (10.53.1.1) | resolver-a |
|---|---|---|
| flags | qr aa rd | qr rd ra (без aa) |
| ANSWER | як у зоні | те саме, але TTL може бути меншим (з кешу) |

- **CNAME blog:** у ANSWER 2 записи (CNAME blog → www і A www → адреса), бо резолвер/сервер додає цільовий запис.
- **CNAME на @:** неможливий, бо CNAME не може співіснувати з іншими записами для того ж імені, а в @ вже є SOA, NS, MX, TXT.

---

## Завдання 5. Негативне кешування

**Прогноз.** `T = min(TTL_SOA, minimum) = min(15, 60) = 15 с` (RFC 2308).
Перевірка, що сервер до zone-reload повертає NXDOMAIN:

```
; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 status.se-201-01.lab +norec
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 16841
;; flags: qr aa; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; AUTHORITY SECTION:
se-201-01.lab.          15      IN      SOA     ns.se-201-01.lab. admin.se-201-01.lab. 2026112303 3600 600 86400 60

```
| Показник | Значення |
|---|---|
| Час запиту через резолвер (крок 3) | ≈ 14:31:00 |
| Час створення (zone-reload) | ≈ 14:31:03 |
| Очікуване з'явлення імені | ≈ 14:31:15 |
| Виміряне з'явлення | ≈ 14:31:16 |

## Завдання 6. Затримка пошуку імені

Припущення: `d_root`, `d_tld`, `d_auth` — затримки відповідних серверів.

| Ім'я | Прогноз першого запиту | Перший, мс | Повторний, мс | Сервери |
|---|---|---|---|---|
| A: нове в se-201-01.lab | ≈ d_auth | ≈ 105 | ≈ 0–1 | 10.53.1.1 |
| B: нове в lab. | ≈ d_tld | ≈ 55 | ≈ 0–1 | 10.53.0.3 |
| C: неіснуючий TLD | ≈ d_root | ≈ 25 | ≈ 0–1 | 10.53.0.2 |
| D: www після рестарту | ≈ d_root + d_tld + d_auth | ≈ 180 | ≈ 0–1 | 10.53.0.2 → .3 → 10.53.1.1 |

```
=== A: x14937.se-201-01.lab ===
;; Query time: 406 msec
;; Query time: 0 msec
=== B: x25585.lab ===
;; Query time: 40 msec
;; Query time: 0 msec
=== C: x29614.z374 ===
;; Query time: 81 msec
;; Query time: 0 msec
```

---

## Завдання 7. Інцидент: зупинка сервера

Прогноз: після `named-stop` і спливання TTL (35 с):
- resolver-a: ≈ `SERVFAIL` або таймаут, відповідь через ≈ 10–30 с (повтори до авторитативного NS);
- resolver-b (serve-stale): `NOERROR` зі застарілою адресою `10.53.0.81` і малим TTL (≈ 30 с, RFC 8767), відповідь миттєва.

Кешування запису в обох резолверах (крок 2) і час зупинки сервера (крок 3):

```
$ dig @10.53.0.53 edge.se-201-01.lab +noall +answer
edge.se-201-02.lab.     35      IN      A       10.53.0.81
$ dig @10.53.0.54 edge.se-201-02.lab +noall +answer
edge.se-201-02.lab.     35      IN      A       10.53.0.81

Час зупинки сервера (named-stop): 14:45:54
```

### Постмортем

1. **Що сталося:** сервер зони se-201-01.lab (10.53.1.1) зупинено, ≈ 14:40:00.
2. **Вплив:** клієнти resolver-a після спливання TTL отримують SERVFAIL/таймаут; resolver-b віддає застарілі дані.
3. **Хронологія:** stop ≈ 14:40:00, запити ≈ 14:40:50, start ≈ 14:41:20, відновлення ≈ 14:41:25.
4. **Роль кешу:** звичайний кеш не допоміг після TTL (35 с); serve-stale допоміг.
5. **Що змінити:** додати другий авторитативний NS у незалежному сегменті; для сервісів, що мігрують, обережно використовувати serve-stale.

---

## Фінальний файл зони

```
; Zone se-201-01.lab – starting version.
; After every change: increase the serial, then run  zone-reload
$TTL 45
@   15 IN SOA ns.se-201-01.lab. admin.se-201-02.lab. (
        2026112304 ; serial
        3600       ; refresh
        600        ; retry
        86400      ; expire
        60         ; minimum
    )
@   IN NS ns.se-201-01.lab.
ns  IN A  10.53.1.1

; ---- add records from your variant card below this line ----
www     IN  A       10.53.0.81
blog    IN  CNAME   www
mail    IN  A       10.53.0.80
@       IN  MX      50 mail
@       IN  TXT     "lab-token=470b3560"
edge 35 IN  A       10.53.0.81
status  IN  A       10.53.0.80
```

---

## Висновки

Розглянуто складові URI та зв'язок між URI й DNS-запитами. Простежено делегування root → TLD → авторитативний сервер. Досліджено вплив TTL на видимість змін, негативне кешування за RFC 2308, затримки пошуку імені та поведінку резолверів під час відмови авторитативного сервера (serve-stale). Кешування зменшує затримки, але відкладає видимість змін і зберігає застарілі та негативні відповіді.

## Декларація використання ШІ

ChatGPT / Claude: допомога зі структурою звіту та рішенні задач.

# Домашнє завдання 4. Інфраструктура та мережі в DevOps

**Студентка:** Вох Ірина  
**Дисципліна:** Комп'ютерні мережі (ІПЗ, 5 семестр)

> **Примітка щодо середовища.** Я працюю на macOS, а в macOS немає `iptables` (там інший файрвол, `pf`). Тому всі команди я виконувала в Linux, який запустила через Docker Desktop. Це привілейований контейнер `docker:27-dind` (Alpine Linux, ядро `6.12.76-linuxkit`) з власним Docker-демоном усередині:
>
> ```bash
> docker run -d --privileged --name dz4-linux docker:27-dind   # Linux із Docker усередині
> docker exec dz4-linux <команда>                              # так виконувалась кожна команда нижче
> ```
>
> Через таку вкладеність «хостом» у завданнях є саме цей Linux (dind):
>
> | Що | Адреса | Пояснення |
> | --- | --- | --- |
> | `eth0` хоста | `172.17.0.2` | інтерфейс dind-контейнера в мережі bridge Docker Desktop |
> | шлюз хоста | `172.17.0.1` | bridge Docker Desktop на Mac |
> | `docker0` хоста | `172.18.0.1`, мережа `172.18.0.0/16` | внутрішній bridge для контейнерів nginx. Внутрішній демон обрав `172.18`, бо `172.17` вже зайнята |
>
> У контейнері не було `curl`, тому замість `curl localhost:8080` я використала `wget -S -O - localhost:8080` (показує заголовки відповіді й тіло сторінки).

---

## Завдання 1. Маршрутизація на хості (2 бали)

### Команди

```bash
docker exec dz4-linux ip route
```

### Результат

```text
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link  src 172.17.0.2
172.18.0.0/16 dev docker0 scope link  src 172.18.0.1
```

### Аналіз

| Маршрут | Що означає |
| --- | --- |
| `default via 172.17.0.1 dev eth0` | Маршрут за замовчуванням. **Шлюз — `172.17.0.1`, інтерфейс — `eth0`.** |
| `172.17.0.0/16 dev eth0 scope link src 172.17.0.2` | Пряме підключення. Адреси `172.17.0.0/16` доступні напряму через `eth0`, без шлюзу. `src 172.17.0.2` — власна адреса хоста, яка ставиться як відправник. |
| `172.18.0.0/16 dev docker0 scope link src 172.18.0.1` | Мережа контейнерів. `docker0` — віртуальний bridge, до якого Docker підключає контейнери. Хост має на ньому адресу `172.18.0.1` і є шлюзом для контейнерів. |

**Що означає `default via X dev Y`.** Це маршрут для всіх адрес, яким не підійшов жоден точніший маршрут (тобто `0.0.0.0/0`). Ядро вибирає маршрут з найдовшим збігом префікса. Якщо збігу немає, пакет відправляється на шлюз `X` через інтерфейс `Y`. Тут: усе, що не в `172.17.0.0/16` і не в `172.18.0.0/16` (наприклад, інтернет), іде на `172.17.0.1` через `eth0`.

На моєму реальному Mac (ДЗ1) аналогічний маршрут був `default → 192.168.88.1` через `en1` (Wi-Fi), тобто шлюзом був домашній роутер.

---

## Завдання 2. Правила файрвола iptables (3 бали)

### Команди

```bash
docker exec dz4-linux iptables -L -n
```

### Результат

```text
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

### Аналіз

`iptables -L` без `-t` показує таблицю `filter`. У ній три стандартні ланцюги:

| Ланцюг | Які пакети | Що у виводі |
| --- | --- | --- |
| **INPUT** | Пакети, адресовані **самому хосту** (наприклад, SSH на сервер). | Правил немає, політика `ACCEPT` — приймається все. |
| **FORWARD** | Пакети, які **проходять через хост** далі, до іншого адресата. Наприклад, трафік до контейнерів через `docker0`: тут хост працює як роутер. | Docker додав свої правила (нижче). |
| **OUTPUT** | Пакети, які **створює сам хост**. | Правил немає, політика `ACCEPT`. |

Правила Docker у FORWARD (перевіряються зверху вниз):

| Правило | Призначення |
| --- | --- |
| `DOCKER-USER` | Ланцюг для правил адміністратора. Docker його не змінює, тому власні обмеження ставлять сюди. Зараз лише `RETURN` (нічого не робить). |
| `DOCKER-ISOLATION-STAGE-1` → `STAGE-2` | Ізоляція мереж. Блокує (`DROP`) трафік між **різними** bridge-мережами Docker, щоб контейнери з різних мереж не бачили один одного. |
| `ACCEPT ... ctstate RELATED,ESTABLISHED` | Пропускає відповіді на вже встановлені з'єднання (connection tracking). |
| `DOCKER` | Сюди Docker додає дозволи для опублікованих портів (`-p`). Поки контейнерів немає, ланцюг порожній. |
| два `ACCEPT` | Дозволяють контейнерам виходити назовні і спілкуватися в межах `docker0`. Без `-v` не видно інтерфейсів, до яких вони прив'язані. |

**Зауваження про безпеку.** Політика `ACCEPT` на INPUT дуже ліберальна: хост приймає з'єднання на будь-який порт. На реальному сервері зазвичай ставлять політику `DROP` і явно відкривають лише потрібні порти (наприклад, 22, 80, 443) плюс `RELATED,ESTABLISHED`.

---

## Завдання 3. Мережа контейнера (3 бали)

### Команди

```bash
docker exec dz4-linux docker run -d --name test-net nginx
docker exec dz4-linux docker inspect test-net --format '{{json .NetworkSettings}}'
docker exec dz4-linux docker network ls
```

### Результат (NetworkSettings скорочено до ключових полів)

```text
{
  "Ports": { "80/tcp": null },
  "Gateway": "172.18.0.1",
  "IPAddress": "172.18.0.2",
  "IPPrefixLen": 16,
  "MacAddress": "02:42:ac:12:00:02",
  "Networks": {
    "bridge": {
      "NetworkID": "6017f9913557...",
      "Gateway": "172.18.0.1",
      "IPAddress": "172.18.0.2",
      "IPPrefixLen": 16,
      "MacAddress": "02:42:ac:12:00:02"
    }
  }
}
```

```text
NETWORK ID     NAME      DRIVER    SCOPE
6017f9913557   bridge    bridge    local
989a1a57bbd3   host      host      local
7c16f112b8d8   none      null      local
```

### Аналіз

| Параметр | Значення | Пояснення |
| --- | --- | --- |
| **IP-адреса** | `172.18.0.2/16` | Адреса контейнера в мережі `docker0`. |
| **Мережа** | `bridge` | Мережа за замовчуванням. Її `NetworkID` `6017f9913557` збігається з рядком `bridge` у `docker network ls`. |
| **Шлюз** | `172.18.0.1` | Адреса `docker0` на хості (див. завдання 1). |
| **MAC** | `02:42:ac:12:00:02` | Docker генерує MAC з IP: `ac:12:00:02` = `172.18.0.2` у hex. |
| **Порти** | `"80/tcp": null` | nginx слухає порт 80 всередині, але порт **не опублікований** на хості. |

Інші мережі: `host` — контейнер використовує мережевий стек хоста без ізоляції; `none` — контейнер без мережі (лише loopback).

**Чому контейнер недоступний з інтернету:**

1. `172.18.0.2` — приватна адреса (діапазон `172.16.0.0/12`, RFC 1918). В інтернеті такі адреси не маршрутизуються.
2. Ця мережа існує лише на віртуальному bridge всередині хоста. Ззовні до неї немає маршруту. Назовні контейнер виходить лише через NAT з адресою хоста.
3. Порт 80 не опублікований (`null`). Щоб сервіс був доступний ззовні, потрібен проброс порту `-p <порт_хоста>:80` (завдання 4).

---

## Завдання 4. Проброс портів і NAT (4 бали)

### Команди

```bash
docker exec dz4-linux docker run -d -p 8080:80 --name web-server nginx
docker exec dz4-linux wget -S -O - localhost:8080    # замість curl localhost:8080
docker exec dz4-linux iptables -t nat -L -n
```

Контейнер `web-server` отримав адресу `172.18.0.3` (наступну після `test-net`).

### Результат

```text
Connecting to localhost:8080 ([::1]:8080)
  HTTP/1.1 200 OK
  Server: nginx/1.31.6
  Date: Sat, 10 Oct 2026 10:44:45 GMT
  Content-Type: text/html
  Content-Length: 896
  Connection: close
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

```text
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.18.0.0/16        0.0.0.0/0
MASQUERADE  tcp  --  172.18.0.3           172.18.0.3           tcp dpt:80

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.18.0.3:80
```

### Аналіз

**Перевірка:** сервер відповів `HTTP/1.1 200 OK`, `Server: nginx/1.31.6`, сторінка «Welcome to nginx!». Проброс працює.

`wget` підключився через `[::1]` (IPv6 localhost). Правило OUTPUT виключає `127.0.0.0/8`, тобто трафік на localhost не проходить через DNAT. Такі з'єднання приймає userland proxy Docker (`docker-proxy`), який теж слухає порт 8080 і пересилає їх у контейнер.

Правила таблиці `nat`:

| Правило | Що робить |
| --- | --- |
| `PREROUTING → DOCKER` (`dst-type LOCAL`) | Вхідні пакети, адресовані будь-якій адресі хоста, перевіряються в ланцюгу `DOCKER`. |
| `OUTPUT → DOCKER` (`!127.0.0.0/8`) | Те саме для пакетів, які створює сам хост (крім localhost). |
| `DNAT tcp dpt:8080 to:172.18.0.3:80` | **Головне правило.** Замінює адресу призначення `<хост>:8080` на `172.18.0.3:80` (контейнер nginx). |
| `MASQUERADE 172.18.0.0/16 → 0.0.0.0/0` | SNAT для вихідного трафіку контейнерів: їхня приватна адреса замінюється на адресу хоста. Так контейнери виходять в інтернет. |
| `MASQUERADE 172.18.0.3 → 172.18.0.3 dpt:80` | Hairpin NAT: для випадку, коли контейнер звертається сам до себе через опублікований порт. |

**Як зовнішній клієнт потрапляє до сервісу, крок за кроком:**

1. Клієнт надсилає TCP-пакет на `<IP хоста>:8080`.
2. Пакет потрапляє в `PREROUTING` таблиці `nat`. Адреса належить хосту (`dst-type LOCAL`), тому перехід у ланцюг `DOCKER`.
3. Правило `DNAT` змінює адресу призначення на `172.18.0.3:80`. Conntrack запам'ятовує цю заміну.
4. Тепер адреса вже не локальна, тому пакет іде не в INPUT, а в **FORWARD** таблиці `filter`. Там Docker дозволив трафік до опублікованого порту контейнера.
5. За маршрутом `172.18.0.0/16 dev docker0` пакет іде через `docker0` у контейнер, nginx його обробляє.
6. Відповідь nginx іде назад через хост. Conntrack автоматично виконує зворотну заміну: адреса відправника `172.18.0.3:80` знову стає `<IP хоста>:8080`. Клієнт бачить відповідь від хоста і не знає про контейнер.

```text
клієнт ──► хост:8080 ──[PREROUTING → DOCKER: DNAT]──► FORWARD ──► docker0 ──► 172.18.0.3:80 (nginx)
клієнт ◄── хост:8080 ◄──────────[conntrack: зворотний NAT]────────────────── 172.18.0.3:80
```

---

## Завдання 5. Мережа в Kubernetes, теорія (2 бали)

**1. Як спілкуються контейнери в одному Pod?**
Усі контейнери Pod мають **один спільний network namespace**: одну IP-адресу та один простір портів. Цей namespace створює службовий pause-контейнер, а інші контейнери до нього приєднуються. Тому вони звертаються один до одного через `localhost:<порт>`. Наслідок: два контейнери в одному Pod не можуть слухати той самий порт.

**2. Навіщо потрібен Service, якщо в Pod є IP?**
IP Pod тимчасовий. Pod може перезапуститися, переїхати на інший вузол або бути створений заново при масштабуванні, і отримає нову адресу. До того ж реплік зазвичай кілька. Service (тип ClusterIP) дає **стабільну віртуальну IP-адресу і DNS-ім'я**. Він вибирає Pod за мітками (labels), веде їх актуальний список (Endpoints/EndpointSlices) і розподіляє навантаження між ними.

**3. Як працює kube-proxy?**
kube-proxy працює на кожному вузлі. Він стежить за об'єктами Service і Endpoints через API-сервер і програмує правила iptables або IPVS (деякі CNI, наприклад Cilium, замінюють його на eBPF). Ці правила роблять DNAT: пакет на `ClusterIP:порт` перенаправляється на IP і порт одного з актуальних Pod. Це той самий механізм DNAT, який Docker використав для `-p 8080:80` у завданні 4.

---

> **Прибирання.** Після виконання завдань тестове середовище видалено разом з усіма контейнерами всередині: `docker rm -f dz4-linux`.

# Дипломная работа: Отказоустойчивая инфраструктура

## Задача
Разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура размещена в Yandex Cloud.

## Архитектура

### Структура инфраструктуры

Инфраструктура состоит из следующих компонентов:

**Публичная подсеть (10.1.0.0/24):**
- Bastion Host (10.1.0.21) — SSH-доступ к приватным ВМ
- Zabbix Server (10.1.0.27) — система мониторинга
- Kibana (10.1.0.29) — визуализация логов

**Приватные подсети:**
- Приватная подсеть A (10.2.0.0/24): Web1 (10.2.0.10)
- Приватная подсеть B (10.3.0.0/24): Web2 (10.3.0.14)
- Приватная подсеть ES (10.4.0.0/24): Elasticsearch (10.4.0.29)

**Балансировщик нагрузки:**
- Application Load Balancer (84.201.146.158) распределяет трафик между Web1 и Web2

### Потоки данных

| От | К | Порт | Протокол |
|---|---|---|---|
| Балансировщик | Web1, Web2 | 80 | HTTP |
| Web1, Web2 | Zabbix Server | 10050 | Zabbix Agent |
| Web1, Web2 | Elasticsearch | 9200 | HTTP |
| Kibana | Elasticsearch | 9200 | HTTP |
| Bastion | Все ВМ | 22 | SSH |
| Клиенты | Zabbix | 80 | HTTP |
| Клиенты | Kibana | 5601 | HTTP |

### Доступ к сервисам

| Сервис | URL | Тип доступа |
|---|---|---|
| Сайт | http://84.201.146.158 | Публичный |
| Zabbix | http://51.250.66.248/zabbix | Публичный |
| Kibana | http://51.250.93.81:5601 | Публичный |
| Elasticsearch | http://10.4.0.29:9200 | Приватный |
| SSH | через bastion 111.88.241.109 | Публичный |

### Сеть

- **VPC**: default
- **Публичная подсеть**: `public-subnet-a` — 10.1.0.0/24 (зона A)
- **Приватные подсети**:
  - `private-subnet-a` — 10.2.0.0/24 (зона A) — для web1
  - `private-subnet-b` — 10.3.0.0/24 (зона B) — для web2
  - `private-subnet-es` — 10.4.0.0/24 (зона A) — для elasticsearch
- **NAT-шлюз** для доступа приватных ВМ в интернет
- **Security Groups** для ограничения трафика

### Виртуальные машины

| ВМ | Зона | Внутренний IP | Внешний IP | Назначение | ПО |
|---|---|---|---|---|---|
| bastion | A | 10.1.0.21 | 111.88.241.109 | SSH-доступ | Ubuntu 22.04 |
| web1 | A | 10.2.0.10 | — | Веб-сервер | Nginx, Zabbix Agent, Filebeat |
| web2 | B | 10.3.0.14 | — | Веб-сервер | Nginx, Zabbix Agent, Filebeat |
| zabbix | A | 10.1.0.27 | 51.250.66.248 | Мониторинг | Zabbix Server 6.4, MySQL, Apache |
| elasticsearch | A | 10.4.0.29 | — | Хранение логов | Elasticsearch 8.11 (Docker) |
| kibana | A | 10.1.0.29 | 51.250.90.121 | Визуализация | Kibana 8.11 (Docker) |

## Компоненты

### Веб-серверы и балансировка

- Два веб-сервера (web1, web2) в разных зонах доступности
- Nginx установлен на оба сервера
- Статический HTML-сайт
- **Application Load Balancer**:
  - Публичный IP: `84.201.146.158`
  - Target Group: web1, web2
  - Backend Group: `web-backend-group`
  - Healthcheck: HTTP GET / на порт 80
  - HTTP Router: `web-http-router` с маршрутом `/`

**Проверка**: `curl http://84.201.146.158:80`

### Мониторинг (Zabbix)

- **Zabbix Server 6.4** на ВМ zabbix
- MySQL как база данных
- Zabbix Agents установлены на:
  - web1 (10.2.0.10)
  - web2 (10.3.0.14)
  - zabbix (localhost)
- **Дашборд USE Metrics**:
  - CPU Utilization
  - Memory Utilization
  - Disk Utilization
  - Network Traffic

**Доступ**: `http://51.250.66.248/zabbix` (Admin / zabbix)

### Логирование

- **Elasticsearch 8.11** (Docker) на ВМ elasticsearch — порт 9200
- **Kibana 8.11** (Docker) на ВМ kibana — порт 5601
- **Filebeat 8.11** (Docker) на web1 и web2
- Собираемые логи:
  - `/var/log/nginx/access.log`
  - `/var/log/nginx/error.log`
- Index Pattern: `nginx-logs-*`
- Data View: `nginx-logs`

**Доступ**: `http://51.250.90.121:5601`

### Резервное копирование

- Ручные snapshots всех дисков ВМ
- Расписание `daily-snapshots`:
  - Ежедневно в 2:00
  - Хранение: 7 дней (168 часов)
  - Включает все 6 дисков

## Безопасность

- Приватные ВМ (web1, web2, elasticsearch) не имеют внешних IP
- SSH-доступ только через bastion host
- Security Groups ограничивают входящий трафик:
  - `sg-bastion`: SSH (22) отовсюду
  - `sg-web`: HTTP (80) от балансировщика, SSH (22) от bastion, Zabbix agent (10050)
  - `sg-zabbix`: HTTP (80), SSH (22), Zabbix agent/trapper (10050/10051)
  - `sg-elasticsearch`: ES (9200) от внутренней сети, SSH (22) от bastion
  - `sg-kibana`: HTTP (5601), SSH (22)

## Выполнение

### Этапы работы:

1. **Создание инфраструктуры** через Yandex Cloud CLI
   - VPC, подсети, NAT-шлюз
   - Виртуальные машины
   - Security Groups

2. **Настройка веб-серверов**
   - Установка Nginx
   - Создание статических сайтов
   - Настройка Load Balancer

3. **Настройка мониторинга**
   - Установка Zabbix Server + MySQL
   - Установка Zabbix Agents
   - Создание дашбордов USE

4. **Настройка логирования**
   - Установка Elasticsearch (Docker)
   - Установка Kibana (Docker)
   - Установка Filebeat (Docker)
   - Настройка сбора логов Nginx

5. **Резервное копирование**
   - Создание snapshots
   - Настройка расписания

## Компромиссы и решения

1. **Terraform не использовался** — вместо этого использован Yandex Cloud CLI из-за ограничений Cloud Shell (проблемы с chmod при установке Terraform провайдера). Ресурсы созданы через YC CLI.

2. **Elasticsearch и Kibana в Docker** — использованы Docker-контейнеры из-за недоступности репозиториев Elastic (403 Forbidden из России).

3. **Filebeat в Docker** — аналогично, использован Docker для обхода ограничений доступа.

4. **NAT на bastion** — для доступа приватных ВМ в интернет использован NAT на bastion host, так как NAT-шлюз Yandex Cloud создавался с типом `shared_egress_gateway`, который не обеспечивал нужную функциональность.

5. **Zabbix вместо Prometheus** — выбран Zabbix как более традиционное решение для мониторинга в Yandex Cloud.

## Тестирование

### 1. Проверка сайта через Load Balancer:
```bash
curl -v http://84.201.146.158:80

*   Trying 84.201.146.158:80...
* Connected to 84.201.146.158 (84.201.146.158) port 80
> GET / HTTP/1.1
> Host: 84.201.146.158
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< server: ycalb
< date: Fri, 04 Sep 2026 15:29:29 GMT
< content-type: text/html
< content-length: 43
< last-modified: Thu, 03 Sep 2026 18:26:49 GMT
< etag: "6a99bbe9-2b"
< accept-ranges: bytes
< 
<h1>Web Server 1</h1><p>Hostname: web1</p>
```

### 1.1. Проверка балансировки (запросы распределяются между серверами):
```bash
for i in 1 2 3 4 5 6; do curl -s http://84.201.146.158:80; echo ""; done

<h1>Web Server 2</h1><p>Hostname: web2</p>
<h1>Web Server 2</h1><p>Hostname: web2</p>
<h1>Web Server 1</h1><p>Hostname: web1</p>
<h1>Web Server 2</h1><p>Hostname: web2</p>
<h1>Web Server 2</h1><p>Hostname: web2</p>
<h1>Web Server 1</h1><p>Hostname: web1</p>
```

### 2. Проверка Zabbix:
```bash
curl -I http://51.250.66.248/zabbix

HTTP/1.1 301 Moved Permanently
Date: Fri, 04 Sep 2026 16:21:57 GMT
Server: Apache/2.4.52 (Ubuntu)
Location: http://51.250.66.248/zabbix/
Content-Type: text/html; charset=iso-8859-1
```

### 3. Проверка Kibana:
```bash
curl -I http://51.250.90.121:5601

HTTP/1.1 302 Found
location: /spaces/enter
x-content-type-options: nosniff
referrer-policy: no-referrer-when-downgrade
permissions-policy: camera=(), display-capture=(), fullscreen=(self), geolocation=(), microphone=(), web-share=()
cross-origin-opener-policy: same-origin
content-security-policy: script-src 'self'; worker-src blob: 'self'; style-src 'unsafe-inline' 'self'
kbn-name: c51dbef1932b
kbn-license-sig: 055fb1e864d26853b03a03a2f9d02a097558087c56f7be4af5209aec8bfeaf67
cache-control: private, no-cache, no-store, must-revalidate
content-length: 0
Date: Fri, 04 Sep 2026 16:22:10 GMT
Connection: keep-alive
```

### 4. Проверка Elasticsearch:
```bash
curl http://localhost:9200

{
  "name" : "1166167ddf75",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "8aE4gBxAQbCYOnjvD3WfWg",
  "version" : {
    "number" : "8.11.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "d9ec3fa628c7b0ba3d25692e277ba26814820b20",
    "build_date" : "2023-11-04T10:04:57.184859352Z",
    "build_snapshot" : false,
    "lucene_version" : "9.8.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

### 5. Проверка логов в Elasticsearch:
```bash
curl http://localhost:9200/_cat/indices?v

health status index                                              uuid                   pri rep docs.count docs.deleted store.size pri.store.size dataset.size
yellow open   .ds-nginx-logs-8.11.0-2026.09.04-2026.09.04-000001 Q96qJXkwQ46_XiJ-gcBOPw   1   1      28740            0      2.4mb        2.4mb        2.4mb
```

### 6. Проверка Zabbix хостов:
```bash
В веб-интерфейсе Zabbix → Monitoring → Hosts:
Хост	Статус	Availability
web1	Enabled	ZBX (зелёный)
web2	Enabled	ZBX (зелёный)
Zabbix server	Enabled	ZBX (зелёный)
```

### 7. Проверка Snapshots:
```bash
yc compute snapshot list

+----------------------+------------------------+----------------------+--------+
|          ID          |          NAME          |     PRODUCT IDS      | STATUS |
+----------------------+------------------------+----------------------+--------+
| fd829g9f2o2d80i3dlu0 | snapshot-web1          | f2eso5tionsq8dgfm97s | READY  |
| fd8afrv32v0fu0dg28pa | snapshot-bastion       | f2eso5tionsq8dgfm97s | READY  |
| fd8dc78pmfc54crnmsk7 | snapshot-web2          | f2eso5tionsq8dgfm97s | READY  |
| fd8k9orgcebn8i107hmr | snapshot-elasticsearch | f2eso5tionsq8dgfm97s | READY  |
| fd8uvr2jrf7n1rhgp4rj | snapshot-kibana        | f2eso5tionsq8dgfm97s | READY  |
| fd8vvitdnd9jma13gmfj | snapshot-zabbix        | f2eso5tionsq8dgfm97s | READY  |
+----------------------+------------------------+----------------------+--------+
```

### 8. Проверка Snapshot Schedule:
```bash
yc compute snapshot-schedule get --name daily-snapshots

Результат:

    Status: ACTIVE

    Expression: 0 2 * * * (ежедневно в 2:00)

    Retention: 604800s (7 дней)

    Диски: 6 дисков привязано
```

### 9. Проверка ВМ:
```bash
yc compute instance list

+----------------------+---------------+---------------+---------+----------------+-------------+
|          ID          |     NAME      |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+---------------+---------------+---------+----------------+-------------+
| epdjt6358vi21orafvqr | web2          | ru-central1-b | RUNNING |                | 10.3.0.14   |
| fhm60vmlp65p4dr255hi | bastion       | ru-central1-a | RUNNING | 111.88.241.109 | 10.1.0.21   |
| fhmc8jfqo69srik99858 | zabbix        | ru-central1-a | RUNNING | 51.250.66.248  | 10.1.0.27   |
| fhmdrrqv32kaf1s2j4nc | web1          | ru-central1-a | RUNNING |                | 10.2.0.10   |
| fhmlki9olrgctc7fmouk | kibana        | ru-central1-a | RUNNING | 51.250.90.121  | 10.1.0.29   |
| fhmtqudqmukvocrdi086 | elasticsearch | ru-central1-a | RUNNING |                | 10.4.0.29   |
+----------------------+---------------+---------------+---------+----------------+-------------+
```

## Скриншоты выполнения дипломной работы:

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/1239d687-44cc-4694-b6f9-04e163f25b69" />

<img width="1366" height="696" alt="image" src="https://github.com/user-attachments/assets/1992401d-7a61-4ae8-9587-fd91f33c0491" />




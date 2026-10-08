# Отчет по лабораторной работе №3

**Студент:** Ращинский Назар Андреевич  
**Группа:** 11б

---

## Цель работы

Изучение средств мониторинга для сбора и обработки телеметрии приложений (MLT: Metrics, Logs, Traces) и настройка системы наблюдаемости (Observability) для микросервисного Python-приложения на базе фреймворка Flask с использованием OpenTelemetry, Jaeger, Prometheus, Loki и Grafana.

---

## Задачи работы

1. Подготовить Python-приложение (Flask) к сбору трассировок и развернуть инструмент трассировки **Jaeger**.
2. Упаковать приложение и сопутствующие сервисы мониторинга в Docker-контейнеры с помощью **Docker Compose**.
3. Обеспечить экспорт телеметрии в Jaeger, Prometheus и Grafana.
4. Оформить отчёт и опубликовать все конфигурационные файлы и результаты в Git-репозитории.

---

## Задание 1: Подключение OpenTelemetry и настройка трассировки (Jaeger)

### Описание шагов выполнения

В рамках первого задания было создано REST-приложение на Flask (`dice-server`), генерирующее случайные числа (бросок кубика). Приложение было инструментировано с помощью SDK OpenTelemetry для автоматического сбора HTTP-трассировок и ручного создания кастомных спанов (`roll_dice_operation`).

#### Созданные файлы проекта (`/task1`):

- `app.py` — исходный код Flask-приложения с интеграцией OpenTelemetry и экспортером Jaeger (UDP 6831).
- `requirements.txt` — зависимости проекта (`flask`, `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-instrumentation-flask`, `opentelemetry-exporter-jaeger`).
- `Dockerfile` — инструкция сборки образа контейнера на базе `python:3.10-slim`.
- `docker-compose.yml` — конфигурация для совместного запуска контейнеров `flask_trace_app` и `jaeger`.

---

### Команды терминала и ход выполнения

Запуск стенда с помощью Docker Compose:

```
docker compose up --build -d
```

```
[+] up 7/7
 ✔ Image jaegertracing/all-in-one:1.53 Pulled                                                          82.5s
[+] Building 37.1s (12/12) FINISHED
[+] up 11/11king to docker.io/library/task1-web:latest                                                 0.2s
 ✔ Image jaegertracing/all-in-one:1.53 Pulled                                                          82.5s
 ✔ Image task1-web                     Built                                                           37.1s
 ✔ Network task1_monitoring            Created                                                          0.0s
 ✔ Container jaeger                    Started                                                          0.2s
 ✔ Container flask_trace_app           Started                                                          0.2s
```

Генерация тестовых HTTP-запросов для проверки трассировки:

```
curl "http://localhost:8080/rolldice?player=Nazar"
Nazar rolled: 6
curl "http://localhost:8080/rolldice?player=TestUser"
TestUser rolled: 4
```

После отправки тестовых запросов был открыт веб-интерфейс Jaeger UI по адресу http://localhost:16686.
В строке поиска сервисов был выбран сервис dice-server. В результате выполнения запроса Find Traces были успешно обнаружены распределенные трассировки вызовов эндпоинта /rolldice.

Каждый трейс содержит базовый спан HTTP-запроса Flask и вложенный кастомный спан roll_dice_operation.

В атрибутах спанов корректно отображаются переданные параметры запроса (player) и сгенерированный результат (result).

---

### Задание 2: Настройка сбора метрик и интеграция с Prometheus

### Описание шагов выполнения

#### Созданные файлы проекта (`/task2`):

- `app.py` — Flask-приложение с интеграцией `OTLPSpanExporter` и `OTLPMetricExporter` (экспорт данных по gRPC на `otel-collector:4317`). Описан кастомный счетчик метрики `dice_rolls_total`.
- `requirements.txt` — добавлены OTLP-экспортеры (`opentelemetry-exporter-otlp`).
- `Dockerfile` — контейнеризация Flask-сервиса.
- `otel-collector-config.yml` — конфигурация пайплайнов Collector (прием OTLP gRPC/HTTP, экспорт трассировок в Jaeger и метрик в Prometheus).
- `prometheus.yml` — конфигурация сбора метрик с эндпоинта `otel-collector:8889`.
- `docker-compose.yml` — оркестрация 4 сервисов (`web`, `otel-collector`, `jaeger`, `prometheus`).

Запуск полного стека мониторинга:

```
docker compose up --build -d
```

Проверка состояния контейнеров:

```
docker compose ps
```

```
NAME                IMAGE                                       STATUS
flask_metrics_app   task2-web                                   Up 5 seconds
jaeger              jaegertracing/all-in-one:1.53               Up 5 seconds
otel-collector      otel/opentelemetry-collector-contrib:0.95.0 Up 5 seconds
prometheus          prom/prometheus:v2.50.1                     Up 5 seconds
```

Генерация метрик с помощью HTTP-запросов:

```
curl "http://localhost:8080/rolldice?player=Nazar"
# Ответ: Nazar rolled: 1

curl "http://localhost:8080/rolldice?player=Nazar"
# Ответ: Nazar rolled: 5

curl "http://localhost:8080/rolldice?player=Alex"
# Ответ: Alex rolled: 1
```

Результаты проверки в Prometheus UI:
dice_rolls_total{exported_job="dice-server", instance="otel-collector:8889", job="otel-collector", player="Alex", result="1"}

1dice_rolls_total{exported_job="dice-server", instance="otel-collector:8889", job="otel-collector", player="Nazar", result="1"}

1dice_rolls_total{exported_job="dice-server", instance="otel-collector:8889", job="otel-collector", player="Nazar", result="5"}

---

## Заданиe 3: Централизованное логирование (Loki/Promtail) и визуализация в Grafana

### Описание шагов выполнения

В третьем задании был развернут полный стек наблюдаемости (Observability Stack), объединяющий все три компонента: **Traces**, **Metrics** и **Logs**.

#### Добавленные сервисы и файлы (`/task3`):

- `loki-config.yml` — конфигурация хранилища и индексатора логов Loki.
- `promtail-config.yml` — агент сбора логов, подключенный к сокету Docker (`/var/run/docker.sock`) для автоматического сбора stdout/stderr всех запущенных контейнеров.
- `docker-compose.yml` — единая оркестрация 7 контейнеров: `flask_observability_app`, `otel-collector`, `jaeger`, `prometheus`, `loki`, `promtail`, `grafana`.

Команды терминала и запуск:

```
docker compose up --build -d
```

Проверка запущенных сервисов:

```
docker compose ps
```

```
NAME                      IMAGE                                         COMMAND                  SERVICE      CREATED          STATUS         PORTS
flask_observability_app   task3-web                                     "python app.py"          web      8 seconds ago    Up 7 seconds   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
grafana                   grafana/grafana:10.3.3                        "/run.sh"                grafana      8 seconds ago    Up 7 seconds   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
jaeger                    jaegertracing/all-in-one:1.53                 "/go/bin/all-in-one-…"   jaeger      8 seconds ago    Up 8 seconds   0.0.0.0:16686->16686/tcp, [::]:16686->16686/tcp, 0.0.0.0:61477->4317/tcp, [::]:61477->4317/tcp
loki                      grafana/loki:2.9.4                            "/usr/bin/loki -conf…"   loki      32 seconds ago   Up 8 seconds   0.0.0.0:3100->3100/tcp, [::]:3100->3100/tcp
otel-collector            otel/opentelemetry-collector-contrib:0.95.0   "/otelcol-contrib --…"   otel-collector   8 seconds ago    Up 7 seconds   0.0.0.0:4317-4318->4317-4318/tcp, [::]:4317-4318->4317-4318/tcp, 0.0.0.0:8889->8889/tcp, [::]:8889->8889/tcp
prometheus                prom/prometheus:v2.50.1                       "/bin/prometheus --c…"   prometheus      8 seconds ago    Up 8 seconds   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
promtail                  grafana/promtail:2.9.4                        "/usr/bin/promtail -…"   promtail      32 seconds ago   Up 8 seconds
```

```
nazar@MacBook-Air-Nazar task3 % curl http://localhost:8080/
Welcome to the Dice Rolling Service! Try /rolldice%

nazar@MacBook-Air-Nazar task3 % curl http://localhost:8080/rolldice
Anonymous rolled: 3

nazar@MacBook-Air-Nazar task3 % curl "http://localhost:8080/rolldice?player=Nazar"
Nazar rolled: 4
```

Настройка и проверка в Grafana UI:
В Grafana (http://localhost:3000) успешно подключены 3 источника данных:

Prometheus (http://prometheus:9090)

Loki (http://loki:3100)

Jaeger (http://jaeger:16686)

В интерфейсе Explore выполнен LogQL-запрос {container="flask_observability_app"} в Loki, подтверждающий корректное поступление логов приложения.

Все трассировки, метрики и логи объединены в единую систему мониторинга.

---

## Ответы на контрольные вопросы

### 1. Что такое OpenTelemetry? Атрибуты, события, контекст, журналы, трассировки, показатели?

**OpenTelemetry (OTel)** — это фреймворк и стандарт с открытым исходным кодом (проект CNCF), предоставляющий единый набор API, SDK и инструментов для генерации, сбора, обработки и экспорта телеметрических данных (метрики, логи, трассировки) из программных систем[cite: 5, 11].

- **Атрибуты (Attributes):** Пары «ключ-значение», содержащие метаданные для добавления контекста к спанам, метрикам или логам (например, `http.status_code=200`, `user.id="Nazar"`).
- **События (Events):** Аннотированные временные метки внутри спана, обозначающие специфический момент времени (например, момент выброса исключения).
- **Контекст (Context):** Объект, передаваемый между функциями и потоками (и через сетевые границы с помощью W3C Trace Context headers), который несет информацию о текущей трассировке (Trace ID, Span ID).
- **Журналы (Logs):** Записи о событиях во времени с текстовым или структурированным содержимым[cite: 5].
- **Трассировки (Traces):** Записи полного пути выполнения запроса через все компоненты распределенной системы[cite: 5].
- **Показатели / Метрики (Metrics):** Числовые агрегированные данные о состоянии и производительности ресурсов системы за промежуток времени[cite: 4].

---

### 2. Что такое наблюдаемость? Надежность и показатели?

- **Наблюдаемость (Observability):** Свойство системы, позволяющее понимать ее внутреннее состояние на основе ее внешних выходов (телеметрии: метрик, логов и трассировок)[cite: 4].
- **Надежность (Reliability):** Способность системы выполнять требуемые функции без сбоев в заданных условиях в течение определенного времени.
- **Показатели (SLI / SLO / SLA):**
  - _SLI (Service Level Indicator)_ — числовая метрика производительности (например, задержка ответа).
  - _SLO (Service Level Objective)_ — целевое значение SLI (например, 99.9% запросов обрабатываются быстрее 200 мс).
  - _SLA (Service Level Agreement)_ — соглашение с пользователями о последствиях нарушения SLO.

---

### 3. Что такое трассировка? Что такое распределенная трассировка? Как собирать распределенную трассировку и какие проблемы имеются в распределенной архитектуре?

- **Трассировка (Trace):** Отображение последовательности операций при выполнении кода в рамках одного запроса[cite: 5].
- **Распределенная трассировка (Distributed Tracing):** Отслеживание пути прохождения запроса сквозь множество независимых микросервисов и сетевых границ[cite: 5].
- **Сбор распределенной трассировки:** Осуществляется за счет передачи контекста трассировки (`traceparent` header по стандарту W3C) в HTTP/gRPC заголовках от сервиса к сервису (Context Propagation).
- **Проблемы распределенной архитектуры:** Сложность поиска узких мест (latency bottlenecks), каскадные сбои, асинхронные вызовы, высокое накладные расходы на передачу и хранение огромных объемов данных телеметрии (требуется сэмплирование/sampling).

---

### 4. Сигналы OpenTelemetry: трассировки, метрики, логи?

- **Трассировки (Traces):** Показывают «где» и «в какой последовательности» происходила обработка запроса[cite: 5]. Состоят из дерева спанов (Spans).
- **Метрики (Metrics):** Показывают «насколько хорошо» работает система (количество запросов, использование CPU, память)[cite: 4, 5].
- **Логи (Logs):** Показывают «что именно произошло» в конкретный момент времени (сообщения об ошибках, отладочный вывод)[cite: 5].

---

### 5. Инструменты OpenTelemetry: автоматические, ручные, библиотеки?

- **Автоматические инструменты (Auto-instrumentation):** Инструментация без изменения исходного кода приложения (например, с помощью агентов Java/Python `opentelemetry-instrument`), перехватывающая вызовы стандартных библиотек (Flask, HTTP, SQLAlchemy).
- **Ручные инструменты (Manual instrumentation):** Явное использование OpenTelemetry API в коде приложения для создания кастомных спанов, добавления специфических атрибутов и метрик.
- **Библиотеки (SDK / API):** Разделение на API (абстрактные интерфейсы для генерации телеметрии) и SDK (реализация отправки, буферизации и сэмплирования).

---

### 6. Компоненты OpenTelemetry: спецификация, сборщики, библиотеки инструментальных средств, экспортеры, автоматические измерительные инструменты?

- **Спецификация (Specification):** Набор формальных требований к API, SDK и протоколам (OTLP)[cite: 11].
- **Сборщики (OTel Collector):** Прокси-сервисы для приема, обработки (batching, filtering), трансформирования и экспорта телеметрии во внешние системы (Jaeger, Prometheus, Loki)[cite: 11].
- **Библиотеки инструментальных средств (SDKs):** Реализации стандарта для конкретных языков программирования (Python, Go, Java)[cite: 6, 11].
- **Экспортеры (Exporters):** Модули SDK/Collector, отвечающие за трансляцию внутренних данных OTel во внешние форматы и их отправку по OTLP/gRPC/HTTP[cite: 11].
- **Автоматические измерительные инструменты:** Плагины и агенты для автоматического перехвата событий[cite: 11].

---

### 7. Ресурс OpenTelemetry (телеметрия)?

**Resource (Ресурс)** в OpenTelemetry — это объект, представляющий сущность, которая генерирует телеметрию (например, имя сервиса `service.name`, версия приложения `service.version`, имя хоста `host.name` или имя Docker-контейнера). Атрибуты ресурса привязываются ко всем сигналам (метрикам, логам, трейсам), создаваемым этой сущностью.

---

### 8. Экспортеры OpenTelemetry?

**Экспортеры (Exporters)** — компоненты, отправляющие собранные данные телеметрии в целевые бэкенды хранения и аналитики[cite: 11].
Примеры:

- `OTLPSpanExporter` / `OTLPMetricExporter` (экспорт по OTLP gRPC/HTTP)
- `PrometheusExporter` (открытие HTTP-эндпоинта для scrape)
- `ConsoleSpanExporter` (вывод в stdout/консоль)

---

### 9. Что такое Prometheus? Как работает Prometheus?

- **Prometheus:** СУБД временных рядов (TSDB) и система мониторинга с открытым исходным кодом[cite: 6, 11].
- **Принцип работы:** Prometheus работает по **Pull-модели** — он периодически запрашивает (scrapes) метрики по протоколу HTTP с целевых эндпоинтов (например, `:8889/metrics`), указанных в конфигурационном файле `prometheus.yml`[cite: 6, 8].

---

### 10. Концепции Prometheus?

- **Time Series (Временные ряды):** Потоки числовых данных, индексированные по времени.
- **Metric Name & Labels:** Идентификация метрики через имя и пары «ключ-значение» (например, `dice_rolls_total{player="Nazar"}`).
- **PromQL:** Язык запросов к метрикам в Prometheus.
- **Targets & Scrape:** Целевые объекты мониторинга и процесс опроса их эндпоинтов.

---

### 11. Типы метрик?

1. **Counter (Счетчик):** Накапливаемая метрика, значение которой может только расти или сбрасываться в 0 при перезапуске (например, количество обработанных запросов `dice_rolls_total`).
2. **Gauge (Даччик):** Метрика, значение которой может свободно увеличиваться и уменьшаться (например, загрузка CPU, количество свободной памяти).
3. **Histogram (Гистограмма):** Измеряет размерности (например, длительность выполнения запроса) и распределяет их по настраиваемым бакетам (buckets).
4. **Summary (Сводка):** Аналогично гистограмме рассчитывает скользящие квантили за промежуток времени на стороне клиента.

---

### 12. Запрос и преобразование данных Grafana?

- **Запросы (Queries):** Позволяют извлекать данные из подключенных Data Sources с использованием их родных языков (PromQL для Prometheus, LogQL для Loki, TraceQL/Jaeger)[cite: 11].
- **Преобразования (Transformations):** Встроенные функции Grafana для обработки результатов запросов на клиентской стороне (фильтрация колонок, объединение серий, переименование полей, математические расчеты).

---

### 13. Источники данных Grafana?

**Data Sources** — это плагины интеграции Grafana с внешними хранилищами телеметрии[cite: 11].
Основные источники:

- **Prometheus** (для метрик)[cite: 6]
- **Loki** (для логов)[cite: 6]
- **Jaeger / Tempo** (для распределенных трейсов)[cite: 6]
- **PostgreSQL / MySQL / Elasticsearch**

---

### 14. Панели мониторинга Grafana?

**Dashboards (Панели мониторинга):** Визуальные представления данных, состоящие из отдельных виджетов (Panels)[cite: 6, 11].
Типы визуализаций:

- **Time series** (линейные графики)
- **Stat / Gauge** (числовые показатели)
- **Logs** (поток логов из Loki)
- **Traces** (дерево выполнения из Jaeger)
- **Table** (табличный вид)

---

### 15. Процесс обработки журналов Grafana Loki?

1. **Сбор:** Агент (Promtail) считывает логи из файлов или сокета Docker[cite: 6].
2. **Маркировка (Labeling):** Promtail добавляет индексируемые метки (например, `container="flask_observability_app"`).
3. **Отправка:** Логи передаются по HTTP в Loki[cite: 6].
4. **Индексация и хранение:** Loki индексирует **только метки** (labels), а сам текст логов сжимает и сохраняет в чанки (chunks) на файловой системе или S3-хранилище.
5. **Запрос:** Интерфейс Grafana запрашивает логи с помощью языка **LogQL**.

---

### 16. Варианты сборки журналов для Grafana Loki?

1. **Promtail:** Официальный агент Grafana для парсинга и передачи логов из локальных файлов и Docker[cite: 6].
2. **Grafana Alloy (ранее Grafana Agent):** Универсальный сборщик для всех видов телеметрии (MLT).
3. **Fluentd / Fluent Bit:** Популярные open-source сборщики с плагином вывода в Loki.
4. **OpenTelemetry Collector:** Использование встроенного `loki exporter` в OTel Collector для отправки логов прямо из пайплайнов телеметрии.

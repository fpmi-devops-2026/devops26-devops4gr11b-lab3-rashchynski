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

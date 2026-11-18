# SmartTJ Infrastructure & Monitoring (`smarttj-docker`)

Централизованная инфраструктура проекта **SmartTJ SaaS**: базы данных (`PostgreSQL 18`, `Redis 8`), система сбора логов (`Loki` + `Promtail`) и система мониторинга (`Prometheus` + `Grafana` + экспортеры).

---

## 🏗️ Состав сервисов

| Сервис | Контейнер | Назначение |
| --- | --- | --- |
| **PostgreSQL 18** | `postgres` | Основная реляционная БД проекта |
| **Redis 8** | `redis` | Кэш, сессии, WebSocket-координаты и очереди BullMQ |
| **Grafana** | `grafana` | Единый UI-интерфейс мониторинга и логов (порт из `.env` — `GRAFANA_PORT`) |
| **Prometheus** | `prometheus` | Сбор и хранение метрик системы, БД и микросервисов (30 дней) |
| **Loki** | `loki` | Хранилище логов всех микросервисов и контейнеров (30 дней) |
| **Promtail** | `promtail` | Агент сбора логов из папок `/logs/*` (`backend`, `ai`, `realtime`, `worker`) и из Docker (`stdout`/`stderr`) |
| **Node Exporter** | `node-exporter` | Мониторинг сервера: CPU, RAM (ОЗУ), Swap, Диск, Сеть, Load Average, Uptime |
| **cAdvisor** | `cadvisor` | Мониторинг контейнеров: CPU, RAM, Сеть и состояние каждого микросервиса |
| **Postgres Exporter** | `postgres-exporter` | Метрики PostgreSQL (`pg_up`, подключения, транзакции, размер БД) |
| **Redis Exporter** | `redis-exporter` | Метрики Redis (`redis_up`, память, клиенты, ключи BullMQ, ops/sec) |
| **Blackbox Exporter** | `blackbox-exporter` | Проверка доступности (HTTP Healthcheck) и времени ответа микросервисов |

---

## ⚙️ Настройка `.env`

Скопируйте `.env.example` в `.env` и укажите нужные параметры (включая порт UI интерфейса `GRAFANA_PORT`):

```env
# Database
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=smarttj

# Monitoring UI (Grafana)
GRAFANA_PORT=3001
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin

# Microservices Logs Paths (host directories with /logs/*)
BACKEND_LOGS_PATH=../smarttj-backend/logs
AI_LOGS_PATH=../smarttj-ai/logs
REALTIME_LOGS_PATH=../smarttj-realtime/logs
WORKER_LOGS_PATH=../smarttj-worker/logs
```

---

## 📜 Как работает сбор логов

Логи собираются **одновременно из двух источников** и передаются через **Promtail → Loki → Grafana UI**:

1. **Файловые логи (`/logs/*`) микросервисов (`source="file"`)**:
   - `smarttj-backend` (`open-logger` / Winston): `combined-YYYY-MM-DD.log`, `error-YYYY-MM-DD.log`, а также `pm2/*.log`
   - `smarttj-ai` (Winston): `combined-YYYY-MM-DD.log`, `error-YYYY-MM-DD.log`
   - `smarttj-realtime` (Winston): `combined-YYYY-MM-DD.log`, `error-YYYY-MM-DD.log`
   - `smarttj-worker` (Winston): `combined-YYYY-MM-DD.log`, `error-YYYY-MM-DD.log`
   - Во всех 4 микросервисах в `docker-compose.yml` примонтирован том `./logs:/app/logs`, поэтому логи из контейнеров синхронизируются с папкой `logs/` каждого проекта.
2. **Контейнерные логи Docker (`source="docker"`)**:
   - Автоматическое обнаружение всех контейнеров через `/var/run/docker.sock` (`smarttj-backend`, `smarttj-ai`, `smarttj-realtime`, `smarttj-worker`, `smarttj-n8n`, `postgres`, `redis`) с очисткой ANSI-цветов и извлечением уровня лога (`info`, `warn`, `error`, `debug`).

---

## 📊 Преднастроенные дашборды в Grafana

При входе в Grafana (`http://localhost:<GRAFANA_PORT>`) автоматически загружаются 2 дашборда в папке **SmartTJ Monitoring**:

1. **SmartTJ — Системный мониторинг и Микросервисы** (открывается на главной):
   - Статусы (`ONLINE` / `DOWN`) всех микросервисов: `Backend API`, `AI Service`, `Realtime WS`, `Worker (BullMQ)`, `n8n`, `PostgreSQL 18`, `Redis 8` и Uptime сервера.
   - Детальный мониторинг процессора (`CPU %`, ядра, `Load Average`), оперативной памяти (`RAM %`, Всего / Занято / Доступно / Кэш / Swap), дисковой памяти (`Disk %`, GB, Disk I/O) и сети.
   - Потребление CPU, RAM, сети и время отклика (`HTTP Latency`) по каждому микросервису отдельно.
   - Метрики PostgreSQL и Redis.
2. **SmartTJ — Логи микросервисов (Loki: `/logs/*` и Docker)**:
   - Фильтрация по микросервису (`service`), источнику (`file` из `/logs/*` или `docker`), уровню (`info`, `warn`, `error`, `debug`), типу файла (`combined`, `error`, `pm2`, `container`) и текстовому поиску.
   - Выделенная панель ошибок (`ERROR`) и живой поток всех логов.

---

## 🚀 Запуск

```bash
# 1. Создать внешнюю сеть (если ещё не создана)
docker network create smarttj-network

# 2. Запустить инфраструктуру и мониторинг
docker compose up -d
```

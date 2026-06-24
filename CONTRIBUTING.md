# Contributing to InversionWharf

Это проект-спецификация (образовательный). Кода в production пока нет — только документация, OpenAPI, AsyncAPI и учебный прототип (Spring Boot).

---

## Структура репозитория

```
InversionWharf/
├── TECHNICAL_SPEC.md       # Основная спецификация
├── EDU_PROJECT_PLAN.md     # Учебный план реализации
├── JIRA_BACKLOG.md         # Product Backlog
├── CHANGELOG.md            # История изменений
├── README.md               # Обзор проекта
├── .env.example            # Переменные окружения (шаблон)
├── docker-compose.yml      # Учебный запуск (Spring Boot прототип)
├── openapi/
│   ├── agent-gateway.yaml  # OpenAPI: API для агентов
│   └── operator-gateway.yaml # OpenAPI: API для операторов
├── asyncapi/
│   └── events.yaml         # AsyncAPI: Kafka топики
├── docs/
│   ├── adr/                # Architecture Decision Records
│   ├── runbooks/           # Операционные сценарии
│   ├── threat-model.md     # STRIDE threat model
│   ├── nfr-slo.md          # NFR и SLO
│   ├── edge-cases.md       # Edge cases и failure scenarios
│   └── key-rotation.md     # Ротация ключей
└── demo/                   # Демо-материалы
```

---

## Запуск учебного прототипа

### Предварительные требования

- Docker Engine ≥ 24
- Docker Compose v2 (`docker compose version`)
- Java 21+ (для локальной разработки Spring Boot сервисов)
- Go 1.22+ (для агента)

### Быстрый старт

```bash
# Скопировать конфиг
cp .env.example .env
# Отредактировать .env — заменить changeme_* на реальные значения

# Запустить инфраструктуру (Kafka, PostgreSQL, Redis)
docker compose up -d kafka postgres redis

# Подождать готовности
docker compose ps

# Запустить сервисы
docker compose up -d
```

### Проверка

```bash
# Health check
curl http://localhost:8080/actuator/health

# Swagger UI
open http://localhost:8080/swagger-ui.html
```

---

## Внесение изменений в спецификацию

### Принципы

1. **Изменения требований → обновить TECHNICAL_SPEC.md** — это источник истины.
2. **Архитектурные решения → создать ADR** в `docs/adr/` (следующий номер по порядку).
3. **Изменения API → обновить OpenAPI** в `openapi/`.
4. **Изменения событий → обновить AsyncAPI** в `asyncapi/events.yaml`.
5. **Каждое изменение → запись в CHANGELOG.md**.

### Формат ADR

Используйте существующие файлы `docs/adr/0001-*.md` как шаблон:
- **Статус:** Принято / Отклонено / Заменено
- **Контекст:** проблема и её бизнес-контекст
- **Рассмотренные варианты:** минимум 2, с плюсами и минусами
- **Решение:** выбранный вариант с обоснованием
- **Последствия:** позитивные и негативные

### Версионирование CHANGELOG

Следуем [Semantic Versioning](https://semver.org/):
- `MAJOR` — ломающие изменения API или архитектуры
- `MINOR` — новые фичи/компоненты без ломающих изменений
- `PATCH` — исправления, уточнения, документация

---

## Стиль OpenAPI

- Версия: `openapi: "3.1.0"`
- Ошибки: `application/problem+json` с `$ref: "#/components/schemas/Problem"` (RFC 7807)
- Безопасность: в каждом endpoint через `security:` или `securitySchemes`
- Примеры: включать `example:` для ключевых полей

## Стиль AsyncAPI

- Версия: `asyncapi: "3.0.0"`
- Kafka bindings: указывать `partitions`, `replicas`, `retention.ms`
- Payload schemas — полные, с `required` и `description`

---

## Git Workflow

Ветка для разработки: `claude/idea-problems-b0zheh`

```bash
git checkout claude/idea-problems-b0zheh
git pull origin claude/idea-problems-b0zheh

# Внести изменения
git add <files>
git commit -m "feat(spec): описание изменения"
git push -u origin claude/idea-problems-b0zheh
```

### Commit message format

```
<type>(<scope>): <description>

type: feat | fix | docs | refactor | security
scope: spec | openapi | asyncapi | adr | runbook | docker
```

---

## Вопросы и предложения

Открывайте issue или обращайтесь к команде платформы (контакты в `TECHNICAL_SPEC.md`).

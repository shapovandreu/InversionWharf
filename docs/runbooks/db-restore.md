# Runbook: Восстановление PostgreSQL из резервной копии (PITR)

**Применяется когда:**
- Критическая ошибка данных (случайный DROP, некорректная миграция)
- Коррупция данных
- Восстановление после инцидента безопасности

**Ссылки:** `TECHNICAL_SPEC.md §2.14 NFR`, `§3.10`

> **Предупреждение:** Восстановление из бэкапа — деструктивная операция. Согласуйте с командой до выполнения.

---

## Подготовка

### Параметры резервного копирования (production)

| Параметр | Значение |
|---|---|
| Тип бэкапа | PITR (Point-in-Time Recovery) через WAL архивирование |
| Инструмент | pgBackRest или Barman |
| Полный бэкап | Еженедельно (воскресенье 03:00 UTC) |
| WAL архивирование | Непрерывное (каждые 5 мин) |
| Retention | 30 дней |
| Хранилище | S3-совместимый объектный storage |

### RTO / RPO

| Метрика | Целевое значение |
|---|---|
| RPO (потеря данных) | ≤ 5 минут |
| RTO (время восстановления) | ≤ 2 часа |

---

## Диагностика

Перед восстановлением определить точку восстановления:

```bash
# Найти время последней корректной операции в audit log
curl -H "Authorization: Bearer $OPERATOR_JWT" \
  "https://api.vendor.example.com/v1/audit?limit=100&sort=desc" | jq '.[] | select(.action | test("DELETE|DROP|TRUNCATE"))'

# Посмотреть PostgreSQL logs на момент инцидента
kubectl logs deployment/postgres --since=2h | grep -i "error\|fatal\|panic"
```

---

## Процедура восстановления

### Шаг 1: Поставить Control Plane на maintenance

```bash
# Включить maintenance mode (API gateway возвращает 503 с Retry-After)
kubectl scale deployment/api-gateway --replicas=0
kubectl scale deployment/agent-gateway --replicas=0

# Убедиться что нет активных транзакций
kubectl exec -it postgres-0 -- psql -U invwharf -c \
  "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"
```

> Агенты перейдут на stale cache и grace period — продукты продолжают работать.

### Шаг 2: Создать бэкап текущего состояния (перед восстановлением)

```bash
# Дамп "как есть" — на случай если нужно будет откатить откат
kubectl exec -it postgres-0 -- pg_dump -U invwharf invwharf \
  | gzip > /backup/pre-restore-$(date +%Y%m%d_%H%M%S).sql.gz
```

### Шаг 3: Определить точку восстановления

```bash
# Список доступных точек восстановления
pgbackrest --stanza=invwharf info

# Пример: восстановить на момент 2026-06-24 14:30:00 UTC
RESTORE_TARGET="2026-06-24 14:30:00 UTC"
```

### Шаг 4: Остановить PostgreSQL

```bash
kubectl scale statefulset/postgres --replicas=0
# Подождать полной остановки
kubectl wait pod/postgres-0 --for=delete --timeout=120s
```

### Шаг 5: Выполнить восстановление

```bash
# Запустить отдельный pod для восстановления
kubectl run pg-restore --image=pgbackrest:2.47 --restart=Never \
  --env="PGBACKREST_STANZA=invwharf" \
  --command -- pgbackrest restore \
    --target="$RESTORE_TARGET" \
    --target-action=promote \
    --delta
```

### Шаг 6: Запустить PostgreSQL в recovery mode

```bash
# Запустить pod
kubectl scale statefulset/postgres --replicas=1
kubectl wait pod/postgres-0 --for=condition=Ready --timeout=300s

# Следить за процессом восстановления
kubectl logs postgres-0 -f | grep -E "recovery|restored|ready to accept"
```

Ожидаемое сообщение об успехе:
```
LOG: database system is ready to accept connections
LOG: recovery stopping before commit of transaction ...
LOG: pausing at the end of recovery
```

### Шаг 7: Проверить целостность данных

```bash
kubectl exec -it postgres-0 -- psql -U invwharf -c \
  "SELECT schemaname, tablename, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 20;"

# Проверить последнюю запись в audit_events
kubectl exec -it postgres-0 -- psql -U invwharf -c \
  "SELECT created_at, action, actor_id FROM audit_events ORDER BY created_at DESC LIMIT 5;"

# Проверить count агентов
kubectl exec -it postgres-0 -- psql -U invwharf -c \
  "SELECT count(*) FROM agents WHERE status = 'active';"
```

### Шаг 8: Запустить Flyway миграции (если нужно)

Если восстановление откатило схему БД:

```bash
# Запустить catalog-service — Flyway применит миграции автоматически
kubectl scale deployment/catalog-service --replicas=1
kubectl logs deployment/catalog-service -f | grep -i "flyway\|migration"
```

### Шаг 9: Поднять Control Plane

```bash
# Запустить сервисы в порядке зависимостей
kubectl scale deployment/config-server --replicas=2
kubectl scale deployment/service-registry --replicas=2
kubectl scale deployment/auth-service --replicas=2
kubectl scale deployment/catalog-service --replicas=2
kubectl scale deployment/entitlement-service --replicas=2
kubectl scale deployment/license-service --replicas=2
kubectl scale deployment/fleet-service --replicas=2
kubectl scale deployment/api-gateway --replicas=2
kubectl scale deployment/agent-gateway --replicas=3

# Проверить health всех сервисов
kubectl get pods -l app.kubernetes.io/part-of=invwharf
```

---

## Верификация после восстановления

```bash
# Health check
curl https://api.vendor.example.com/actuator/health

# Тестовый запрос от имени оператора
curl -H "Authorization: Bearer $OPERATOR_JWT" \
  https://api.vendor.example.com/v1/fleet/agents | jq '.total'

# Проверить что агенты начали heartbeat
kubectl logs deployment/agent-gateway -f | grep heartbeat
```

Агенты должны восстановить связь в течение 1-2 минут после поднятия agent-gateway.

---

## Rollback (восстановление из pre-restore бэкапа)

Если восстановленная точка оказалась некорректной:

```bash
# Остановить CP
kubectl scale deployment/api-gateway --replicas=0
kubectl scale deployment/agent-gateway --replicas=0

# Восстановить из pre-restore dump
kubectl exec -it postgres-0 -- psql -U invwharf < /backup/pre-restore-<timestamp>.sql.gz

# Поднять CP заново (Шаг 9)
```

---

## Post-incident

1. Написать post-mortem: причина, timeline, impact, root cause, preventive actions.
2. Проверить alert rules: не сработал ли алерт на аномалию вовремя?
3. Проверить backup retention: все ли WAL сегменты были доступны?
4. Обновить RTO/RPO в `docs/nfr-slo.md` если фактические значения превысили целевые.

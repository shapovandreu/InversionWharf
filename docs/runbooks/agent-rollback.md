# Runbook: Откат версии агента (appliance)

**Применяется когда:**
- Новая версия агента вызывает сбои (не подключается, крашится, вызывает проблемы с продуктами)
- Обнаружена критическая ошибка в новой версии после раскатки

**Ссылки:** `TECHNICAL_SPEC.md §2.8`, `§2.9 S2`

---

## Контекст

Агент-апплайанс — это systemd-сервис на хосте организации. Обновление агента выполняется:
1. Оператором вручную (через Dashboard: **Fleet → Agent → Update**).
2. В будущем: автоматически через self-update механизм агента (не реализован в MVP).

Откат производится на сервере организации администратором при поддержке оператора вендора.

---

## Диагностика — убедиться в необходимости отката

### На стороне вендора (Dashboard)

```
Fleet → Agent <agentId> → Status: Offline / Error
Installation: health=Failed
```

### На сервере агента

```bash
# Статус сервиса
systemctl status inversionwharf-agent

# Логи последних 50 строк
journalctl -u inversionwharf-agent -n 50 --no-pager

# Посмотреть код выхода при краше
journalctl -u inversionwharf-agent | grep "Main process exited"
```

Типичные признаки необходимости отката:
- `SIGSEGV`, `panic: ...` в логах
- Цикличный рестарт (systemd restart loop)
- Heartbeat не отправляется > 5 минут (алерт `agent_offline_5m`)
- После обновления продукты перестали запускаться

---

## Процедура отката

### Шаг 1: Определить предыдущую версию

```bash
# На сервере агента — проверить историю установок
ls -lt /opt/inversionwharf/versions/
# Обычно структура: /opt/inversionwharf/versions/<version>/invwharf-agent

# Или из systemd unit
cat /etc/systemd/system/inversionwharf-agent.service | grep ExecStart
```

В Dashboard: **Fleet → Agent → History** — список версий с датами.

### Шаг 2: Остановить текущую версию

```bash
sudo systemctl stop inversionwharf-agent
```

### Шаг 3: Переключить на предыдущую версию

```bash
# Обновить симлинк на предыдущую версию
PREV_VERSION="1.2.3"  # заменить на фактическую предыдущую версию
sudo ln -sfn /opt/inversionwharf/versions/${PREV_VERSION}/invwharf-agent \
  /opt/inversionwharf/current/invwharf-agent

# Или восстановить из бэкапа бинаря (если симлинк не используется)
sudo cp /opt/inversionwharf/backup/invwharf-agent-${PREV_VERSION} \
  /usr/local/bin/invwharf-agent
```

### Шаг 4: Запустить предыдущую версию

```bash
sudo systemctl start inversionwharf-agent
sudo systemctl status inversionwharf-agent
```

### Шаг 5: Верификация

```bash
# Проверить версию
invwharf-agent --version

# Проверить что heartbeat пошёл
journalctl -u inversionwharf-agent -f | grep -m5 heartbeat
```

В Dashboard агент должен перейти в `Online` через ≤ 60 сек.

---

## Массовый откат (несколько агентов)

Если проблемная версия раскачена на несколько агентов:

1. В Dashboard: **Fleet → Agents → Filter by version=<problematic>**
2. Получить список `agentId` затронутых агентов.
3. Для каждой организации: уведомить администратора с инструкцией по откату (шаги 1-5 выше).
4. Если у вендора есть SSH-доступ к серверам (нетипично, только при явном соглашении) — можно выполнить скрипт массового отката.

---

## Предотвращение повторения

После отката:

### 1. Заблокировать проблемную версию в каталоге

В Dashboard: **Catalog → Product: invwharf-agent → Release <version> → Mark as Yanked**

Агенты не будут предлагать эту версию к установке.

### 2. Опубликовать patch-версию

```bash
# После исправления bug:
git tag v1.3.1
git push origin v1.3.1
# CI/CD → сборка → публикация в Harbor → event catalog.release.published
```

### 3. Post-mortem

Документировать в Jira/Confluence:
- Какая версия вызвала проблему?
- Сколько агентов затронуто?
- Время между раскаткой и обнаружением?
- Почему не поймали в тестах?
- Action items: добавить тест, улучшить smoke-тест при обновлении, добавить alerta.

---

## Связанные документы

- [re-enroll.md](re-enroll.md) — если после отката агент не может подключиться (сертификат)
- `docs/edge-cases.md §7` — сценарий Rollout Failure
- `TECHNICAL_SPEC.md §2.9 S2` — сценарий обновления продукта (аналогичная логика)

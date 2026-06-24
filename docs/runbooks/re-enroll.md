# Runbook: Re-enrollment агента

**Применяется когда:**
- mTLS-сертификат агента истёк и агент не может подключиться к agent-gateway
- Агент скомпрометирован и нужно выпустить новый сертификат
- Агент переустанавливается на новом сервере

**Ссылки:** `TECHNICAL_SPEC.md §2.8`, ADR-0004, `openapi/agent-gateway.yaml`

---

## Диагностика — убедиться в необходимости re-enrollment

```bash
# На сервере агента: проверить срок действия сертификата
openssl x509 -in /etc/inversionwharf/agent.crt -noout -dates
# Если notAfter в прошлом → сертификат истёк

# Проверить статус агента
systemctl status inversionwharf-agent

# Посмотреть логи ошибок
journalctl -u inversionwharf-agent -n 50 | grep -i "tls\|cert\|401\|enroll"
```

Признаки истёкшего сертификата в логах:
```
ERRO tls: certificate has expired or is not yet valid
ERRO heartbeat failed: status 401
ERRO lease renewal failed: mTLS handshake error
```

---

## Процедура

### Шаг 1: Деактивировать старого агента в Dashboard (опционально)

Если агент скомпрометирован или переустанавливается на другом сервере:

1. Войти в Dashboard → **Fleet** → найти агента по `agentId`.
2. Нажать **Deactivate** → сертификат добавляется в CRL Vault.
3. Audit log: `AGENT_DEACTIVATED`.

> Если сертификат просто истёк (штатная ситуация) — деактивация не нужна.

### Шаг 2: Создать нового агента и получить персонализированный installer

1. Dashboard → **Fleet** → **Add Agent**.
2. Заполнить поля: имя агента, организация, описание.
3. Нажать **Generate Installer**.
4. Скачать персонализированный installer (подписанный архив с вшитым токеном).

### Шаг 3: Остановить старый агент

```bash
sudo systemctl stop inversionwharf-agent
sudo systemctl disable inversionwharf-agent
```

### Шаг 4: Удалить старые credentials (если переустановка)

```bash
sudo rm -f /etc/inversionwharf/agent.crt
sudo rm -f /etc/inversionwharf/agent.key
sudo rm -f /etc/inversionwharf/ca-bundle.pem
sudo rm -f /etc/inversionwharf/registry-credentials.json
```

> **Внимание:** Если данные нужно сохранить (конфиги продуктов, compose-файлы), сначала сделайте бэкап `/etc/inversionwharf/`.

### Шаг 5: Запустить новый installer

```bash
# Распаковать и запустить
chmod +x invwharf-installer-<agentName>.run
sudo ./invwharf-installer-<agentName>.run

# Installer автоматически:
# 1. Проверяет Docker Engine версию
# 2. Генерирует CSR
# 3. Вызывает POST /v1/enroll с токеном из архива
# 4. Сохраняет сертификат и CA bundle
# 5. Сохраняет Harbor robot credentials
# 6. Запускает и включает systemd unit
```

### Шаг 6: Верификация

```bash
# Проверить что агент запущен
systemctl status inversionwharf-agent

# Проверить heartbeat (в Dashboard: агент должен появиться как Online)
journalctl -u inversionwharf-agent -n 20

# Проверить срок действия нового сертификата
openssl x509 -in /etc/inversionwharf/agent.crt -noout -dates
```

Ожидаемый вывод в Dashboard: агент переходит в статус `Online` через ≤ 60 секунд (первый heartbeat).

---

## Rollback

Если новый enrollment не удался (installer завершился с ошибкой):

```bash
# Посмотреть лог installer
cat /var/log/invwharf-install.log

# Типичные ошибки:
# "enrollment token already used" → токен уже был использован, нужен новый installer
# "connection refused" → agent-gateway недоступен, проверить DNS и сеть
# "docker version check failed" → Docker Engine не установлен или версия < 24
```

Если нужно вернуть предыдущие credentials:
```bash
# Восстановить из бэкапа
sudo cp /backup/inversionwharf/agent.crt /etc/inversionwharf/agent.crt
sudo systemctl start inversionwharf-agent
# Если сертификат ещё действителен — агент должен подключиться
```

---

## Связанные документы

- [ca-rotation.md](ca-rotation.md) — ротация CA
- `docs/key-rotation.md` — ротация ключей лицензирования
- ADR-0004 — решение о персонализированном installer

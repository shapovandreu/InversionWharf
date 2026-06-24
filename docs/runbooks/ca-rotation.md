# Runbook: Ротация CA (Vault PKI — агентские mTLS-сертификаты)

**Применяется когда:**
- CA-сертификат вендора (Vault intermediate CA) скомпрометирован
- Плановая ротация (срок действия CA подходит к концу, обычно раз в 3-5 лет)
- Регуляторное требование смены алгоритма (например, миграция с RSA на Ed25519)

**Ссылки:** `TECHNICAL_SPEC.md §2.7 Vault PKI`, `§3.7`

> **Предупреждение:** Ротация CA требует re-enrollment ВСЕХ агентов. Планируйте maintenance window.

---

## Подготовка (за 30 дней до ротации)

### 1. Уведомить всех операторов

Разослать уведомление: «Ротация CA запланирована на <дата>. Все агенты должны будут пройти re-enrollment.»

### 2. Оценить fleet

```bash
# Количество зарегистрированных агентов
curl -H "Authorization: Bearer $OPERATOR_JWT" \
  https://api.vendor.example.com/v1/fleet/agents?status=active | jq '.total'
```

### 3. Создать резервную копию текущего CA

```bash
# На сервере с доступом к Vault
vault read pki_int/cert/ca > /secure/backup/ca-cert-$(date +%Y%m%d).pem
vault read pki_int/cert/ca_chain > /secure/backup/ca-chain-$(date +%Y%m%d).pem
```

---

## Процедура ротации

### Фаза 1: Создать новый intermediate CA в Vault

```bash
# Сгенерировать новый intermediate CA
vault pki tune -max-lease-ttl=87600h pki_int_v2

vault write pki_int_v2/intermediate/generate/internal \
  common_name="InversionWharf Intermediate CA v2" \
  key_type=rsa \
  key_bits=4096 \
  > /tmp/pki_int_v2_csr.json

# Подписать новый CA корневым CA
vault write pki/root/sign-intermediate \
  csr=$(jq -r '.data.csr' /tmp/pki_int_v2_csr.json) \
  format=pem_bundle \
  ttl="43800h" \
  > /tmp/pki_int_v2_cert.json

# Установить подписанный сертификат
vault write pki_int_v2/intermediate/set-signed \
  certificate=$(jq -r '.data.certificate' /tmp/pki_int_v2_cert.json)
```

### Фаза 2: Период двойной поддержки

agent-gateway должен принимать сертификаты, выданные ОБОИМИ CA:

```yaml
# Конфиг agent-gateway (добавить новый CA bundle)
tls:
  trusted_cas:
    - /etc/ssl/ca/pki_int_v1.pem   # старый CA
    - /etc/ssl/ca/pki_int_v2.pem   # новый CA
```

Перезапустить agent-gateway с новой конфигурацией:
```bash
kubectl rollout restart deployment/agent-gateway
kubectl rollout status deployment/agent-gateway
```

### Фаза 3: Re-enrollment всех агентов

Для каждого агента в fleet:

1. Сгенерировать новый персонализированный installer в Dashboard (новый токен + новый CA bundle).
2. Передать installer администратору организации (как при первоначальной установке).
3. Администратор запускает re-enrollment (см. [re-enroll.md](re-enroll.md)).

Для массовой рассылки через API:
```bash
# Получить список всех организаций
curl -H "Authorization: Bearer $OPERATOR_JWT" \
  https://api.vendor.example.com/v1/fleet/tenants | jq -r '.[].tenantId' \
  > /tmp/tenant_ids.txt

# Для каждого тенанта — создать задачу на re-enrollment
while read TENANT_ID; do
  curl -X POST -H "Authorization: Bearer $OPERATOR_JWT" \
    https://api.vendor.example.com/v1/fleet/tenants/$TENANT_ID/agents/re-enroll-required \
    -d '{"reason": "CA rotation scheduled 2026-09-01"}'
done < /tmp/tenant_ids.txt
```

### Фаза 4: Мониторинг прогресса

```bash
# Agents still on old CA (certificate issuer = pki_int_v1)
# В Dashboard → Fleet → фильтр по "CA version = v1"

# Или через API: проверить lastSeen и версию CA в агентских сертификатах
curl -H "Authorization: Bearer $OPERATOR_JWT" \
  "https://api.vendor.example.com/v1/fleet/agents?ca_version=v1" | jq '.total'
```

**Целевой показатель:** 100% агентов перешли на новый CA.

### Фаза 5: Отзыв старого CA

Только после того как ВСЕ агенты прошли re-enrollment:

```bash
# Убрать старый CA из trusted CAs в agent-gateway
# Обновить конфиг:
tls:
  trusted_cas:
    - /etc/ssl/ca/pki_int_v2.pem   # только новый CA

kubectl rollout restart deployment/agent-gateway

# Отозвать все сертификаты, выданные старым CA
vault write pki_int/root/sign-revocation \
  issuer_ref="pki_int_v1"
```

---

## Экстренная ротация (при компрометации)

При компрометации ключа старого CA выполнить всё то же, но:

1. **Немедленно** перейти к Фазе 1 и Фазе 2.
2. Немедленно отозвать все сертификаты старого CA через CRL:
   ```bash
   vault write pki_int/revoke serial_number="*"  # все сертификаты
   ```
3. agent-gateway начинает отклонять старые сертификаты.
4. Агенты теряют доступ — нужен экстренный re-enrollment.
5. Уведомить всех операторов об инциденте с инструкцией по re-enrollment.

---

## Проверка после ротации

```bash
# Убедиться что новые агенты получают сертификаты от нового CA
openssl x509 -in /etc/inversionwharf/agent.crt -noout -issuer
# Должно содержать "InversionWharf Intermediate CA v2"

# Проверить срок действия
openssl x509 -in /etc/inversionwharf/agent.crt -noout -dates

# Проверить что heartbeat работает
journalctl -u inversionwharf-agent -n 10 | grep heartbeat
```

---

## Rollback

Если новый CA вызвал проблемы:

```bash
# Вернуть старый CA в trusted list (если ещё не отозван)
tls:
  trusted_cas:
    - /etc/ssl/ca/pki_int_v1.pem
    - /etc/ssl/ca/pki_int_v2.pem

kubectl rollout restart deployment/agent-gateway
```

Агенты со старыми сертификатами снова получат доступ.

---

## Связанные документы

- [re-enroll.md](re-enroll.md) — re-enrollment одного агента
- `docs/key-rotation.md` — ротация ключей подписи лицензий

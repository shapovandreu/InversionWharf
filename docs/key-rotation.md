# Ротация ключа подписи лицензий

**Ссылки:** ADR-0003, `TECHNICAL_SPEC.md §2.15`, `§3.7`

---

## Когда выполнять

- **Компрометация:** немедленно, после подтверждения инцидента
- **Плановая:** раз в 2 года или по регламенту ИБ

---

## Используемые ключи

| Ключ | Где хранится | Что подписывает |
|---|---|---|
| `license-signing-key` (Ed25519 или RSA-2048) | Vault, mount `secret/license` | JWT-лизы (`leaseToken`) |
| `vendor-ca-key` (RSA-4096) | Vault PKI, mount `pki_int` | mTLS-сертификаты агентов |

> Данный документ описывает ротацию `license-signing-key`. Ротацию `vendor-ca-key` см. в [ca-rotation.md](runbooks/ca-rotation.md).

---

## Цена ротации

Публичный ключ вендора **пиннут в License SDK** внутри каждого Docker-образа продукта.
Ротация требует:
1. Обновления SDK во всех языках (Go, Java, Node).
2. Пересборки всех продуктовых образов с новым SDK.
3. Переподписи всех образов cosign.
4. Раскатки обновлённых образов всем клиентам **до** момента, когда старый ключ перестаёт работать на license-service.

Неправильная последовательность → массовый HardStop у клиентов.

---

## Процедура ротации

### Фаза 0 — Подготовка (за 2 недели до ротации)

1. Создать задачу в Jira с типом `Security — Key Rotation`, назначить ответственного.
2. Уведомить команды разработки SDK (Go, Java, Node) о дедлайне.
3. Согласовать maintenance window с бизнесом.

### Фаза 1 — Генерация нового ключа

```bash
# На защищённой рабочей станции с доступом к Vault
vault write pki/license/keys/generate/exported \
  key_type=ed25519 \
  key_name=license-signing-key-v2

# Сохранить публичный ключ для встройки в SDK
vault read -field=public_key pki/license/keys/by-name/license-signing-key-v2 \
  > /secure/license-public-key-v2.pem
```

### Фаза 2 — Период двойной поддержки

license-service должен поддерживать **оба ключа** одновременно:
- Новые лизы подписываются новым ключом.
- Существующие лизы (подписанные старым ключом) продолжают приниматься до истечения.
- Максимальный TTL лиза + grace period = **72 ч + 7 дней ≈ 10 дней**.

```yaml
# Конфиг license-service (application.yaml)
license:
  signing:
    active-key: "license-signing-key-v2"
    trusted-keys:
      - "license-signing-key-v1"   # принимать до X дата
      - "license-signing-key-v2"
  trusted-keys-retire-date: "2026-08-01T00:00:00Z"
```

### Фаза 3 — Обновление SDK

Для каждого языка:

```go
// go/license-sdk/keys.go
var VendorPublicKey = []byte(`-----BEGIN PUBLIC KEY-----
<новый публичный ключ>
-----END PUBLIC KEY-----`)
```

- Тег релиза SDK: `vX.Y.Z` (семвер, мажор при смене алгоритма, минор при ротации).
- Матрица тестирования: Go 1.22+, Java 21+, Node 20+ LTS.

### Фаза 4 — Пересборка и переподпись образов

```bash
# Для каждого продукта
docker build --build-arg LICENSE_SDK_VERSION=vX.Y.Z \
  -t harbor.vendor.example.com/<product>/gateway:<version> .

# Переподпись cosign
cosign sign --key /secure/cosign.key \
  harbor.vendor.example.com/<product>/gateway@sha256:<digest>
```

### Фаза 5 — Раскатка клиентам

1. Опубликовать новые образы в Harbor.
2. Kafka-событие `catalog.release.published` → агенты получат новые образы при очередном опросе.
3. **Контрольная точка:** ≥ 95% агентов перешли на новые образы (мониторинг в Grafana → Fleet dashboard).

### Фаза 6 — Отзыв старого ключа

Только после того как все агенты обновились (или истёк период двойной поддержки):

```bash
# Убрать старый ключ из trusted-keys в license-service
vault delete secret/license/trusted-keys/license-signing-key-v1

# Перезапустить license-service для применения конфига
kubectl rollout restart deployment/license-service
```

### Фаза 7 — Верификация

```bash
# Проверить что старые JWT больше не принимаются
curl -X POST https://agent-gateway.vendor.example.com/v1/licenses/lease \
  -H "Content-Type: application/json" \
  -d '{"productId": "<test-uuid>"}' \
  # Ожидаем 200 с лизом, подписанным новым ключом

# Попытка использовать старый лиз (заголовок с kid старого ключа)
# Ожидаем 401
```

---

## Откат

Если на Фазе 5 обнаружена проблема (массовый HardStop):

1. Немедленно восстановить старый ключ в trusted-keys на license-service.
2. license-service снова выдаёт лизы, подписанные новым ключом, но принимает оба.
3. Диагностировать причину (неверный публичный ключ в SDK? ошибка пересборки?).
4. Повторить Фазы 3-5 с исправлением.

---

## Связанные документы

- [ca-rotation.md](runbooks/ca-rotation.md) — ротация CA для mTLS агентов
- ADR-0003 — архитектурное решение о двухслойном лицензировании
- `TECHNICAL_SPEC.md §2.15` — механика лицензирования

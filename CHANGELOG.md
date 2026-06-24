# Changelog

Все значимые изменения в спецификации и документации фиксируются здесь.
Формат: [Semantic Versioning](https://semver.org/). Тип изменений: `Added`, `Changed`, `Fixed`, `Removed`, `Security`.

---

## [4.1.0] — 2026-06-24

### Security
- Закрыта дыра: cached-образы оставались на диске после HardStop/отзыва гранта. Добавлен `CleanupManager` в агент (`TECHNICAL_SPEC.md §2.8`).
- Закрыта дыра: Harbor robot credentials не обновлялись при отзыве гранта. Добавлен Kafka-топик `entitlement.grant.status.changed`; fleet-service сужает scope robot-аккаунта автоматически (`§2.4`, `§2.7`).
- Закрыта дыра: enrollment-токен передавался отдельно от installer в открытом виде. Installer теперь персонализированный (токен вшит в подписанный архив) (`§2.8`, `§2.9 S1`).

### Added
- ADR-0001..0004 в `docs/adr/`.
- OpenAPI 3.1 контракты для agent-gateway и operator-gateway в `openapi/`.
- AsyncAPI контракт для Kafka-топиков в `asyncapi/`.
- `docs/threat-model.md` — полная STRIDE-модель с trust boundaries.
- `docs/nfr-slo.md` — SLO и capacity-модель.
- `docs/edge-cases.md` — сценарии отказов и граничные случаи.
- `docs/key-rotation.md` — процедура ротации ключей подписи лиза.
- `docs/runbooks/` — четыре операционные инструкции.
- `docker-compose.yml` и `.env.example` для учебной версии.
- `README.md` и `CONTRIBUTING.md`.

### Changed
- Агент-апплайанс: убран вендоринг Docker Engine; Docker ≥ 24 — предусловие хоста (`§1.1`, `§2.8`, `§2.14`).
- `§3.5`: Harbor isolation теперь включает автоматическое сужение scope при revocation.
- `§3.9`: таблица угроз обновлена по всем трём закрытым дырам.

---

## [4.0.0] — 2026-06 (до начала работы в этой сессии)

### Added
- Полная спецификация системы (`TECHNICAL_SPEC.md`): три раздела — бизнес-идея, реализация, безопасность.
- Учебный план (`EDU_PROJECT_PLAN.md`): учебная версия под 3–4 ДЗ на Spring.
- Jira-бэклог (`JIRA_BACKLOG.md`): 13 эпиков, 30+ историй с SP и AC.
- Статические HTML-демо (`demo/`): Operator Dashboard и Agent LocalUI.

---

## [3.0.0] — предыдущая версия (не зафиксирована)

История версий до v4.0 в репозитории не сохранена.

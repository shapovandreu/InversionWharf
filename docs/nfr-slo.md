# NFR и SLO — InversionWharf

**Ссылки:** `TECHNICAL_SPEC.md §2.14`, `§3.10`

---

## 1. Service Level Objectives (SLO)

### 1.1 Agent Gateway (критичный путь)

| Метрика | SLO | Измерение |
|---|---|---|
| Availability | 99.9% / месяц (≤ 43 мин даунтайма) | % успешных запросов (non-5xx) |
| Latency p99 `/v1/heartbeat` | < 200 мс | Percentile по всем агентам |
| Latency p99 `/v1/licenses/lease` | < 500 мс | Percentile |
| Latency p99 `/v1/products` | < 1 сек (stale-ok) | Percentile |
| Throughput | 10 000 агентов × heartbeat / 60 сек ≈ **167 RPS** пиковый | Ratelimit конфиг |

> Heartbeat-шторм при рестарте: 10 000 агентов одновременно ≈ 10 000 RPS за 1 минуту.
> Требуется exponential backoff + jitter в агенте.

### 1.2 Operator API (Dashboard)

| Метрика | SLO |
|---|---|
| Availability | 99.5% / месяц |
| Latency p95 | < 500 мс |
| Latency p99 | < 2 сек |

### 1.3 License Service (критичный для лизов)

| Метрика | SLO |
|---|---|
| Availability | 99.9% / месяц |
| Latency p99 `/licenses/leases` | < 300 мс |
| Lease issuance throughput | ≥ 1 000 лизов/сек |

### 1.4 Kafka (Event Bus)

| Метрика | SLO |
|---|---|
| Consumer lag `fleet.heartbeat` | < 10 000 сообщений |
| Consumer lag `entitlement.grant.status.changed` | < 1 000 сообщений |
| End-to-end latency события | < 30 сек p99 |

---

## 2. Capacity Model

### 2.1 Целевые параметры роста

| Горизонт | Агентов | Организаций | Продуктов |
|---|---|---|---|
| Launch (MVP) | 500 | 50 | 5 |
| Year 1 | 5 000 | 500 | 20 |
| Year 3 | 50 000 | 5 000 | 100 |

### 2.2 Трафик agent-gateway (Year 1, 5 000 агентов)

| Событие | Частота | RPS |
|---|---|---|
| Heartbeat | каждые 60 сек | **83 RPS** steady |
| Installation report | каждые 10 мин | **8.3 RPS** |
| License lease | каждые 48 ч | **0.03 RPS** (negligible) |
| Products poll | каждые 30 мин | **2.8 RPS** |
| **Итого steady-state** | | **~95 RPS** |
| **Пик (рестарт 20% агентов)** | | **~400 RPS** |

### 2.3 Sizing (Year 1)

| Компонент | Минимум | Рекомендация |
|---|---|---|
| agent-gateway | 2 pods × 2 CPU / 4 GB | 3 pods × 4 CPU / 8 GB |
| license-service | 2 pods × 2 CPU / 4 GB | 2 pods × 4 CPU / 8 GB |
| catalog-service | 2 pods × 1 CPU / 2 GB | 2 pods × 2 CPU / 4 GB |
| entitlement-service | 2 pods × 1 CPU / 2 GB | 2 pods × 2 CPU / 4 GB |
| fleet-service | 2 pods × 1 CPU / 2 GB | 2 pods × 2 CPU / 4 GB |
| PostgreSQL | 8 CPU / 32 GB / 500 GB SSD | + Read replica |
| Redis | 2 CPU / 8 GB | Redis Cluster 3 nodes |
| Kafka | 3 brokers × 4 CPU / 16 GB | 3 brokers + ZooKeeper / KRaft |
| Harbor | 4 CPU / 16 GB + 5 TB storage | Distributed mode |

### 2.4 Database sizing

| Таблица | Строк (Year 1) | Рост |
|---|---|---|
| agents | 5 000 | линейный по агентам |
| heartbeats (rolling 30d) | ~216 M (5k агентов × 24h × 30d × 60) | partitioned by date |
| installation_snapshots | ~2.5 M (5k × 20 продуктов × 25 версий) | upsert, не растёт быстро |
| leases | ~1.5 M (5k агентов × 20 продуктов × 15 выдач) | TTL cleanup |
| audit_events | ~50 M/год | append-only, 365d retention |

> heartbeats должны быть в time-series БД (TimescaleDB, InfluxDB) или в партиционированной таблице PostgreSQL с DROP PARTITION.

---

## 3. NFR (Non-Functional Requirements)

### 3.1 Доступность и отказоустойчивость

| # | Требование |
|---|---|
| NFR-1 | Потеря одного AZ не влияет на обслуживание агентов (multi-AZ deployment) |
| NFR-2 | Плановое обслуживание CP не вызывает HardStop (grace period ≥ 7 дней покрывает maintenance) |
| NFR-3 | Агент продолжает работу при недоступности CP ≥ 7 дней (кэшированный лиз) |
| NFR-4 | Circuit breaker открывается при error rate > 50% за 10 сек (Resilience4j) |
| NFR-5 | Stale-ok ответ от agent-gateway при деградации catalog-service (Redis cache TTL 1 ч) |

### 3.2 Безопасность

| # | Требование |
|---|---|
| NFR-6 | Все внешние соединения agent-gateway — mTLS (TLS 1.3+) |
| NFR-7 | JWT TTL ≤ 4 ч для операторов, ≤ 72 ч для лизов агентов |
| NFR-8 | Vault auto-unseal; audit log Vault включён |
| NFR-9 | Ротация robot credentials Harbor — раз в 90 дней (автоматическая, fleet-service) |
| NFR-10 | Все Docker-образы подписаны cosign; агент отказывает неподписанным образам |

### 3.3 Наблюдаемость

| # | Требование |
|---|---|
| NFR-11 | Structured logging (JSON) во всех сервисах; traceId пробрасывается через X-Trace-Id |
| NFR-12 | Prometheus метрики на `/actuator/prometheus` (Spring) и `/metrics` (agent) |
| NFR-13 | SLO dashboards в Grafana: availability, latency p50/p95/p99, error rate |
| NFR-14 | Алерт при agent offline > 5 мин (heartbeat gap) |
| NFR-15 | Алерт при consumer lag `entitlement.grant.status.changed` > 1 000 |

### 3.4 Совместимость хоста (агент)

| # | Требование |
|---|---|
| NFR-16 | Linux x86_64 и arm64 |
| NFR-17 | Docker Engine ≥ 24 или containerd ≥ 1.7 + Compose v2 |
| NFR-18 | systemd ≥ 232 |
| NFR-19 | Installer ≤ 10 MB (Go static binary + embedded token) |
| NFR-20 | RAM агента (daemon) ≤ 128 MB idle; ≤ 256 MB под нагрузкой |

### 3.5 Data Retention

| Данные | Retention |
|---|---|
| Heartbeats | 30 дней (rolling) |
| Installation snapshots | бессрочно (upsert) |
| Audit events | 365 дней (compliance) |
| License leases | 90 дней после истечения |
| Kafka `fleet.heartbeat` | 7 дней |
| Kafka `audit.operator.action` | 365 дней |

---

## 4. Alerting Runbook Links

| Алерт | Runbook |
|---|---|
| `agent_offline_5m` | [agent-rollback.md](runbooks/agent-rollback.md) |
| `license_service_down` | Restart pod; if persistent → check Vault unsealed |
| `kafka_consumer_lag_high` | Scale consumer pods; check for poison-pill messages |
| `vault_sealed` | Manual unseal procedure (ops team only) |
| `harbor_robot_expired` | [re-enroll.md](runbooks/re-enroll.md) |

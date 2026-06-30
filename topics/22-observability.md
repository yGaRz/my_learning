# 22. Наблюдаемость (логи, метрики, трейсинг)

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание трёх столпов наблюдаемости, структурного логирования, распределённого трейсинга и алертинга.

## Ключевые вопросы (чеклист)
- [x] Три столпа: logs, metrics, traces
- [x] Monitoring vs Observability
- [x] Структурное логирование (Serilog)
- [x] ILogger, scopes, message templates
- [x] Correlation/trace id
- [x] Distributed tracing (OpenTelemetry)
- [x] Метрики: counter/gauge/histogram, RED/USE
- [x] Prometheus/Grafana
- [x] APM
- [x] Health checks
- [x] Алертинг, SLI/SLO/SLA
- [x] Перцентили
- [x] Sampling
- [x] Логи в контейнерах

## Разбор вопросов

### Вопрос: Три столпа наблюдаемости
**Краткий ответ:**
- **Logs** — дискретные события с контекстом (что произошло).
- **Metrics** — числовые агрегаты во времени (сколько/как быстро): счётчики, latency.
- **Traces** — путь запроса через систему/сервисы (где время теряется).
Вместе дают полную картину поведения системы.

---

### Вопрос: Monitoring vs Observability
**Краткий ответ:** **Monitoring** — отслеживание известных метрик/состояний (предопределённые дашборды/алерты). **Observability** — способность понять **новые, неизвестные заранее** проблемы по выходным данным системы (богатая телеметрия позволяет задавать вопросы постфактум).

---

### Вопрос: Структурное логирование
**Краткий ответ:** Логи как структурированные данные (ключ-значение/JSON), а не плоский текст → можно фильтровать/искать/агрегировать. Serilog — популярный выбор; sinks в разные хранилища.

```csharp
logger.LogInformation("Order {OrderId} created for {CustomerId}", orderId, customerId);
// orderId/customerId — структурированные поля, а не просто текст
```

---

### Вопрос: ILogger, scopes, message templates
**Краткий ответ:** `ILogger<T>` — абстракция логирования с уровнями (Trace..Critical). **Message templates** (`{OrderId}`) сохраняют структуру (не интерполяция строк!). **Scopes** (`BeginScope`) добавляют контекст ко всем логам внутри (например, requestId).

**Подводные камни:** строковая интерполяция в логах ломает структуру и всегда вычисляется (даже если уровень отключён).

---

### Вопрос: Correlation / trace id
**Краткий ответ:** Сквозной идентификатор запроса, проходящий через все сервисы/логи → можно собрать всю цепочку одного запроса. В ASP.NET Core есть `Activity.TraceId`; пробрасывается через заголовки (W3C `traceparent`).

---

### Вопрос: Distributed tracing (OpenTelemetry)
**Краткий ответ:** **OpenTelemetry** — стандарт сбора telemetry (traces/metrics/logs). Trace состоит из **spans** (операций) с родительско-дочерними связями; контекст передаётся между сервисами (**W3C Trace Context**). Экспорт в Jaeger/Zipkin/Tempo/APM. В .NET — через `System.Diagnostics.Activity` + OTel SDK.

---

### Вопрос: Метрики — типы, RED/USE
**Краткий ответ:**
- **Counter** — монотонно растёт (число запросов).
- **Gauge** — текущее значение (использование памяти, размер очереди).
- **Histogram** — распределение (latency по бакетам → перцентили).
- **RED** (для сервисов): Rate, Errors, Duration.
- **USE** (для ресурсов): Utilization, Saturation, Errors.

---

### Вопрос: Prometheus / Grafana
**Краткий ответ:** **Prometheus** — pull-based сбор и хранение метрик (scrape endpoint), запросы PromQL. **Grafana** — визуализация/дашборды/алерты. В .NET метрики через `System.Diagnostics.Metrics` + OTel/Prometheus exporter.

---

### Вопрос: APM
**Краткий ответ:** Application Performance Monitoring — комплексные платформы (Application Insights, Datadog, New Relic, Dynatrace, Elastic APM): traces+metrics+logs+алерты в одном месте, автоинструментирование.

---

### Вопрос: Health checks
**Краткий ответ:** см. [темы 10, 21](./10-aspnet-core.md). Эндпоинты liveness/readiness + проверки зависимостей; используются оркестратором и мониторингом.

---

### Вопрос: Алертинг, SLI / SLO / SLA
**Краткий ответ:**
- **SLI** — измеряемый индикатор (например, % успешных запросов, p99 latency).
- **SLO** — целевое значение SLI (99.9% успешных).
- **SLA** — внешнее обязательство с последствиями за нарушение.
- **Error budget** — допустимый объём ошибок в рамках SLO; исчерпан → стоп фич, фокус на надёжности.
Алерты — на симптомы для пользователя (SLO violation), а не на каждый шум.

---

### Вопрос: Перцентили
**Краткий ответ:** см. [тему 20](./20-performance.md). p50/p95/p99 latency отражают реальный опыт и «хвосты»; среднее вводит в заблуждение.

---

### Вопрос: Sampling
**Краткий ответ:** Сбор всех трейсов дорог (объём/стоимость). Sampling (head/tail-based) сохраняет часть трейсов (и/или все ошибочные/медленные). Баланс полноты и стоимости.

---

### Вопрос: Логи в контейнерах
**Краткий ответ:** Писать в **stdout/stderr** (12-factor), а не в файлы; сбор и агрегация — внешними средствами (ELK/Loki/Fluentd → централизованный поиск). Не хранить состояние логов в контейнере.

---

## Полезные ссылки
- Observability in .NET: https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel
- OpenTelemetry: https://opentelemetry.io/docs/
- Google SRE (SLI/SLO): https://sre.google/books/
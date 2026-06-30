# 21. DevOps, CI/CD, Docker, Kubernetes

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание сборки/доставки, контейнеризации, оркестрации и того, как .NET-приложение живёт в проде.

## Ключевые вопросы (чеклист)
- [x] CI и CD (delivery vs deployment)
- [x] Пайплайн
- [x] Стратегии деплоя
- [x] Версионирование, semver
- [x] Trunk-based vs GitFlow
- [x] Docker: образ vs контейнер, слои
- [x] Multi-stage build для .NET
- [x] Размер образа (chiseled/distroless)
- [x] Конфигурация в контейнере
- [x] Healthcheck, graceful shutdown
- [x] k8s: Pod, Deployment, Service, Ingress
- [x] ConfigMap, Secret
- [x] Probes
- [x] Requests/limits, HPA
- [x] Rolling update, rollback
- [x] Helm
- [x] .NET в k8s (GC под лимиты)

## CI/CD

### CI и CD
**Краткий ответ:**
- **CI (Continuous Integration)** — частые слияния в общую ветку с автоматической сборкой и тестами (раннее обнаружение проблем).
- **Continuous Delivery** — артефакт всегда готов к деплою, релиз — по кнопке.
- **Continuous Deployment** — автоматический деплой в прод после прохождения пайплайна.

### Пайплайн
**Краткий ответ:** restore → build → test → static analysis (линтеры, SonarQube, security scan) → package (NuGet/Docker image) → deploy (по окружениям dev/stage/prod) → smoke tests. Быстрая обратная связь, fail fast.

### Стратегии деплоя
**Краткий ответ:**
- **Rolling** — постепенная замена инстансов (по умолчанию в k8s).
- **Blue-Green** — два окружения, переключение трафика мгновенно, лёгкий откат.
- **Canary** — выкатка на малую долю трафика, постепенное расширение при отсутствии ошибок.
- **Feature flags** — включение функционала без передеплоя.

### Версионирование, semver
**Краткий ответ:** SemVer — MAJOR.MINOR.PATCH (ломающие/фичи/фиксы). Артефакты версионируются, образы тегируются (не только `latest`).

### Trunk-based vs GitFlow
**Краткий ответ:**
- **Trunk-based** — короткие ветки, частые мерджи в main, feature flags. Хорошо для CI/CD.
- **GitFlow** — develop/release/hotfix ветки; тяжелее, подходит для редких релизов/версионируемых продуктов.

## Docker

### Образ vs контейнер, слои
**Краткий ответ:** **Образ** — неизменяемый шаблон (слои файловой системы). **Контейнер** — запущенный экземпляр образа. Слои кэшируются и переиспользуются → порядок инструкций в Dockerfile влияет на скорость сборки.

### Multi-stage build для .NET
**Краткий ответ:** Сборка в SDK-образе, запуск — в маленьком runtime-образе (без SDK) → меньше размер и поверхность атаки.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore           # кэшируется, если csproj не менялся
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Размер образа
**Краткий ответ:** Использовать chiseled/distroless/alpine runtime-образы, multi-stage, `.dockerignore`, Native AOT (минимальный образ). Меньше размер → быстрее деплой и меньше уязвимостей.

### Конфигурация в контейнере
**Краткий ответ:** Через переменные окружения (12-factor), не хардкодить. Секреты — через секрет-механизмы оркестратора, не в образ.

### Healthcheck, graceful shutdown
**Краткий ответ:** HEALTHCHECK/health endpoints для оркестратора. Приложение должно корректно обрабатывать **SIGTERM** (дозавершить запросы, закрыть соединения) — в .NET через `IHostApplicationLifetime`/graceful shutdown хоста.

## Kubernetes

### Pod, Deployment, Service, Ingress
**Краткий ответ:**
- **Pod** — минимальная единица (один+ контейнер).
- **Deployment** — управляет ReplicaSet (число реплик, rolling update).
- **Service** — стабильная сетевая точка/балансировка к подам.
- **Ingress** — HTTP-маршрутизация снаружи в сервисы (хосты/пути, TLS).

### ConfigMap, Secret
**Краткий ответ:** ConfigMap — неконфиденциальная конфигурация; Secret — чувствительные данные (base64, желательно с шифрованием at-rest/внешним менеджером). Монтируются как env/файлы.

### Probes
**Краткий ответ:**
- **Liveness** — жив ли контейнер (если нет — рестарт).
- **Readiness** — готов ли принимать трафик (если нет — убрать из балансировки).
- **Startup** — для медленного старта (защищает от преждевременного liveness).

### Requests/limits, HPA
**Краткий ответ:** **Requests** — гарантированные ресурсы (для планирования), **limits** — потолок (CPU throttling, OOM kill при превышении памяти). **HPA** — автоскейлинг подов по метрикам (CPU/память/кастомные).

### Rolling update, rollback
**Краткий ответ:** Deployment обновляет поды постепенно (maxSurge/maxUnavailable), при проблеме — `kubectl rollout undo` к предыдущей версии.

### Helm
**Краткий ответ:** Пакетный менеджер k8s: шаблоны манифестов + values для окружений, версионирование релизов.

### .NET в k8s (GC под лимиты)
**Краткий ответ:** .NET учитывает cgroup-лимиты (CPU/память). Под memory limit важно следить за Server GC (heap на ядро) — настраивать `DOTNET_gcServer`/`DOTNET_GCHeapHardLimit`, иначе OOM kill. Логи в stdout (см. тему 22).

---

## Полезные ссылки
- .NET in containers: https://learn.microsoft.com/dotnet/core/docker/build-container
- Kubernetes docs: https://kubernetes.io/docs/
- .NET + cgroup limits / GC: https://learn.microsoft.com/dotnet/core/runtime-config/garbage-collector
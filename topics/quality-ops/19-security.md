<!-- nav-top -->
[← К оглавлению](../../README.md)

# 19. Безопасность (auth, OWASP)

<a id="top"></a>

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание аутентификации/авторизации, токенов, типичных уязвимостей и защиты от них.

## Ключевые вопросы (чеклист)
- [x] [Аутентификация vs авторизация](#q1)
- [x] [Cookie vs token auth](#q2)
- [x] [JWT: структура, валидация](#q3)
- [x] [Access vs refresh token](#q4)
- [x] [OAuth 2.0 flows](#q5)
- [x] [OpenID Connect](#q6)
- [x] [IdentityServer/Keycloak/Entra](#q7)
- [x] [Authorization в ASP.NET Core](#q8)
- [x] [Где хранить токены](#q9)
- [x] [SQL injection, XSS, CSRF](#q10)
- [x] [Broken access control (IDOR)](#q11)
- [x] [Шифрование at-rest/in-transit](#q12)
- [x] [Хеширование паролей](#q13)
- [x] [Secrets management](#q14)
- [x] [Rate limiting](#q15)
- [x] [Mass assignment](#q16)
- [x] [Безопасные заголовки](#q17)
- [x] [Симметричное vs асимметричное, хеш vs шифрование](#q18)
- [x] [HTTPS/TLS](#q19)
- [x] [Supply-chain](#q20)

## Разбор вопросов

> ⭐ — вопросы, которые реально задавали на собеседовании.

<a id="q-http-security"></a>
### ⭐ Вопрос: Как обеспечить безопасность при работе по HTTP
**Краткий ответ:** Это широкий вопрос — отвечать слоями («defense in depth»):

**1. Транспорт:**
- **HTTPS/TLS** обязателен; HSTS (форсировать https), редирект http→https, актуальные версии TLS, валидные сертификаты.

**2. Аутентификация и авторизация:**
- Надёжная аутентификация (OAuth2/OIDC, JWT/cookie), проверка авторизации на **каждом** защищённом endpoint (не только в UI).
- Принцип наименьших привилегий, проверка владения ресурсом (против IDOR).

**3. Защита от типовых атак (OWASP):**
- **SQL injection** — параметризованные запросы/ORM.
- **XSS** — экранирование вывода, Content Security Policy.
- **CSRF** — anti-forgery токены и `SameSite` cookie (для cookie-based auth).
- **Mass assignment/over-posting** — DTO вместо привязки к доменным сущностям.

**4. Валидация и лимиты:**
- Валидация и санитизация всех входных данных (никогда не доверять клиенту).
- **Rate limiting** и защита от brute-force (см. тему 11), ограничение размера запроса.

**5. Заголовки безопасности:**
- HSTS, `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options`/frame-ancestors, `Referrer-Policy`.
- Корректный **CORS** (только доверенные origin, не `*` для credential-запросов).

**6. Данные и секреты:**
- Шифрование чувствительных данных at-rest и in-transit; хеширование паролей (bcrypt/Argon2).
- Секреты — не в коде/репозитории (Key Vault, user-secrets, env).
- Минимизация данных в ответах/логах (не логировать токены/PII).

**7. Эксплуатация:**
- Не раскрывать стектрейсы/детали в проде (ProblemDetails без внутренностей), актуальные зависимости (audit), мониторинг/алерты на подозрительную активность.

**Пример (заголовки + https):**
```csharp
app.UseHsts();
app.UseHttpsRedirection();
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
    ctx.Response.Headers["X-Frame-Options"] = "DENY";
    ctx.Response.Headers["Content-Security-Policy"] = "default-src 'self'";
    await next();
});
```

**Подводные камни:** авторизация только на фронте; `*` в CORS с credentials; хранение JWT в localStorage (XSS-кража); отсутствие ротации/инвалидации токенов; логирование секретов.

[↑ Наверх](#top)

---

<a id="q1"></a>
### Вопрос: Аутентификация vs авторизация
**Краткий ответ:** Аутентификация — «кто ты» (проверка личности). Авторизация — «что тебе можно» (права на действие/ресурс). Сначала аутентификация, потом авторизация.

[↑ Наверх](#top)

---

<a id="q2"></a>
### Вопрос: Cookie vs token auth
**Краткий ответ:**
- **Cookie-based** — сервер хранит/проверяет сессию или подписанную cookie; браузер шлёт автоматически (нужна защита от CSRF, `SameSite`/`HttpOnly`/`Secure`). Хорошо для одного домена/SSR.
- **Token-based (JWT)** — клиент шлёт токен в заголовке `Authorization`; stateless, удобно для API/SPA/мобильных/микросервисов.

[↑ Наверх](#top)

---

<a id="q3"></a>
### Вопрос: JWT — структура и валидация
**Краткий ответ:** Три части (base64url): **header** (алгоритм), **payload** (claims: sub, exp, iss, aud, roles), **signature** (подпись header+payload секретом/приватным ключом). Сервер валидирует подпись, `exp` (срок), `iss`/`aud`. **JWT не шифруется** — payload читаем (base64), не класть туда секреты.

[↑ Наверх](#top)

---

<a id="q4"></a>
### Вопрос: Access vs refresh token
**Краткий ответ:** **Access token** — короткоживущий (минуты), даёт доступ. **Refresh token** — долгоживущий, хранится надёжно, используется для получения нового access без повторного логина. Ротация refresh-токенов и возможность их отзыва (revocation).

[↑ Наверх](#top)

---

<a id="q5"></a>
### Вопрос: OAuth 2.0 flows
**Краткий ответ:**
- **Authorization Code + PKCE** — для web/SPA/mobile (стандарт сейчас).
- **Client Credentials** — сервис-сервис (без пользователя).
- Устаревшие: Implicit, Password grant — не рекомендуются.
OAuth — про **авторизацию** (делегированный доступ), не аутентификацию.

[↑ Наверх](#top)

---

<a id="q6"></a>
### Вопрос: OpenID Connect
**Краткий ответ:** Слой аутентификации поверх OAuth 2.0; добавляет **id_token** (кто пользователь) и стандартизирует получение профиля. OAuth = доступ, OIDC = логин.

[↑ Наверх](#top)

---

<a id="q7"></a>
### Вопрос: IdentityServer / Keycloak / Entra ID
**Краткий ответ:** Готовые провайдеры identity (IdP), реализующие OAuth2/OIDC: IdentityServer (Duende), Keycloak (open source), Azure Entra ID (бывш. Azure AD), Auth0. Не «изобретать» аутентификацию самим.

[↑ Наверх](#top)

---

<a id="q8"></a>
### Вопрос: Authorization в ASP.NET Core
**Краткий ответ:** Роли (`[Authorize(Roles=...)]`), **policy-based** (`AddPolicy` + requirements/handlers), **claims-based**. Для сложной логики — кастомные `AuthorizationHandler`. Resource-based авторизация для проверки владения конкретным ресурсом.

[↑ Наверх](#top)

---

<a id="q9"></a>
### Вопрос: Где хранить токены
**Краткий ответ:** В браузере: `localStorage` уязвим к XSS (токен украдут). Безопаснее — `HttpOnly` `Secure` `SameSite` cookie (недоступна JS). Trade-off: cookie требует защиты от CSRF.

[↑ Наверх](#top)

---

<a id="q10"></a>
### Вопрос: SQL injection, XSS, CSRF
**Краткий ответ:**
- **SQLi** — внедрение SQL через ввод; защита: параметризация/ORM, никогда не конкатенировать ввод в запрос.
- **XSS** — внедрение скрипта; защита: экранирование вывода, CSP, не вставлять непроверенный HTML.
- **CSRF** — подделка запроса от имени аутентифицированного пользователя; защита: anti-forgery токен, `SameSite` cookie.

[↑ Наверх](#top)

---

<a id="q11"></a>
### Вопрос: Broken access control (IDOR)
**Краткий ответ:** №1 в OWASP Top 10. IDOR — доступ к чужому объекту по угадыванию id (`/orders/123`). Защита: проверять, что ресурс принадлежит текущему пользователю; авторизация на сервере для каждого запроса.

[↑ Наверх](#top)

---

<a id="q12"></a>
### Вопрос: Шифрование at-rest/in-transit
**Краткий ответ:** In-transit — TLS. At-rest — шифрование БД/дисков/полей (TDE, column encryption). Ключи — в KMS/Key Vault, ротация.

[↑ Наверх](#top)

---

<a id="q13"></a>
### Вопрос: Хеширование паролей
**Краткий ответ:** Только медленные адаптивные хеши с солью: **bcrypt/scrypt/Argon2/PBKDF2**. Никогда MD5/SHA-1/«просто SHA-256» (быстрые → brute-force). Соль уникальна на пользователя; опционально pepper.

[↑ Наверх](#top)

---

<a id="q14"></a>
### Вопрос: Secrets management
**Краткий ответ:** Секреты не в коде/git/appsettings в репозитории. Использовать user-secrets (dev), env vars, Azure Key Vault/HashiCorp Vault/AWS Secrets Manager (prod), managed identity. Ротация и аудит доступа.

[↑ Наверх](#top)

---

<a id="q15"></a>
### Вопрос: Rate limiting
**Краткий ответ:** см. [тему 11](./11-request-pipeline-http-highload.md). Защита от brute-force/DoS/abuse; лимиты по IP/пользователю/ключу.

[↑ Наверх](#top)

---

<a id="q16"></a>
### Вопрос: Mass assignment / over-posting
**Краткий ответ:** Привязка лишних полей из запроса к модели (например `IsAdmin`). Защита: DTO только с разрешёнными полями, не биндить напрямую доменные сущности.

[↑ Наверх](#top)

---

<a id="q17"></a>
### Вопрос: Безопасные заголовки
**Краткий ответ:** HSTS, CSP, `X-Content-Type-Options: nosniff`, `X-Frame-Options`/frame-ancestors, `Referrer-Policy`, `Permissions-Policy`. Снижают классы атак (clickjacking, MIME-sniffing, XSS).

[↑ Наверх](#top)

---

<a id="q18"></a>
### Вопрос: Симметричное vs асимметричное, хеш vs шифрование vs кодирование
**Краткий ответ:**
- **Симметричное** (AES) — один ключ; быстро, для данных.
- **Асимметричное** (RSA/ECC) — пара ключей; обмен ключами/подписи.
- **Хеш** — односторонний (нельзя расшифровать), для проверки целостности/паролей.
- **Шифрование** — обратимо при наличии ключа (конфиденциальность).
- **Кодирование** (base64) — НЕ безопасность, просто представление.

[↑ Наверх](#top)

---

<a id="q19"></a>
### Вопрос: HTTPS/TLS handshake
**Краткий ответ:** Клиент и сервер согласуют версию/шифры, сервер предъявляет сертификат (проверка цепочки доверия), обмен ключами (асимметрия) → согласуется симметричный сеансовый ключ → дальше симметричное шифрование. TLS 1.3 упрощает и ускоряет handshake.

[↑ Наверх](#top)

---

<a id="q20"></a>
### Вопрос: Supply-chain security
**Краткий ответ:** Зависимости — вектор атаки. `dotnet list package --vulnerable`, NuGet audit, фиксация версий, проверенные источники, SBOM, обновления. Осторожно с typosquatting пакетами.

[↑ Наверх](#top)

---

## Полезные ссылки
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- ASP.NET Core security: https://learn.microsoft.com/aspnet/core/security/
- OAuth2/OIDC: https://oauth.net/2/ , https://openid.net/connect/

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../README.md)

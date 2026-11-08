# Ітерація 1 «Скелет» — план реалізації

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Уся система піднімається однією командою через Aspire; користувач входить через Keycloak, викладач створює курс, студент бачить каталог і записується (US-1, US-2); є щонайменше по одному модульному, інтеграційному та e2e тесту.

**Architecture:** Aspire AppHost (C#) оркеструє Keycloak, Postgres (дві БД), два FastAPI-сервіси, C#-gateway на YARP і React SPA. SPA ходить на `/api/*` через проксі Vite → gateway; gateway валідує JWT і грубі ролі, сервіси повторно валідують JWT і перевіряють доступ до ресурсу. Спільний для Python-сервісів код автентифікації та телеметрії винесено в локальний пакет `lms-common`.

**Tech Stack:** .NET 10, Aspire 13.6.4 (`Aspire.Hosting.Python`, `Aspire.Hosting.JavaScript`, `Aspire.Hosting.PostgreSQL`, `Aspire.Hosting.Keycloak` 13.6.4-preview), YARP + `Microsoft.Extensions.ServiceDiscovery.Yarp`, xUnit, `Aspire.Hosting.Testing`; Python ≥ 3.12 з `uv`, FastAPI, SQLAlchemy 2 (sync, psycopg 3), Alembic, pydantic-settings, PyJWT[crypto], OpenTelemetry, pytest, testcontainers[postgres], pytest-playwright; React 19 + Vite + TypeScript, react-router-dom, react-oidc-context + oidc-client-ts.

**Spec:** `docs/superpowers/specs/2026-11-08-lms-mvp-design.md`

## Global Constraints

- Realm Keycloak: `lms`. Ролі: `teacher`, `student`. Клієнти: `lms-web` (public, Authorization Code + PKCE), `lms-tests` (public, Direct Access Grants — лише dev-realm).
- JWT audience для API: `lms-api` (audience mapper у обох клієнтах). Issuer: `{KEYCLOAK_URL}/realms/lms`.
- Ефективна роль користувача: `teacher`, якщо `teacher` ∈ `realm_access.roles`; інакше `student`, якщо `student` ∈ ролей; інакше ролі немає → 403. Однакове правило в C# і Python.
- `user_id` = claim `sub`; ім'я = claim `name`, якщо його немає — `preferred_username`.
- Самореєстрація увімкнена; нові користувачі отримують `student` (через `default-roles-lms`).
- Фіксовані порти dev: Keycloak `8080`, web `5173`.
- Тіло помилки сервісів: `{"detail": "текст"}`.
- Ідентифікатори сутностей — UUID.
- Python: `uv` для залежностей і запуску; лінтер `ruff`.
- CI не робимо; усі тести запускає `scripts/test-all.ps1`.
- TDD: для правил ролей і доступу — окремі коміти `test: …` (червоний) і `feat: …` (зелений).

## Review Focus

1. Назва курсу з одних пробілів (`"   "`) → 422, а не порожній курс. Тест: Task 4 `test_create_course_rejects_blank_title`.
2. Подвійний клік «Записатися» / паралельні запити на запис → обидва 204, жодного 500. Тест: Task 4 `test_enroll_is_idempotent` (запис через `INSERT … ON CONFLICT DO NOTHING`).
3. Протухлий токен, токен з іншим issuer або audience → 401, а не 500. Тест: Task 2 `test_validate_rejects_expired`, `test_validate_rejects_wrong_audience`, `test_validate_rejects_wrong_issuer`.
4. Не-UUID у шляху (`/api/courses/abc`) → 422; `GET /mine` не перехоплюється маршрутом `/{course_id}`. Тест: Task 4 `test_get_course_invalid_id_returns_422`, `test_mine_route_not_shadowed`.
5. `/api/courses/internal/...` через gateway → 404 (внутрішні ендпоінти не відкриті). Тест: Task 7 `Internal_endpoints_are_not_exposed`.

---

## Карта файлів

```
lms/
├─ lms.slnx                                  ← додаються всі .NET-проєкти
├─ infra/keycloak/realm-lms.json             realm, ролі, клієнти, тестові юзери
├─ src/
│  ├─ Lms.AppHost/AppHost.cs                 уся оркестрація
│  ├─ Lms.ServiceDefaults/                   шаблон Aspire без змін
│  ├─ Lms.Gateway/
│  │  ├─ Program.cs                          auth, політики, YARP, блок /internal
│  │  ├─ Auth/RealmRoles.cs                  ParseRealmRoles, ResolveRole
│  │  ├─ Auth/KeycloakRoleClaimsTransformation.cs
│  │  └─ Proxy/ProxyConfig.cs                маршрути й кластери YARP
│  ├─ lms-common/                            Python-пакет, path-залежність сервісів
│  │  ├─ pyproject.toml
│  │  ├─ lms_common/auth.py                  CurrentUser, resolve_role, TokenValidator, FastAPI-залежності
│  │  ├─ lms_common/telemetry.py             configure_telemetry(app)
│  │  ├─ lms_common/testing.py               TokenFactory для тестів сервісів
│  │  └─ tests/test_auth.py
│  ├─ courses-service/
│  │  ├─ pyproject.toml, alembic.ini
│  │  ├─ alembic/env.py, alembic/versions/0001_courses.py
│  │  ├─ app/main.py                         app = create_app()
│  │  ├─ app/factory.py                      create_app(settings, validator)
│  │  ├─ app/settings.py, app/db.py, app/migrations.py
│  │  ├─ app/models.py                       Course, Enrollment
│  │  ├─ app/schemas.py                      CourseCreate, CourseSummary, CourseDetail
│  │  ├─ app/access.py                       course_access()
│  │  ├─ app/routes.py
│  │  └─ tests/conftest.py, test_access.py, test_api.py
│  ├─ assessment-service/                    та сама структура, без моделей і маршрутів домену
│  └─ web/                                   Vite react-ts
│     ├─ vite.config.ts
│     └─ src/auth/*, src/api/*, src/components/Layout.tsx, src/pages/*, src/App.tsx, src/main.tsx
├─ tests/
│  ├─ Lms.Gateway.Tests/RealmRolesTests.cs, KeycloakRoleClaimsTransformationTests.cs
│  ├─ Lms.IntegrationTests/SystemFixture.cs, CoursesFlowTests.cs
│  └─ e2e/pyproject.toml, conftest.py, test_enroll_flow.py
├─ scripts/test-all.ps1
└─ README.md                                 як запустити й протестувати
```

Відхилення від специфікації (уточнення, не зміна рішень): спільний пакет `src/lms-common` (щоб не дублювати JWT-код у двох сервісах); CORS у gateway не потрібен, бо SPA ходить на `/api` через проксі dev-сервера Vite (одне походження); Postgres без тому даних — БД очищується при кожному запуску. Усі три пункти — внести в специфікацію в Task 9.

---

### Task 1: AppHost, ServiceDefaults, Keycloak realm, Postgres

**Files:**
- Create: `src/Lms.AppHost/*` (шаблон `aspire-apphost`), `src/Lms.ServiceDefaults/*` (шаблон `aspire-servicedefaults`)
- Create: `infra/keycloak/realm-lms.json`
- Modify: `lms.slnx`
- Delete: `.gitkeep` у заповнених папках

**Interfaces:**
- Produces: ресурси Aspire з іменами `keycloak`, `postgres`, `coursesdb`, `assessmentdb`; тестові юзери з паролем, що дорівнює логіну: `teacher1` (`teacher`), `student1`, `student2` (`student`). Наступні таски додають у `AppHost.cs` свої ресурси.

- [ ] **Step 1: Створити проєкти й додати в рішення**

```bash
dotnet new aspire-apphost -n Lms.AppHost -o src/Lms.AppHost
dotnet new aspire-servicedefaults -n Lms.ServiceDefaults -o src/Lms.ServiceDefaults
dotnet sln lms.slnx add src/Lms.AppHost src/Lms.ServiceDefaults
cd src/Lms.AppHost
dotnet add package Aspire.Hosting.Keycloak --prerelease
dotnet add package Aspire.Hosting.PostgreSQL
dotnet add package Aspire.Hosting.Python
dotnet add package Aspire.Hosting.JavaScript
```

- [ ] **Step 2: Написати `infra/keycloak/realm-lms.json`**

Обов'язкові поля realm (решту Keycloak заповнить за замовчуванням):
- `"realm": "lms"`, `"enabled": true`, `"registrationAllowed": true`, `"sslRequired": "none"`.
- `roles.realm`: `teacher`, `student`.
- `defaultRole`: `{"name": "default-roles-lms", "composite": true, "composites": {"realm": ["offline_access", "uma_authorization", "student"]}}`.
- Клієнт `lms-web`: `publicClient: true`, `standardFlowEnabled: true`, `directAccessGrantsEnabled: false`, `redirectUris: ["http://localhost:5173/*"]`, `webOrigins: ["http://localhost:5173"]`, `post.logout.redirect.uris` = `http://localhost:5173/*`, атрибут `pkce.code.challenge.method: "S256"`.
- Клієнт `lms-tests`: `publicClient: true`, `standardFlowEnabled: false`, `directAccessGrantsEnabled: true`.
- В обох клієнтах protocol mapper `oidc-audience-mapper` з `included.custom.audience: "lms-api"`, `access.token.claim: "true"`.
- `users`: `teacher1`, `student1`, `student2` — `enabled: true`, `emailVerified: true`, `firstName`/`lastName` (наприклад «Олена Викладач», «Іван Студент», «Марія Студентка»), фіксований `id` (UUID — щоб `sub` був стабільним), `credentials: [{"type": "password", "value": <логін>, "temporary": false}]`, `realmRoles`: `["teacher"]` або `["default-roles-lms"]`.

- [ ] **Step 3: Описати ресурси в `src/Lms.AppHost/AppHost.cs`**

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var keycloak = builder.AddKeycloak("keycloak", 8080)
    .WithRealmImport("../../infra/keycloak");

var postgres = builder.AddPostgres("postgres");
var coursesDb = postgres.AddDatabase("coursesdb");
var assessmentDb = postgres.AddDatabase("assessmentdb");

builder.Build().Run();
```

- [ ] **Step 4: Перевірити запуск**

Run: `dotnet build lms.slnx`, потім `aspire run` (або `dotnet run --project src/Lms.AppHost`).
Expected: у дашборді `keycloak` і `postgres` у стані Running/Healthy; на `http://localhost:8080/realms/lms/account` вхід під `teacher1`/`teacher1` успішний.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: add Aspire AppHost, ServiceDefaults and Keycloak realm"
```

---

### Task 2: Пакет `lms-common` — автентифікація й телеметрія

**Files:**
- Create: `src/lms-common/pyproject.toml`, `src/lms-common/lms_common/__init__.py`, `auth.py`, `telemetry.py`, `testing.py`
- Test: `src/lms-common/tests/test_auth.py`

**Interfaces:**
- Produces (`lms_common.auth`):
  - `Role = Literal["teacher", "student"]`
  - `@dataclass(frozen=True) class CurrentUser: id: str; name: str; role: Role`
  - `def resolve_role(realm_roles: Iterable[str]) -> Role | None`
  - `class InvalidTokenError(Exception)`, `class MissingRoleError(Exception)`
  - `class TokenValidator: __init__(self, issuer: str, audience: str, key_resolver: Callable[[str], Any])`; `@classmethod from_keycloak(cls, keycloak_url: str, realm: str, audience: str) -> TokenValidator` (JWKS: `{keycloak_url}/realms/{realm}/protocol/openid-connect/certs` через `jwt.PyJWKClient`); `def validate(self, token: str) -> CurrentUser`
  - FastAPI-залежності: `def get_current_user(request: Request) -> CurrentUser` (бере `request.app.state.token_validator`; 401 `{"detail": "Необхідна автентифікація"}`; 403 `{"detail": "Немає доступу"}` при `MissingRoleError`); `def require_role(role: Role) -> Callable[..., CurrentUser]` (403, якщо роль інша)
- Produces (`lms_common.telemetry`): `def configure_telemetry(app: FastAPI, service_name: str) -> None` — OTLP-експорт трейсів і логів, якщо задано `OTEL_EXPORTER_OTLP_ENDPOINT`; інакше нічого не робить.
- Produces (`lms_common.testing`): `class TokenFactory` — генерує RSA-ключ; `validator(issuer="http://test/realms/lms", audience="lms-api") -> TokenValidator`; `token(sub: str, name: str, roles: list[str], *, issuer=..., audience=..., expires_in: int = 300) -> str`.

- [ ] **Step 1: `uv init --lib --name lms-common src/lms-common`; додати залежності**

`uv add fastapi "pyjwt[crypto]" opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-instrumentation-fastapi`; dev: `uv add --dev pytest ruff`.

- [ ] **Step 2: Написати падаючі тести `tests/test_auth.py`**

```python
@pytest.mark.parametrize("roles, expected", [
    (["teacher", "student"], "teacher"),
    (["student", "offline_access"], "student"),
    (["teacher"], "teacher"),
    (["offline_access", "uma_authorization"], None),
    ([], None),
])
def test_resolve_role(roles, expected):
    assert resolve_role(roles) == expected

def test_validate_returns_current_user():
    f = TokenFactory()
    user = f.validator().validate(f.token("u-1", "Іван", ["student"]))
    assert user == CurrentUser(id="u-1", name="Іван", role="student")

def test_validate_falls_back_to_preferred_username()  # токен без name, з preferred_username="ivan" → name == "ivan"
def test_validate_rejects_expired()                   # expires_in=-10 → InvalidTokenError
def test_validate_rejects_wrong_audience()            # audience="account" → InvalidTokenError
def test_validate_rejects_wrong_issuer()              # issuer="http://evil/realms/lms" → InvalidTokenError
def test_validate_rejects_garbage()                   # "not-a-jwt" → InvalidTokenError
def test_validate_without_role_raises_missing_role()  # roles=["offline_access"] → MissingRoleError
```

`TokenFactory.token` кладе ролі в `{"realm_access": {"roles": roles}}`, як Keycloak.

- [ ] **Step 3: Запустити — тести падають**

Run: `cd src/lms-common && uv run pytest -v`
Expected: FAIL (ImportError: `resolve_role` не визначено). Commit: `test: auth rules for lms-common`.

- [ ] **Step 4: Реалізувати `auth.py`, `testing.py`, `telemetry.py`**

`validate`: `jwt.decode(token, key, algorithms=["RS256"], audience=..., issuer=...)`; будь-який `jwt.PyJWTError` → `InvalidTokenError`. `key_resolver(token)` у `from_keycloak` = `PyJWKClient(...).get_signing_key_from_jwt(token).key`.

- [ ] **Step 5: Тести проходять**

Run: `uv run pytest -v` → усі PASS; `uv run ruff check .` → без помилок.

- [ ] **Step 6: Commit**

```bash
git add src/lms-common && git commit -m "feat: lms-common auth and telemetry"
```

---

### Task 3: courses-service — каркас, БД, міграції, підключення до AppHost

**Files:**
- Create: `src/courses-service/pyproject.toml`, `alembic.ini`, `alembic/env.py`, `alembic/versions/0001_courses.py`
- Create: `app/__init__.py`, `app/main.py`, `app/factory.py`, `app/settings.py`, `app/db.py`, `app/migrations.py`, `app/models.py`
- Test: `tests/conftest.py`, `tests/test_health.py`
- Modify: `src/Lms.AppHost/AppHost.cs`

**Interfaces:**
- Consumes: `lms_common.auth.TokenValidator`, `lms_common.telemetry.configure_telemetry`, `lms_common.testing.TokenFactory`.
- Produces:
  - `class Settings(BaseSettings)`: `database_url: str` (env `DATABASE_URL`), `keycloak_url: str` (env `KEYCLOAK_URL`), `keycloak_realm: str = "lms"`, `jwt_audience: str = "lms-api"`.
  - `app.db`: `def make_engine(database_url: str) -> Engine` (замінює схему `postgresql://` на `postgresql+psycopg://`); `def get_session(request: Request) -> Iterator[Session]`.
  - `app.migrations.run_migrations(database_url: str) -> None` (Alembic `upgrade head` програмно).
  - `app.factory.create_app(settings: Settings | None = None, validator: TokenValidator | None = None) -> FastAPI`: у lifespan запускає міграції, кладе `engine`/session factory і `token_validator` у `app.state`; маршрут `GET /health` → `{"status": "ok"}` після `SELECT 1`.
  - `app.main.app = create_app()` (точка входу uvicorn `app.main:app`).
  - Моделі: `Course(id: UUID pk, title: str(200), description: Text, teacher_id: str, teacher_name: str, created_at: timestamptz default now())`; `Enrollment(course_id: UUID fk → courses.id ON DELETE CASCADE, student_id: str, student_name: str, enrolled_at: timestamptz default now())`, PK (`course_id`, `student_id`).
  - Фікстури `conftest.py`: `postgres_url` (session, testcontainers `PostgresContainer("postgres:17")`), `tokens` (`TokenFactory`), `client` (`TestClient(create_app(Settings(database_url=..., keycloak_url="http://test"), tokens.validator()))`), `auth(sub, name, roles) -> dict` (заголовок `Authorization`). Таблиці очищуються `TRUNCATE` перед кожним тестом.

- [ ] **Step 1: `uv init --app src/courses-service`; залежності**

`uv add fastapi "uvicorn[standard]" sqlalchemy "psycopg[binary]" alembic pydantic-settings` і `lms-common` як path-залежність (`[tool.uv.sources] lms-common = { path = "../lms-common", editable = true }`); dev: `uv add --dev pytest "testcontainers[postgres]" httpx ruff`.

- [ ] **Step 2: Написати падаючий тест `tests/test_health.py`**

```python
def test_health_ok_and_tables_created(client, postgres_url):
    assert client.get("/health").json() == {"status": "ok"}
    tables = sqlalchemy.inspect(make_engine(postgres_url)).get_table_names()
    assert {"courses", "enrollments"} <= set(tables)
```

- [ ] **Step 3: Запустити — падає** (`uv run pytest tests/test_health.py -v` → FAIL, немає `app.factory`).

- [ ] **Step 4: Реалізувати settings, db, models, migrations, factory, main, Alembic-міграцію `0001_courses`**

- [ ] **Step 5: Тест проходить** (`uv run pytest -v` → PASS).

- [ ] **Step 6: Підключити до AppHost**

```csharp
var courses = builder.AddUvicornApp("courses-service", "../courses-service", "app.main:app")
    .WithUv()
    .WithEnvironment("DATABASE_URL", coursesDb.Resource.UriExpression)
    .WithEnvironment("KEYCLOAK_URL", keycloak.GetEndpoint("http"))
    .WithHttpHealthCheck("/health")
    .WaitFor(coursesDb)
    .WaitFor(keycloak);
```

Run: `aspire run` → `courses-service` Healthy, у дашборді є його логи й трейс запиту `/health`.

- [ ] **Step 7: Commit** — `feat: courses-service skeleton with migrations`.

---

### Task 4: courses-service — курси й запис (US-2)

**Files:**
- Create: `app/access.py`, `app/schemas.py`, `app/routes.py`
- Modify: `app/factory.py` (підключити router)
- Test: `tests/test_access.py`, `tests/test_api.py`

**Interfaces:**
- Consumes: `CurrentUser`, `get_current_user`, `require_role` (Task 2); `get_session`, моделі (Task 3).
- Produces:
  - `app.access.course_access(teacher_id: str, user_id: str, is_enrolled: bool) -> Literal["owner", "student", "none"]`
  - Схеми: `CourseCreate(title: str — strip, 1..200; description: str = "" — ≤ 5000)`; `CourseSummary(id: UUID, title, description, teacher_name, is_owner: bool, is_enrolled: bool)`; `CourseDetail(id, title, description, teacher_id, teacher_name, created_at: datetime, access: Literal["owner", "student"])`.
  - Маршрути (шляхи всередині сервісу; gateway зрізає префікс `/api/courses`), порядок оголошення саме такий:
    - `GET /` → `list[CourseSummary]`, новіші першими
    - `POST /` (teacher) → 201 `CourseDetail`; `teacher_id = user.id`, `teacher_name = user.name`
    - `GET /mine` → `list[CourseSummary]`: викладачу — власні, студенту — ті, на які записаний
    - `GET /{course_id}` (UUID) → `CourseDetail`; 404 `{"detail": "Курс не знайдено"}`; 403, якщо `course_access == "none"`
    - `POST /{course_id}/enroll` (student) → 204; 404, якщо курсу немає; вставка `INSERT … ON CONFLICT DO NOTHING`

- [ ] **Step 1: Падаючий модульний тест `tests/test_access.py`**

```python
def test_owner():            assert course_access("t1", "t1", False) == "owner"
def test_enrolled_student(): assert course_access("t1", "s1", True) == "student"
def test_stranger():         assert course_access("t1", "s1", False) == "none"
```

Run: `uv run pytest tests/test_access.py -v` → FAIL. Commit: `test: course access rule`.

- [ ] **Step 2: Реалізувати `course_access`, тест PASS.** Commit: `feat: course access rule`.

- [ ] **Step 3: Падаючі інтеграційні тести `tests/test_api.py`**

```python
def test_teacher_creates_course(client, auth)            # POST / → 201; body.title == "Алгоритми"; body.access == "owner"; teacher_name == "Олена"
def test_student_cannot_create_course(client, auth)      # роль student → 403
def test_request_without_token_returns_401(client)
def test_create_course_rejects_blank_title(client, auth) # title "   " → 422
def test_catalog_lists_courses_with_flags(client, auth)  # студент бачить курс, is_enrolled False → після enroll True; is_owner False
def test_enroll_is_idempotent(client, auth)              # два POST /{id}/enroll → обидва 204; у /mine курс один раз
def test_enroll_unknown_course_returns_404(client, auth)
def test_teacher_cannot_enroll(client, auth)             # → 403
def test_mine_for_teacher_and_student(client, auth)
def test_get_course_forbidden_for_stranger(client, auth) # не записаний студент → 403; записаний → 200, access == "student"
def test_get_course_invalid_id_returns_422(client, auth) # GET /abc → 422
def test_mine_route_not_shadowed(client, auth)           # GET /mine → 200 (а не 422)
```

Run → FAIL. 

- [ ] **Step 4: Реалізувати схеми й маршрути, усі тести PASS; `ruff check` чистий.**

- [ ] **Step 5: Commit** — `feat: course catalog, creation and enrollment`.

---

### Task 5: assessment-service — каркас

**Files:**
- Create: `src/assessment-service/` — та сама структура, що в Task 3 (`settings.py`, `db.py`, `migrations.py`, `factory.py`, `main.py`, Alembic без жодної ревізії доменних таблиць), `tests/conftest.py`, `tests/test_health.py`
- Modify: `src/Lms.AppHost/AppHost.cs`

**Interfaces:**
- Consumes: `lms-common` (Task 2); патерн Task 3.
- Produces: ресурс `assessment-service` з `GET /health` → `{"status": "ok"}`. Додатково `GET /me` → `CurrentUser` у JSON (`{"id", "name", "role"}`) — перевірка, що автентифікація в сервісі працює наскрізно; доменні маршрути з'являться в ітерації 2.

- [ ] **Step 1: Падаючі тести** — `test_health_ok`, `test_me_returns_current_user` (`{"id": "s1", "name": "Іван", "role": "student"}`), `test_me_without_token_returns_401`.
- [ ] **Step 2: Реалізувати; тести PASS.**
- [ ] **Step 3: AppHost** — як `courses-service`, але `assessmentDb` і ім'я `assessment-service`; додатково `.WithEnvironment("COURSES_SERVICE_URL", courses.GetEndpoint("http"))` (знадобиться в ітерації 2). `aspire run` → Healthy.
- [ ] **Step 4: Commit** — `feat: assessment-service skeleton`.

---

### Task 6: Gateway — JWT, ролі, YARP

**Files:**
- Create: `src/Lms.Gateway/` (`dotnet new web`), `Auth/RealmRoles.cs`, `Auth/KeycloakRoleClaimsTransformation.cs`, `Proxy/ProxyConfig.cs`, `Program.cs`
- Create: `tests/Lms.Gateway.Tests/` (`dotnet new xunit`), `RealmRolesTests.cs`, `KeycloakRoleClaimsTransformationTests.cs`
- Modify: `lms.slnx`, `src/Lms.AppHost/AppHost.cs`

**Interfaces:**
- Produces:
  - `public static class RealmRoles { public static IReadOnlyList<string> Parse(string? realmAccessJson); public static string? Resolve(IEnumerable<string> realmRoles); }` — `Resolve` за правилом із Global Constraints.
  - `public sealed class KeycloakRoleClaimsTransformation : IClaimsTransformation` — читає claim `realm_access`, додає одну claim типу `"role"` з ефективною роллю; ідемпотентна (повторний виклик не дублює claim).
  - Політики авторизації: `"Teacher"` (`RequireRole("teacher")`), `"Student"` (`RequireRole("student")`), fallback/default — автентифікований користувач.
  - `public static class ProxyConfig { public static IReadOnlyList<RouteConfig> Routes { get; } public static IReadOnlyList<ClusterConfig> Clusters { get; } }`:
    - `courses-create`: `Path = "/api/courses"`, `Methods = ["POST"]`, policy `Teacher`
    - `courses-enroll`: `Path = "/api/courses/{id}/enroll"`, `Methods = ["POST"]`, policy `Student`
    - `courses`: `Path = "/api/courses/{**rest}"`, policy `default`
    - `assessment`: `Path = "/api/assessment/{**rest}"`, policy `default`
    - трансформації `PathRemovePrefix` `/api/courses` і `/api/assessment` відповідно
    - кластери: `courses` → `http://courses-service`, `assessment` → `http://assessment-service` (розв'язуються через service discovery)
  - Блок внутрішніх шляхів: `app.Map("/api/courses/internal/{**rest}", () => Results.NotFound())` і те саме для `/api/assessment/internal/{**rest}` (літеральний сегмент `internal` має вищий пріоритет за catch-all YARP).
  - JwtBearer: `Authority = $"{Keycloak:Url}/realms/lms"`, `Audience = "lms-api"`, `RequireHttpsMetadata = false`, `MapInboundClaims = false`, `TokenValidationParameters.RoleClaimType = "role"`, `NameClaimType = "name"`.

- [ ] **Step 1: Створити проєкти, додати в `lms.slnx`; пакети** `Yarp.ReverseProxy`, `Microsoft.Extensions.ServiceDiscovery.Yarp`, `Microsoft.AspNetCore.Authentication.JwtBearer`; reference на `Lms.ServiceDefaults`. Тест-проєкт посилається на `Lms.Gateway`.

- [ ] **Step 2: Падаючі тести**

```csharp
[Theory]
[InlineData(new[] { "teacher", "student" }, "teacher")]
[InlineData(new[] { "student", "offline_access" }, "student")]
[InlineData(new[] { "offline_access" }, null)]
public void Resolve_applies_teacher_priority(string[] roles, string? expected)
    => Assert.Equal(expected, RealmRoles.Resolve(roles));

[Fact] public void Parse_reads_roles_from_realm_access_json()  // "{\"roles\":[\"student\"]}" → ["student"]
[Fact] public void Parse_returns_empty_for_null_or_invalid_json()
[Fact] public async Task Transformation_adds_effective_role()   // principal з realm_access teacher+student → IsInRole("teacher") true, IsInRole("student") false
[Fact] public async Task Transformation_is_idempotent()         // два виклики → рівно одна claim "role"
```

Run: `dotnet test tests/Lms.Gateway.Tests` → FAIL (не компілюється). Commit: `test: gateway role mapping`.

- [ ] **Step 3: Реалізувати `RealmRoles`, трансформацію; тести PASS.** Commit: `feat: gateway role mapping`.

- [ ] **Step 4: `Program.cs`** — `AddServiceDefaults()`, автентифікація/авторизація як вище, `AddReverseProxy().LoadFromMemory(ProxyConfig.Routes, ProxyConfig.Clusters).AddServiceDiscoveryDestinationResolver()`, блок `/internal`, `MapReverseProxy()`, `MapDefaultEndpoints()`.

- [ ] **Step 5: AppHost**

```csharp
var gateway = builder.AddProject<Projects.Lms_Gateway>("gateway")
    .WithEnvironment("Keycloak__Url", keycloak.GetEndpoint("http"))
    .WithReference(courses).WithReference(assessment)
    .WaitFor(courses).WaitFor(assessment);
```

Run: `aspire run`; `curl -i http://localhost:<порт gateway>/api/courses` → `401`.

- [ ] **Step 6: Commit** — `feat: gateway with JWT validation and YARP routes`.

---

### Task 7: Наскрізні інтеграційні тести на Aspire

**Files:**
- Create: `tests/Lms.IntegrationTests/` (`dotnet new aspire-xunit`), `SystemFixture.cs`, `CoursesFlowTests.cs`
- Modify: `lms.slnx`

**Interfaces:**
- Consumes: усі ресурси AppHost; тестові юзери з Task 1; маршрути Task 4 і Task 6.
- Produces: `SystemFixture : IAsyncLifetime` — `DistributedApplicationTestingBuilder.CreateAsync<Projects.Lms_AppHost>()`, чекає `WaitForResourceHealthyAsync` для `gateway`, `courses-service`, `assessment-service`, `keycloak`; властивості `HttpClient Gateway` (`app.CreateHttpClient("gateway")`); метод `Task<string> GetTokenAsync(string username, string password)` — `POST {keycloak}/realms/lms/protocol/openid-connect/token`, `grant_type=password`, `client_id=lms-tests`.

- [ ] **Step 1: Тести**

```csharp
[Fact] Unauthenticated_request_returns_401()          // GET /api/courses → 401
[Fact] Student_cannot_create_course()                 // student1, POST /api/courses → 403 (відсікає gateway)
[Fact] Teacher_creates_course_and_student_enrolls()   // teacher1 POST → 201 з унікальною назвою;
                                                      // student1 POST /api/courses/{id}/enroll → 204;
                                                      // student1 GET /api/courses/mine містить курс; GET /api/courses/{id} → 200
[Fact] Assessment_service_sees_user_through_gateway() // student1 GET /api/assessment/me → role == "student"
[Fact] Internal_endpoints_are_not_exposed()           // GET /api/courses/internal/courses/x/access → 404
```

- [ ] **Step 2: Запустити** — `dotnet test tests/Lms.IntegrationTests` (Docker запущений, `aspire run` зупинений — порт 8080 зайнятий інакше). Expected: усі PASS. Якщо якийсь падає — виправити код відповідної таски, а не тест.

- [ ] **Step 3: Commit** — `test: end-to-end integration tests via Aspire`.

---

### Task 8: Web — логін, каталог, створення курсу, мої курси (US-1, US-2)

**Files:**
- Create: `src/web/` (`npm create vite@latest web -- --template react-ts` у `src/`)
- Create: `src/auth/oidcConfig.ts`, `src/auth/useCurrentUser.ts`, `src/api/client.ts`, `src/api/courses.ts`, `src/components/Layout.tsx`, `src/pages/CatalogPage.tsx`, `MyCoursesPage.tsx`, `CoursePage.tsx`, `CreateCoursePage.tsx`
- Modify: `src/App.tsx`, `src/main.tsx`, `vite.config.ts`, `src/Lms.AppHost/AppHost.cs`

**Interfaces:**
- Consumes: API Task 4 через `/api/courses`; клієнт Keycloak `lms-web`.
- Produces:
  - `vite.config.ts`: `server.port = Number(process.env.PORT) || 5173`, `server.strictPort = true`, `server.proxy['/api'] = { target: process.env.GATEWAY_URL, changeOrigin: true }`.
  - `oidcConfig`: `authority = ${import.meta.env.VITE_KEYCLOAK_URL}/realms/lms`, `client_id = "lms-web"`, `redirect_uri = window.location.origin`, `post_logout_redirect_uri = window.location.origin`, `onSigninCallback` прибирає `?code=…` з URL.
  - `useCurrentUser(): { id: string; name: string; role: "teacher" | "student" | null }` — ролі з payload **access token** (`realm_access.roles`), правило з Global Constraints.
  - `class ApiError extends Error { status: number; detail: string }`; `apiFetch<T>(path: string, token: string, init?: RequestInit): Promise<T>` (204 → `undefined`).
  - `courses.ts`: `listCourses`, `listMyCourses`, `getCourse(id)`, `createCourse({title, description})`, `enroll(id)` — типи відповідають схемам Task 4.
  - Маршрути: `/` каталог, `/mine` мої курси, `/courses/new` (лише teacher), `/courses/:id`. Увесь застосунок за `withAuthenticationRequired`.
  - Точні тексти (на них спираються e2e): посилання «Каталог», «Мої курси»; кнопки «Створити курс», «Записатися», «Вийти»; позначка «Ви записані»; форма: поля з `<label>` «Назва» і «Опис», кнопка «Створити»; після створення — перехід на `/courses/{id}`, де видно назву курсу; картка курсу в каталозі — елемент `<article>`; у шапці — ім'я користувача. Помилки: 401 → `signinRedirect()`; 403 → «Немає доступу»; 404 → «Не знайдено»; 422 → повідомлення під полем; 5xx/мережа → «Сервіс тимчасово недоступний».

- [ ] **Step 1: Створити проєкт; `npm i react-router-dom react-oidc-context oidc-client-ts`.**
- [ ] **Step 2: Реалізувати auth, api, layout, сторінки.**
- [ ] **Step 3: AppHost**

```csharp
builder.AddViteApp("web", "../web")
    .WithEnvironment("GATEWAY_URL", gateway.GetEndpoint("http"))
    .WithEnvironment("VITE_KEYCLOAK_URL", keycloak.GetEndpoint("http"))
    .WithEndpoint("http", e => e.Port = 5173)
    .WaitFor(gateway);
```

- [ ] **Step 4: Ручна перевірка (приймальний сценарій для демо)** — `aspire run`, `http://localhost:5173`: вхід `teacher1` → «Створити курс» → курс у каталозі; «Вийти»; вхід `student1` → у каталозі кнопка «Записатися», кнопки «Створити курс» немає → запис → «Ви записані», курс у «Мої курси»; реєстрація нового користувача на сторінці Keycloak → потрапляє в систему як студент. `npm run build` і `npm run lint` без помилок.
- [ ] **Step 5: Commit** — `feat: web app with login, catalog and enrollment`.

---

### Task 9: E2E-тест, скрипт тестів, README, оновлення специфікації

**Files:**
- Create: `tests/e2e/pyproject.toml`, `tests/e2e/conftest.py`, `tests/e2e/test_enroll_flow.py`, `scripts/test-all.ps1`, `README.md`
- Modify: `docs/superpowers/specs/2026-11-08-lms-mvp-design.md` (три уточнення з «Карти файлів»)

**Interfaces:**
- Consumes: тексти UI з Task 8; тестові юзери з Task 1; запущена система (`aspire run`).
- Produces: `conftest.py` — `base_url = os.environ.get("LMS_BASE_URL", "http://localhost:5173")`; хелпер `login(page, username, password)` (форма Keycloak: `#username`, `#password`, `#kc-login`) і `logout(page)` (кнопка «Вийти»).

- [ ] **Step 1: `uv init --app tests/e2e`; `uv add --dev pytest pytest-playwright`; `uv run playwright install chromium`.**

- [ ] **Step 2: Тест `test_enroll_flow.py`**

```python
def test_teacher_creates_course_and_student_enrolls(page, base_url):
    title = f"E2E курс {uuid4().hex[:6]}"
    login(page, "teacher1", "teacher1")
    page.get_by_role("button", name="Створити курс").click()
    page.get_by_label("Назва").fill(title)
    page.get_by_role("button", name="Створити").click()
    expect(page.get_by_text(title)).to_be_visible()
    logout(page)

    login(page, "student1", "student1")
    card = page.get_by_role("article").filter(has_text=title)  # картка курсу — елемент <article>
    card.get_by_role("button", name="Записатися").click()
    expect(card.get_by_text("Ви записані")).to_be_visible()
    page.get_by_role("link", name="Мої курси").click()
    expect(page.get_by_text(title)).to_be_visible()
```

- [ ] **Step 3: Запустити** — у першому терміналі `aspire run`, у другому `cd tests/e2e && uv run pytest -v`. Expected: PASS.

- [ ] **Step 4: `scripts/test-all.ps1`** — послідовно, зупиняючись на першій помилці (`$ErrorActionPreference = "Stop"` + перевірка `$LASTEXITCODE`): `uv run pytest` у `src/lms-common`, `src/courses-service`, `src/assessment-service`; `dotnet test lms.slnx`; з параметром `-E2E` — ще й `uv run pytest` у `tests/e2e` (потребує запущеного `aspire run`). Вивести підсумок по кожному блоку.

- [ ] **Step 5: `README.md`** — передумови (.NET 10 SDK, Docker, uv, Node 24), запуск `aspire run`, адреси (web 5173, Keycloak 8080, дашборд), тестові акаунти, запуск тестів, правило «перед злиттям у `main` — `scripts/test-all.ps1`».

- [ ] **Step 6: Оновити специфікацію** — розділ 4: пакет `lms-common`; CORS → проксі Vite; розділ 4 «Файли»/AppHost: Postgres без тому даних; розділ 12: додати `src/lms-common/`.

- [ ] **Step 7: Повний прогін** — `aspire run` у фоні, `./scripts/test-all.ps1 -E2E` → усі блоки PASS.

- [ ] **Step 8: Commit і тег релізу**

```bash
git add -A && git commit -m "test: e2e enroll flow, test script and README"
git tag v0.1
```

---

## Відповідність user stories ClickUp

| Story | Таски |
|---|---|
| [US-1. Реєстрація та вхід](https://app.clickup.com/t/1ame4r31ht8) | 1 (realm), 2, 6, 8 |
| [US-2. Створення курсу і запис на курс](https://app.clickup.com/t/1ame4r31ht9) | 3, 4, 8 |
| Тести всіх видів (вимога лабораторної) | 2, 4, 6 (модульні); 4, 7 (інтеграційні); 9 (e2e) |

# Jobby API — Lab 1

## Цель
Реализовать REST API сайта по поиску работы на **TypeScript + Express + TypeORM + PostgreSQL**.
Спецификация: `openapi.yaml`. Схема БД: `HW3.1.drawio`.

---

## Tech Stack
- **Runtime**: Node.js + TypeScript
- **Framework**: Express.js
- **ORM**: TypeORM
- **DB**: PostgreSQL
- **Auth**: JWT в httpOnly cookie (`access_token`)
- **Password**: bcrypt
- **Validation**: class-validator + class-transformer
- **UUID**: pg `gen_random_uuid()` / uuid пакет

---

## Структура проекта

```
src/
  entities/          # TypeORM entities
  dto/               # Request/Response DTOs (class-validator)
  controllers/       # Express route handlers
  services/          # Business logic
  middleware/        # auth, roles, validation
  routes/            # Express routers
  config/            # DB config, env
  app.ts
  index.ts
```

---

## База данных — все сущности

### Справочники
| Entity | Поля |
|--------|------|
| Country | id uuid PK, name varchar |
| City | id uuid PK, country_id FK→Country, name varchar |
| Industry | id uuid PK, name varchar |
| Skill | id uuid PK, name varchar |
| EmploymentType | id uuid PK, name varchar |
| DegreeType | id uuid PK, name varchar |

> **SkillCategory убрана** — Skill не имеет категории.

### Пользователи
| Entity | Поля |
|--------|------|
| User | id uuid PK, role enum('SEEKER','EMPLOYER','ADMIN'), email varchar UQ, phone varchar nullable, password_hash varchar, created_at, updated_at |
| JobSeeker | id uuid PK, user_id FK→User, city_id FK→City, first_name, last_name, middle_name nullable, birth_date date nullable, gender enum('MALE','FEMALE') nullable |
| Employer | id uuid PK, user_id FK→User, company_id FK→Company nullable, first_name, last_name, position nullable |

### Компании
| Entity | Поля |
|--------|------|
| Company | id uuid PK, industry_id FK→Industry nullable, city_id FK→City nullable, name varchar, description text nullable, website varchar nullable, logo_url varchar nullable, updated_at |

### Резюме
| Entity | Поля |
|--------|------|
| Resume | id uuid PK, job_seeker_id FK→JobSeeker, title varchar, summary text nullable, experience_months_cached int default 0, is_published boolean default false, created_at, updated_at |
| WorkExperience | id uuid PK, resume_id FK→Resume, company_name varchar, role varchar, start_date date, end_date date nullable, is_current boolean |
| Education | id uuid PK, resume_id FK→Resume, degree_type_id FK→DegreeType, institution varchar, program_name varchar nullable, start_date date nullable, end_date date nullable |
| ResumeSkill | id uuid PK, resume_id FK→Resume, skill_id FK→Skill, level enum('BEGINNER','INTERMEDIATE','EXPERT') |

### Вакансии
| Entity | Поля |
|--------|------|
| Vacancy | id uuid PK, employer_id FK→Employer, company_id FK→Company, industry_id FK→Industry nullable, city_id FK→City nullable, employment_type_id FK→EmploymentType nullable, title varchar, description text nullable, conditions text nullable, requirements_note varchar nullable, salary_type enum('FIXED','RANGE','FROM','TO','NEGOTIABLE'), salary_fixed int nullable, salary_min int nullable, salary_max int nullable, currency varchar default 'RUB', experience_years_min int nullable, experience_years_max int nullable, is_remote boolean default false, is_published boolean default false, created_at, updated_at |
| VacancySkill | id uuid PK, vacancy_id FK→Vacancy, skill_id FK→Skill |

> **VacancySkill.is_required и min_level убраны.**

### Отклики
| Entity | Поля |
|--------|------|
| Application | id uuid PK, resume_id FK→Resume, vacancy_id FK→Vacancy, cover_letter text nullable, status enum('PENDING','VIEWED','INVITED','REJECTED','ACCEPTED') default 'PENDING', created_at, updated_at |

> **ApplicationStatusHistory убрана.** ApplicationDetail = ApplicationItem.

---

## Auth Flow (двухшаговая регистрация)

1. `POST /auth/register/step1` — создаёт User, возвращает `temporaryToken` (JWT 15 мин, payload: `{sub: userId, role, step:'register'}`)
2. `POST /auth/register/seeker` или `/employer` — принимает `temporaryToken` через `Authorization: Bearer <token>`, создаёт JobSeeker/Employer, устанавливает `access_token` httpOnly cookie, возвращает `AuthResponse`
3. `POST /auth/login` — верифицирует credentials, устанавливает `access_token` httpOnly cookie
4. `POST /auth/logout` — Set-Cookie: access_token=; Max-Age=0

**Полный JWT payload**: `{ sub: userId, role, iat, exp }`

---

## Middleware

| Middleware | Описание |
|-----------|---------|
| `authenticate` | Парсит JWT из cookie `access_token`, кидает 401 если нет/невалидный |
| `authenticateTemp` | Парсит temporaryToken из `Authorization: Bearer`, проверяет step='register' |
| `requireRole(...roles)` | Проверяет роль из JWT, кидает 403 |
| `validate(DtoClass)` | class-validator на req.body, кидает 422 с полями ошибок |

---

## Все эндпоинты (base: `/api/v1`)

### Auth
| Method | Path | Auth | Roles |
|--------|------|------|-------|
| POST | /auth/register/step1 | — | — |
| POST | /auth/register/seeker | temporaryToken | — |
| POST | /auth/register/employer | temporaryToken | — |
| POST | /auth/login | — | — |
| GET | /auth/me | JWT cookie | any |
| POST | /auth/logout | JWT cookie | any |

### Profile
| Method | Path | Auth | Roles |
|--------|------|------|-------|
| PATCH | /profile/seeker | JWT cookie | SEEKER |
| PATCH | /profile/employer | JWT cookie | EMPLOYER |

### Resumes
| Method | Path | Notes |
|--------|------|-------|
| GET | /resumes | SEEKER — свои |
| POST | /resumes | SEEKER |
| GET | /resumes/:resumeId | SEEKER/EMPLOYER |
| PATCH | /resumes/:resumeId | только владелец |
| DELETE | /resumes/:resumeId | только владелец |
| POST | /resumes/:resumeId/skills | владелец |
| DELETE | /resumes/:resumeId/skills/:skillId | владелец |
| POST | /resumes/:resumeId/work-experience | владелец |
| PATCH | /resumes/:resumeId/work-experience/:workId | владелец |
| DELETE | /resumes/:resumeId/work-experience/:workId | владелец |
| POST | /resumes/:resumeId/education | владелец |
| PATCH | /resumes/:resumeId/education/:educationId | владелец |
| DELETE | /resumes/:resumeId/education/:educationId | владелец |

### Vacancies
| Method | Path | Notes |
|--------|------|-------|
| GET | /vacancies | публичный, фильтры + пагинация |
| POST | /vacancies | EMPLOYER |
| GET | /vacancies/:vacancyId | публичный |
| PATCH | /vacancies/:vacancyId | только автор |
| DELETE | /vacancies/:vacancyId | только автор |
| POST | /vacancies/:vacancyId/skills | только автор |
| DELETE | /vacancies/:vacancyId/skills/:skillId | только автор |
| GET | /vacancies/:vacancyId/applications | только автор вакансии |
| GET | /employer/vacancies | EMPLOYER — свои |

### Applications
| Method | Path | Notes |
|--------|------|-------|
| POST | /applications | SEEKER |
| GET | /applications/my | SEEKER |
| GET | /applications/:applicationId | SEEKER (свой) или EMPLOYER (автор вакансии) |
| PATCH | /applications/:applicationId/status | EMPLOYER (автор вакансии) |

### Companies
| Method | Path | Notes |
|--------|------|-------|
| GET | /companies | публичный |
| POST | /companies | EMPLOYER |
| GET | /companies/:companyId | публичный |
| PATCH | /companies/:companyId | EMPLOYER (сотрудник компании) |

### Dictionaries (все публичные)
- GET /dictionaries/countries
- GET /dictionaries/cities
- GET /dictionaries/industries
- GET /dictionaries/skills
- GET /dictionaries/employment-types
- GET /dictionaries/degree-types

### Admin Dictionaries (только ADMIN)
- CRUD: /admin/dictionaries/countries
- CRUD: /admin/dictionaries/cities
- CRUD: /admin/dictionaries/industries
- CRUD: /admin/dictionaries/skills
- CRUD: /admin/dictionaries/employment-types
- CRUD: /admin/dictionaries/degree-types

---

## Формат ответов

```json
// Успех — данные напрямую
{ "id": "...", ... }

// Ошибка
{ "statusCode": 4xx, "message": "..." }

// Ошибка валидации
{ "statusCode": 422, "message": "Validation failed", "errors": [{ "field": "email", "message": "..." }] }
```

---

## Docker

Весь проект разворачивается в Docker. Два сервиса в `docker-compose.yml`:

| Сервис | Образ | Порт |
|--------|-------|------|
| `app` | Dockerfile (node:20-alpine) | 3000:3000 |
| `db` | postgres:16-alpine | 5432:5432 |

**Dockerfile** — многоэтапная сборка:
```
Stage 1 (builder): установка зависимостей + tsc компиляция
Stage 2 (runner): только dist/ + node_modules (prod)
```

**docker-compose.yml** — сервис `app` зависит от `db` через `depends_on` + healthcheck на postgres.

**Окружение передаётся через `.env`** (в docker-compose: `env_file: .env`).

Команды:
```bash
docker compose up --build      # первый запуск / пересборка
docker compose up -d           # в фоне
docker compose down            # остановить
docker compose logs -f app     # логи приложения
```

---

## Окружение (.env)

```
PORT=3000
DB_HOST=db
DB_PORT=5432
DB_NAME=jobby
DB_USER=postgres
DB_PASSWORD=postgres
JWT_SECRET=secret
JWT_EXPIRES_IN=7d
JWT_TEMP_EXPIRES_IN=15m
BCRYPT_ROUNDS=10
NODE_ENV=production
```

> `DB_HOST=db` — имя сервиса в docker-compose сети, не localhost.

---

## Бизнес-правила

1. `experience_months_cached` в Resume пересчитывается при любом изменении WorkExperience
2. Уникальность: одно резюме не может дважды откликнуться на одну вакансию (UNIQUE resume_id + vacancy_id в Application)
3. Уникальность: один навык не может дважды быть в одном резюме (UNIQUE resume_id + skill_id в ResumeSkill)
4. При удалении справочника с зависимыми записями → 409
5. EMPLOYER видит только отклики на свои вакансии

---

## Команды

```bash
npm run dev        # ts-node-dev
npm run build      # tsc
npm run start      # node dist/index.js
```

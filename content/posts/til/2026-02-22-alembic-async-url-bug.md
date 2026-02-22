---
title: "async_engine_from_config에 psycopg2 URL을 넘기면 안 된다"
date: 2026-02-22
categories: ["til"]
tags: ["alembic", "sqlalchemy", "asyncpg", "fastapi", "database", "python"]
project: "podo-budget"
source_sessions: ["2026-02-22-podo-budget-feat-telegram-link-code"]
---

`alembic revision --autogenerate`를 실행했는데 에러가 났습니다. 모델도 문제없고, DB 연결도 잘 되는데 migration 명령만 실패했습니다. 원인은 `alembic/env.py` 안에 있었습니다.

## 🔍 문제의 구조

`env.py`에는 `get_url()`이라는 헬퍼 함수가 있었습니다. 이 함수는 환경변수에서 읽어온 `postgresql+asyncpg://...` URL을 `postgresql+psycopg2://...`로 변환해서 반환했습니다. 동기 엔진용으로 만들어진 함수였습니다.

문제는 이 URL을 `async_engine_from_config()`에 그대로 넘기고 있었다는 점입니다.

```python
# 버그가 있던 코드
async def run_async_migrations() -> None:
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = get_url()  # psycopg2 URL로 변환된 값 전달
    connectable = async_engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
```

`async_engine_from_config()`는 내부적으로 asyncpg 드라이버를 사용합니다. 여기에 psycopg2 URL을 넘기면 드라이버 불일치가 발생합니다.

## ✔ 수정

`settings.DATABASE_URL`을 직접 사용하도록 바꿨습니다. 이 값은 이미 `postgresql+asyncpg://` 형태입니다.

```python
# 수정된 코드
async def run_async_migrations() -> None:
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = settings.DATABASE_URL  # asyncpg URL 그대로 사용
    connectable = async_engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
```

URL 스킴과 엔진 타입의 대응 관계는 명확합니다.

| 함수 | 사용해야 하는 URL 스킴 |
|---|---|
| `create_engine()` | `postgresql+psycopg2://` |
| `create_async_engine()` | `postgresql+asyncpg://` |
| `async_engine_from_config()` | `postgresql+asyncpg://` |

`async_engine_from_config()`는 `create_async_engine()`의 config 기반 래퍼입니다. async 계열 함수는 전부 asyncpg URL이어야 합니다.

## 🏁 덤: SQLite 로컬 환경에서는 alembic upgrade head가 필요 없다

podo-budget은 로컬 Docker 환경에서 SQLite를 사용합니다. SQLite 환경에서는 `alembic upgrade head`를 실행할 필요가 없었습니다. 앱 시작 시 `Base.metadata.create_all()`이 호출되면서 SQLAlchemy가 스키마를 자동으로 생성해 주기 때문입니다.

이번에 추가한 `telegram_link_code`, `telegram_link_code_expires_at` 컬럼도 컨테이너를 재시작하자 자동으로 생성됐습니다. migration 파일을 작성하고 실행하는 과정은 PostgreSQL 프로덕션 환경에서만 필요합니다.

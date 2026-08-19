# Task Spec: миграция SQLite → PostgreSQL 17

Составлено на основе ответов в `questions.md`.

## Цель
Перевести проект с SQLite на PostgreSQL 17 как на единственную СУБД, вынеся конфигурацию в переменные окружения. Перенос данных не требуется — БД стартует пустой.

## Ключевые решения

| Вопрос | Решение |
|---|---|
| СУБД | PostgreSQL 17, только она; SQLite-fallback не нужен |
| Где живёт БД | Вне проекта, поднимается отдельно; подключение по кредам. `docker-compose.yml` не создаём |
| Драйвер | `psycopg[binary]` (v3) |
| Конфигурация | `.env`, отдельные переменные (не `DATABASE_URL`) |
| Чтение `.env` | `python-dotenv` — минимальная зависимость, не переписывает структуру `settings.py` |
| Что выносим в env | `DB_*`, а также `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` |
| Данные | Не переносим. `db.sqlite3` удаляем |
| Токены | Не сохраняем, пользователи перелогинятся |
| Миграции | Схлопываем историю в одну чистую `0001_initial` |
| Тесты | Существующие HTTP-тесты работают против PostgreSQL как есть; отдельная тестовая БД не настраивается |
| Права БД-пользователя | Включая `CREATEDB` |

---

## Объём работ

### 1. Зависимости
- Установить `psycopg[binary]` и `python-dotenv` в `.venv`.
- Пересобрать `requirements.txt` через `pip freeze > requirements.txt`.

### 2. Переменные окружения

Создать `project-3/.env` (не в git) и `project-3/.env.example` (в git):

```
SECRET_KEY=django-insecure-change-me
DEBUG=True
ALLOWED_HOSTS=127.0.0.1,localhost

DB_NAME=taskmanager
DB_USER=taskmanager
DB_PASSWORD=
DB_HOST=127.0.0.1
DB_PORT=5432
```

`.env.example` содержит те же ключи с пустыми или плейсхолдерными значениями — без реальных секретов.

### 3. `.gitignore`
Создать (или дополнить) в корне `project-3`:
```
.env
*.sqlite3
__pycache__/
*.py[cod]
```

### 4. `project/project/settings.py`

- В начале файла — загрузка `.env`:
  ```python
  from dotenv import load_dotenv
  load_dotenv(BASE_DIR.parent / '.env')
  ```
  (`.env` лежит в `project-3/`, а `BASE_DIR` — это `project-3/project/`.)

- `SECRET_KEY = os.environ['SECRET_KEY']` — падать при отсутствии, а не подставлять дефолт.
- `DEBUG` — парсить из строки (`os.environ.get('DEBUG', 'False') == 'True'`).
- `ALLOWED_HOSTS` — split по запятой, пустая строка → пустой список.
- `DATABASES`:
  ```python
  DATABASES = {
      'default': {
          'ENGINE': 'django.db.backends.postgresql',
          'NAME': os.environ['DB_NAME'],
          'USER': os.environ['DB_USER'],
          'PASSWORD': os.environ.get('DB_PASSWORD', ''),
          'HOST': os.environ.get('DB_HOST', '127.0.0.1'),
          'PORT': os.environ.get('DB_PORT', '5432'),
      }
  }
  ```

Обязательные переменные читаются через `os.environ[...]` — отсутствие должно быть громкой ошибкой на старте, а не тихим дефолтом.

### 5. Миграции — схлопывание

1. Удалить `project/app/migrations/0001_initial.py`, `0002_delete_task.py`, `0003_initial.py` и содержимое `__pycache__`. `__init__.py` оставить.
2. Убедиться, что PostgreSQL-база создана и пуста.
3. `python manage.py makemigrations app` → должна появиться единственная `0001_initial.py` с актуальной моделью `Task`.
4. `python manage.py migrate` — прогон с нуля.

Это безопасно только потому, что данные не переносятся. Если где-то существует применённая БД со старой историей — она несовместима и должна быть пересоздана.

### 6. Удаление SQLite
- Удалить `project/db.sqlite3`.
- Убедиться, что в коде не осталось упоминаний sqlite (`grep -ri sqlite project/`).

### 7. Подготовка БД (шаги для оператора, в документацию)
```sql
CREATE USER taskmanager WITH PASSWORD '...' CREATEDB;
CREATE DATABASE taskmanager OWNER taskmanager;
```
`CREATEDB` нужен, чтобы Django мог создавать тестовую БД для `manage.py test`.

### 8. Документация
- `CODEBASE_INVENTORY.md`: раздел «Стек и технологии» — SQLite → PostgreSQL 17 + `psycopg[binary]` + `python-dotenv`; «Полезные команды» — добавить настройку `.env` и создание БД; «Тесты и конфиги» — упомянуть `.env` / `.env.example`; из «Зон повышенного риска» убрать пункты про захардкоженный `SECRET_KEY` и `db.sqlite3` в репозитории.
- `Readme.md`: добавить раздел с требованиями к окружению (PostgreSQL 17, переменные, порядок запуска).

---

## Критерии приёмки

Все пункты обязательны:

1. `python manage.py migrate` проходит без ошибок на чистой PostgreSQL 17.
2. `python manage.py runserver` стартует, `api/` отвечает.
3. Полный прогон `tests/` зелёный (сервер запущен, БД — PostgreSQL).
4. `python manage.py check --deploy` без критичных замечаний.
5. Ручной сценарий проходит целиком: регистрация → логин → создание задачи → взятие исполнителем → отметка выполненной.
6. `project/db.sqlite3` отсутствует; `.env` не в git; `.env.example` в git.
7. В `project/app/migrations/` ровно одна миграция — `0001_initial.py`.
8. Приложение не стартует при отсутствии обязательных переменных окружения (проверить, временно убрав `DB_NAME`).

Пункт 5 стоит прогнать вручную, а не считать покрытым пунктом 3: тесты вызывают `clear_db/` в `setUp` и могут маскировать проблемы с состоянием.

---

## Вне скоупа

Вопросы 8.1 и 8.2 в `questions.md` остались без ответа — фиксирую предположение по умолчанию, скорректируйте, если неверно:

- Connection pooling (`CONN_MAX_AGE`, pgbouncer) — не настраиваем.
- Индексы, оптимизация запросов — не трогаем.
- Бэкапы, CI, деплой — не входят.
- Рефакторинг views (в т.ч. известные проблемы: незащищённый `clear_db/`, `on_delete=CASCADE` у nullable `executor`, `int(executor_id)` без обработки) — отдельная задача, здесь не трогаем.
- Docker-инфраструктура — не создаём.

## Риски

- **Схлопывание миграций необратимо** для любой уже применённой БД. Убедиться, что нигде нет рабочего инстанса со старой историей.
- **`executor` FK с `on_delete=CASCADE`** при `null=True` — поведение не меняется при переезде, но на PostgreSQL с реальными FK-констрейнтами эффект каскада станет заметнее. Не чиним здесь, но стоит знать.
- **Тесты деструктивны** (`clear_db/` стирает всех пользователей). На PostgreSQL это по-прежнему так — не запускать против БД с нужными данными.
- **`DecimalField`** ведёт себя строже на PostgreSQL, чем на SQLite: значения, превышающие `max_digits=8`, теперь дадут ошибку БД вместо тихого прохода. Проверить на шаге приёмки 5.

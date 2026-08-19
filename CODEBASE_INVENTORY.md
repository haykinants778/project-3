# CODEBASE_INVENTORY.md

## Стек и технологии
- Python 3.11/3.12 (в `__pycache__` есть артефакты обеих версий), виртуальное окружение `.venv` на уровне `lesson_3`.
- Django 5.0.6 — веб-фреймворк, ORM, миграции.
- Django REST Framework 3.15.1 — APIView/generics, сериализаторы, `rest_framework.authtoken` (токен-аутентификация).
- SQLite (`project/db.sqlite3`) — единственная БД, файл закоммичен в репозиторий.
- `requests` 2.31 + `unittest` — внешние (black-box) HTTP-тесты в `tests/`.
- Полный список зависимостей: `requirements.txt`.

## Модули и ответственность
| Путь | Ответственность |
|---|---|
| `project/manage.py` | CLI-точка входа Django. |
| `project/project/settings.py` | Конфиг: INSTALLED_APPS (`rest_framework`, `rest_framework.authtoken`, `app`), SQLite, DEBUG=True, SECRET_KEY в коде. |
| `project/project/urls.py` | Корневой роутинг: `admin/`, `api/` → `app.urls`. |
| `project/project/wsgi.py` / `asgi.py` | Точки входа для WSGI/ASGI серверов. |
| `project/app/models.py` | Единственная модель `Task` (creator, executor, name, cost, is_done, deadline). |
| `project/app/serializers.py` | `TaskSerializer` — ModelSerializer со всеми полями Task. |
| `project/app/views.py` | Вся бизнес-логика: 12 view-классов (auth, CRUD задач, статистика, служебная очистка БД). |
| `project/app/urls.py` | Маппинг 12 URL → views, все под префиксом `api/`. |
| `project/app/admin.py` | Пустой — модели в админке не зарегистрированы. |
| `project/app/tests.py` | Пустая заглушка Django TestCase. |
| `project/app/migrations/` | 3 миграции: `0001_initial`, `0002_delete_task`, `0003_initial`. |
| `tests/` | 11 файлов внешних API-тестов + `configs.json` с `BASE_URL`. |
| `Readme.md` | Спецификация задания: требования к каждой view, коды ответов, URL-схема. |

## Точки входа
- **HTTP API**: `http://127.0.0.1:8000/api/...` (см. `project/app/urls.py`).
  - `POST api/user/create/` — регистрация (`UserCreateView`).
  - `POST api/login/` — выдача токена (`LoginView`).
  - `POST api/logout/` — удаление токена (`LogoutView`, auth).
  - `POST api/task/create/` — создание задачи (`TaskCreateView`, auth).
  - `GET  api/tasks-created-by-user/` — задачи, созданные мной (`TasksCreatedByUser`, auth).
  - `GET  api/task/executor/` — все задачи, executor → `undefined` если пуст (`TaskWithExecutorAPIView`, auth).
  - `GET  api/user-tasks/` — задачи, где я исполнитель (`UserTasksAPIView`, auth).
  - `GET  api/user-tasks-stats/` — агрегированная статистика (`UserTasksStatsAPIView`, auth).
  - `GET  api/unassigned-tasks/` — задачи без исполнителя, сортировка по `cost` (`UnassignedTasksAPIView`, auth).
  - `PATCH api/become-executor/<task_id>/` — назначить себя исполнителем (`BecomeExecutorAPIView`, auth).
  - `PATCH api/mark-task-done/<task_id>/` — отметить выполненной (`MarkTaskDoneAPIView`, auth).
  - `GET  api/clear_db/` — **удаляет все Task и User** (`ClearDatabaseView`, без авторизации).
- **Админка**: `http://127.0.0.1:8000/admin/` (моделей нет, только auth).
- **CLI**: `python project/manage.py <command>`.

## Полезные команды
```bash
# из каталога project-3
pip install -r requirements.txt

cd project
python manage.py migrate
python manage.py runserver            # 127.0.0.1:8000
python manage.py createsuperuser
python manage.py makemigrations app

# внешние тесты (сервер должен быть запущен!)
cd ../tests
python -m unittest discover -p "test_*.py"
python -m unittest test_LoginView.py
```

## Тесты и конфиги
- `tests/configs.json` — единственный конфиг тестов: `{"BASE_URL": "http://127.0.0.1:8000/api/"}`.
- Каждый тест-файл сам читает `configs.json` через локальную `get_url()` (дублируется в 11 файлах).
- Тесты — интеграционные по HTTP через `requests`, **не** Django `TestCase`: требуют живого сервера и реальной БД.
- `setUp` каждого теста вызывает `clear_db/`, то есть тесты деструктивны для рабочей БД.
- `project/app/tests.py` — пустая заглушка, юнит-тестов внутри Django нет.
- Конфигурация приложения — только `settings.py`; переменных окружения / `.env` нет.

## Карта зависимостей
```
project/project/urls.py
        └── app/urls.py
                └── app/views.py
                        ├── app/serializers.py ──► app/models.py (Task)
                        ├── django.contrib.auth.models.User
                        ├── rest_framework.authtoken.models.Token
                        └── django.db.models (Sum, Count), django.utils.timezone

app/models.py ──► User (creator: related_name='task_created',
                        executor: related_name='task_executed', nullable)

tests/*.py ──HTTP──► BASE_URL (configs.json) ──► работающий Django-сервер
```
Внешних сервисов, очередей, кэшей нет — весь стек локальный.

## Runtime-потоки
1. **Регистрация**: `POST user/create/` → валидация трёх полей → проверка уникальности username → `User.objects.create_user` → 201 с `{id, username, email}`.
2. **Логин**: `POST login/` → `authenticate()` → `Token.objects.get_or_create` → `{'token': key}` / 401 `{'error': 'Invalid credentials'}`.
3. **Логаут**: заголовок `Authorization: Token <key>` → `TokenAuthentication` → удаление токена + `logout(request)` → 200.
4. **Создание задачи**: auth → `creator = request.user.pk`; если `executor == creator` → 400; если executor не существует → `executor = None`; `TaskSerializer` → 201 или 400 с `serializer.errors`.
5. **Взятие задачи**: `PATCH become-executor/<id>/` → 404 если нет задачи → 400 если я создатель → 400 если исполнитель уже есть → назначение + 200.
6. **Завершение**: `PATCH mark-task-done/<id>/` → 404 если нет → 403 если я не исполнитель → `is_done=True` → 200 с телом задачи.
7. **Статистика**: агрегаты `Count`/`Sum` по `creator`/`executor`, `deadline__lt=now()` для просрочек, `None → 0`.
8. **Сброс данных**: `GET clear_db/` → `Task.objects.all().delete()` + `User.objects.all().delete()` → 200.

## Зоны повышенного риска
- **`ClearDatabaseView` без аутентификации и по GET** — любой запрос стирает всех пользователей и задачи. В `Readme.md` описан как `post`, в коде — `get` (тесты полагаются на GET). Критическая уязвимость для любого не-локального запуска.
- **`SECRET_KEY` захардкожен**, `DEBUG = True`, `ALLOWED_HOSTS = []` — конфиг непригоден для продакшена.
- **`db.sqlite3` в репозитории** — данные и возможные учётки уезжают в git.
- **`MarkTaskDoneAPIView` возвращает 403**, тогда как `Readme.md` требует 400 `{'error': 'You are not authorized...'}` — расхождение спецификации и кода.
- **`UserTasksStatsAPIView`**: `completed_tasks`/`pending_tasks` считаются по `creator`, а `total_earned` — по `executor`; семантика «completed» неоднозначна относительно спецификации.
- **`TaskCreateView`**: `int(executor_id)` без try/except → 500 на нечисловом `executor`; мутация `request.data` напрямую (для QueryDict может быть immutable).
- **Отладочные `print()`** в `TaskCreateView` и `TaskWithExecutorAPIView` — шум в логах продакшена.
- **`from rest_framework.views import *` и `from .models import *`** — неявные импорты, тени имён.
- **`executor` с `on_delete=CASCADE`** при `null=True`: удаление исполнителя удаляет задачу целиком вместо `SET_NULL`.
- **Тесты деструктивны**: прогон по BASE_URL стирает БД — запуск против непустой среды недопустим.
- **Миграции**: `0002_delete_task` + `0003_initial` — история пересоздания модели, применение на существующей БД теряет данные.
- **`LogoutView`** использует `get_or_create` вместо `get` — создаёт токен, чтобы тут же его удалить.

## Открытые вопросы
- Должен ли `clear_db/` остаться в кодовой базе, и если да — как его защитить (только DEBUG / только POST / токен)?
- Какой код ответа правильный для `mark-task-done` при чужой задаче: 400 (Readme) или 403 (код)?
- `completed_tasks` в статистике — это задачи, созданные мной и завершённые, или выполненные мной? Требуется уточнение спецификации.
- Нужны ли Django-юнит-тесты (`app/tests.py`) вместо/в дополнение к HTTP-тестам, чтобы не зависеть от живого сервера?
- Нужна ли регистрация `Task` в админке?
- Планируется ли деплой (тогда обязательны env-переменные, `DEBUG=False`, PostgreSQL)?
- Нужна ли пагинация для списковых эндпоинтов (`task/executor/`, `unassigned-tasks/`)?
- Почему `Readme.md` описывает URL с префиксом `app/`, а фактический префикс — `api/`? Какой считать каноничным?

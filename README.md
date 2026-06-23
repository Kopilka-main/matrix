# Matrix (Synapse) + Element для FitBond

Корпоративный мессенджер на базе Matrix. Разворачивается **параллельно** с прототипом
в `/opt/messenger` — ничего не удаляет, использует отдельную БД и отдельные поддомены.

- Сервер: `matrix.fitbond.ru` (Synapse)
- Клиент: `chat.fitbond.ru` (Element Web)
- ID пользователей: `@login:fitbond.ru` (через well-known делегацию)
- Изолированный on-premise: федерация выключена, регистрация закрыта, E2E включён по умолчанию.

## Состав

| Файл | Назначение |
|---|---|
| `docker-compose.yml` | Synapse + PostgreSQL + Element |
| `synapse/conf.d/override.yaml` | Несекретные настройки (коммитится) |
| `synapse/conf.d/local.yaml` | Секреты: БД, TURN (НЕ коммитится) |
| `element/config.json` | Настройки клиента + брендинг (название, цвета) |
| `caddy/fitbond.matrix.caddy` | Что добавить в общий Caddy |

---

## Развёртывание на VPS (по шагам)

Предполагается, что репозиторий склонирован в `/opt/matrix`.

### 1. DNS
Добавьте A-записи на тот же IP, что и прототип:
```
matrix.fitbond.ru  -> <IP VPS>
chat.fitbond.ru     -> <IP VPS>
```

### 2. Секреты
```bash
cd /opt/matrix
cp .env.example .env
# впишите длинный случайный SYNAPSE_DB_PASSWORD
nano .env

cp synapse/conf.d/local.yaml.example synapse/conf.d/local.yaml
# впишите тот же пароль БД + TURN_USER/TURN_PASSWORD из /opt/messenger/.env
nano synapse/conf.d/local.yaml
```

### 3. Сгенерировать базовый конфиг Synapse (создаёт ключи и секреты)
```bash
docker run --rm \
  -v "$(pwd)/synapse:/data" \
  -e SYNAPSE_SERVER_NAME=fitbond.ru \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```
Создаст `synapse/homeserver.yaml` и `synapse/fitbond.ru.signing.key`.
Наши `override.yaml` и `local.yaml` загружаются поверх и переопределяют нужное.

### 4. Запуск
```bash
docker compose up -d
docker compose logs -f synapse   # дождаться "Synapse now listening on TCP port 8008"
```

### 5. Caddy
Подключите контейнер Caddy к сети `proxy` (если ещё не подключён) и добавьте блоки
из `caddy/fitbond.matrix.caddy`, затем перезагрузите Caddy. Проверка:
```bash
curl -s https://matrix.fitbond.ru/_matrix/client/versions   # должен вернуть JSON
```

### 6. Создать админа
```bash
docker compose exec synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008 -a
```
Затем зайдите на `https://chat.fitbond.ru` под этим логином.

### 7. Завести пользователей
Тем же `register_new_matrix_user` без `-a` (обычный пользователь). Позже подключим
SSO/LDAP, если нужно (отдельный шаг).

---

## Обновление
```bash
cd /opt/matrix
docker compose pull
docker compose up -d
```
Следите за бюллетенями безопасности Synapse и обновляйтесь регулярно.

## Что дальше (после пилота)
- Логотип/фавикон Element (пересборка образа с подменой ассетов).
- SSO/LDAP-вход (Keycloak или ваш каталог).
- Групповые видеозвонки: Element Call (LiveKit) поверх текущего coturn.
- Нагрузочное тестирование и worker'ы Synapse под тысячи пользователей.

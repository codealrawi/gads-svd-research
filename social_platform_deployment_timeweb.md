# Развёртывание демо-режима платформы на сервере Timeweb (инструкция, не выполнялась)

**Статус.** Инструкция подготовлена; **ничего не публиковалось и не создавалось**, образ Docker не собирался (на машине разработки Docker не установлен). Документация Timeweb для этого документа не использовалась: используется общий путь «облачный сервер (VPS) + Docker Compose + Caddy», как в `docs/deployment_timeweb.md`; названия пунктов панели управления здесь не приводятся. Публично размещать можно **только режим `demo`** (синтетические данные, только чтение).

## 0. Что понадобится

- Облачный сервер с Ubuntu 22.04/24.04 LTS, ≥ 1 vCPU, ≥ 2 ГБ RAM, ≥ 10 ГБ диска (оценка; пилот на 120 аккаунтах × 40 окон выполняется за секунды–десятки секунд на ноутбуке, нагрузочных замеров нет). Создание сервера и оплата — решение владельца проекта.
- Домен и доступ к его DNS (для HTTPS) либо доступ по SSH-туннелю без публикации.
- SSH-ключ. Не используйте сервер, на котором уже работают другие проекты.

## 1. Подготовка сервера

См. `docs/deployment_timeweb.md`, разделы 1–2 (пользователь без root, SSH по ключу, `ufw`: 22, 80, 443; Docker по официальной инструкции). Порт приложения 8100 наружу не открывать: в `compose.yaml` он привязан к `127.0.0.1`.

## 2. Доставка кода

С машины разработки, из корня репозитория (без `.venv`, `data*`, `results`, `research/sources`):

```bash
rsync -av --exclude .venv --exclude 'data*' --exclude results --exclude research/sources --exclude __pycache__ ./ <пользователь>@<IP>:/opt/gads-platform/
```

Нужны: `requirements-app.txt`, `src/`, `social_platform/`, `adaptive_spec/ki_gate_v1_3.py`, `adaptive_spec/e0_runner.py`.

## 3. Конфигурация

```bash
cd /opt/gads-platform/social_platform/deploy
cp env.example .env && chmod 600 .env
# PLATFORM_MODE=demo   PLATFORM_DOMAIN=demo.<ваш-домен>   COOKIE_SECURE=true
# RESEARCH_MODE_ENABLED=false  (не менять)
```

Секреты (`ADMIN_PASSWORD`) в `.env` не хранить: передаются только в окружение одной команды.

## 4. Первый запуск, миграции, данные демо

```bash
docker compose --profile prod up -d --build
docker compose exec app python -m social_platform.cli init-db        # миграции применяются и при старте
docker compose exec app python -m social_platform.cli seed-demo      # синтетические данные
docker compose exec -e ADMIN_PSEUDONYM=<имя> -e ADMIN_PASSWORD=<пароль> app python -m social_platform.cli create-admin
curl -s https://demo.<ваш-домен>/api/health
```

Проверьте в браузере: баннер «ДЕМО», регистрация закрыта, публикации помечены «синтетика», панель исследования доступна только администратору и показывает «сбор событий участников: выключен».

## 5. HTTPS

A-запись домена → IP сервера; Caddy получает сертификат автоматически при первом обращении (нужны открытые 80 и 443). `COOKIE_SECURE=true` обязателен за HTTPS. Заголовки безопасности добавляет приложение; HSTS — Caddy.

## 6. Резервные копии и проверка восстановления

```bash
docker compose exec app python -m social_platform.cli backup           # копия + проверка целостности и числа строк
docker compose cp app:/data/backups ./backups                          # вынести копию с сервера по защищённому каналу
```

Восстановление: остановить `app`, заменить файл `/data/social_platform.sqlite3` копией, запустить, проверить `/api/health`, число записей в панели, `PRAGMA integrity_check`. Периодичность и срок хранения копий — по решению владельца (в режиме `research` не дольше документированного срока хранения).

## 7. Обновление и откат

```bash
cd /opt/gads-platform/social_platform/deploy
docker compose exec app python -m social_platform.cli backup            # 1) копия
# 2) залить новую версию кода (rsync, как в п. 2)
docker compose --profile prod up -d --build                            # 3) пересборка; миграции — при старте
docker compose logs --tail=50 app && curl -s https://demo.<ваш-домен>/api/health
```

Откат: вернуть предыдущий код (из архива предыдущей версии), `docker compose --profile prod up -d --build`; если миграция изменила схему — восстановить копию БД из п. 1. Миграции в прототипе только добавляют; откат схемы вручную не автоматизирован.

## 8. Что нельзя делать на публичном сервере

Не менять `PLATFORM_MODE` на `research`; не загружать реальные данные пользователей; не включать внешнюю аналитику; не открывать порт 8100 и файл БД наружу. Исследовательский режим на публичном демо-сервере технически закрыт: база, созданная в `demo`, не может быть открыта в `research`, а запуск `research` требует файла подтверждений.

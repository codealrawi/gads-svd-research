# Развёртывание на сервере Timeweb (пошагово)

**Статус:** инструкция подготовлена, **фактическое развёртывание не выполнялось**; `Dockerfile`, `compose.yaml` и `deploy/Caddyfile` на этой машине не собирались (Docker не установлен). Первый запуск на сервере считается проверкой развёртываемости — см. раздел 9. Интерфейс панели управления Timeweb здесь не воспроизводится (названия пунктов могут отличаться): ниже указано, *что* нужно получить, а не где нажимать.

Прототип — демонстрационный: только синтетические данные, оценка аномальности ≠ вывод о нарушении, метод GADS v0 не валидирован. Не загружайте на сервер персональные и медицинские данные.

## 0. Что понадобится

- Облачный сервер/VPS (Ubuntu 22.04 или 24.04 LTS), **1–2 vCPU, ≥ 2 ГБ RAM, ≥ 10 ГБ диска** (оценка: наборы до 3000 аккаунтов × 80 окон; запуск B3/B4/B5 заметно медленнее B0–B2). Создание сервера — ваше решение и ваши расходы; **этот проект ничего не создаёт и не заказывает**.
- Отдельный сервер или чистая установка: **не используйте сервер проекта архитектурного бенчмарка** и не меняйте его конфигурацию.
- Домен (для HTTPS) и возможность править его DNS-записи; либо только IP + HTTP для закрытой проверки (без HTTPS токен передаётся открытым текстом — не рекомендуется).
- SSH-ключ.

## 1. Первичная настройка сервера

```bash
ssh root@<IP_СЕРВЕРА>
adduser gads && usermod -aG sudo gads            # пользователь без root
rsync --archive --chown=gads:gads ~/.ssh /home/gads
# в /etc/ssh/sshd_config: PasswordAuthentication no ; PermitRootLogin no ; затем:
systemctl restart ssh
apt update && apt -y upgrade
apt -y install ufw unattended-upgrades
ufw default deny incoming && ufw default allow outgoing
ufw allow OpenSSH && ufw allow 80/tcp && ufw allow 443/tcp
ufw enable
```

Порты 8000 (приложение) наружу **не открываются**: в `compose.yaml` он привязан к `127.0.0.1`. База (SQLite) — файл в томе Docker, сетевого доступа к ней нет.

## 2. Установка Docker

Следуйте официальной инструкции Docker для Ubuntu (репозиторий `docs.docker.com/engine/install/ubuntu/`), затем:

```bash
sudo usermod -aG docker gads   # перелогиньтесь
docker --version && docker compose version
```

## 3. Доставка кода

Скопируйте каталог проекта на сервер без `.venv`, `data*`, `results/raw`, `research/sources`:

```bash
# с вашей машины (PowerShell/WSL/Git Bash), из папки проекта:
rsync -av --exclude .venv --exclude 'data*' --exclude results/raw --exclude research/sources --exclude __pycache__ ./ gads@<IP>:/opt/gads-svd/
```

Нужны: `Dockerfile`, `compose.yaml`, `requirements-app.txt`, `src/`, `deploy/`, `scripts/backup.py`, `.env.example`.

## 4. Конфигурация (`.env`)

```bash
cd /opt/gads-svd
cp .env.example .env
python3 -c "import secrets; print(secrets.token_urlsafe(32))"   # скопируйте в GADS_ACCESS_TOKEN
nano .env      # GADS_ACCESS_TOKEN=<токен>   GADS_DOMAIN=demo.<ваш-домен>
chmod 600 .env
```

Токен нужен всегда на публичном сервере: без него API открыт всем. Токен вводится в интерфейсе один раз за сессию вкладки. Сервер не хранит учётных записей.

## 5. Домен и DNS

Создайте у регистратора/в DNS-панели **A-запись** `demo.<ваш-домен> → <IP_СЕРВЕРА>` (и AAAA при наличии IPv6). Дождитесь распространения (`dig +short demo.<ваш-домен>`). Caddy получает сертификат Let's Encrypt автоматически при первом обращении; для этого должны быть открыты 80 и 443 и корректен DNS.

## 6. Запуск

```bash
cd /opt/gads-svd
docker compose --profile prod up -d --build
docker compose ps
docker compose logs -f app caddy       # Ctrl+C для выхода
curl -s https://demo.<ваш-домен>/api/health        # {"status":"ok",...}
```

Откройте `https://demo.<ваш-домен>/`, введите токен, создайте демонстрационный набор и запустите анализ.

Без домена (проверка по IP/через SSH-туннель): `docker compose up -d --build` (профиль без Caddy) и `ssh -L 8000:127.0.0.1:8000 gads@<IP>`, затем `http://127.0.0.1:8000`.

## 7. Обновление версии

```bash
cd /opt/gads-svd
python3 scripts/backup.py /var/lib/docker/volumes/gads-svd_gads-data/_data /opt/gads-backups   # сначала резервная копия (sudo при необходимости)
# загрузите новую версию кода (rsync, как в п. 3)
docker compose --profile prod up -d --build
docker compose logs --tail=50 app
curl -s https://demo.<ваш-домен>/api/health
```

Схема БД версионируется (`schema_version`, миграции применяются при старте). Откат: верните предыдущий код, пересоберите; при несовместимой миграции восстановите резервную копию (п. 8).

## 8. Резервное копирование и восстановление

Все данные приложения — каталог тома `gads-data` (`gads.sqlite3` + `datasets/*.npz`).

```bash
# консистентная копия «на ходу» (SQLite online backup + файлы наборов + контрольные суммы):
sudo python3 scripts/backup.py /var/lib/docker/volumes/gads-svd_gads-data/_data /opt/gads-backups
# ежедневно в 03:15 (crontab -e под пользователем с доступом к тому):
15 3 * * * /usr/bin/python3 /opt/gads-svd/scripts/backup.py /var/lib/docker/volumes/gads-svd_gads-data/_data /opt/gads-backups
# храните ≥ 7 копий, переносите их за пределы сервера (объектное хранилище/другая машина)
```

Восстановление: `docker compose stop app`, скопируйте `gads.sqlite3` и `datasets/` из копии в каталог тома, `docker compose start app`. Проверьте `MANIFEST.sha256`. Размер: ориентировочно 4 МБ на один запуск набора 1000×30 (оценки хранятся в БД); удаление набора через интерфейс удаляет его запуски и файлы.

## 9. Проверка после развёртывания (чек-лист)

1. `GET /api/health` → `status: ok`, `token_required: true`.
2. Запрос без токена к `/api/datasets` → 401.
3. Создание демо-набора, запуск B2 и GADS v0, открытие результатов, карточки аккаунта, экспорт CSV/JSON.
4. Загрузка файла > 10 МБ → отказ; загрузка `.txt` → отказ.
5. `curl -I http://<IP>:8000` снаружи → соединение не устанавливается (порт закрыт).
6. `ufw status`: открыты только 22, 80, 443.
7. Перезапуск `docker compose restart app`: данные на месте.

## 10. Безопасность и эксплуатация

- Обновляйте ОС и образы (`docker compose pull && docker compose up -d --build`), следите за логами.
- Не публикуйте токен; при утечке замените его в `.env` и перезапустите `app`.
- Не добавляйте реальные данные; для реальной платформы нужны правовые основания, обезличивание и отдельная модель угроз.
- Мониторинг: `docker compose ps`, `docker stats`, `curl /api/health`; healthcheck встроен в образ.
- Ограничения: один рабочий процесс и очередь внутри процесса; без многопользовательской авторизации; для нагрузки выше учебной демонстрации архитектуру придётся менять (внешняя очередь, Postgres).
- Домен/HTTPS: если сертификат не выдаётся, проверьте DNS и порты 80/443, `docker compose logs caddy`.
- Не подключайте этот сервер к инфраструктуре и данным проекта архитектурных экспериментов.

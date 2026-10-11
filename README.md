# GADS-SVD — исследовательское ядро и веб-прототип

Проект состоит из двух связанных частей, использующих **один и тот же код ядра**:

| Часть | Что внутри |
|---|---|
| A. Исследовательское ядро | генератор синтетических временных данных (`src/data`), baseline B0–B5 (`src/baselines`), конструкция GADS-SVD v0 (`src/gads_svd`), префиксная оценка и метрики (`src/evaluation`), тесты причинности/утечек, пилот |
| B. Веб-прототип | FastAPI (`src/api`) + статический интерфейс на русском (`src/ui/static`), SQLite, запуск анализа, результаты, карточка аккаунта, сравнение, экспорт CSV/JSON |

**Научный статус.** Основная научная серия экспериментов **не запускалась**; гипотезы H1–H5 не подтверждены. GADS-SVD v0 — исследовательская конструкция с невалидированными параметрами, новизна не подтверждена (`research/research_gap.md`). Оценка аномальности — степень отклонения от модели поведения, не доказательство нарушения; система ничего не блокирует. Демонстрационный режим (синтетика) не смешивается с результатами научной серии.

## Быстрый запуск (локально, Windows/Linux/macOS, Python 3.12)

```bash
python -m venv .venv
.venv\Scripts\activate            # Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt   # полное окружение (включая тесты); для одного приложения хватит requirements-app.txt
python -m uvicorn src.api.main:create_app --factory --host 127.0.0.1 --port 8000
```

Откройте <http://127.0.0.1:8000> → «Данные» → «Создать набор» → «Анализ» → «Запустить анализ» → результаты → экспорт.
Данные хранятся в `./data` (переменная `GADS_DATA_DIR`). Токен доступа и лимиты — через `.env` (см. `.env.example`; без `GADS_ACCESS_TOKEN` токен не требуется — только для локального использования).

Одной командой через Docker (на этой машине **не проверялось**, Docker не установлен):

```bash
docker compose up --build         # http://127.0.0.1:8000
```

Развёртывание на сервере Timeweb: `docs/deployment_timeweb.md`.

## Тесты

```bash
python -m pytest tests -q
```

150 тестов: математические тождества, генератор, baseline, причинность/утечки/калибровка порога, метрики, GADS v0 (механика), API и сквозной сценарий «загрузка → анализ → результаты → экспорт» (сравнение с прямым вызовом ядра). Прохождение тестов не доказывает научную гипотезу.

## Структура

```
src/data/          генератор S0–S9 (WindowView отделён от WindowTruth)
src/baselines/     B0 static SVD, B1 rolling SVD, B2 forgetting PCA, B2' incremental SVD, B3 RPCA(PCP), B4 IF, B5 LOF
src/gads_svd/      subspace_alignment.py, detector.py (GADS v0)
src/evaluation/    harness.py (префиксный цикл, калибровка), metrics.py
src/api/           config, db (миграции), csv_io, algorithms (реестр), services, main (FastAPI)
src/ui/static/     index.html, app.css, app.js, charts.js (SVG), i18n/ru.json, example_synthetic.csv
docs/              модель, гипотезы, карточка датасета, baseline, архитектура, формат CSV, развёртывание
research/          литература, пробел, протокол, журнал изменений, манифесты
reports/           отчёты по контрольным точкам
scripts/           пилот, сводка, манифесты, backup.py
tests/
```

## Документы

`docs/prototype_architecture.md` · `docs/csv_format.md` · `docs/deployment_timeweb.md` · `docs/gads_v0_implementation.md` · `docs/mathematical_model.md` · `docs/hypotheses.md` · `docs/dataset_card.md` · `docs/baselines.md` · `research/experiment_protocol.md` · `research/protocol_amendments.md` · `reports/checkpoint_*.md`.

## Ограничения и безопасность (кратко)

Один рабочий процесс, очередь в памяти процесса, SQLite — для демонстрации. Загрузка: только `.csv`, лимиты размера/строк/аккаунтов, строгая проверка схемы, данные не исполняются; идентификаторы аккаунтов — только псевдонимы. Секреты — через окружение; `.env` в `.gitignore`. Реальные персональные и медицинские данные в демо не используются.

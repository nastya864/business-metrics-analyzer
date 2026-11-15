# Business Metrics Analyzer

Учебный проект по дисциплине «Проектирование и разработка бизнес-ориентированного программного обеспечения».

## Бизнес-задача

Расчёт и визуализация ключевых показателей эффективности (KPI) предприятия.

## Планируемые метрики

- Выручка (Revenue)
- Себестоимость (Cost)
- Рентабельность (Profitability)
- Средний чек (Average Check)

## Текущий прототип

- Python 3.10+
- pandas
- matplotlib

Целевая архитектура предусматривает FastAPI, PostgreSQL, Redis, S3/MinIO, Nginx и Docker. Подробности приведены в разделе [«Архитектура приложения»](docs/architecture/README.md).

## Запуск

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python main.py
```

## Проектная документация

- [Бизнес-требования](docs/business_requirements.md)
- [Протокол интервью](docs/interview_protocol.md)
- [User Stories](docs/user_stories.md)
- [Описание ключевых Use Cases](docs/use_cases.md)
- [Use Case-диаграмма](docs/use_case_diagram.png)
- [Product Backlog](docs/product_backlog.md)
- [Ответы на контрольные вопросы](docs/control_questions.md)
- [Архитектура приложения](docs/architecture/README.md)

## Автор

Студент группы Б1123.38.03.05(2) Шишигина Анастасия Евгеньевна.

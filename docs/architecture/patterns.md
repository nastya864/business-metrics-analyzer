# Паттерны проектирования Business Metrics Analyzer

## 1. Выбранные паттерны

| Паттерн | Где применяется | Обоснование | Альтернатива |
|---|---|---|---|
| Repository | Доступ к данным о продажах и пользователях | Изолирует бизнес-логику от SQLAlchemy и PostgreSQL, упрощает модульное тестирование | Прямые ORM-запросы из сервисов |
| Unit of Work | Импорт продаж и операции, изменяющие несколько сущностей | Обеспечивает единую транзакцию и откат всех изменений при ошибке | Ручное управление транзакциями в каждом сервисе |
| Service Layer | `AuthService`, `SalesImportService`, `KpiService`, `ReportService`, `UserService` | Сосредотачивает пользовательские сценарии вне HTTP-контроллеров | Бизнес-логика в контроллерах FastAPI |
| Strategy | Расчёт рентабельности, среднего чека, выручки и других KPI | Позволяет добавлять и заменять формулы расчёта без изменения `KpiService` | Условные операторы для каждого типа KPI |
| Factory | Создание Excel- и PDF-отчётов | Скрывает выбор конкретного генератора и упрощает добавление форматов | Создание генераторов непосредственно в `ReportService` |
| DTO | Запросы и ответы REST API, результаты импорта и расчёта KPI | Формирует явные контракты API и не раскрывает внутренние ORM-модели | Передача словарей или ORM-объектов |
| Adapter | Интеграции с 1С, файловым хранилищем и SMTP | Изолирует внешние сервисы и позволяет заменять их реализации | Прямые вызовы внешних библиотек из сервисов |
| Observer | Уведомление пользователей после обновления KPI | Отделяет расчёт показателей от отправки email-уведомлений | Явный вызов уведомлений из каждого сценария |

## 2. Repository и Unit of Work

`SaleRepository` задаёт интерфейс работы с продажами. Реализация для PostgreSQL находится в инфраструктурном слое. `UnitOfWork` объединяет репозитории и управляет границами транзакции.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal


@dataclass
class Sale:
    month: str
    revenue: Decimal
    cost: Decimal
    id: int | None = None


class SaleRepository(ABC):
    @abstractmethod
    def add(self, sale: Sale) -> None:
        raise NotImplementedError

    @abstractmethod
    def get_by_period(self, period: str) -> list[Sale]:
        raise NotImplementedError


class SqlSaleRepository(SaleRepository):
    def __init__(self, session) -> None:
        self.session = session

    def add(self, sale: Sale) -> None:
        self.session.add(sale)

    def get_by_period(self, period: str) -> list[Sale]:
        return self.session.query(Sale).filter_by(month=period).all()


class UnitOfWork:
    def __init__(self, session_factory) -> None:
        self.session_factory = session_factory

    def __enter__(self):
        self.session = self.session_factory()
        self.sales = SqlSaleRepository(self.session)
        return self

    def __exit__(self, exc_type, exc_value, traceback) -> None:
        if exc_type is None:
            self.session.commit()
        else:
            self.session.rollback()
        self.session.close()
```

Пример использования в прикладном сервисе:

```python
class SalesImportService:
    def __init__(self, unit_of_work_factory) -> None:
        self.unit_of_work_factory = unit_of_work_factory

    def import_sales(self, records: list[Sale]) -> int:
        with self.unit_of_work_factory() as unit_of_work:
            for record in records:
                unit_of_work.sales.add(record)
        return len(records)
```

## 3. Strategy для расчёта KPI

Общий интерфейс позволяет `KpiService` работать с любой формулой расчёта.

```python
from abc import ABC, abstractmethod


class KpiStrategy(ABC):
    @abstractmethod
    def calculate(self, **values: float) -> float:
        raise NotImplementedError


class ProfitabilityStrategy(KpiStrategy):
    def calculate(self, **values: float) -> float:
        revenue = values["revenue"]
        cost = values["cost"]
        return 0.0 if revenue == 0 else (revenue - cost) / revenue * 100


class AverageCheckStrategy(KpiStrategy):
    def calculate(self, **values: float) -> float:
        revenue = values["revenue"]
        orders = values["orders"]
        return 0.0 if orders == 0 else revenue / orders


class KpiService:
    def __init__(self, strategies: dict[str, KpiStrategy]) -> None:
        self.strategies = strategies

    def calculate(self, metric: str, **values: float) -> float:
        strategy = self.strategies[metric]
        return strategy.calculate(**values)
```

## 4. Factory для формирования отчётов

Фабрика выбирает генератор по запрошенному формату. `ReportService` не зависит от конкретной библиотеки создания файла.

```python
from abc import ABC, abstractmethod


class ReportGenerator(ABC):
    @abstractmethod
    def generate(self, data: list[dict]) -> bytes:
        raise NotImplementedError


class ExcelReportGenerator(ReportGenerator):
    def generate(self, data: list[dict]) -> bytes:
        # В рабочей реализации здесь используется openpyxl.
        return b"excel report"


class PdfReportGenerator(ReportGenerator):
    def generate(self, data: list[dict]) -> bytes:
        return b"pdf report"


class ReportGeneratorFactory:
    _generators = {
        "xlsx": ExcelReportGenerator,
        "pdf": PdfReportGenerator,
    }

    @classmethod
    def create(cls, report_type: str) -> ReportGenerator:
        try:
            return cls._generators[report_type]()
        except KeyError as error:
            raise ValueError(f"Неподдерживаемый формат: {report_type}") from error
```

## 5. Правила применения

- Интерфейсы репозиториев и стратегий располагаются в Domain или Application.
- Реализации, зависящие от PostgreSQL, Redis, S3/MinIO и SMTP, располагаются в Infrastructure.
- Контроллеры FastAPI работают с DTO и вызывают Service Layer.
- Доменные модели не импортируют FastAPI, SQLAlchemy, pandas или openpyxl.
- Внедрение зависимостей выполняется при запуске приложения, а не внутри бизнес-сервисов.

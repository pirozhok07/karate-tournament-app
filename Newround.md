Вот реализация SQL-запроса на SQLAlchemy ORM с использованием оконных функций и CTE:

1. Определение модели

```python
from sqlalchemy import Column, Integer, Float, func, desc, over, case_, text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import Session
from sqlalchemy.sql.expression import cast

Base = declarative_base()

class Score(Base):
    __tablename__ = 'оценки'
    
    id = Column(Integer, primary_key=True)
    id_спортсмена = Column(Integer, nullable=False)
    оценка_1 = Column(Float, nullable=False)
    оценка_2 = Column(Float, nullable=False)
    оценка_3 = Column(Float, nullable=False)
    оценка_4 = Column(Float, nullable=False)
    оценка_5 = Column(Float, nullable=False)
    сумма_3_средних = Column(Float, nullable=False)
```

2. Реализация запроса с оконными функциями

```python
from sqlalchemy import select, func, desc, and_, or_
from sqlalchemy.sql.expression import literal

def select_athletes_with_rankings(session: Session, n: int):
    """
    Выбирает спортсменов с их рейтингом и позицией
    Возвращает запрос с колонками:
    - id_спортсмена
    - сумма_3_средних
    - максимальная_оценка
    - минимальная_оценка
    - место (rank)
    """
    
    # Вычисление максимальной и минимальной оценки
    max_score = func.greatest(
        Score.оценка_1,
        Score.оценка_2, 
        Score.оценка_3,
        Score.оценка_4,
        Score.оценка_5
    ).label('максимальная_оценка')
    
    min_score = func.least(
        Score.оценка_1,
        Score.оценка_2, 
        Score.оценка_3,
        Score.оценка_4,
        Score.оценка_5
    ).label('минимальная_оценка')
    
    # Используем оконную функцию для ранжирования
    row_number = func.row_number().over(
        order_by=[
            desc(Score.сумма_3_средних),
            desc(max_score),
            desc(min_score)
        ]
    ).label('место')
    
    query = select(
        Score.id_спортсмена,
        Score.сумма_3_средних,
        max_score,
        min_score,
        row_number
    ).select_from(Score)
    
    return query
```

3. Функция для отбора спортсменов с обработкой пограничных случаев

```python
def select_next_round_athletes(session: Session, n: int):
    """
    Отбирает n спортсменов для следующего раунда
    Возвращает кортеж: (прошедшие, нужен_доп_раунд, спортсмены_доп_раунда)
    """
    # Получаем всех спортсменов с ранжированием
    ranking_query = select_athletes_with_rankings(session, n)
    
    # Выполняем запрос
    result = session.execute(ranking_query).fetchall()
    
    if len(result) <= n:
        # Если всех спортсменов <= n, то все проходят
        return result, False, []
    
    # Берем первых n спортсменов
    selected = result[:n]
    border_athlete = selected[-1]  # Пограничный спортсмен
    
    # Проверяем, есть ли спортсмены с такими же параметрами после n-го места
    border_sum = border_athlete.сумма_3_средних
    border_max = border_athlete.максимальная_оценка
    border_min = border_athlete.минимальная_оценка
    
    # Ищем спортсменов с такими же параметрами, начиная с n-го места
    same_params_athletes = []
    for athlete in result[n-1:]:  # Начинаем с n-го (индекс n-1)
        if (athlete.сумма_3_средних == border_sum and
            athlete.максимальная_оценка == border_max and
            athlete.минимальная_оценка == border_min):
            same_params_athletes.append(athlete)
    
    if len(same_params_athletes) > 1:
        # Нужен дополнительный раунд для всей группы
        # Проходят только спортсмены до этой группы
        passed_athletes = result[:n-1]
        playoff_athletes = same_params_athletes
        return passed_athletes, True, playoff_athletes
    else:
        # Все определено четко
        return selected, False, []
```

4. Альтернативная реализация с использованием CTE

```python
def select_athletes_cte(session: Session, n: int):
    """
    Реализация с использованием Common Table Expression (CTE)
    """
    from sqlalchemy.sql.expression import literal_column
    
    # CTE с вычисленными полями
    athletes_cte = (
        select(
            Score.id_спортсмена,
            Score.сумма_3_средних,
            func.greatest(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('max_score'),
            func.least(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('min_score')
        )
        .cte('athletes_cte')
    )
    
    # Основной запрос с ранжированием
    ranked_query = (
        select(
            athletes_cte.c.id_спортсмена,
            athletes_cte.c.сумма_3_средних,
            athletes_cte.c.max_score,
            athletes_cte.c.min_score,
            func.row_number().over(
                order_by=[
                    desc(athletes_cte.c.сумма_3_средних),
                    desc(athletes_cte.c.max_score),
                    desc(athletes_cte.c.min_score)
                ]
            ).label('rank')
        )
        .select_from(athletes_cte)
    )
    
    # Выполняем запрос
    result = session.execute(ranked_query).fetchall()
    
    # Анализ результатов аналогичен предыдущей функции
    if len(result) <= n:
        return result, False, []
    
    selected = result[:n]
    border = selected[-1]
    
    # Находим спортсменов с одинаковыми параметрами
    same_group = []
    for athlete in result:
        if (athlete.rank >= n and
            athlete.сумма_3_средних == border.сумма_3_средних and
            athlete.max_score == border.max_score and
            athlete.min_score == border.min_score):
            same_group.append(athlete)
    
    if len(same_group) > 1:
        passed = result[:n-1]
        playoff = same_group
        return passed, True, playoff
    
    return selected, False, []
```

5. Универсальная функция с параметрами

```python
from dataclasses import dataclass
from typing import List, Tuple, Optional

@dataclass
class SelectionResult:
    selected_athletes: List
    needs_playoff: bool
    playoff_athletes: List
    border_values: Optional[dict] = None

def select_for_next_round(
    session: Session,
    n: int,
    use_cte: bool = False
) -> SelectionResult:
    """
    Основная функция для отбора спортсменов
    
    Args:
        session: SQLAlchemy сессия
        n: количество спортсменов для отбора
        use_cte: использовать ли CTE подход
    
    Returns:
        SelectionResult с результатами отбора
    """
    
    if use_cte:
        selected, needs_playoff, playoff = select_athletes_cte(session, n)
    else:
        selected, needs_playoff, playoff = select_next_round_athletes(session, n)
    
    border_values = None
    if selected and not needs_playoff:
        # Значения пограничного спортсмена
        border = selected[-1]
        border_values = {
            'сумма_3_средних': border.сумма_3_средних,
            'максимальная_оценка': getattr(border, 'максимальная_оценка', 
                                         getattr(border, 'max_score', None)),
            'минимальная_оценка': getattr(border, 'минимальная_оценка', 
                                        getattr(border, 'min_score', None))
        }
    
    return SelectionResult(
        selected_athletes=selected,
        needs_playoff=needs_playoff,
        playoff_athletes=playoff,
        border_values=border_values
    )
```

6. Пример использования

```python
from sqlalchemy import create_engine

# Создаем соединение
engine = create_engine('postgresql://user:password@localhost/dbname')
SessionLocal = sessionmaker(bind=engine)

# Использование
with SessionLocal() as session:
    result = select_for_next_round(session, n=10)
    
    if result.needs_playoff:
        print(f"Отобрано {len(result.selected_athletes)} спортсменов")
        print(f"Требуется дополнительный раунд для {len(result.playoff_athletes)} спортсменов")
        
        # ID спортсменов для дополнительного раунда
        playoff_ids = [a.id_спортсмена for a in result.playoff_athletes]
        print(f"ID для доп. раунда: {playoff_ids}")
    else:
        print(f"Отобрано {len(result.selected_athletes)} спортсменов")
        
        # ID прошедших спортсменов
        selected_ids = [a.id_спортсмена for a in result.selected_athletes]
        print(f"Прошедшие: {selected_ids}")
```

7. Дополнительный запрос для получения детальной информации

```python
def get_detailed_scores(session: Session, athlete_ids: List[int]):
    """
    Получает детальные оценки для указанных спортсменов
    """
    query = (
        select(Score)
        .where(Score.id_спортсмена.in_(athlete_ids))
        .order_by(
            desc(Score.сумма_3_средних),
            desc(func.greatest(
                Score.оценка_1, Score.оценка_2, Score.оценка_3,
                Score.оценка_4, Score.оценка_5
            )),
            desc(func.least(
                Score.оценка_1, Score.оценка_2, Score.оценка_3,
                Score.оценка_4, Score.оценка_5
            ))
        )
    )
    
    return session.execute(query).fetchall()
```

Ключевые особенности реализации:

1. Использование оконных функций для правильного ранжирования
2. Вычисление MAX/MIN оценок через func.greatest() и func.least()
3. Обработка пограничных случаев при равенстве всех критериев
4. Гибкость подходов - как с CTE, так и без него
5. Возврат структурированного результата с флагом необходимости дополнительного раунда

Эта реализация эффективно работает с базой данных, выполняя сортировку и ранжирование на стороне СУБД, что оптимально для больших наборов данных.



Вы абсолютно правы! Если три или более человек имеют одинаковые граничные оценки, все они попадают в дополнительный раунд. Вот исправленная логика с учетом этого случая:

Исправленный алгоритм

Основная проблема:

При равенстве всех трех параметров у нескольких спортсменов на границе отсечения:

· Все такие спортсмены участвуют в дополнительном раунде
· В текущий раунд проходят только те, кто строго выше этой группы

Обновленная реализация на SQLAlchemy ORM

```python
from sqlalchemy import select, func, desc, and_
from sqlalchemy.orm import Session
from typing import List, Tuple
from dataclasses import dataclass

@dataclass
class SelectionResult:
    """Результат отбора спортсменов"""
    passed_athletes: List  # Прошедшие в следующий раунд
    playoff_athletes: List  # Спортсмены для дополнительного раунда
    needs_playoff: bool  # Флаг необходимости доп. раунда
    border_values: dict  # Значения на границе отсечения


def select_athletes_for_next_round(session: Session, n: int) -> SelectionResult:
    """
    Отбирает n спортсменов для следующего раунда с учетом всех правил
    
    Args:
        session: SQLAlchemy сессия
        n: количество спортсменов для отбора
    
    Returns:
        SelectionResult с результатами
    """
    # 1. Подзапрос с вычислением max/min оценок
    from sqlalchemy.sql.expression import literal
    
    subquery = (
        select(
            Score.id_спортсмена,
            Score.сумма_3_средних,
            Score.оценка_1,
            Score.оценка_2,
            Score.оценка_3,
            Score.оценка_4,
            Score.оценка_5,
            func.greatest(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('max_score'),
            func.least(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('min_score')
        )
        .subquery()
    )
    
    # 2. Основной запрос с ранжированием
    ranked_query = (
        select(
            subquery,
            func.row_number().over(
                order_by=[
                    desc(subquery.c.сумма_3_средних),
                    desc(subquery.c.max_score),
                    desc(subquery.c.min_score)
                ]
            ).label('rank')
        )
        .select_from(subquery)
    )
    
    # 3. Выполняем запрос
    all_results = session.execute(ranked_query).fetchall()
    
    if len(all_results) <= n:
        # Все проходят, если спортсменов <= n
        return SelectionResult(
            passed_athletes=all_results,
            playoff_athletes=[],
            needs_playoff=False,
            border_values={}
        )
    
    # 4. Находим значения на границе (n-й спортсмен)
    border_athlete = all_results[n-1]  # Индексация с 0
    border_values = {
        'sum': border_athlete.сумма_3_средних,
        'max': border_athlete.max_score,
        'min': border_athlete.min_score
    }
    
    # 5. Ищем ВСЕХ спортсменов с такими же параметрами
    # (могут быть как выше, так и ниже границы)
    equal_athletes = []
    for athlete in all_results:
        if (athlete.сумма_3_средних == border_values['sum'] and
            athlete.max_score == border_values['max'] and
            athlete.min_score == border_values['min']):
            equal_athletes.append(athlete)
    
    # 6. Сортируем группу одинаковых спортсменов для определения минимального ранга в группе
    equal_athletes.sort(key=lambda x: x.rank)
    min_rank_in_group = equal_athletes[0].rank
    
    # 7. Логика отбора в зависимости от ситуации
    if len(equal_athletes) == 1:
        # Ситуация 1: Граничный спортсмен один (нет равных)
        passed = all_results[:n]
        return SelectionResult(
            passed_athletes=passed,
            playoff_athletes=[],
            needs_playoff=False,
            border_values=border_values
        )
    else:
        # Ситуация 2: Есть группа одинаковых спортсменов
        # Ситуация 2.1: Группа начинается ДО границы (min_rank < n)
        # Пример: есть 3 одинаковых, ранги 4,5,6 при n=5
        # Тогда проходят 1,2,3 (ранги < 4), а 4,5,6 в доп. раунд
        if min_rank_in_group < n:
            # Все спортсмены с рангом < min_rank_in_group проходят
            passed = [a for a in all_results if a.rank < min_rank_in_group]
            
            # Все спортсмены из группы в доп. раунд
            playoff_group = equal_athletes
            
            return SelectionResult(
                passed_athletes=passed,
                playoff_athletes=playoff_group,
                needs_playoff=True,
                border_values=border_values
            )
        
        # Ситуация 2.2: Группа начинается НА границе (min_rank == n)
        # Пример: есть 3 одинаковых, ранги 5,6,7 при n=5
        # Тогда проходят 1,2,3,4 (ранги < 5), а 5,6,7 в доп. раунд
        elif min_rank_in_group == n:
            # Все спортсмены с рангом < n проходят
            passed = all_results[:n-1]  # До границы
            
            # Все спортсмены из группы в доп. раунд
            playoff_group = equal_athletes
            
            return SelectionResult(
                passed_athletes=passed,
                playoff_athletes=playoff_group,
                needs_playoff=True,
                border_values=border_values
            )
        
        # Ситуация 2.3: Группа начинается ПОСЛЕ границы (min_rank > n)
        # Это не должно происходить при правильной сортировке,
        # но на всякий случай обработаем
        else:
            # Все спортсмены до n проходят
            passed = all_results[:n]
            return SelectionResult(
                passed_athletes=passed,
                playoff_athletes=[],
                needs_playoff=False,
                border_values=border_values
            )
```

Примеры работы алгоритма

Пример 1: Три спортсмена с одинаковыми параметрами на границе

```
Данные:
- Всего спортсменов: 10
- n = 5 (нужно отобрать 5)

Ранги и сумма_3_средних:
1. 95 (проходит)
2. 90 (проходит) 
3. 85 (проходит)
4. 80 (проходит)
5. 75 (граница) ←
6. 75 (такая же сумма)
7. 75 (такая же сумма)
8. 70
9. 65
10. 60

Результат:
- Проходят: ранги 1,2,3,4 (4 человека)
- Доп. раунд: ранги 5,6,7 (3 человека)
```

Пример 2: Группа начинается до границы

```
Данные:
- n = 5

Ранги и сумма_3_средних:
1. 95 (проходит)
2. 90 (проходит)
3. 85 (проходит) 
4. 80 ← начало группы
5. 80 (граница) ←
6. 80 ← продолжение группы
7. 75
8. 70

Результат:
- Проходят: ранги 1,2,3 (3 человека) ← все с рангом < 4
- Доп. раунд: ранги 4,5,6 (вся группа из 3 человек)
```

Пример 3: Граничный спортсмен один (нет равных)

```
Данные:
- n = 5

Ранги:
1. 95
2. 90
3. 85
4. 80
5. 75 ← уникальный
6. 70
7. 65

Результат:
- Проходят: ранги 1-5 (5 человек)
- Доп. раунд: не требуется
```

Вспомогательные функции для работы с результатами

```python
def format_selection_result(result: SelectionResult) -> dict:
    """Форматирует результат отбора для вывода"""
    return {
        'passed_count': len(result.passed_athletes),
        'passed_ids': [a.id_спортсмена for a in result.passed_athletes],
        'playoff_count': len(result.playoff_athletes),
        'playoff_ids': [a.id_спортсмена for a in result.playoff_athletes],
        'needs_playoff': result.needs_playoff,
        'border_values': result.border_values
    }


def print_selection_summary(result: SelectionResult):
    """Выводит сводку по отбору"""
    print(f"=== Результаты отбора ===")
    print(f"Проходят в следующий раунд: {len(result.passed_athletes)} спортсменов")
    print(f"ID прошедших: {[a.id_спортсмена for a in result.passed_athletes]}")
    
    if result.needs_playoff:
        print(f"\nТребуется дополнительный раунд!")
        print(f"Участники доп. раунда: {len(result.playoff_athletes)} спортсменов")
        print(f"ID участников: {[a.id_спортсмена for a in result.playoff_athletes]}")
        print(f"Параметры на границе: сумма={result.border_values['sum']}, "
              f"макс={result.border_values['max']}, мин={result.border_values['min']}")
    else:
        print(f"\nДополнительный раунд не требуется.")
        if result.border_values:
            print(f"Параметры последнего прошедшего: сумма={result.border_values['sum']}, "
                  f"макс={result.border_values['max']}, мин={result.border_values['min']}")


# Пример использования
if __name__ == "__main__":
    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker
    
    engine = create_engine('postgresql://user:pass@localhost/sports_db')
    SessionLocal = sessionmaker(bind=engine)
    
    with SessionLocal() as session:
        # Отбираем 8 спортсменов для следующего раунда
        result = select_athletes_for_next_round(session, n=8)
        
        # Выводим результаты
        print_selection_summary(result)
        
        # Получаем отформатированные данные
        formatted = format_selection_result(result)
        print(f"\nФорматированный результат: {formatted}")
```

Ключевые моменты логики:

1. Группа одинаковых спортсменов может включать любого количества участников (2, 3, 5 и т.д.)
2. Позиция группы относительно границы определяет, кто проходит:
   · Если группа начинается ДО границы → проходят только те, кто ВЫШЕ группы
   · Если группа начинается НА границе → проходят те, кто ДО границы
   · В обоих случаях вся группа идет в доп. раунд
3. Дополнительный раунд нужен для разрешения неоднозначности внутри группы
4. Итоговое количество прошедших может быть меньше n, если группа занимает место на границе

Эта логика корректно обрабатывает любые комбинации одинаковых параметров у спортсменов на границе отсечения.




Понял! Если это последний раунд, то нужно определить конкретные места (1-е, 2-е, 3-е и т.д.) для всех участников с учетом правил распределения мест при равенстве.

Вот полная реализация с распределением мест в последнем раунде:

```python
from sqlalchemy import select, func, desc, and_, case_, literal
from sqlalchemy.orm import Session, aliased
from typing import List, Tuple, Dict, Any
from dataclasses import dataclass
from itertools import groupby


@dataclass
class AthleteRank:
    """Информация о спортсмене с его местом"""
    id_спортсмена: int
    сумма_3_средних: float
    максимальная_оценка: float
    минимальная_оценка: float
    место: int  # 1, 2, 3 и т.д.
    делит_место: bool = False  # Флаг, делит ли место с другими


@dataclass
class FinalRoundResult:
    """Результаты последнего раунда"""
    rankings: List[AthleteRank]  # Список спортсменов с местами
    medalists: Dict[int, List[int]]  # Словарь {место: [id спортсменов]}
    total_athletes: int


def calculate_final_rankings(session: Session) -> FinalRoundResult:
    """
    Рассчитывает итоговые места в последнем раунде
    
    Правила:
    1. Места распределяются по убыванию суммы_3_средних
    2. При равенстве суммы - по максимальной оценке
    3. При равенстве максимальной оценки - по минимальной оценке
    4. При полном равенстве всех параметров - спортсмены делят место
       (например, два спортсмена на 1-м месте, следующий на 3-м)
    """
    
    # 1. Создаем подзапрос с вычисленными полями
    scores_subquery = (
        select(
            Score.id_спортсмена,
            Score.сумма_3_средних,
            func.greatest(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('max_score'),
            func.least(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('min_score'),
            Score.оценка_1,
            Score.оценка_2,
            Score.оценка_3,
            Score.оценка_4,
            Score.оценка_5
        )
        .subquery('scores_subquery')
    )
    
    # 2. Запрос для получения отсортированных данных
    sorted_query = (
        select(
            scores_subquery.c.id_спортсмена,
            scores_subquery.c.сумма_3_средних,
            scores_subquery.c.max_score,
            scores_subquery.c.min_score,
            scores_subquery.c.оценка_1,
            scores_subquery.c.оценка_2,
            scores_subquery.c.оценка_3,
            scores_subquery.c.оценка_4,
            scores_subquery.c.оценка_5,
            func.row_number().over(
                order_by=[
                    desc(scores_subquery.c.сумма_3_средних),
                    desc(scores_subquery.c.max_score),
                    desc(scores_subquery.c.min_score)
                ]
            ).label('row_num')
        )
        .select_from(scores_subquery)
    )
    
    # 3. Получаем все данные
    all_results = session.execute(sorted_query).fetchall()
    
    if not all_results:
        return FinalRoundResult([], {}, 0)
    
    # 4. Группируем спортсменов по одинаковым параметрам
    # Создаем список для хранения групп
    groups = []
    current_group = []
    
    for i, athlete in enumerate(all_results):
        if i == 0:
            current_group.append(athlete)
        else:
            prev_athlete = all_results[i-1]
            # Проверяем, равны ли параметры с предыдущим спортсменом
            if (athlete.сумма_3_средних == prev_athlete.сумма_3_средних and
                athlete.max_score == prev_athlete.max_score and
                athlete.min_score == prev_athlete.min_score):
                # Добавляем в текущую группу
                current_group.append(athlete)
            else:
                # Завершаем текущую группу и начинаем новую
                groups.append(current_group.copy())
                current_group = [athlete]
    
    # Добавляем последнюю группу
    if current_group:
        groups.append(current_group)
    
    # 5. Распределяем места с учетом общих мест
    rankings = []
    medalists = {}
    current_place = 1
    
    for group in groups:
        # Все спортсмены в группе получают одинаковое место
        place = current_place
        shares_place = len(group) > 1  # Делит ли место с другими
        
        # Добавляем спортсменов в группу мест
        medalists[place] = [a.id_спортсмена for a in group]
        
        for athlete in group:
            rankings.append(AthleteRank(
                id_спортсмена=athlete.id_спортсмена,
                сумма_3_средних=athlete.сумма_3_средних,
                максимальная_оценка=athlete.max_score,
                минимальная_оценка=athlete.min_score,
                место=place,
                делит_место=shares_place
            ))
        
        # Увеличиваем текущее место на количество спортсменов в группе
        current_place += len(group)
    
    return FinalRoundResult(
        rankings=rankings,
        medalists=medalists,
        total_athletes=len(all_results)
    )


# Альтернативная реализация с использованием SQL для расчета мест
def calculate_final_rankings_sql(session: Session) -> FinalRoundResult:
    """
    Более эффективная реализация с использованием оконных функций SQL
    для расчета мест непосредственно в запросе
    """
    # Создаем CTE с вычисленными полями и ранжированием
    scores_cte = (
        select(
            Score.id_спортсмена,
            Score.сумма_3_средних,
            func.greatest(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('max_score'),
            func.least(
                Score.оценка_1,
                Score.оценка_2,
                Score.оценка_3,
                Score.оценка_4,
                Score.оценка_5
            ).label('min_score')
        )
        .cte('scores_cte')
    )
    
    # Используем DENSE_RANK для учета общих мест
    # Но DENSE_RANK не подходит, так как при равенстве места должны "пропускать" следующие
    # Например: два на 1-м месте -> следующий на 3-м, а не на 2-м
    
    # Поэтому используем комбинацию ROW_NUMBER и анализ групп
    ranked_cte = (
        select(
            scores_cte.c.id_спортсмена,
            scores_cte.c.сумма_3_средних,
            scores_cte.c.max_score,
            scores_cte.c.min_score,
            # Номер строки для сортировки
            func.row_number().over(
                order_by=[
                    desc(scores_cte.c.сумма_3_средних),
                    desc(scores_cte.c.max_score),
                    desc(scores_cte.c.min_score)
                ]
            ).label('row_num'),
            # Ранг для определения групп
            func.dense_rank().over(
                order_by=[
                    desc(scores_cte.c.сумма_3_средних),
                    desc(scores_cte.c.max_score),
                    desc(scores_cte.c.min_score)
                ]
            ).label('dense_rank')
        )
        .select_from(scores_cte)
        .cte('ranked_cte')
    )
    
    # Вычисляем места с учетом пропусков
    # Место = ROW_NUMBER первой строки в группе
    final_query = (
        select(
            ranked_cte.c.id_спортсмена,
            ranked_cte.c.сумма_3_средних,
            ranked_cte.c.max_score,
            ranked_cte.c.min_score,
            # Место = минимальный row_num в группе dense_rank
            func.min(ranked_cte.c.row_num).over(
                partition_by=ranked_cte.c.dense_rank
            ).label('place'),
            # Флаг, делит ли место
            (func.count().over(partition_by=ranked_cte.c.dense_rank) > 1).label('shares_place'),
            # Порядковый номер в группе (для отладки)
            ranked_cte.c.row_num,
            ranked_cte.c.dense_rank
        )
        .order_by(ranked_cte.c.row_num)
    )
    
    results = session.execute(final_query).fetchall()
    
    # Формируем результат
    rankings = []
    medalists = {}
    
    for result in results:
        place = result.place
        athlete_id = result.id_спортсмена
        
        # Добавляем в список медалистов
        if place not in medalists:
            medalists[place] = []
        medalists[place].append(athlete_id)
        
        # Создаем объект AthleteRank
        rankings.append(AthleteRank(
            id_спортсмена=athlete_id,
            сумма_3_средних=result.сумма_3_средних,
            максимальная_оценка=result.max_score,
            минимальная_оценка=result.min_score,
            место=place,
            делит_место=result.shares_place
        ))
    
    return FinalRoundResult(
        rankings=rankings,
        medalists=medalists,
        total_athletes=len(results)
    )


def print_final_results(result: FinalRoundResult):
    """Выводит итоговые результаты последнего раунда"""
    print("=" * 80)
    print("ИТОГОВЫЕ РЕЗУЛЬТАТЫ ПОСЛЕДНЕГО РАУНДА")
    print("=" * 80)
    
    print(f"\nВсего участников: {result.total_athletes}")
    print(f"Распределение мест:")
    
    # Выводим пьедестал (первые 3 места)
    print("\n🏆 ПОДИУМ 🏆")
    for place in sorted(result.medalists.keys()):
        if place <= 3:
            athletes = result.medalists[place]
            place_symbol = {1: "🥇", 2: "🥈", 3: "🥉"}.get(place, "")
            shared = " (делят место)" if len(athletes) > 1 else ""
            
            print(f"\n{place_symbol} {place}-е место{shared}:")
            for athlete_id in athletes:
                # Находим полную информацию о спортсмене
                athlete_info = next(a for a in result.rankings 
                                   if a.id_спортсмена == athlete_id)
                print(f"   Спортсмен #{athlete_id}: сумма={athlete_info.сумма_3_средних}, "
                      f"макс={athlete_info.максимальная_оценка}, мин={athlete_info.минимальная_оценка}")
    
    # Полная таблица результатов
    print(f"\n📊 ПОЛНАЯ ТАБЛИЦА РЕЗУЛЬТАТОВ:")
    print("-" * 80)
    print(f"{'Место':<6} {'ID':<8} {'Сумма':<10} {'Макс':<8} {'Мин':<8} {'Примечание':<15}")
    print("-" * 80)
    
    for athlete in result.rankings:
        note = "делит место" if athlete.делит_место else ""
        print(f"{athlete.место:<6} {athlete.id_спортсмена:<8} "
              f"{athlete.сумма_3_средних:<10.2f} "
              f"{athlete.максимальная_оценка:<8.2f} "
              f"{athlete.минимальная_оценка:<8.2f} "
              f"{note:<15}")
    
    # Статистика
    print(f"\n📈 СТАТИСТИКА:")
    print(f"- Уникальных мест: {len(result.medalists)}")
    
    # Места, которые делятся
    shared_places = [place for place, athletes in result.medalists.items() 
                     if len(athletes) > 1]
    if shared_places:
        print(f"- Места, которые делятся: {', '.join(map(str, shared_places))}")
        for place in shared_places:
            print(f"  {place}-е место делят {len(result.medalists[place])} спортсмена")
    else:
        print(f"- Все места распределены уникально")


def get_detailed_athlete_info(session: Session, athlete_ids: List[int]) -> List[Dict[str, Any]]:
    """Получает детальную информацию о спортсменах"""
    query = (
        select(Score)
        .where(Score.id_спортсмена.in_(athlete_ids))
        .order_by(Score.id_спортсмена)
    )
    
    athletes = session.execute(query).fetchall()
    
    detailed_info = []
    for athlete in athletes:
        # Вычисляем max и min для каждого спортсмена
        scores = [athlete.оценка_1, athlete.оценка_2, athlete.оценка_3, 
                 athlete.оценка_4, athlete.оценка_5]
        max_score = max(scores)
        min_score = min(scores)
        
        detailed_info.append({
            'id': athlete.id_спортсмена,
            'scores': {
                'score_1': athlete.оценка_1,
                'score_2': athlete.оценка_2,
                'score_3': athlete.оценка_3,
                'score_4': athlete.оценка_4,
                'score_5': athlete.оценка_5,
            },
            'max_score': max_score,
            'min_score': min_score,
            'sum_of_three': athlete.сумма_3_средних,
            'average': sum(scores) / len(scores)
        })
    
    return detailed_info


# Универсальная функция, которая работает для любого раунда
def process_round(session: Session, n: int = None, is_final: bool = False):
    """
    Обрабатывает раунд (финальный или нет)
    
    Args:
        session: SQLAlchemy сессия
        n: количество проходящих в следующий раунд (не используется при is_final=True)
        is_final: флаг финального раунда
    """
    if is_final:
        # Финальный раунд - распределяем места
        result = calculate_final_rankings_sql(session)
        print_final_results(result)
        
        # Возвращаем детальную информацию для пьедестала
        podium_ids = []
        for place in sorted(result.medalists.keys()):
            if place <= 3:
                podium_ids.extend(result.medalists[place])
        
        detailed_podium = get_detailed_athlete_info(session, podium_ids)
        
        return {
            'final': True,
            'rankings': result.rankings,
            'medalists': result.medalists,
            'podium_details': detailed_podium
        }
    else:
        # Обычный раунд - отбор в следующий
        from предыдущий_код import select_athletes_for_next_round
        result = select_athletes_for_next_round(session, n)
        
        # Форматируем вывод
        print(f"\n📋 РЕЗУЛЬТАТЫ ОТБОРА В СЛЕДУЮЩИЙ РАУНД (n={n})")
        print(f"Проходят: {len(result.passed_athletes)} спортсменов")
        
        if result.needs_playoff:
            print(f"\n⚔ ТРЕБУЕТСЯ ДОПОЛНИТЕЛЬНЫЙ РАУНД!")
            print(f"Участники доп. раунда ({len(result.playoff_athletes)}):")
            for athlete in result.playoff_athletes:
                print(f"  Спортсмен #{athlete.id_спортсмена}")
        
        return {
            'final': False,
            'passed': [a.id_спортсмена for a in result.passed_athletes],
            'needs_playoff': result.needs_playoff,
            'playoff_athletes': [a.id_спортсмена for a in result.playoff_athletes]
        }


# Пример использования
if __name__ == "__main__":
    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker
    
    engine = create_engine('postgresql://user:pass@localhost/sports_db')
    SessionLocal = sessionmaker(bind=engine)
    
    with SessionLocal() as session:
        print("=" * 80)
        print("СИСТЕМА ПОДСЧЕТА ОЧКОВ И РАНЖИРОВАНИЯ")
        print("=" * 80)
        
        # Пример 1: Обычный раунд (отбор в следующий)
        print("\n1. ОБЫЧНЫЙ РАУНД (отбор 8 спортсменов в следующий раунд):")
        result1 = process_round(session, n=8, is_final=False)
        
        # Пример 2: Финальный раунд
        print("\n\n2. ФИНАЛЬНЫЙ РАУНД (распределение мест):")
        result2 = process_round(session, is_final=True)
        
        # Дополнительный анализ для финального раунда
        print("\n\n3. АНАЛИЗ РЕЗУЛЬТАТОВ ФИНАЛА:")
        
        # Группируем по местам
        for place, athletes in result2['medalists'].items():
            if len(athletes) > 1:
                print(f"\nНа {place}-м месте {len(athletes)} спортсменов:")
                
                # Детальная информация о каждом
                for athlete_id in athletes:
                    athlete_info = next(a for a in result2['rankings'] 
                                       if a.id_спортсмена == athlete_id)
                    print(f"  ID {athlete_id}: сумма={athlete_info.сумма_3_средних:.2f}, "
                          f"макс={athlete_info.максимальная_оценка:.2f}, "
                          f"мин={athlete_info.минимальная_оценка:.2f}")
```

Ключевые особенности реализации:

1. Распределение мест с учетом равенства

· При полном равенстве всех параметров спортсмены делят место
· Следующее место рассчитывается с учетом количества спортсменов, занявших предыдущие места

2. Примеры распределения

```
Спортсмены: A(95), B(95), C(90), D(85), E(85), F(80)

Результаты:
1-е место: A, B (делят)
3-е место: C
4-е место: D, E (делят)
6-е место: F
```

3. Алгоритм работы

1. Сортировка по убыванию: сумма → макс → мин
2. Группировка спортсменов с одинаковыми параметрами
3. Назначение мест:
   · Каждая группа получает место = текущее_место
   · Следующая группа получает место = текущее_место + размер_предыдущей_группы

4. Два подхода реализации

· Python-логика: Группировка и расчет мест в Python (проще для понимания)
· SQL-логика: Использование оконных функций SQL (более эффективно)

5. Вывод результатов

· 🏆 Пьедестал (первые 3 места с эмодзи)
· 📊 Полная таблица результатов
· 📈 Статистика по распределению мест

6. Универсальная функция process_round

· Автоматически определяет логику в зависимости от типа раунда
· Возвращает структурированные данные для дальнейшей обработки

Эта реализация корректно обрабатывает все возможные сценарии, включая множественные равенства на разных позициях в таблице результатов.

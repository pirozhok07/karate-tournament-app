🔧 ИСПРАВЛЕНИЕ: Object of type Athlete is not JSON serializable

Эта ошибка возникает при попытке сериализовать объекты SQLAlchemy в JSON. Нужно преобразовать объекты в словари. Вот исправленные функции:

📝 1. Обновите функцию categorize_athletes в app.py:

```python
def categorize_athletes(athletes, categories):
    """Распределение спортсменов по категориям (возвращает словари вместо объектов)"""
    categorized = {}
    
    for athlete in athletes:
        category_name = "Без категории"
        
        # Простая логика распределения по полу
        athlete_gender = athlete.gender
        athlete_birth_date = athlete.birth_date
        athlete_weight = athlete.weight
        
        # Рассчитываем возраст, если есть дата рождения
        athlete_age = None
        if athlete_birth_date:
            from datetime import datetime
            today = datetime.today()
            athlete_age = today.year - athlete_birth_date.year - (
                (today.month, today.day) < (athlete_birth_date.month, athlete_birth_date.day)
            )
        
        for category in categories:
            # Проверка пола
            if category.gender and category.gender != athlete_gender:
                continue
            
            # Проверка возраста
            if athlete_age:
                if category.min_age and athlete_age < category.min_age:
                    continue
                if category.max_age and athlete_age > category.max_age:
                    continue
            
            # Проверка веса
            if athlete_weight:
                if category.min_weight and athlete_weight < category.min_weight:
                    continue
                if category.max_weight and athlete_weight > category.max_weight:
                    continue
            
            category_name = category.name
            break
        
        if category_name not in categorized:
            categorized[category_name] = []
        
        # Преобразуем объект Athlete в словарь
        athlete_dict = {
            'id': athlete.id,
            'first_name': athlete.first_name,
            'last_name': athlete.last_name,
            'birth_date': athlete.birth_date.strftime('%Y-%m-%d') if athlete.birth_date else None,
            'gender': athlete.gender,
            'weight': athlete.weight,
            'height': athlete.height,
            'club': athlete.club,
            'registration_number': athlete.registration_number,
            'category_id': athlete.category_id
        }
        categorized[category_name].append(athlete_dict)
    
    return categorized
```

📝 2. Обновите функцию generate_draw в app.py:

```python
import random

def generate_draw(categorized):
    """Генерация сетки соревнований (работает со словарями)"""
    draw_data = {}
    
    for category_name, athletes in categorized.items():
        # Перемешиваем спортсменов
        random.shuffle(athletes)
        
        # Создаем пары для первого раунда
        pairs = []
        for i in range(0, len(athletes), 2):
            if i + 1 < len(athletes):
                pairs.append([
                    athletes[i]['id'],
                    athletes[i + 1]['id']
                ])
            else:
                pairs.append([athletes[i]['id'], None])  # Свободный жребий
        
        draw_data[category_name] = {
            'athletes': athletes,  # Уже словари
            'pairs': pairs,
            'order': [athlete['id'] for athlete in athletes]
        }
    
    return draw_data
```

📝 3. Обновите функцию create_competition_with_draw:

```python
def create_competition_with_draw(name, date, location, description=''):
    """Создание соревнования с автоматической генерацией сетки"""
    try:
        # Создание соревнования
        competition = Competition(
            name=name,
            date=date,
            location=location,
            description=description,
            status='pending'
        )
        db.session.add(competition)
        db.session.flush()  # Получаем ID без коммита
        
        # Получаем спортсменов и категории
        athletes = Athlete.query.all()
        categories = Category.query.all()
        
        if athletes and categories:
            # Распределяем спортсменов по категориям
            categorized = categorize_athletes(athletes, categories)
            
            # Генерируем сетку
            draw = generate_draw(categorized)
            
            # Сохраняем сетку
            competition.draw_data = json.dumps(draw, ensure_ascii=False)
        
        db.session.commit()
        return competition
        
    except Exception as e:
        db.session.rollback()
        raise e
```

📝 4. Обновите API endpoints для работы со словарями:

```python
@app.route('/api/athletes')
def api_athletes():
    """API для получения списка спортсменов (возвращает словари)"""
    athletes = Athlete.query.all()
    result = []
    for athlete in athletes:
        result.append({
            'id': athlete.id,
            'first_name': athlete.first_name,
            'last_name': athlete.last_name,
            'birth_date': athlete.birth_date.strftime('%Y-%m-%d') if athlete.birth_date else None,
            'gender': athlete.gender,
            'weight': athlete.weight,
            'height': athlete.height,
            'club': athlete.club,
            'registration_number': athlete.registration_number,
            'category': athlete.category.name if athlete.category else None,
            'created_at': athlete.created_at.strftime('%Y-%m-%d %H:%M:%S') if athlete.created_at else None
        })
    return jsonify(result)

@app.route('/api/categories')
def api_categories():
    """API для получения списка категорий (возвращает словари)"""
    categories = Category.query.all()
    result = []
    for category in categories:
        result.append({
            'id': category.id,
            'name': category.name,
            'min_age': category.min_age,
            'max_age': category.max_age,
            'min_weight': category.min_weight,
            'max_weight': category.max_weight,
            'gender': category.gender,
            'created_at': category.created_at.strftime('%Y-%m-%d %H:%M:%S') if category.created_at else None
        })
    return jsonify(result)
```

📝 5. Обновите функцию calculate_final_results:

```python
def calculate_final_results(competition_id):
    """Расчет финальных результатов (возвращает словари)"""
    athletes = Athlete.query.all()
    results = []
    
    for athlete in athletes:
        scores = Score.query.filter_by(
            competition_id=competition_id,
            athlete_id=athlete.id
        ).order_by(Score.round_number).all()
        
        if scores:
            round1 = scores[0].average if len(scores) > 0 else None
            round2 = scores[1].average if len(scores) > 1 else None
            round3 = scores[2].average if len(scores) > 2 else None
            
            # Сумма лучших двух раундов
            valid_scores = [s for s in [round1, round2, round3] if s is not None]
            if len(valid_scores) >= 2:
                valid_scores.sort(reverse=True)
                total = sum(valid_scores[:2])
                average = total / 2
            else:
                total = sum(valid_scores) if valid_scores else 0
                average = total / len(valid_scores) if valid_scores else 0
            
            results.append({
                'athlete_id': athlete.id,
                'first_name': athlete.first_name,
                'last_name': athlete.last_name,
                'club': athlete.club,
                'category': athlete.category.name if athlete.category else 'Без категории',
                'round1': round1,
                'round2': round2,
                'round3': round3,
                'total': total,
                'average': average
            })
    
    # Сортировка по среднему баллу
    results.sort(key=lambda x: x['average'], reverse=True)
    
    # Присвоение мест
    for i, result in enumerate(results):
        result['place'] = i + 1
    
    return results
```

📝 6. Обновите маршрут view_competition для правильной обработки draw_data:

```python
@app.route('/competition/<int:id>')
def view_competition(id):
    competition = Competition.query.get_or_404(id)
    
    # Преобразуем competition в словарь для шаблона
    competition_dict = {
        'id': competition.id,
        'name': competition.name,
        'date': competition.date.strftime('%Y-%m-%d') if competition.date else None,
        'location': competition.location,
        'description': competition.description,
        'status': competition.status,
        'current_round': competition.current_round,
        'created_at': competition.created_at.strftime('%Y-%m-%d %H:%M:%S') if competition.created_at else None
    }
    
    # Проверяем наличие draw_data
    draw = {}
    if competition.draw_data:
        try:
            draw = json.loads(competition.draw_data)
        except json.JSONDecodeError as e:
            print(f"Ошибка декодирования draw_data: {e}")
            draw = {}
    
    # Получение результатов
    scores = Score.query.filter_by(competition_id=id).all()
    
    # Преобразуем scores в словари
    scores_list = []
    for score in scores:
        scores_list.append({
            'id': score.id,
            'athlete_id': score.athlete_id,
            'competition_id': score.competition_id,
            'round_number': score.round_number,
            'judge1': score.judge1,
            'judge2': score.judge2,
            'judge3': score.judge3,
            'judge4': score.judge4,
            'judge5': score.judge5,
            'total': score.total,
            'average': score.average,
            'created_at': score.created_at.strftime('%Y-%m-%d %H:%M:%S') if score.created_at else None
        })
    
    return render_template('competition.html', 
                         competition=competition_dict, 
                         draw=draw,
                         scores=scores_list)
```

📝 7. Добавьте вспомогательную функцию для сериализации объектов:

```python
def serialize_athlete(athlete):
    """Преобразование объекта Athlete в словарь"""
    return {
        'id': athlete.id,
        'first_name': athlete.first_name,
        'last_name': athlete.last_name,
        'birth_date': athlete.birth_date.strftime('%Y-%m-%d') if athlete.birth_date else None,
        'gender': athlete.gender,
        'weight': athlete.weight,
        'height': athlete.height,
        'club': athlete.club,
        'registration_number': athlete.registration_number,
        'category_id': athlete.category_id,
        'created_at': athlete.created_at.strftime('%Y-%m-%d %H:%M:%S') if athlete.created_at else None
    }

def serialize_category(category):
    """Преобразование объекта Category в словарь"""
    return {
        'id': category.id,
        'name': category.name,
        'min_age': category.min_age,
        'max_age': category.max_age,
        'min_weight': category.min_weight,
        'max_weight': category.max_weight,
        'gender': category.gender,
        'created_at': category.created_at.strftime('%Y-%m-%d %H:%M:%S') if category.created_at else None
    }

def serialize_competition(competition):
    """Преобразование объекта Competition в словарь"""
    return {
        'id': competition.id,
        'name': competition.name,
        'date': competition.date.strftime('%Y-%m-%d') if competition.date else None,
        'location': competition.location,
        'description': competition.description,
        'status': competition.status,
        'current_round': competition.current_round,
        'draw_data': competition.draw_data,
        'created_at': competition.created_at.strftime('%Y-%m-%d %H:%M:%S') if competition.created_at else None
    }
```

📝 8. Обновите функцию generate_draw_route:

```python
@app.route('/competition/<int:id>/generate_draw')
def generate_draw_route(id):
    """Генерация сетки для существующего соревнования"""
    competition = Competition.query.get_or_404(id)
    
    try:
        athletes = Athlete.query.all()
        categories = Category.query.all()
        
        if not athletes:
            flash('Нет спортсменов для генерации сетки', 'error')
            return redirect(url_for('view_competition', id=id))
        
        if not categories:
            flash('Нет категорий для распределения спортсменов', 'error')
            return redirect(url_for('view_competition', id=id))
        
        # Распределяем спортсменов по категориям (возвращает словари)
        categorized = categorize_athletes(athletes, categories)
        
        # Генерируем сетку
        draw = generate_draw(categorized)
        
        # Сохраняем сетку
        competition.draw_data = json.dumps(draw, ensure_ascii=False)
        competition.status = 'active'
        db.session.commit()
        
        flash('Сетка соревнований успешно сгенерирована!', 'success')
        return redirect(url_for('view_competition', id=id))
        
    except Exception as e:
        flash(f'Ошибка при генерации сетки: {str(e)}', 'error')
        return redirect(url_for('view_competition', id=id))
```

🎯 Ключевые изменения:

1. Все функции, возвращающие данные для JSON, теперь возвращают словари, а не объекты SQLAlchemy
2. Добавлены вспомогательные функции сериализации
3. Функция categorize_athletes теперь возвращает словари спортсменов
4. Функция generate_draw работает со словарями
5. API endpoints возвращают JSON-совместимые данные

После этих изменений ошибка "Object of type Athlete is not JSON serializable" должна быть исправлена!

🔧 РЕШЕНИЕ ПРОБЛЕМЫ: КНОПКА "СОЗДАТЬ КАТЕГОРИЮ" НЕ РАБОТАЕТ

Вот исправленный код, который решит проблему:

📝 1. Обновите файл app.py (маршрут /categories):

```python
@app.route('/categories', methods=['GET', 'POST'])
def manage_categories():
    form = CategoryForm()
    
    if form.validate_on_submit():
        try:
            category = Category(
                name=form.name.data,
                min_age=form.min_age.data if form.min_age.data else None,
                max_age=form.max_age.data if form.max_age.data else None,
                min_weight=form.min_weight.data if form.min_weight.data else None,
                max_weight=form.max_weight.data if form.max_weight.data else None,
                gender=form.gender.data if form.gender.data else None
            )
            db.session.add(category)
            db.session.commit()
            flash('Категория успешно создана!', 'success')
            return redirect(url_for('manage_categories'))
        except Exception as e:
            db.session.rollback()
            flash(f'Ошибка при создании категории: {str(e)}', 'danger')
    
    # Получаем данные для отображения
    categories = Category.query.all()
    athletes = Athlete.query.all()
    
    # Функция распределения по категориям
    categorized = {}
    if athletes and categories:
        for athlete in athletes:
            # Логика распределения (упрощенная)
            category_name = "Без категории"
            for category in categories:
                if (not category.gender or athlete.gender == category.gender):
                    category_name = category.name
                    break
            
            if category_name not in categorized:
                categorized[category_name] = []
            categorized[category_name].append(athlete)
    
    return render_template('categories.html', 
                         form=form, 
                         categories=categories, 
                         athletes=athletes,
                         categorized=categorized,
                         now=datetime.now())
```

📝 2. Обновите шаблон categories.html:

```html
{% extends "base.html" %}

{% block content %}
<div class="container mt-4">
    <h2 class="mb-4">
        <i class="bi bi-tags-fill"></i> Управление категориями
    </h2>
    
    <!-- Форма создания категории -->
    <div class="card mb-4">
        <div class="card-header bg-primary text-white">
            <h5 class="mb-0">Создать новую категорию</h5>
        </div>
        <div class="card-body">
            <form method="POST" action="{{ url_for('manage_categories') }}">
                {{ form.hidden_tag() }}
                
                <div class="row mb-3">
                    <div class="col-md-6">
                        <label class="form-label">{{ form.name.label }}</label>
                        {{ form.name(class="form-control", placeholder="Например: Юноши 14-16 лет") }}
                        {% if form.name.errors %}
                            <div class="text-danger">
                                {% for error in form.name.errors %}
                                    <small>{{ error }}</small>
                                {% endfor %}
                            </div>
                        {% endif %}
                    </div>
                    
                    <div class="col-md-6">
                        <label class="form-label">{{ form.gender.label }}</label>
                        {{ form.gender(class="form-select") }}
                        {% if form.gender.errors %}
                            <div class="text-danger">
                                {% for error in form.gender.errors %}
                                    <small>{{ error }}</small>
                                {% endfor %}
                            </div>
                        {% endif %}
                    </div>
                </div>
                
                <div class="row mb-3">
                    <div class="col-md-3">
                        <label class="form-label">{{ form.min_age.label }}</label>
                        {{ form.min_age(class="form-control", placeholder="14") }}
                    </div>
                    
                    <div class="col-md-3">
                        <label class="form-label">{{ form.max_age.label }}</label>
                        {{ form.max_age(class="form-control", placeholder="16") }}
                    </div>
                    
                    <div class="col-md-3">
                        <label class="form-label">{{ form.min_weight.label }}</label>
                        {{ form.min_weight(class="form-control", placeholder="50.0") }}
                    </div>
                    
                    <div class="col-md-3">
                        <label class="form-label">{{ form.max_weight.label }}</label>
                        {{ form.max_weight(class="form-control", placeholder="60.0") }}
                    </div>
                </div>
                
                <div class="d-grid">
                    {{ form.submit(class="btn btn-primary btn-lg") }}
                </div>
            </form>
        </div>
    </div>
    
    <!-- Существующие категории -->
    <div class="card mb-4">
        <div class="card-header bg-success text-white">
            <h5 class="mb-0">
                Существующие категории
                <span class="badge bg-light text-dark float-end">{{ categories|length }}</span>
            </h5>
        </div>
        <div class="card-body">
            {% if categories %}
                <div class="table-responsive">
                    <table class="table table-hover">
                        <thead>
                            <tr>
                                <th>Название</th>
                                <th>Пол</th>
                                <th>Возраст</th>
                                <th>Вес (кг)</th>
                                <th>Дата создания</th>
                            </tr>
                        </thead>
                        <tbody>
                            {% for category in categories %}
                            <tr>
                                <td><strong>{{ category.name }}</strong></td>
                                <td>
                                    {% if category.gender == 'М' %}
                                        <span class="badge bg-primary">Мужской</span>
                                    {% elif category.gender == 'Ж' %}
                                        <span class="badge bg-danger">Женский</span>
                                    {% else %}
                                        <span class="badge bg-secondary">Любой</span>
                                    {% endif %}
                                </td>
                                <td>
                                    {% if category.min_age or category.max_age %}
                                        {{ category.min_age or '?' }} - {{ category.max_age or '?' }} лет
                                    {% else %}
                                        <span class="text-muted">Не ограничено</span>
                                    {% endif %}
                                </td>
                                <td>
                                    {% if category.min_weight or category.max_weight %}
                                        {{ category.min_weight or '?' }} - {{ category.max_weight or '?' }} кг
                                    {% else %}
                                        <span class="text-muted">Не ограничено</span>
                                    {% endif %}
                                </td>
                                <td>{{ category.created_at.strftime('%d.%m.%Y') if category.created_at else '—' }}</td>
                            </tr>
                            {% endfor %}
                        </tbody>
                    </table>
                </div>
            {% else %}
                <div class="text-center py-4">
                    <i class="bi bi-tag display-4 text-muted"></i>
                    <p class="mt-3">Категории еще не созданы</p>
                </div>
            {% endif %}
        </div>
    </div>
    
    <!-- Распределение спортсменов -->
    <div class="card">
        <div class="card-header bg-info text-white">
            <h5 class="mb-0">Распределение спортсменов по категориям</h5>
        </div>
        <div class="card-body">
            {% if athletes %}
                <div class="row">
                    {% for category_name, athletes_list in categorized.items() %}
                    <div class="col-md-6 mb-3">
                        <div class="card">
                            <div class="card-header">
                                <h6 class="mb-0">
                                    {{ category_name }}
                                    <span class="badge bg-primary float-end">{{ athletes_list|length }}</span>
                                </h6>
                            </div>
                            <div class="card-body" style="max-height: 200px; overflow-y: auto;">
                                <ul class="list-group list-group-flush">
                                    {% for athlete in athletes_list %}
                                    <li class="list-group-item d-flex justify-content-between align-items-center">
                                        {{ athlete.last_name }} {{ athlete.first_name }}
                                        <small class="text-muted">{{ athlete.club or '—' }}</small>
                                    </li>
                                    {% endfor %}
                                </ul>
                            </div>
                        </div>
                    </div>
                    {% endfor %}
                </div>
            {% else %}
                <div class="text-center py-4">
                    <i class="bi bi-person-slash display-4 text-muted"></i>
                    <p class="mt-3">Спортсменов нет</p>
                    <a href="{{ url_for('upload_athletes') }}" class="btn btn-primary">
                        <i class="bi bi-upload"></i> Загрузить спортсменов
                    </a>
                </div>
            {% endif %}
        </div>
    </div>
</div>
{% endblock %}
```

📝 3. Убедитесь, что форма CategoryForm правильно определена:

В файле app.py убедитесь, что форма CategoryForm имеет правильные поля:

```python
from wtforms.validators import DataRequired, Optional

class CategoryForm(FlaskForm):
    """Форма создания категории"""
    name = StringField('Название категории', validators=[DataRequired()])
    min_age = IntegerField('Минимальный возраст', validators=[Optional()])
    max_age = IntegerField('Максимальный возраст', validators=[Optional()])
    min_weight = FloatField('Минимальный вес (кг)', validators=[Optional()])
    max_weight = FloatField('Максимальный вес (кг)', validators=[Optional()])
    gender = SelectField('Пол', choices=[
        ('', 'Любой'), ('М', 'Мужской'), ('Ж', 'Женский')
    ], validators=[Optional()])
    submit = SubmitField('Создать категорию')
```

📝 4. Проверьте базовый шаблон base.html:

Убедитесь, что в templates/base.html есть блок для отображения flash-сообщений:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Система соревнований{% endblock %}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.8.1/font/bootstrap-icons.css">
</head>
<body>
    <div class="container mt-4">
        <!-- Сообщения -->
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category if category != 'message' else 'info' }} alert-dismissible fade show">
                        {{ message }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        <!-- Основной контент -->
        {% block content %}{% endblock %}
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

🔍 Диагностика проблемы:

Если кнопка все еще не работает, добавьте отладочную информацию:

1. Добавьте отладку в app.py:

```python
@app.route('/categories', methods=['GET', 'POST'])
def manage_categories():
    form = CategoryForm()
    print(f"Метод запроса: {request.method}")  # Отладочный вывод
    print(f"Форма отправлена: {form.is_submitted()}")  # Отладочный вывод
    print(f"Форма валидна: {form.validate_on_submit()}")  # Отладочный вывод
    
    if form.validate_on_submit():
        print("Форма прошла валидацию!")  # Отладочный вывод
        print(f"Данные формы: name={form.name.data}, gender={form.gender.data}")
        # ... остальной код ...
```

2. Проверьте консоль сервера на наличие ошибок.

3. Убедитесь, что форма имеет атрибут action:

В шаблоне форма должна выглядеть так:

```html
<form method="POST" action="{{ url_for('manage_categories') }}">
    <!-- поля формы -->
</form>
```

🚀 Быстрое решение:

Создайте новый файл categories_fixed.html с этим минимальным рабочим кодом:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Категории - Система соревнований</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <div class="container mt-4">
        <h2>Создание категории</h2>
        
        {% with messages = get_flashed_messages() %}
            {% if messages %}
                {% for message in messages %}
                    <div class="alert alert-info">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        <form method="POST" action="/categories">
            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
            
            <div class="mb-3">
                <label class="form-label">Название категории *</label>
                <input type="text" class="form-control" name="name" required placeholder="Юноши 14-16 лет">
            </div>
            
            <div class="mb-3">
                <label class="form-label">Пол</label>
                <select class="form-select" name="gender">
                    <option value="">Любой</option>
                    <option value="М">Мужской</option>
                    <option value="Ж">Женский</option>
                </select>
            </div>
            
            <div class="row mb-3">
                <div class="col">
                    <label class="form-label">Минимальный возраст</label>
                    <input type="number" class="form-control" name="min_age" placeholder="14">
                </div>
                <div class="col">
                    <label class="form-label">Максимальный возраст</label>
                    <input type="number" class="form-control" name="max_age" placeholder="16">
                </div>
            </div>
            
            <div class="row mb-3">
                <div class="col">
                    <label class="form-label">Минимальный вес (кг)</label>
                    <input type="number" step="0.1" class="form-control" name="min_weight" placeholder="50.0">
                </div>
                <div class="col">
                    <label class="form-label">Максимальный вес (кг)</label>
                    <input type="number" step="0.1" class="form-control" name="max_weight" placeholder="60.0">
                </div>
            </div>
            
            <button type="submit" class="btn btn-primary">Создать категорию</button>
            <a href="/" class="btn btn-secondary">На главную</a>
        </form>
    </div>
</body>
</html>
```

📋 Проверочный список:

1. ✅ Проверьте, что Flask-WTF установлен: pip install flask-wtf
2. ✅ Убедитесь, что в форме есть CSRF-токен: {{ form.hidden_tag() }}
3. ✅ Проверьте метод формы: method="POST"
4. ✅ Проверьте action формы: action="{{ url_for('manage_categories') }}"
5. ✅ Проверьте консоль сервера на наличие ошибок
6. ✅ Убедитесь, что база данных инициализирована

Попробуйте эти исправления, и кнопка "Создать категорию" должна заработать!

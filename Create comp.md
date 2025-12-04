📝 КОД ФАЙЛА templates/create_competition.html

```html
{% extends "base.html" %}

{% block content %}
<div class="row">
    <div class="col-md-8 offset-md-2">
        <h2 class="mb-4">
            <i class="bi bi-calendar-plus"></i> Создание нового соревнования
        </h2>
        
        <!-- Форма создания соревнования -->
        <div class="card mb-4">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0">Основная информация о соревновании</h5>
            </div>
            <div class="card-body">
                <form method="POST" action="{{ url_for('create_competition') }}">
                    {{ form.hidden_tag() }}
                    
                    <div class="mb-3">
                        <label class="form-label">{{ form.name.label }} *</label>
                        {{ form.name(class="form-control", placeholder="Чемпионат города по гимнастике") }}
                        {% if form.name.errors %}
                            <div class="text-danger">
                                {% for error in form.name.errors %}
                                    <small>{{ error }}</small>
                                {% endfor %}
                            </div>
                        {% endif %}
                    </div>
                    
                    <div class="row mb-3">
                        <div class="col-md-6">
                            <label class="form-label">{{ form.date.label }} *</label>
                            {{ form.date(class="form-control", type="date") }}
                            {% if form.date.errors %}
                                <div class="text-danger">
                                    {% for error in form.date.errors %}
                                        <small>{{ error }}</small>
                                    {% endfor %}
                                </div>
                            {% endif %}
                        </div>
                        
                        <div class="col-md-6">
                            <label class="form-label">{{ form.location.label }} *</label>
                            {{ form.location(class="form-control", placeholder="Спорткомплекс 'Олимп'") }}
                            {% if form.location.errors %}
                                <div class="text-danger">
                                    {% for error in form.location.errors %}
                                        <small>{{ error }}</small>
                                    {% endfor %}
                                </div>
                            {% endif %}
                        </div>
                    </div>
                    
                    <div class="mb-3">
                        <label class="form-label">{{ form.description.label }}</label>
                        {{ form.description(class="form-control", rows="4", 
                                           placeholder="Опишите соревнование: вид спорта, правила, особенности...") }}
                    </div>
                    
                    <div class="d-grid gap-2 d-md-flex justify-content-md-end mt-4">
                        <a href="{{ url_for('index') }}" class="btn btn-secondary me-md-2">
                            <i class="bi bi-arrow-left"></i> Отмена
                        </a>
                        <button type="submit" class="btn btn-primary">
                            <i class="bi bi-check-circle"></i> Создать соревнование
                        </button>
                    </div>
                </form>
            </div>
        </div>
        
        <!-- Информация о процессе проведения -->
        <div class="card">
            <div class="card-header bg-info text-white">
                <h5 class="mb-0"><i class="bi bi-info-circle"></i> Процесс проведения соревнования</h5>
            </div>
            <div class="card-body">
                <div class="row">
                    <div class="col-md-3 text-center mb-3">
                        <div class="bg-primary text-white rounded-circle p-3 d-inline-block">
                            <i class="bi bi-1-circle-fill fs-3"></i>
                        </div>
                        <h6 class="mt-2">Создание</h6>
                        <p class="small text-muted">Заполните информацию о соревновании</p>
                    </div>
                    
                    <div class="col-md-3 text-center mb-3">
                        <div class="bg-secondary text-white rounded-circle p-3 d-inline-block">
                            <i class="bi bi-2-circle-fill fs-3"></i>
                        </div>
                        <h6 class="mt-2">Распределение</h6>
                        <p class="small text-muted">Распределите спортсменов по категориям</p>
                    </div>
                    
                    <div class="col-md-3 text-center mb-3">
                        <div class="bg-warning text-white rounded-circle p-3 d-inline-block">
                            <i class="bi bi-3-circle-fill fs-3"></i>
                        </div>
                        <h6 class="mt-2">Проведение</h6>
                        <p class="small text-muted">Вводите оценки по раундам</p>
                    </div>
                    
                    <div class="col-md-3 text-center mb-3">
                        <div class="bg-success text-white rounded-circle p-3 d-inline-block">
                            <i class="bi bi-4-circle-fill fs-3"></i>
                        </div>
                        <h6 class="mt-2">Завершение</h6>
                        <p class="small text-muted">Получите результаты и отчеты</p>
                    </div>
                </div>
                
                <div class="alert alert-warning mt-3">
                    <h6><i class="bi bi-exclamation-triangle"></i> Важно!</h6>
                    <ul class="mb-0">
                        <li>После создания соревнования необходимо распределить спортсменов по категориям</li>
                        <li>Для каждого соревнования создается отдельная жеребьевка</li>
                        <li>Соревнование можно проводить в несколько раундов (рекомендуется 3 раунда)</li>
                        <li>По окончании можно сформировать отчеты в Excel и PDF форматах</li>
                    </ul>
                </div>
            </div>
        </div>
        
        <!-- Примеры названий соревнований -->
        <div class="card mt-4">
            <div class="card-header bg-success text-white">
                <h5 class="mb-0"><i class="bi bi-lightbulb"></i> Примеры названий соревнований</h5>
            </div>
            <div class="card-body">
                <div class="row">
                    <div class="col-md-6">
                        <ul class="list-group">
                            <li class="list-group-item">Чемпионат города по художественной гимнастике</li>
                            <li class="list-group-item">Первенство области по спортивной акробатике</li>
                            <li class="list-group-item">Кубок школы по вольным упражнениям</li>
                            <li class="list-group-item">Межрегиональные соревнования по прыжкам на батуте</li>
                        </ul>
                    </div>
                    <div class="col-md-6">
                        <ul class="list-group">
                            <li class="list-group-item">Турнир памяти заслуженного тренера</li>
                            <li class="list-group-item">Открытый чемпионат клуба "Спартак"</li>
                            <li class="list-group-item">Соревнования "Новые звезды гимнастики"</li>
                            <li class="list-group-item">Фестиваль спортивной гимнастики</li>
                        </ul>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
// Установка минимальной даты на сегодня
document.addEventListener('DOMContentLoaded', function() {
    const dateField = document.querySelector('input[type="date"]');
    if (dateField) {
        const today = new Date().toISOString().split('T')[0];
        dateField.min = today;
        
        // Если поле пустое, установить сегодняшнюю дату
        if (!dateField.value) {
            dateField.value = today;
        }
    }
    
    // Автоматический расчет даты окончания
    const startDateField = document.querySelector('input[name="start_date"]');
    const endDateField = document.querySelector('input[name="end_date"]');
    
    if (startDateField && endDateField) {
        startDateField.addEventListener('change', function() {
            const startDate = new Date(this.value);
            if (!isNaN(startDate.getTime())) {
                const endDate = new Date(startDate);
                endDate.setDate(endDate.getDate() + 2); // +2 дня
                endDateField.value = endDate.toISOString().split('T')[0];
                endDateField.min = this.value;
            }
        });
    }
});

// Подсказка для поля названия
function showNameSuggestion() {
    const examples = [
        "Чемпионат города по гимнастике",
        "Первенство области по акробатике", 
        "Кубок школы по спортивной гимнастике",
        "Турнир на призы клуба 'Спартак'"
    ];
    const randomExample = examples[Math.floor(Math.random() * examples.length)];
    alert('Пример названия: ' + randomExample);
}
</script>
{% endblock %}
```

📋 ТРЕБУЕМЫЕ ИКОНКИ BOOTSTRAP

Убедитесь, что в вашем base.html подключены Bootstrap и иконки:

```html
<!-- В шапке base.html -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.8.1/font/bootstrap-icons.css">
```

🎯 АЛЬТЕРНАТИВНЫЙ ВАРИАНТ (без иконок)

Если иконки не загружаются, вот упрощенная версия без иконок:

```html
{% extends "base.html" %}

{% block content %}
<div class="row">
    <div class="col-md-8 offset-md-2">
        <h2 class="mb-4">Создание нового соревнования</h2>
        
        <!-- Форма создания соревнования -->
        <div class="card mb-4">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0">Основная информация о соревновании</h5>
            </div>
            <div class="card-body">
                <form method="POST" action="{{ url_for('create_competition') }}">
                    {{ form.hidden_tag() }}
                    
                    <div class="mb-3">
                        <label class="form-label">Название соревнования *</label>
                        <input type="text" class="form-control" name="name" required 
                               placeholder="Чемпионат города по гимнастике">
                    </div>
                    
                    <div class="row mb-3">
                        <div class="col-md-6">
                            <label class="form-label">Дата проведения *</label>
                            <input type="date" class="form-control" name="date" required>
                        </div>
                        
                        <div class="col-md-6">
                            <label class="form-label">Место проведения *</label>
                            <input type="text" class="form-control" name="location" required 
                                   placeholder="Спорткомплекс 'Олимп'">
                        </div>
                    </div>
                    
                    <div class="mb-3">
                        <label class="form-label">Описание (необязательно)</label>
                        <textarea class="form-control" name="description" rows="4" 
                                  placeholder="Опишите соревнование: вид спорта, правила, особенности..."></textarea>
                    </div>
                    
                    <div class="d-grid gap-2 d-md-flex justify-content-md-end mt-4">
                        <a href="{{ url_for('index') }}" class="btn btn-secondary me-md-2">
                            ← Отмена
                        </a>
                        <button type="submit" class="btn btn-primary">
                            Создать соревнование
                        </button>
                    </div>
                </form>
            </div>
        </div>
        
        <!-- Информация -->
        <div class="card">
            <div class="card-header bg-info text-white">
                <h5 class="mb-0">Процесс проведения соревнования</h5>
            </div>
            <div class="card-body">
                <div class="alert alert-warning">
                    <h6>Важно!</h6>
                    <ul class="mb-0">
                        <li>После создания соревнования необходимо распределить спортсменов по категориям</li>
                        <li>Для каждого соревнования создается отдельная жеребьевка</li>
                        <li>Соревнование можно проводить в несколько раундов (рекомендуется 3 раунда)</li>
                        <li>По окончании можно сформировать отчеты в Excel и PDF форматах</li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
// Установка минимальной даты на сегодня
document.addEventListener('DOMContentLoaded', function() {
    const dateField = document.querySelector('input[type="date"]');
    if (dateField) {
        const today = new Date().toISOString().split('T')[0];
        dateField.min = today;
        
        // Если поле пустое, установить сегодняшнюю дату
        if (!dateField.value) {
            dateField.value = today;
        }
    }
});
</script>
{% endblock %}
```

🔧 СОВМЕСТИМЫЙ КОД ДЛЯ app.py

Убедитесь, что в app.py есть соответствующий маршрут:

```python
@app.route('/create_competition', methods=['GET', 'POST'])
def create_competition():
    form = CompetitionForm()
    
    if form.validate_on_submit():
        try:
            competition = Competition(
                name=form.name.data,
                date=form.date.data,
                location=form.location.data,
                description=form.description.data,
                status='pending'
            )
            db.session.add(competition)
            db.session.commit()
            
            flash(f'Соревнование "{form.name.data}" успешно создано!', 'success')
            return redirect(url_for('index'))
            
        except Exception as e:
            db.session.rollback()
            flash(f'Ошибка при создании соревнования: {str(e)}', 'danger')
    
    return render_template('create_competition.html', form=form)
```

Этот шаблон создаст полностью рабочую форму для создания соревнования!

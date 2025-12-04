Веб-приложение для проведения соревнований

Структура проекта

```
competition_app/
├── app.py
├── database.py
├── models.py
├── requirements.txt
├── static/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── main.js
└── templates/
    ├── base.html
    ├── index.html
    ├── create_competition.html
    ├── categories.html
    ├── register_participant.html
    ├── draw.html
    ├── schedule.html
    ├── scoring.html
    └── results.html
```

Шаг 1: Установка зависимостей

requirements.txt:

```txt
Flask==2.3.3
Flask-SQLAlchemy==3.0.5
Flask-WTF==1.1.1
WTForms==3.0.1
python-dotenv==1.0.0
```

Установите зависимости:

```bash
pip install -r requirements.txt
```

Шаг 2: Настройка базы данных

database.py:

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
```

Шаг 3: Создание моделей данных

models.py:

```python
from datetime import datetime
from database import db

class Competition(db.Model):
    __tablename__ = 'competitions'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(200), nullable=False)
    description = db.Column(db.Text)
    start_date = db.Column(db.DateTime)
    end_date = db.Column(db.DateTime)
    status = db.Column(db.String(20), default='planned')  # planned, active, finished
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    categories = db.relationship('Category', backref='competition', lazy=True, cascade='all, delete-orphan')

class Category(db.Model):
    __tablename__ = 'categories'
    
    id = db.Column(db.Integer, primary_key=True)
    competition_id = db.Column(db.Integer, db.ForeignKey('competitions.id'), nullable=False)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text)
    min_age = db.Column(db.Integer)
    max_age = db.Column(db.Integer)
    gender = db.Column(db.String(20))  # male, female, mixed
    
    participants = db.relationship('Participant', backref='category', lazy=True, cascade='all, delete-orphan')

class Participant(db.Model):
    __tablename__ = 'participants'
    
    id = db.Column(db.Integer, primary_key=True)
    category_id = db.Column(db.Integer, db.ForeignKey('categories.id'), nullable=False)
    first_name = db.Column(db.String(50), nullable=False)
    last_name = db.Column(db.String(50), nullable=False)
    age = db.Column(db.Integer)
    gender = db.Column(db.String(10))
    country = db.Column(db.String(50))
    registration_date = db.Column(db.DateTime, default=datetime.utcnow)
    order_number = db.Column(db.Integer)  # Порядковый номер выступления
    is_active = db.Column(db.Boolean, default=True)
    
    scores = db.relationship('Score', backref='participant', lazy=True, cascade='all, delete-orphan')

class Score(db.Model):
    __tablename__ = 'scores'
    
    id = db.Column(db.Integer, primary_key=True)
    participant_id = db.Column(db.Integer, db.ForeignKey('participants.id'), nullable=False)
    judge_id = db.Column(db.Integer, db.ForeignKey('judges.id'), nullable=False)
    criteria_1 = db.Column(db.Float, default=0)
    criteria_2 = db.Column(db.Float, default=0)
    criteria_3 = db.Column(db.Float, default=0)
    criteria_4 = db.Column(db.Float, default=0)
    criteria_5 = db.Column(db.Float, default=0)
    total_score = db.Column(db.Float, default=0)
    notes = db.Column(db.Text)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)

class Judge(db.Model):
    __tablename__ = 'judges'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    specialization = db.Column(db.String(100))
    login_code = db.Column(db.String(20), unique=True)
    
    scores = db.relationship('Score', backref='judge', lazy=True)
```

Шаг 4: Создание основного приложения

app.py:

```python
from flask import Flask, render_template, request, redirect, url_for, flash, jsonify
from flask_wtf import FlaskForm
from wtforms import StringField, TextAreaField, IntegerField, SelectField, DateField, FloatField
from wtforms.validators import DataRequired, Optional
from datetime import datetime
import random

from database import db
from models import Competition, Category, Participant, Score, Judge

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key-here'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///competitions.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db.init_app(app)

# Формы
class CompetitionForm(FlaskForm):
    name = StringField('Название соревнования', validators=[DataRequired()])
    description = TextAreaField('Описание')
    start_date = DateField('Дата начала', format='%Y-%m-%d')
    end_date = DateField('Дата окончания', format='%Y-%m-%d')

class CategoryForm(FlaskForm):
    name = StringField('Название категории', validators=[DataRequired()])
    description = TextAreaField('Описание')
    min_age = IntegerField('Минимальный возраст', validators=[Optional()])
    max_age = IntegerField('Максимальный возраст', validators=[Optional()])
    gender = SelectField('Пол', choices=[('mixed', 'Смешанный'), ('male', 'Мужской'), ('female', 'Женский')])

class ParticipantForm(FlaskForm):
    first_name = StringField('Имя', validators=[DataRequired()])
    last_name = StringField('Фамилия', validators=[DataRequired()])
    age = IntegerField('Возраст', validators=[Optional()])
    gender = SelectField('Пол', choices=[('male', 'Мужской'), ('female', 'Женский')])
    country = StringField('Страна')

class ScoreForm(FlaskForm):
    criteria_1 = FloatField('Критерий 1 (0-10)', validators=[Optional()])
    criteria_2 = FloatField('Критерий 2 (0-10)', validators=[Optional()])
    criteria_3 = FloatField('Критерий 3 (0-10)', validators=[Optional()])
    criteria_4 = FloatField('Критерий 4 (0-10)', validators=[Optional()])
    criteria_5 = FloatField('Критерий 5 (0-10)', validators=[Optional()])
    notes = TextAreaField('Комментарии')

# Создание базы данных
with app.app_context():
    db.create_all()
    
    # Создаем тестовых судей, если их нет
    if Judge.query.count() == 0:
        judges = [
            Judge(name='Иванов И.И.', specialization='Техника', login_code='J001'),
            Judge(name='Петрова А.С.', specialization='Артистичность', login_code='J002'),
            Judge(name='Сидоров В.В.', specialization='Общее впечатление', login_code='J003')
        ]
        db.session.add_all(judges)
        db.session.commit()

# Главная страница
@app.route('/')
def index():
    competitions = Competition.query.order_by(Competition.created_at.desc()).all()
    return render_template('index.html', competitions=competitions)

# Создание соревнования
@app.route('/competition/create', methods=['GET', 'POST'])
def create_competition():
    form = CompetitionForm()
    
    if form.validate_on_submit():
        competition = Competition(
            name=form.name.data,
            description=form.description.data,
            start_date=form.start_date.data,
            end_date=form.end_date.data
        )
        db.session.add(competition)
        db.session.commit()
        flash('Соревнование успешно создано!', 'success')
        return redirect(url_for('index'))
    
    return render_template('create_competition.html', form=form)

# Управление категориями соревнования
@app.route('/competition/<int:competition_id>/categories', methods=['GET', 'POST'])
def manage_categories(competition_id):
    competition = Competition.query.get_or_404(competition_id)
    form = CategoryForm()
    
    if form.validate_on_submit():
        category = Category(
            competition_id=competition_id,
            name=form.name.data,
            description=form.description.data,
            min_age=form.min_age.data,
            max_age=form.max_age.data,
            gender=form.gender.data
        )
        db.session.add(category)
        db.session.commit()
        flash('Категория успешно добавлена!', 'success')
        return redirect(url_for('manage_categories', competition_id=competition_id))
    
    return render_template('categories.html', competition=competition, form=form)

# Регистрация участников
@app.route('/category/<int:category_id>/register', methods=['GET', 'POST'])
def register_participant(category_id):
    category = Category.query.get_or_404(category_id)
    form = ParticipantForm()
    
    if form.validate_on_submit():
        participant = Participant(
            category_id=category_id,
            first_name=form.first_name.data,
            last_name=form.last_name.data,
            age=form.age.data,
            gender=form.gender.data,
            country=form.country.data
        )
        db.session.add(participant)
        db.session.commit()
        flash('Участник успешно зарегистрирован!', 'success')
        return redirect(url_for('register_participant', category_id=category_id))
    
    participants = Participant.query.filter_by(category_id=category_id).order_by(Participant.last_name).all()
    return render_template('register_participant.html', category=category, form=form, participants=participants)

# Жеребьевка участников
@app.route('/category/<int:category_id>/draw', methods=['GET', 'POST'])
def draw_participants(category_id):
    category = Category.query.get_or_404(category_id)
    participants = Participant.query.filter_by(category_id=category_id, is_active=True).all()
    
    if request.method == 'POST':
        # Случайное перемешивание участников
        random.shuffle(participants)
        
        # Назначение порядковых номеров
        for i, participant in enumerate(participants, 1):
            participant.order_number = i
        
        db.session.commit()
        flash('Жеребьевка проведена успешно!', 'success')
        return redirect(url_for('draw_participants', category_id=category_id))
    
    # Сортировка по порядковому номеру
    participants_sorted = sorted(participants, key=lambda x: x.order_number or 999)
    
    return render_template('draw.html', category=category, participants=participants_sorted)

# Корректировка порядка выступлений
@app.route('/category/<int:category_id>/schedule', methods=['GET', 'POST'])
def manage_schedule(category_id):
    category = Category.query.get_or_404(category_id)
    participants = Participant.query.filter_by(category_id=category_id, is_active=True).all()
    
    if request.method == 'POST':
        # Обновление порядковых номеров
        for participant in participants:
            new_order = request.form.get(f'order_{participant.id}')
            if new_order and new_order.isdigit():
                participant.order_number = int(new_order)
        
        db.session.commit()
        flash('Порядок выступлений обновлен!', 'success')
        return redirect(url_for('manage_schedule', category_id=category_id))
    
    # Сортировка по текущему порядку
    participants_sorted = sorted(participants, key=lambda x: x.order_number or 999)
    
    return render_template('schedule.html', category=category, participants=participants_sorted)

# Выставление оценок
@app.route('/category/<int:category_id>/scoring', methods=['GET', 'POST'])
def scoring(category_id):
    category = Category.query.get_or_404(category_id)
    participants = Participant.query.filter_by(category_id=category_id, is_active=True).all()
    judges = Judge.query.all()
    
    # Получаем или создаем форму для судьи
    judge_code = request.args.get('judge', '')
    judge = Judge.query.filter_by(login_code=judge_code).first() if judge_code else None
    
    if not judge and request.method == 'GET':
        return render_template('judge_login.html', category=category)
    
    form = ScoreForm()
    
    if request.method == 'POST' and judge:
        participant_id = request.form.get('participant_id')
        participant = Participant.query.get(participant_id)
        
        if participant:
            # Проверяем, есть ли уже оценка от этого судьи
            existing_score = Score.query.filter_by(
                participant_id=participant_id,
                judge_id=judge.id
            ).first()
            
            if existing_score:
                # Обновляем существующую оценку
                existing_score.criteria_1 = form.criteria_1.data or 0
                existing_score.criteria_2 = form.criteria_2.data or 0
                existing_score.criteria_3 = form.criteria_3.data or 0
                existing_score.criteria_4 = form.criteria_4.data or 0
                existing_score.criteria_5 = form.criteria_5.data or 0
                existing_score.notes = form.notes.data
                
                # Расчет общей оценки
                existing_score.total_score = (
                    existing_score.criteria_1 +
                    existing_score.criteria_2 +
                    existing_score.criteria_3 +
                    existing_score.criteria_4 +
                    existing_score.criteria_5
                )
            else:
                # Создаем новую оценку
                total_score = (
                    (form.criteria_1.data or 0) +
                    (form.criteria_2.data or 0) +
                    (form.criteria_3.data or 0) +
                    (form.criteria_4.data or 0) +
                    (form.criteria_5.data or 0)
                )
                
                score = Score(
                    participant_id=participant_id,
                    judge_id=judge.id,
                    criteria_1=form.criteria_1.data or 0,
                    criteria_2=form.criteria_2.data or 0,
                    criteria_3=form.criteria_3.data or 0,
                    criteria_4=form.criteria_4.data or 0,
                    criteria_5=form.criteria_5.data or 0,
                    total_score=total_score,
                    notes=form.notes.data
                )
                db.session.add(score)
            
            db.session.commit()
            flash('Оценка сохранена!', 'success')
    
    # Сортировка участников по порядку выступления
    participants_sorted = sorted(participants, key=lambda x: x.order_number or 999)
    
    # Получаем оценки для отображения
    scores = {}
    for participant in participants_sorted:
        participant_scores = Score.query.filter_by(participant_id=participant.id).all()
        scores[participant.id] = participant_scores
    
    return render_template('scoring.html', 
                         category=category, 
                         participants=participants_sorted, 
                         judges=judges,
                         scores=scores,
                         judge=judge,
                         form=form)

# Результаты соревнований
@app.route('/category/<int:category_id>/results')
def competition_results(category_id):
    category = Category.query.get_or_404(category_id)
    participants = Participant.query.filter_by(category_id=category_id, is_active=True).all()
    judges = Judge.query.all()
    
    # Рассчитываем результаты для каждого участника
    results = []
    for participant in participants:
        scores = Score.query.filter_by(participant_id=participant.id).all()
        
        if scores:
            # Средняя оценка
            avg_score = sum(score.total_score for score in scores) / len(scores)
            
            # Максимальная и минимальная оценки (исключая крайние значения)
            score_values = [score.total_score for score in scores]
            if len(score_values) > 2:
                score_values.sort()
                score_values = score_values[1:-1]  # Убираем одну минимальную и одну максимальную
            
            final_score = sum(score_values) / len(score_values) if score_values else 0
            
            results.append({
                'participant': participant,
                'scores': scores,
                'average_score': avg_score,
                'final_score': final_score,
                'judge_count': len(scores)
            })
    
    # Сортировка по итоговой оценке (по убыванию)
    results.sort(key=lambda x: x['final_score'], reverse=True)
    
    # Присваиваем места
    for i, result in enumerate(results):
        result['place'] = i + 1
    
    return render_template('results.html', category=category, results=results, judges=judges)

# API для получения данных в реальном времени
@app.route('/api/category/<int:category_id>/scores')
def get_scores_api(category_id):
    participants = Participant.query.filter_by(category_id=category_id, is_active=True).all()
    
    data = []
    for participant in participants:
        scores = Score.query.filter_by(participant_id=participant.id).all()
        
        if scores:
            avg_score = sum(score.total_score for score in scores) / len(scores)
            data.append({
                'id': participant.id,
                'name': f"{participant.first_name} {participant.last_name}",
                'order': participant.order_number,
                'score_count': len(scores),
                'average_score': round(avg_score, 2)
            })
    
    return jsonify(data)

if __name__ == '__main__':
    app.run(debug=True)
```

Шаг 5: Создание HTML шаблонов

templates/base.html:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Система проведения соревнований{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
    <nav class="navbar">
        <div class="container">
            <a href="{{ url_for('index') }}" class="logo">Соревнования</a>
            <div class="nav-links">
                <a href="{{ url_for('index') }}">Главная</a>
                <a href="{{ url_for('create_competition') }}">Создать соревнование</a>
            </div>
        </div>
    </nav>

    <div class="container">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }}">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}

        {% block content %}{% endblock %}
    </div>

    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
</body>
</html>
```

templates/index.html:

```html
{% extends "base.html" %}

{% block title %}Главная - Система проведения соревнований{% endblock %}

{% block content %}
<div class="header">
    <h1>Система проведения соревнований</h1>
    <a href="{{ url_for('create_competition') }}" class="btn btn-primary">Создать новое соревнование</a>
</div>

<div class="competitions-grid">
    {% for competition in competitions %}
    <div class="competition-card">
        <h3>{{ competition.name }}</h3>
        <p>{{ competition.description }}</p>
        <div class="competition-meta">
            <span>Дата: {{ competition.start_date.strftime('%d.%m.%Y') if competition.start_date else 'Не указана' }}</span>
            <span class="status status-{{ competition.status }}">{{ competition.status }}</span>
        </div>
        <div class="competition-actions">
            <a href="{{ url_for('manage_categories', competition_id=competition.id) }}" class="btn btn-sm">Категории</a>
            {% if competition.categories %}
                <a href="{{ url_for('competition_results', category_id=competition.categories[0].id) }}" class="btn btn-sm">Результаты</a>
            {% endif %}
        </div>
    </div>
    {% else %}
    <p>Нет созданных соревнований. Создайте первое соревнование!</p>
    {% endfor %}
</div>
{% endblock %}
```

templates/categories.html:

```html
{% extends "base.html" %}

{% block title %}Категории - {{ competition.name }}{% endblock %}

{% block content %}
<div class="header">
    <h1>{{ competition.name }}</h1>
    <h2>Управление категориями</h2>
</div>

<div class="two-column-layout">
    <div class="column">
        <h3>Создать новую категорию</h3>
        <form method="POST">
            {{ form.hidden_tag() }}
            <div class="form-group">
                {{ form.name.label }}
                {{ form.name(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.description.label }}
                {{ form.description(class="form-control", rows=3) }}
            </div>
            <div class="form-row">
                <div class="form-group">
                    {{ form.min_age.label }}
                    {{ form.min_age(class="form-control") }}
                </div>
                <div class="form-group">
                    {{ form.max_age.label }}
                    {{ form.max_age(class="form-control") }}
                </div>
            </div>
            <div class="form-group">
                {{ form.gender.label }}
                {{ form.gender(class="form-control") }}
            </div>
            {{ form.submit(class="btn btn-primary") }}
        </form>
    </div>

    <div class="column">
        <h3>Существующие категории</h3>
        {% for category in competition.categories %}
        <div class="category-card">
            <h4>{{ category.name }}</h4>
            <p>{{ category.description }}</p>
            <div class="category-info">
                <span>Возраст: {{ category.min_age }}-{{ category.max_age }}</span>
                <span>Пол: {{ category.gender }}</span>
            </div>
            <div class="category-actions">
                <a href="{{ url_for('register_participant', category_id=category.id) }}" class="btn btn-sm">Участники</a>
                <a href="{{ url_for('draw_participants', category_id=category.id) }}" class="btn btn-sm">Жеребьевка</a>
                <a href="{{ url_for('scoring', category_id=category.id) }}" class="btn btn-sm">Оценивание</a>
            </div>
        </div>
        {% else %}
        <p>Нет созданных категорий.</p>
        {% endfor %}
    </div>
</div>
{% endblock %}
```

Шаг 6: Создание CSS стилей

static/css/style.css:

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    line-height: 1.6;
    color: #333;
    background-color: #f5f5f5;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

.navbar {
    background-color: #2c3e50;
    color: white;
    padding: 1rem 0;
    margin-bottom: 2rem;
}

.navbar .container {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.logo {
    font-size: 1.5rem;
    font-weight: bold;
    color: white;
    text-decoration: none;
}

.nav-links a {
    color: white;
    text-decoration: none;
    margin-left: 1.5rem;
    transition: color 0.3s;
}

.nav-links a:hover {
    color: #3498db;
}

.header {
    margin-bottom: 2rem;
    text-align: center;
}

.competitions-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 1.5rem;
    margin-top: 2rem;
}

.competition-card, .category-card {
    background: white;
    border-radius: 8px;
    padding: 1.5rem;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    transition: transform 0.3s, box-shadow 0.3s;
}

.competition-card:hover, .category-card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.competition-meta, .category-info {
    display: flex;
    justify-content: space-between;
    margin: 1rem 0;
    font-size: 0.9rem;
    color: #666;
}

.status {
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.8rem;
    font-weight: bold;
}

.status-planned { background-color: #f39c12; color: white; }
.status-active { background-color: #27ae60; color: white; }
.status-finished { background-color: #7f8c8d; color: white; }

.competition-actions, .category-actions {
    display: flex;
    gap: 0.5rem;
    margin-top: 1rem;
}

.btn {
    display: inline-block;
    padding: 0.5rem 1rem;
    background-color: #3498db;
    color: white;
    text-decoration: none;
    border-radius: 4px;
    border: none;
    cursor: pointer;
    transition: background-color 0.3s;
}

.btn:hover {
    background-color: #2980b9;
}

.btn-primary {
    background-color: #e74c3c;
}

.btn-primary:hover {
    background-color: #c0392b;
}

.btn-sm {
    padding: 0.25rem 0.5rem;
    font-size: 0.9rem;
}

.two-column-layout {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 2rem;
    margin-top: 2rem;
}

.form-group {
    margin-bottom: 1rem;
}

.form-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem;
}

.form-control {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 1rem;
}

textarea.form-control {
    min-height: 100px;
    resize: vertical;
}

.alert {
    padding: 1rem;
    margin-bottom: 1rem;
    border-radius: 4px;
}

.alert-success {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

.alert-error {
    background-color: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
}

.participants-table {
    width: 100%;
    background: white;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.participants-table table {
    width: 100%;
    border-collapse: collapse;
}

.participants-table th,
.participants-table td {
    padding: 1rem;
    text-align: left;
    border-bottom: 1px solid #eee;
}

.participants-table th {
    background-color: #f8f9fa;
    font-weight: 600;
}

.participants-table tr:hover {
    background-color: #f8f9fa;
}

.score-form {
    background: white;
    padding: 1.5rem;
    border-radius: 8px;
    margin-bottom: 1.5rem;
}

.score-criteria {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    gap: 1rem;
    margin-bottom: 1rem;
}

.results-container {
    background: white;
    border-radius: 8px;
    padding: 1.5rem;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.results-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 1rem;
}

.results-table th {
    background-color: #2c3e50;
    color: white;
    padding: 1rem;
    text-align: left;
}

.results-table td {
    padding: 1rem;
    border-bottom: 1px solid #eee;
}

.place-1 { background-color: #ffd700; }
.place-2 { background-color: #c0c0c0; }
.place-3 { background-color: #cd7f32; }

@media (max-width: 768px) {
    .two-column-layout {
        grid-template-columns: 1fr;
    }
    
    .competitions-grid {
        grid-template-columns: 1fr;
    }
    
    .form-row {
        grid-template-columns: 1fr;
    }
}
```

Шаг 7: Создание JavaScript для интерактивности

static/js/main.js:

```javascript
// Функция для обновления результатов в реальном времени
function updateLiveResults(categoryId) {
    fetch(`/api/category/${categoryId}/scores`)
        .then(response => response.json())
        .then(data => {
            // Обновление интерфейса с результатами
            const resultsContainer = document.getElementById('live-results');
            if (resultsContainer) {
                resultsContainer.innerHTML = data.map(participant => `
                    <div class="participant-score">
                        <span>${participant.order}. ${participant.name}</span>
                        <span>Оценок: ${participant.score_count}</span>
                        <span>Среднее: ${participant.average_score}</span>
                    </div>
                `).join('');
            }
        })
        .catch(error => console.error('Error fetching results:', error));
}

// Автоматическое обновление результатов каждые 10 секунд
document.addEventListener('DOMContentLoaded', function() {
    const categoryId = document.body.dataset.categoryId;
    if (categoryId) {
        // Обновляем сразу при загрузке
        updateLiveResults(categoryId);
        
        // И каждые 10 секунд
        setInterval(() => updateLiveResults(categoryId), 10000);
    }
    
    // Валидация форм
    const forms = document.querySelectorAll('form');
    forms.forEach(form => {
        form.addEventListener('submit', function(e) {
            const numberInputs = this.querySelectorAll('input[type="number"]');
            let isValid = true;
            
            numberInputs.forEach(input => {
                if (input.hasAttribute('min') || input.hasAttribute('max')) {
                    const value = parseFloat(input.value);
                    const min = parseFloat(input.getAttribute('min')) || -Infinity;
                    const max = parseFloat(input.getAttribute('max')) || Infinity;
                    
                    if (value < min || value > max) {
                        isValid = false;
                        input.classList.add('error');
                        alert(`Поле "${input.name}" должно быть между ${min} и ${max}`);
                    } else {
                        input.classList.remove('error');
                    }
                }
            });
            
            if (!isValid) {
                e.preventDefault();
            }
        });
    });
    
    // Сортировка таблиц
    const tables = document.querySelectorAll('table.sortable');
    tables.forEach(table => {
        const headers = table.querySelectorAll('th[data-sort]');
        headers.forEach(header => {
            header.style.cursor = 'pointer';
            header.addEventListener('click', function() {
                const column = this.dataset.sort;
                const tbody = table.querySelector('tbody');
                const rows = Array.from(tbody.querySelectorAll('tr'));
                
                const isAscending = !this.classList.contains('asc');
                this.classList.toggle('asc', isAscending);
                this.classList.toggle('desc', !isAscending);
                
                rows.sort((a, b) => {
                    const aVal = a.querySelector(`td:nth-child(${this.cellIndex + 1})`).textContent;
                    const bVal = b.querySelector(`td:nth-child(${this.cellIndex + 1})`).textContent;
                    
                    if (!isNaN(aVal) && !isNaN(bVal)) {
                        return isAscending ? aVal - bVal : bVal - aVal;
                    }
                    
                    return isAscending 
                        ? aVal.localeCompare(bVal)
                        : bVal.localeCompare(aVal);
                });
                
                rows.forEach(row => tbody.appendChild(row));
            });
        });
    });
});
```

Шаг 8: Запуск приложения

1. Создайте виртуальное окружение:

```bash
python -m venv venv
```

1. Активируйте виртуальное окружение:

· Windows: venv\Scripts\activate
· Linux/Mac: source venv/bin/activate

1. Установите зависимости:

```bash
pip install -r requirements.txt
```

1. Запустите приложение:

```bash
python app.py
```

1. Откройте браузер и перейдите по адресу: http://localhost:5000

Функциональность приложения:

1. Создание соревнований: Укажите название, описание, даты проведения
2. Создание категорий: Разделение соревнований по возрасту, полу, уровню
3. Регистрация участников: Внесение данных об участниках
4. Жеребьевка: Случайное распределение порядка выступлений
5. Корректировка порядка: Ручное изменение порядка выступлений
6. Выставление оценок: Система оценок от нескольких судей
7. Выявление победителей: Автоматический расчет результатов с учетом всех оценок

Дальнейшее развитие:

1. Добавить аутентификацию пользователей
2. Реализовать экспорт результатов в PDF/Excel
3. Добавить систему уведомлений
4. Реализовать онлайн-трансляцию результатов
5. Добавить модуль для проведения квалификационных отборов
6. Интеграция с платежными системами для регистрационных взносов
7. Мобильное приложение для судей
8. Система видеоповторов для арбитража

Это приложение предоставляет полный цикл управления соревнованиями и может быть адаптировано под конкретные виды спорта или конкурсов.

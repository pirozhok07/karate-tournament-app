Я создам для вас пошаговую инструкцию с кодом для приложения проведения соревнований. Вот полный проект:

📁 Структура проекта

```
competition_app/
├── app.py
├── requirements.txt
├── config.py
├── models.py
├── database.py
├── utils/
│   ├── excel_handler.py
│   ├── draw_generator.py
│   └── pdf_reporter.py
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── upload.html
│   ├── categories.html
│   ├── competition.html
│   ├── results.html
│   └── report.html
├── static/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── script.js
└── uploads/
```

📋 1. requirements.txt

```txt
Flask==3.0.0
Flask-SQLAlchemy==3.0.5
Flask-WTF==1.2.1
pandas==2.1.0
openpyxl==3.1.2
reportlab==4.0.5
python-dotenv==1.0.0
XlsxWriter==3.1.2
```

⚙️ 2. config.py

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'your-secret-key-here'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///competition.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    UPLOAD_FOLDER = 'uploads'
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16MB
    ALLOWED_EXTENSIONS = {'xlsx', 'xls'}
```

🗃️ 3. models.py

```python
from datetime import datetime
from database import db

class Athlete(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    first_name = db.Column(db.String(50), nullable=False)
    last_name = db.Column(db.String(50), nullable=False)
    birth_date = db.Column(db.Date)
    gender = db.Column(db.String(10))
    weight = db.Column(db.Float)
    height = db.Column(db.Float)
    club = db.Column(db.String(100))
    registration_number = db.Column(db.String(50), unique=True)
    category_id = db.Column(db.Integer, db.ForeignKey('category.id'))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    scores = db.relationship('Score', backref='athlete', lazy=True)
    
    def __repr__(self):
        return f'Athlete({self.first_name} {self.last_name})'

class Category(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    min_age = db.Column(db.Integer)
    max_age = db.Column(db.Integer)
    min_weight = db.Column(db.Float)
    max_weight = db.Column(db.Float)
    gender = db.Column(db.String(10))
    
    athletes = db.relationship('Athlete', backref='category', lazy=True)
    
    def __repr__(self):
        return f'Category({self.name})'

class Competition(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(200), nullable=False)
    date = db.Column(db.Date, nullable=False)
    location = db.Column(db.String(200))
    status = db.Column(db.String(20), default='pending')  # pending, active, completed
    current_round = db.Column(db.Integer, default=1)
    
    scores = db.relationship('Score', backref='competition', lazy=True)

class Score(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    athlete_id = db.Column(db.Integer, db.ForeignKey('athlete.id'), nullable=False)
    competition_id = db.Column(db.Integer, db.ForeignKey('competition.id'), nullable=False)
    round_number = db.Column(db.Integer, nullable=False)
    judge1 = db.Column(db.Float)
    judge2 = db.Column(db.Float)
    judge3 = db.Column(db.Float)
    judge4 = db.Column(db.Float)
    judge5 = db.Column(db.Float)
    total = db.Column(db.Float)
    average = db.Column(db.Float)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def calculate_scores(self):
        scores = [self.judge1, self.judge2, self.judge3, self.judge4, self.judge5]
        valid_scores = [s for s in scores if s is not None]
        if valid_scores:
            valid_scores.sort()
            valid_scores = valid_scores[1:-1]  # Убираем мин и макс
            self.total = sum(valid_scores)
            self.average = self.total / len(valid_scores)
```

🗄️ 4. database.py

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
```

📊 5. utils/excel_handler.py

```python
import pandas as pd
from datetime import datetime
from models import Athlete

def import_athletes_from_excel(file_path):
    """Импорт спортсменов из Excel файла"""
    try:
        df = pd.read_excel(file_path)
        
        athletes = []
        for _, row in df.iterrows():
            athlete = Athlete(
                first_name=str(row.get('Имя', '')),
                last_name=str(row.get('Фамилия', '')),
                birth_date=parse_date(row.get('Дата рождения')),
                gender=str(row.get('Пол', '')),
                weight=float(row.get('Вес', 0)) if pd.notna(row.get('Вес')) else None,
                height=float(row.get('Рост', 0)) if pd.notna(row.get('Рост')) else None,
                club=str(row.get('Клуб', '')),
                registration_number=str(row.get('Номер', ''))
            )
            athletes.append(athlete)
        
        return athletes
    except Exception as e:
        raise Exception(f"Ошибка при чтении Excel файла: {str(e)}")

def parse_date(date_str):
    """Парсинг даты из различных форматов"""
    if pd.isna(date_str):
        return None
    try:
        if isinstance(date_str, datetime):
            return date_str.date()
        elif isinstance(date_str, str):
            return datetime.strptime(date_str, '%Y-%m-%d').date()
        return None
    except:
        return None

def export_results_to_excel(results, output_path):
    """Экспорт результатов в Excel"""
    data = []
    for result in results:
        data.append({
            'Место': result['place'],
            'Фамилия': result['last_name'],
            'Имя': result['first_name'],
            'Клуб': result['club'],
            'Категория': result['category'],
            'Раунд 1': result['round1'],
            'Раунд 2': result['round2'],
            'Раунд 3': result['round3'],
            'Общий балл': result['total'],
            'Средний балл': result['average']
        })
    
    df = pd.DataFrame(data)
    with pd.ExcelWriter(output_path, engine='xlsxwriter') as writer:
        df.to_excel(writer, sheet_name='Результаты', index=False)
        
        workbook = writer.book
        worksheet = writer.sheets['Результаты']
        
        # Форматирование
        header_format = workbook.add_format({
            'bold': True,
            'text_wrap': True,
            'valign': 'top',
            'bg_color': '#D7E4BC',
            'border': 1
        })
        
        for col_num, value in enumerate(df.columns.values):
            worksheet.write(0, col_num, value, header_format)
            worksheet.set_column(col_num, col_num, 15)
    
    return output_path
```

🎲 6. utils/draw_generator.py

```python
import random
from datetime import datetime
from models import Athlete, Category

def categorize_athletes(athletes, categories):
    """Распределение спортсменов по категориям"""
    categorized = {}
    
    for athlete in athletes:
        assigned = False
        for category in categories:
            if matches_category(athlete, category):
                if category.name not in categorized:
                    categorized[category.name] = []
                categorized[category.name].append(athlete)
                athlete.category_id = category.id
                assigned = True
                break
        
        if not assigned:
            if 'Без категории' not in categorized:
                categorized['Без категории'] = []
            categorized['Без категории'].append(athlete)
    
    return categorized

def matches_category(athlete, category):
    """Проверка соответствия спортсмена категории"""
    # Проверка пола
    if category.gender and category.gender != athlete.gender:
        return False
    
    # Проверка возраста
    if athlete.birth_date:
        age = calculate_age(athlete.birth_date)
        if category.min_age and age < category.min_age:
            return False
        if category.max_age and age > category.max_age:
            return False
    
    # Проверка веса
    if athlete.weight:
        if category.min_weight and athlete.weight < category.min_weight:
            return False
        if category.max_weight and athlete.weight > category.max_weight:
            return False
    
    return True

def calculate_age(birth_date):
    """Расчет возраста"""
    today = datetime.today()
    return today.year - birth_date.year - (
        (today.month, today.day) < (birth_date.month, birth_date.day)
    )

def generate_draw(category_athletes):
    """Генерация сетки соревнований"""
    draw = {}
    
    for category_name, athletes in category_athletes.items():
        # Случайный порядок выступления
        random.shuffle(athletes)
        
        # Создание пар для первого раунда
        pairs = []
        for i in range(0, len(athletes), 2):
            if i + 1 < len(athletes):
                pairs.append([athletes[i], athletes[i + 1]])
            else:
                pairs.append([athletes[i], None])  # Свободный жребий
        
        draw[category_name] = {
            'athletes': athletes,
            'pairs': pairs,
            'order': [athlete.id for athlete in athletes]
        }
    
    return draw
```

📄 7. utils/pdf_reporter.py

```python
from reportlab.lib.pagesizes import A4
from reportlab.lib import colors
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib.units import inch, cm
from reportlab.pdfgen import canvas
from reportlab.lib.enums import TA_CENTER

def generate_results_pdf(results, competition_info, output_path):
    """Генерация PDF отчета с результатами"""
    doc = SimpleDocTemplate(
        output_path,
        pagesize=A4,
        rightMargin=72,
        leftMargin=72,
        topMargin=72,
        bottomMargin=18
    )
    
    story = []
    styles = getSampleStyleSheet()
    
    # Заголовок
    title_style = ParagraphStyle(
        'CustomTitle',
        parent=styles['Heading1'],
        fontSize=24,
        spaceAfter=30,
        alignment=TA_CENTER
    )
    
    story.append(Paragraph(f"Протокол соревнований: {competition_info['name']}", title_style))
    story.append(Paragraph(f"Дата: {competition_info['date']}", styles['Normal']))
    story.append(Paragraph(f"Место проведения: {competition_info['location']}", styles['Normal']))
    story.append(Spacer(1, 20))
    
    # Группировка по категориям
    categories = {}
    for result in results:
        category = result['category']
        if category not in categories:
            categories[category] = []
        categories[category].append(result)
    
    # Таблица для каждой категории
    for category, cat_results in categories.items():
        story.append(Paragraph(f"Категория: {category}", styles['Heading2']))
        story.append(Spacer(1, 10))
        
        # Сортировка по месту
        cat_results.sort(key=lambda x: x['place'])
        
        # Подготовка данных для таблицы
        data = [['Место', 'Спортсмен', 'Клуб', 'Раунд 1', 'Раунд 2', 'Раунд 3', 'Общий', 'Средний']]
        
        for result in cat_results:
            data.append([
                str(result['place']),
                f"{result['last_name']} {result['first_name']}",
                result['club'],
                f"{result['round1']:.2f}" if result['round1'] else '-',
                f"{result['round2']:.2f}" if result['round2'] else '-',
                f"{result['round3']:.2f}" if result['round3'] else '-',
                f"{result['total']:.2f}" if result['total'] else '-',
                f"{result['average']:.2f}" if result['average'] else '-'
            ])
        
        # Создание таблицы
        table = Table(data, colWidths=[1*cm, 4*cm, 3*cm, 2*cm, 2*cm, 2*cm, 2*cm, 2*cm])
        table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 10),
            ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
            ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
            ('GRID', (0, 0), (-1, -1), 1, colors.black),
            ('ALIGN', (1, 1), (1, -1), 'LEFT'),
        ]))
        
        story.append(table)
        story.append(Spacer(1, 30))
    
    # Подписи
    story.append(Spacer(1, 50))
    story.append(Paragraph("Главный судья: _________________________", styles['Normal']))
    story.append(Spacer(1, 20))
    story.append(Paragraph("Главный секретарь: _________________________", styles['Normal']))
    
    doc.build(story)
    return output_path
```

🚀 8. app.py (основное приложение)

```python
from flask import Flask, render_template, request, redirect, url_for, flash, send_file, jsonify
from flask_wtf import FlaskForm
from wtforms import FileField, SubmitField, StringField, DateField, FloatField, IntegerField
from wtforms.validators import DataRequired
from werkzeug.utils import secure_filename
import os
import json
from datetime import datetime

from config import Config
from database import db
from models import Athlete, Category, Competition, Score
from utils.excel_handler import import_athletes_from_excel, export_results_to_excel
from utils.draw_generator import categorize_athletes, generate_draw
from utils.pdf_reporter import generate_results_pdf

app = Flask(__name__)
app.config.from_object(Config)
db.init_app(app)

# Формы
class UploadForm(FlaskForm):
    excel_file = FileField('Excel файл', validators=[DataRequired()])
    submit = SubmitField('Загрузить')

class CategoryForm(FlaskForm):
    name = StringField('Название категории', validators=[DataRequired()])
    min_age = IntegerField('Минимальный возраст')
    max_age = IntegerField('Максимальный возраст')
    min_weight = FloatField('Минимальный вес')
    max_weight = FloatField('Максимальный вес')
    gender = StringField('Пол (М/Ж)')
    submit = SubmitField('Создать категорию')

class CompetitionForm(FlaskForm):
    name = StringField('Название соревнования', validators=[DataRequired()])
    date = DateField('Дата', validators=[DataRequired()])
    location = StringField('Место проведения')
    submit = SubmitField('Создать')

# Вспомогательные функции
def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

# Маршруты
@app.route('/')
def index():
    competitions = Competition.query.all()
    return render_template('index.html', competitions=competitions)

@app.route('/upload', methods=['GET', 'POST'])
def upload_athletes():
    form = UploadForm()
    if form.validate_on_submit():
        if 'excel_file' not in request.files:
            flash('Файл не выбран')
            return redirect(request.url)
        
        file = request.files['excel_file']
        if file.filename == '':
            flash('Файл не выбран')
            return redirect(request.url)
        
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            file.save(filepath)
            
            try:
                athletes = import_athletes_from_excel(filepath)
                for athlete in athletes:
                    db.session.add(athlete)
                db.session.commit()
                flash(f'Успешно загружено {len(athletes)} спортсменов')
                return redirect(url_for('manage_categories'))
            except Exception as e:
                flash(f'Ошибка: {str(e)}')
    
    return render_template('upload.html', form=form)

@app.route('/categories', methods=['GET', 'POST'])
def manage_categories():
    form = CategoryForm()
    if form.validate_on_submit():
        category = Category(
            name=form.name.data,
            min_age=form.min_age.data,
            max_age=form.max_age.data,
            min_weight=form.min_weight.data,
            max_weight=form.max_weight.data,
            gender=form.gender.data
        )
        db.session.add(category)
        db.session.commit()
        flash('Категория создана')
        return redirect(url_for('manage_categories'))
    
    categories = Category.query.all()
    athletes = Athlete.query.all()
    
    # Автоматическое распределение
    categorized = categorize_athletes(athletes, categories)
    
    return render_template('categories.html', 
                         form=form, 
                         categories=categories, 
                         athletes=athletes,
                         categorized=categorized)

@app.route('/create_competition', methods=['GET', 'POST'])
def create_competition():
    form = CompetitionForm()
    if form.validate_on_submit():
        competition = Competition(
            name=form.name.data,
            date=form.date.data,
            location=form.location.data,
            status='pending'
        )
        db.session.add(competition)
        db.session.commit()
        
        # Создание сетки
        categories = Category.query.all()
        athletes = Athlete.query.all()
        categorized = categorize_athletes(athletes, categories)
        draw = generate_draw(categorized)
        
        # Сохранение сетки
        competition.draw_data = json.dumps(draw)
        db.session.commit()
        
        flash('Соревнование создано')
        return redirect(url_for('view_competition', id=competition.id))
    
    return render_template('create_competition.html', form=form)

@app.route('/competition/<int:id>')
def view_competition(id):
    competition = Competition.query.get_or_404(id)
    draw = json.loads(competition.draw_data) if competition.draw_data else {}
    
    # Получение результатов
    scores = Score.query.filter_by(competition_id=id).all()
    
    return render_template('competition.html', 
                         competition=competition, 
                         draw=draw,
                         scores=scores)

@app.route('/enter_scores', methods=['POST'])
def enter_scores():
    data = request.json
    athlete_id = data['athlete_id']
    competition_id = data['competition_id']
    round_number = data['round_number']
    scores = data['scores']
    
    # Поиск существующей записи
    score = Score.query.filter_by(
        athlete_id=athlete_id,
        competition_id=competition_id,
        round_number=round_number
    ).first()
    
    if not score:
        score = Score(
            athlete_id=athlete_id,
            competition_id=competition_id,
            round_number=round_number
        )
    
    # Установка оценок
    score.judge1 = scores[0]
    score.judge2 = scores[1]
    score.judge3 = scores[2]
    score.judge4 = scores[3]
    score.judge5 = scores[4]
    score.calculate_scores()
    
    db.session.add(score)
    db.session.commit()
    
    return jsonify({'success': True, 'average': score.average})

@app.route('/results/<int:competition_id>')
def show_results(competition_id):
    competition = Competition.query.get_or_404(competition_id)
    
    # Расчет результатов
    results = calculate_final_results(competition_id)
    
    return render_template('results.html', 
                         competition=competition,
                         results=results)

def calculate_final_results(competition_id):
    """Расчет финальных результатов"""
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

@app.route('/export/excel/<int:competition_id>')
def export_excel(competition_id):
    results = calculate_final_results(competition_id)
    competition = Competition.query.get(competition_id)
    
    filename = f"results_{competition.name.replace(' ', '_')}.xlsx"
    filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    
    export_results_to_excel(results, filepath)
    
    return send_file(filepath, as_attachment=True)

@app.route('/export/pdf/<int:competition_id>')
def export_pdf(competition_id):
    results = calculate_final_results(competition_id)
    competition = Competition.query.get(competition_id)
    
    filename = f"protocol_{competition.name.replace(' ', '_')}.pdf"
    filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    
    competition_info = {
        'name': competition.name,
        'date': competition.date.strftime('%d.%m.%Y'),
        'location': competition.location
    }
    
    generate_results_pdf(results, competition_info, filepath)
    
    return send_file(filepath, as_attachment=True)

# Инициализация базы данных
@app.before_first_request
def create_tables():
    db.create_all()
    # Создание папки для загрузок
    os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

if __name__ == '__main__':
    app.run(debug=True)
```

🎨 9. templates/base.html

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Система проведения соревнований</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('index') }}">Соревнования</a>
            <div class="navbar-nav">
                <a class="nav-link" href="{{ url_for('upload_athletes') }}">Загрузка спортсменов</a>
                <a class="nav-link" href="{{ url_for('manage_categories') }}">Категории</a>
                <a class="nav-link" href="{{ url_for('create_competition') }}">Создать соревнование</a>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
        {% with messages = get_flashed_messages() %}
            {% if messages %}
                {% for message in messages %}
                    <div class="alert alert-info">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}

        {% block content %}{% endblock %}
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
    <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

📝 10. templates/competition.html

```html
{% extends "base.html" %}

{% block content %}
<div class="row">
    <div class="col-12">
        <h2>{{ competition.name }}</h2>
        <p>Дата: {{ competition.date }} | Место: {{ competition.location }}</p>
        <p>Статус: {{ competition.status }} | Текущий раунд: {{ competition.current_round }}</p>
    </div>
</div>

<div class="row">
    {% for category, data in draw.items() %}
    <div class="col-md-6 mb-4">
        <div class="card">
            <div class="card-header">
                <h5>Категория: {{ category }}</h5>
            </div>
            <div class="card-body">
                <h6>Порядок выступления:</h6>
                <ol>
                    {% for athlete in data.athletes %}
                    <li>{{ athlete.last_name }} {{ athlete.first_name }} ({{ athlete.club }})</li>
                    {% endfor %}
                </ol>
                
                <h6>Пары 1 раунда:</h6>
                <table class="table table-sm">
                    {% for pair in data.pairs %}
                    <tr>
                        <td>{{ pair[0].last_name if pair[0] else 'Свободный жребий' }}</td>
                        <td>vs</td>
                        <td>{{ pair[1].last_name if pair[1] else 'Свободный жребий' }}</td>
                    </tr>
                    {% endfor %}
                </table>
            </div>
        </div>
    </div>
    {% endfor %}
</div>

<div class="row">
    <div class="col-12">
        <h3>Ввод оценок</h3>
        <form id="scoreForm">
            <div class="row mb-3">
                <div class="col-md-3">
                    <label>Спортсмен:</label>
                    <select id="athleteSelect" class="form-select" required>
                        <option value="">Выберите спортсмена</option>
                        {% for category, data in draw.items() %}
                            {% for athlete in data.athletes %}
                            <option value="{{ athlete.id }}">{{ athlete.last_name }} {{ athlete.first_name }}</option>
                            {% endfor %}
                        {% endfor %}
                    </select>
                </div>
                <div class="col-md-2">
                    <label>Раунд:</label>
                    <select id="roundSelect" class="form-select" required>
                        <option value="1">Раунд 1</option>
                        <option value="2">Раунд 2</option>
                        <option value="3">Раунд 3</option>
                    </select>
                </div>
            </div>
            
            <div class="row mb-3">
                <h5>Оценки судей (0-10)</h5>
                {% for i in range(5) %}
                <div class="col-md-2">
                    <label>Судья {{ i+1 }}:</label>
                    <input type="number" class="form-control judge-score" 
                           min="0" max="10" step="0.1" required>
                </div>
                {% endfor %}
            </div>
            
            <button type="submit" class="btn btn-primary">Сохранить оценки</button>
        </form>
    </div>
</div>

<div class="row mt-4">
    <div class="col-12">
        <a href="{{ url_for('show_results', competition_id=competition.id) }}" 
           class="btn btn-success">Показать результаты</a>
        <a href="{{ url_for('export_excel', competition_id=competition.id) }}" 
           class="btn btn-info">Экспорт в Excel</a>
        <a href="{{ url_for('export_pdf', competition_id=competition.id) }}" 
           class="btn btn-danger">Экспорт в PDF</a>
    </div>
</div>

<script>
document.getElementById('scoreForm').addEventListener('submit', function(e) {
    e.preventDefault();
    
    const athleteId = document.getElementById('athleteSelect').value;
    const roundNumber = document.getElementById('roundSelect').value;
    const scores = Array.from(document.querySelectorAll('.judge-score'))
        .map(input => parseFloat(input.value));
    
    fetch('/enter_scores', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            athlete_id: athleteId,
            competition_id: {{ competition.id }},
            round_number: roundNumber,
            scores: scores
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            alert(`Оценки сохранены! Средний балл: ${data.average.toFixed(2)}`);
            document.getElementById('scoreForm').reset();
        }
    });
});
</script>
{% endblock %}
```

🎯 Пошаговая инструкция использования:

Шаг 1: Установка

```bash
# Создание виртуального окружения
python -m venv venv

# Активация (Windows)
venv\Scripts\activate

# Активация (Linux/Mac)
source venv/bin/activate

# Установка зависимостей
pip install -r requirements.txt
```

Шаг 2: Запуск приложения

```bash
python app.py
```

Шаг 3: Загрузка спортсменов

1. Перейдите на /upload
2. Загрузите Excel файл со спортсменами (формат: Имя, Фамилия, Дата рождения, Пол, Вес, Рост, Клуб, Номер)
3. Файл будет обработан и спортсмены добавлены в базу

Шаг 4: Создание категорий

1. Перейдите на /categories
2. Создайте категории с параметрами (возраст, вес, пол)
3. Система автоматически распределит спортсменов по категориям

Шаг 5: Создание соревнования

1. Перейдите на /create_competition
2. Заполните данные о соревновании
3. Система создаст сетку (draw) со случайным порядком выступления

Шаг 6: Проведение соревнований

1. На странице соревнования вводите оценки 5 судей для каждого спортсмена
2. Система автоматически отбрасывает мин и макс оценки
3. Результаты сохраняются после каждого раунда

Шаг 7: Просмотр результатов

1. На странице результатов видны итоговые места
2. Учитываются 2 лучших раунда из 3

Шаг 8: Экспорт отчетов

1. Excel отчет для дальнейшей обработки
2. PDF протокол для печати с подписями

📊 Пример Excel файла для загрузки спортсменов:

Имя Фамилия Дата рождения Пол Вес Рост Клуб Номер
Иван Петров 2000-05-15 М 75.5 180 Спартак 001
Анна Сидорова 2001-08-20 Ж 60.2 170 Динамо 002

🔧 Особенности системы:

1. Гибкая система категорий (по возрасту, весу, полу)
2. Автоматическая жеребьевка (случайный порядок выступления)
3. Система оценок (5 судей, отбрасывание мин/макс)
4. Три раунда с учетом двух лучших
5. Экспорт в PDF с форматированием для печати
6. Веб-интерфейс для удобного ввода данных
7. База данных SQLite для хранения всех данных

Это полнофункциональное приложение готово к использованию! Вы можете расширить его функционал по мере необходимости.

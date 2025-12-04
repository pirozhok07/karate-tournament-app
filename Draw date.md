🔧 ИСПРАВЛЕНИЕ ОШИБКИ: Competition object has no attribute draw_data

Эта ошибка возникает потому, что в модели Competition нет поля draw_data. Давайте исправим это:

📝 1. Обновите models.py (добавьте поле draw_data)

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
    description = db.Column(db.Text)  # Добавлено поле описания
    status = db.Column(db.String(20), default='pending')  # pending, active, completed
    current_round = db.Column(db.Integer, default=1)
    draw_data = db.Column(db.Text)  # ДОБАВЬТЕ ЭТУ СТРОЧКУ - хранит JSON с сеткой
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    scores = db.relationship('Score', backref='competition', lazy=True)
    
    def __repr__(self):
        return f'Competition({self.name})'

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

📝 2. Обновите app.py с правильной работой с draw_data

```python
from flask import Flask, render_template, request, redirect, url_for, flash, send_file, jsonify
from flask_wtf import FlaskForm
from wtforms import FileField, SubmitField, StringField, DateField, FloatField, IntegerField, TextAreaField
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

# Флаг инициализации
app_initialized = False

@app.before_request
def initialize_on_first_request():
    """Инициализация при первом запросе"""
    global app_initialized
    
    if not app_initialized:
        # Создание таблиц в базе данных
        with app.app_context():
            db.create_all()
            
            # Создание папки для загрузок
            uploads_dir = app.config['UPLOAD_FOLDER']
            os.makedirs(uploads_dir, exist_ok=True)
            print(f"✅ Приложение инициализировано: создана папка {uploads_dir}")
        
        app_initialized = True

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
    description = TextAreaField('Описание')
    submit = SubmitField('Создать')

# Вспомогательные функции
def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

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

# Маршруты
@app.route('/')
def index():
    competitions = Competition.query.all()
    athletes_count = Athlete.query.count()
    active_competitions = Competition.query.filter_by(status='active').count()
    
    return render_template('index.html', 
                         competitions=competitions,
                         athletes_count=athletes_count,
                         active_competitions=active_competitions)

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
        try:
            # Используем новую функцию для создания соревнования с сеткой
            competition = create_competition_with_draw(
                name=form.name.data,
                date=form.date.data,
                location=form.location.data,
                description=form.description.data
            )
            
            flash(f'Соревнование "{competition.name}" создано!', 'success')
            return redirect(url_for('view_competition', id=competition.id))
            
        except Exception as e:
            flash(f'Ошибка при создании соревнования: {str(e)}', 'error')
    
    return render_template('create_competition.html', form=form)

@app.route('/competition/<int:id>')
def view_competition(id):
    competition = Competition.query.get_or_404(id)
    
    # Проверяем наличие draw_data
    draw = {}
    if competition.draw_data:
        try:
            draw = json.loads(competition.draw_data)
        except json.JSONDecodeError:
            draw = {}
    
    # Получение результатов
    scores = Score.query.filter_by(competition_id=id).all()
    
    return render_template('competition.html', 
                         competition=competition, 
                         draw=draw,
                         scores=scores)

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
        
        # Распределяем спортсменов по категориям
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

# Новые API эндпоинты для расширенного функционала
@app.route('/api/athletes')
def api_athletes():
    """API для получения списка спортсменов"""
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
            'category': athlete.category.name if athlete.category else None
        })
    return jsonify(result)

@app.route('/api/categories')
def api_categories():
    """API для получения списка категорий"""
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
            'gender': category.gender
        })
    return jsonify(result)

@app.route('/api/create_competition_full', methods=['POST'])
def api_create_competition_full():
    """API для создания полного соревнования с участниками и сеткой"""
    data = request.json
    
    try:
        # Создание соревнования
        competition = Competition(
            name=data['name'],
            date=datetime.strptime(data['date'], '%Y-%m-%d'),
            location=data['location'],
            description=data.get('description'),
            status='pending',
            current_round=1
        )
        db.session.add(competition)
        db.session.flush()  # Получаем ID без коммита
        
        # Сохранение сетки если есть
        if data.get('draw'):
            competition.draw_data = json.dumps(data['draw'], ensure_ascii=False)
        
        db.session.commit()
        
        return jsonify({
            'success': True,
            'competition_id': competition.id,
            'message': f'Соревнование "{competition.name}" создано'
        })
        
    except Exception as e:
        db.session.rollback()
        return jsonify({
            'success': False,
            'error': str(e)
        }), 500

@app.route('/api/competition/<int:id>/draw')
def api_competition_draw(id):
    """API для получения сетки соревнования"""
    competition = Competition.query.get_or_404(id)
    
    if not competition.draw_data:
        return jsonify({'error': 'Сетка не сгенерирована'}), 404
    
    try:
        draw = json.loads(competition.draw_data)
        return jsonify(draw)
    except json.JSONDecodeError:
        return jsonify({'error': 'Ошибка при чтении сетки'}), 500

if __name__ == '__main__':
    app.run(debug=True)
```

📝 3. Обновите базу данных

Поскольку мы добавили новое поле в модель, нужно обновить базу данных:

Вариант A: Удалить старую базу данных (простой способ)

```bash
# Удалите файл базы данных
rm competition.db  # Linux/Mac
# или
del competition.db  # Windows

# Перезапустите приложение
python app.py
```

Вариант B: Использовать миграции (рекомендуется)

```bash
# Установите Flask-Migrate
pip install flask-migrate

# Обновите app.py, добавив:
from flask_migrate import Migrate

app = Flask(__name__)
# ... остальной код ...

migrate = Migrate(app, db)

# Создайте миграцию
flask db init
flask db migrate -m "Add draw_data field to Competition"
flask db upgrade
```

📝 4. Упрощенная версия generate_draw (если у вас нет этого файла)

Если у вас нет файла utils/draw_generator.py, создайте его с таким содержанием:

```python
import random

def categorize_athletes(athletes, categories):
    """Распределение спортсменов по категориям"""
    categorized = {}
    
    # Простая логика распределения по полу
    for athlete in athletes:
        category_name = "Без категории"
        
        for category in categories:
            if category.gender and athlete.gender == category.gender:
                category_name = category.name
                break
        
        if category_name not in categorized:
            categorized[category_name] = []
        categorized[category_name].append(athlete)
    
    return categorized

def generate_draw(categorized):
    """Генерация сетки соревнований"""
    draw_data = {}
    
    for category_name, athletes in categorized.items():
        # Перемешиваем спортсменов
        random.shuffle(athletes)
        
        # Создаем пары
        pairs = []
        for i in range(0, len(athletes), 2):
            if i + 1 < len(athletes):
                pairs.append([
                    {'id': athletes[i].id, 'name': f"{athletes[i].last_name} {athletes[i].first_name}"},
                    {'id': athletes[i+1].id, 'name': f"{athletes[i+1].last_name} {athletes[i+1].first_name}"}
                ])
            else:
                pairs.append([
                    {'id': athletes[i].id, 'name': f"{athletes[i].last_name} {athletes[i].first_name}"},
                    None  # Свободный жребий
                ])
        
        draw_data[category_name] = {
            'athletes': [{'id': a.id, 'name': f"{a.last_name} {a.first_name}"} for a in athletes],
            'pairs': pairs,
            'order': [a.id for a in athletes]
        }
    
    return draw_data
```

🔧 Проверка работоспособности:

1. Проверьте модель Competition - убедитесь, что поле draw_data добавлено
2. Обновите базу данных - удалите старый файл или используйте миграции
3. Перезапустите приложение - python app.py
4. Проверьте создание соревнования - поле draw_data должно сохраняться

Теперь ошибка "Competition object has no attribute draw_data" должна быть исправлена!

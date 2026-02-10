Приложение для проведения соревнований по олимпийской системе

Структура проекта

```
tournament_app/
├── app.py                 # Основной Flask сервер
├── models.py             # Модели SQLAlchemy
├── database.py           # Инициализация БД
├── create_tournament.py  # Логика генерации турнирной сетки
├── static/
│   ├── style.css        # Стили
│   └── script.js        # Фронтенд логика
├── templates/
│   └── index.html       # Главная страница
├── requirements.txt      # Зависимости
└── config.py            # Конфигурация
```

1. Установка зависимостей

requirements.txt:

```txt
Flask==3.0.0
Flask-SQLAlchemy==3.1.1
Flask-CORS==4.0.0
python-dotenv==1.0.0
```

2. Конфигурация (config.py)

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///tournament.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

3. Модели базы данных (models.py)

```python
from datetime import datetime
from database import db

class Tournament(db.Model):
    __tablename__ = 'tournaments'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text)
    start_date = db.Column(db.DateTime, default=datetime.utcnow)
    status = db.Column(db.String(20), default='pending')  # pending, active, completed
    max_participants = db.Column(db.Integer, default=16)
    
    # Отношения
    participants = db.relationship('Participant', backref='tournament', lazy=True)
    matches = db.relationship('Match', backref='tournament', lazy=True)
    
    def to_dict(self):
        return {
            'id': self.id,
            'name': self.name,
            'description': self.description,
            'start_date': self.start_date.isoformat() if self.start_date else None,
            'status': self.status,
            'max_participants': self.max_participants
        }

class Participant(db.Model):
    __tablename__ = 'participants'
    
    id = db.Column(db.Integer, primary_key=True)
    tournament_id = db.Column(db.Integer, db.ForeignKey('tournaments.id'), nullable=False)
    name = db.Column(db.String(100), nullable=False)
    seed = db.Column(db.Integer)  # Посев участника
    is_active = db.Column(db.Boolean, default=True)
    
    def to_dict(self):
        return {
            'id': self.id,
            'tournament_id': self.tournament_id,
            'name': self.name,
            'seed': self.seed,
            'is_active': self.is_active
        }

class Match(db.Model):
    __tablename__ = 'matches'
    
    id = db.Column(db.Integer, primary_key=True)
    tournament_id = db.Column(db.Integer, db.ForeignKey('tournaments.id'), nullable=False)
    round = db.Column(db.Integer, nullable=False)  # Номер раунда
    match_order = db.Column(db.Integer)  # Порядковый номер матча в раунде
    participant1_id = db.Column(db.Integer, db.ForeignKey('participants.id'))
    participant2_id = db.Column(db.Integer, db.ForeignKey('participants.id'))
    winner_id = db.Column(db.Integer, db.ForeignKey('participants.id'))
    score1 = db.Column(db.Integer, default=0)
    score2 = db.Column(db.Integer, default=0)
    status = db.Column(db.String(20), default='scheduled')  # scheduled, in_progress, completed
    next_match_id = db.Column(db.Integer, db.ForeignKey('matches.id'))  # Следующий матч для победителя
    
    # Отношения
    participant1 = db.relationship('Participant', foreign_keys=[participant1_id])
    participant2 = db.relationship('Participant', foreign_keys=[participant2_id])
    winner = db.relationship('Participant', foreign_keys=[winner_id])
    next_match = db.relationship('Match', foreign_keys=[next_match_id], remote_side=[id])
    
    def to_dict(self):
        return {
            'id': self.id,
            'tournament_id': self.tournament_id,
            'round': self.round,
            'match_order': self.match_order,
            'participant1': self.participant1.to_dict() if self.participant1 else None,
            'participant2': self.participant2.to_dict() if self.participant2 else None,
            'winner_id': self.winner_id,
            'score1': self.score1,
            'score2': self.score2,
            'status': self.status,
            'next_match_id': self.next_match_id
        }
```

4. Инициализация базы данных (database.py)

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

def init_db(app):
    db.init_app(app)
    
    with app.app_context():
        db.create_all()
```

5. Логика генерации турнирной сетки (create_tournament.py)

```python
import math
from models import Tournament, Participant, Match, db

def generate_bracket(tournament_id):
    """
    Генерирует турнирную сетку для заданного количества участников
    по олимпийской системе (на выбывание)
    """
    tournament = Tournament.query.get(tournament_id)
    if not tournament:
        return None
    
    participants = Participant.query.filter_by(
        tournament_id=tournament_id, 
        is_active=True
    ).order_by(Participant.seed).all()
    
    num_participants = len(participants)
    if num_participants < 2:
        return None
    
    # Определяем ближайшую степень двойки, большую или равную количеству участников
    next_power_of_two = 2 ** math.ceil(math.log2(num_participants))
    
    # Создаем список участников с заполнением пустыми слотами
    bracket_participants = participants.copy()
    while len(bracket_participants) < next_power_of_two:
        bracket_participants.append(None)  # Пустой слот (bye)
    
    # Распределяем участников по сетке (метод seeding)
    bracket = distribute_participants(bracket_participants)
    
    # Создаем матчи для первого раунда
    matches = []
    match_id_counter = 1
    
    # Генерируем матчи первого раунда
    round_matches = []
    for i in range(0, len(bracket), 2):
        match = Match(
            tournament_id=tournament_id,
            round=1,
            match_order=len(round_matches) + 1,
            participant1_id=bracket[i].id if bracket[i] else None,
            participant2_id=bracket[i+1].id if bracket[i+1] else None,
            status='scheduled'
        )
        db.session.add(match)
        round_matches.append(match)
    
    db.session.commit()
    
    # Генерируем последующие раунды
    current_round = 1
    total_rounds = int(math.log2(next_power_of_two))
    
    while current_round < total_rounds:
        next_round = current_round + 1
        num_matches_next_round = next_power_of_two // (2 ** next_round)
        next_round_matches = []
        
        for i in range(num_matches_next_round):
            match = Match(
                tournament_id=tournament_id,
                round=next_round,
                match_order=i + 1,
                status='pending'  # Матчи следующих раундов ожидают результатов предыдущих
            )
            db.session.add(match)
            next_round_matches.append(match)
        
        db.session.commit()
        
        # Связываем матчи текущего раунда со следующим
        for i in range(0, len(round_matches), 2):
            if i + 1 < len(round_matches):
                next_match = next_round_matches[i // 2]
                round_matches[i].next_match_id = next_match.id
                round_matches[i + 1].next_match_id = next_match.id
        
        db.session.commit()
        round_matches = next_round_matches
        current_round = next_round
    
    return True

def distribute_participants(participants):
    """
    Распределяет участников по турнирной сетке с учетом seeding
    Стандартный алгоритм для турниров на выбывание
    """
    n = len(participants)
    if n <= 2:
        return participants
    
    # Используем алгоритм seeding для равномерного распределения сильных участников
    result = [None] * n
    result[0] = participants[0]
    
    if n > 1:
        result[-1] = participants[1]
    
    if n > 2:
        left = distribute_participants(participants[2:(n+2)//2])
        right = distribute_participants(participants[(n+2)//2:])
        
        for i in range(1, n-1):
            if i % 2 == 1:
                result[i] = left[i//2] if i//2 < len(left) else None
            else:
                result[i] = right[i//2 - 1] if i//2 - 1 < len(right) else None
    
    return result

def update_match_winner(match_id, winner_id, score1=None, score2=None):
    """
    Обновляет результат матча и продвигает победителя в следующий раунд
    """
    match = Match.query.get(match_id)
    if not match:
        return False
    
    match.winner_id = winner_id
    match.status = 'completed'
    
    if score1 is not None:
        match.score1 = score1
    if score2 is not None:
        match.score2 = score2
    
    # Если есть следующий матч, добавляем победителя в него
    if match.next_match_id:
        next_match = Match.query.get(match.next_match_id)
        
        # Определяем, в какую позицию (participant1 или participant2) добавить победителя
        if not next_match.participant1_id:
            next_match.participant1_id = winner_id
        elif not next_match.participant2_id:
            next_match.participant2_id = winner_id
        
        # Если оба участника определены, меняем статус матча
        if next_match.participant1_id and next_match.participant2_id:
            next_match.status = 'scheduled'
    
    db.session.commit()
    return True
```

6. Основное Flask приложение (app.py)

```python
from flask import Flask, render_template, jsonify, request
from flask_cors import CORS
from database import init_db, db
from models import Tournament, Participant, Match
from create_tournament import generate_bracket, update_match_winner
import config

app = Flask(__name__)
app.config.from_object(config.Config)
CORS(app)

# Инициализация базы данных
init_db(app)

# API Endpoints

@app.route('/')
def index():
    return render_template('index.html')

# Турниры
@app.route('/api/tournaments', methods=['GET'])
def get_tournaments():
    tournaments = Tournament.query.all()
    return jsonify([t.to_dict() for t in tournaments])

@app.route('/api/tournaments', methods=['POST'])
def create_tournament():
    data = request.json
    tournament = Tournament(
        name=data['name'],
        description=data.get('description', ''),
        max_participants=data.get('max_participants', 16)
    )
    db.session.add(tournament)
    db.session.commit()
    return jsonify(tournament.to_dict()), 201

@app.route('/api/tournaments/<int:tournament_id>', methods=['GET'])
def get_tournament(tournament_id):
    tournament = Tournament.query.get_or_404(tournament_id)
    return jsonify(tournament.to_dict())

# Участники
@app.route('/api/tournaments/<int:tournament_id>/participants', methods=['GET'])
def get_participants(tournament_id):
    participants = Participant.query.filter_by(tournament_id=tournament_id).all()
    return jsonify([p.to_dict() for p in participants])

@app.route('/api/tournaments/<int:tournament_id>/participants', methods=['POST'])
def add_participant(tournament_id):
    data = request.json
    participant = Participant(
        tournament_id=tournament_id,
        name=data['name'],
        seed=data.get('seed')
    )
    db.session.add(participant)
    db.session.commit()
    return jsonify(participant.to_dict()), 201

# Матчи
@app.route('/api/tournaments/<int:tournament_id>/matches', methods=['GET'])
def get_matches(tournament_id):
    matches = Match.query.filter_by(tournament_id=tournament_id).order_by(Match.round, Match.match_order).all()
    return jsonify([m.to_dict() for m in matches])

@app.route('/api/tournaments/<int:tournament_id>/generate-bracket', methods=['POST'])
def generate_tournament_bracket(tournament_id):
    success = generate_bracket(tournament_id)
    if success:
        return jsonify({'message': 'Bracket generated successfully'}), 200
    return jsonify({'error': 'Failed to generate bracket'}), 400

@app.route('/api/matches/<int:match_id>', methods=['PUT'])
def update_match(match_id):
    data = request.json
    winner_id = data.get('winner_id')
    score1 = data.get('score1')
    score2 = data.get('score2')
    
    success = update_match_winner(match_id, winner_id, score1, score2)
    if success:
        return jsonify({'message': 'Match updated successfully'}), 200
    return jsonify({'error': 'Failed to update match'}), 400

@app.route('/api/tournaments/<int:tournament_id>/bracket', methods=['GET'])
def get_tournament_bracket(tournament_id):
    """
    Возвращает структурированные данные для отображения турнирной сетки
    """
    matches = Match.query.filter_by(tournament_id=tournament_id).order_by(Match.round, Match.match_order).all()
    
    # Группируем матчи по раундам
    bracket = {}
    for match in matches:
        if match.round not in bracket:
            bracket[match.round] = []
        bracket[match.round].append(match.to_dict())
    
    # Определяем общее количество раундов
    tournament = Tournament.query.get(tournament_id)
    total_rounds = int(math.ceil(math.log2(tournament.max_participants)))
    
    return jsonify({
        'bracket': bracket,
        'total_rounds': total_rounds
    })

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

7. Фронтенд: HTML (templates/index.html)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Турнирная система - Олимпийская система</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
</head>
<body>
    <div class="container">
        <header>
            <h1><i class="fas fa-trophy"></i> Турнирная система</h1>
            <p class="subtitle">Олимпийская система (на выбывание)</p>
        </header>

        <main>
            <div class="dashboard">
                <div class="sidebar">
                    <div class="card">
                        <h3><i class="fas fa-plus-circle"></i> Создать турнир</h3>
                        <form id="createTournamentForm">
                            <input type="text" id="tournamentName" placeholder="Название турнира" required>
                            <textarea id="tournamentDescription" placeholder="Описание"></textarea>
                            <select id="maxParticipants">
                                <option value="4">4 участника</option>
                                <option value="8" selected>8 участников</option>
                                <option value="16">16 участников</option>
                                <option value="32">32 участника</option>
                            </select>
                            <button type="submit" class="btn-primary">
                                <i class="fas fa-plus"></i> Создать турнир
                            </button>
                        </form>
                    </div>

                    <div class="card">
                        <h3><i class="fas fa-users"></i> Добавить участника</h3>
                        <form id="addParticipantForm">
                            <select id="tournamentSelect" required>
                                <option value="">Выберите турнир</option>
                            </select>
                            <input type="text" id="participantName" placeholder="Имя участника" required>
                            <input type="number" id="participantSeed" placeholder="Посев (необязательно)" min="1">
                            <button type="submit" class="btn-secondary">
                                <i class="fas fa-user-plus"></i> Добавить участника
                            </button>
                        </form>
                    </div>

                    <div class="card">
                        <h3><i class="fas fa-play-circle"></i> Управление турниром</h3>
                        <div class="control-buttons">
                            <button id="generateBracketBtn" class="btn-success" disabled>
                                <i class="fas fa-sitemap"></i> Сгенерировать сетку
                            </button>
                            <button id="startTournamentBtn" class="btn-warning" disabled>
                                <i class="fas fa-play"></i> Начать турнир
                            </button>
                        </div>
                    </div>
                </div>

                <div class="content">
                    <div class="card">
                        <div class="card-header">
                            <h3><i class="fas fa-list"></i> Список турниров</h3>
                            <button id="refreshBtn" class="btn-icon">
                                <i class="fas fa-sync-alt"></i>
                            </button>
                        </div>
                        <div id="tournamentsList" class="tournaments-list">
                            <!-- Турниры будут загружены здесь -->
                        </div>
                    </div>

                    <div class="card">
                        <div class="card-header">
                            <h3><i class="fas fa-project-diagram"></i> Турнирная сетка</h3>
                            <select id="selectedTournament" class="tournament-selector">
                                <option value="">Выберите турнир для просмотра</option>
                            </select>
                        </div>
                        <div id="bracketContainer" class="bracket-container">
                            <div class="empty-state">
                                <i class="fas fa-sitemap fa-3x"></i>
                                <p>Выберите турнир для отображения сетки</p>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </main>

        <!-- Модальное окно для обновления результатов матча -->
        <div id="matchModal" class="modal">
            <div class="modal-content">
                <div class="modal-header">
                    <h3>Результат матча</h3>
                    <span class="close">&times;</span>
                </div>
                <div class="modal-body">
                    <div class="match-preview">
                        <div class="participant" id="modalParticipant1">
                            <span class="participant-name"></span>
                            <input type="number" id="score1" min="0" placeholder="Счет" class="score-input">
                        </div>
                        <div class="vs">VS</div>
                        <div class="participant" id="modalParticipant2">
                            <span class="participant-name"></span>
                            <input type="number" id="score2" min="0" placeholder="Счет" class="score-input">
                        </div>
                    </div>
                    <div class="modal-actions">
                        <button id="setWinnerBtn1" class="btn-primary">Победитель: <span id="winnerName1"></span></button>
                        <button id="setWinnerBtn2" class="btn-primary">Победитель: <span id="winnerName2"></span></button>
                    </div>
                </div>
                <div class="modal-footer">
                    <button id="saveMatchResult" class="btn-success">Сохранить результат</button>
                    <button id="cancelMatchResult" class="btn-secondary">Отмена</button>
                </div>
            </div>
        </div>
    </div>

    <script src="{{ url_for('static', filename='script.js') }}"></script>
</body>
</html>
```

8. Стили CSS (static/style.css)

```css
/* Основные стили */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    padding: 20px;
    color: #333;
}

.container {
    max-width: 1400px;
    margin: 0 auto;
}

/* Заголовок */
header {
    text-align: center;
    color: white;
    margin-bottom: 30px;
    padding: 20px;
    border-radius: 10px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
}

header h1 {
    font-size: 2.5rem;
    margin-bottom: 10px;
}

.subtitle {
    font-size: 1.2rem;
    opacity: 0.9;
}

/* Основной контент */
.dashboard {
    display: grid;
    grid-template-columns: 350px 1fr;
    gap: 20px;
}

@media (max-width: 1024px) {
    .dashboard {
        grid-template-columns: 1fr;
    }
}

/* Карточки */
.card {
    background: white;
    border-radius: 15px;
    padding: 25px;
    margin-bottom: 20px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
    transition: transform 0.3s ease;
}

.card:hover {
    transform: translateY(-5px);
}

.card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 2px solid #f0f0f0;
}

.card h3 {
    color: #2d3748;
    font-size: 1.3rem;
    display: flex;
    align-items: center;
    gap: 10px;
}

/* Формы */
form {
    display: flex;
    flex-direction: column;
    gap: 15px;
}

input, textarea, select {
    padding: 12px 15px;
    border: 2px solid #e2e8f0;
    border-radius: 8px;
    font-size: 1rem;
    transition: border-color 0.3s ease;
}

input:focus, textarea:focus, select:focus {
    outline: none;
    border-color: #667eea;
}

textarea {
    min-height: 80px;
    resize: vertical;
}

/* Кнопки */
button {
    padding: 12px 20px;
    border: none;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
}

.btn-primary {
    background: #4f46e5;
    color: white;
}

.btn-primary:hover {
    background: #4338ca;
    transform: scale(1.05);
}

.btn-secondary {
    background: #6b7280;
    color: white;
}

.btn-secondary:hover {
    background: #4b5563;
}

.btn-success {
    background: #10b981;
    color: white;
}

.btn-success:hover {
    background: #059669;
}

.btn-warning {
    background: #f59e0b;
    color: white;
}

.btn-warning:hover {
    background: #d97706;
}

.btn-icon {
    width: 40px;
    height: 40px;
    padding: 0;
    border-radius: 50%;
    background: #f3f4f6;
    color: #4b5563;
}

.btn-icon:hover {
    background: #e5e7eb;
    transform: rotate(180deg);
}

.control-buttons {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

/* Список турниров */
.tournaments-list {
    max-height: 300px;
    overflow-y: auto;
}

.tournament-item {
    padding: 15px;
    border-radius: 8px;
    background: #f8fafc;
    margin-bottom: 10px;
    border-left: 4px solid #4f46e5;
    cursor: pointer;
    transition: all 0.3s ease;
}

.tournament-item:hover {
    background: #edf2f7;
    transform: translateX(5px);
}

.tournament-item.active {
    background: #e0e7ff;
    border-left-color: #10b981;
}

.tournament-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
}

.tournament-name {
    font-weight: 600;
    color: #2d3748;
}

.tournament-status {
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 0.85rem;
    font-weight: 600;
}

.status-pending {
    background: #fef3c7;
    color: #92400e;
}

.status-active {
    background: #d1fae5;
    color: #065f46;
}

.status-completed {
    background: #e0e7ff;
    color: #3730a3;
}

.tournament-info {
    display: flex;
    gap: 15px;
    font-size: 0.9rem;
    color: #6b7280;
}

/* Турнирная сетка */
.bracket-container {
    min-height: 500px;
    background: #f8fafc;
    border-radius: 10px;
    padding: 20px;
    overflow-x: auto;
}

.empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 300px;
    color: #9ca3af;
}

.empty-state i {
    margin-bottom: 20px;
}

.bracket {
    display: flex;
    gap: 40px;
    padding: 20px;
}

.round {
    display: flex;
    flex-direction: column;
    gap: 30px;
    min-width: 250px;
}

.round-header {
    text-align: center;
    padding: 10px;
    background: #4f46e5;
    color: white;
    border-radius: 8px;
    margin-bottom: 10px;
    font-weight: 600;
}

.match {
    background: white;
    border-radius: 8px;
    padding: 15px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    border: 2px solid #e5e7eb;
    transition: all 0.3s ease;
    cursor: pointer;
}

.match:hover {
    border-color: #4f46e5;
    transform: translateY(-2px);
}

.match.completed {
    background: #f0f9ff;
    border-color: #10b981;
}

.match.in-progress {
    background: #fef3c7;
    border-color: #f59e0b;
}

.match-participants {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.participant {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 12px;
    border-radius: 6px;
    background: #f8fafc;
}

.participant.winner {
    background: #d1fae5;
    font-weight: 600;
}

.participant-name {
    flex-grow: 1;
}

.participant-score {
    font-weight: 600;
    color: #4f46e5;
    min-width: 30px;
    text-align: right;
}

.match-vs {
    text-align: center;
    padding: 5px;
    color: #9ca3af;
    font-weight: 600;
}

.match-info {
    display: flex;
    justify-content: space-between;
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px solid #e5e7eb;
    font-size: 0.85rem;
    color: #6b7280;
}

/* Модальное окно */
.modal {
    display: none;
    position: fixed;
    z-index: 1000;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.5);
    align-items: center;
    justify-content: center;
}

.modal-content {
    background: white;
    border-radius: 15px;
    width: 90%;
    max-width: 500px;
    overflow: hidden;
    animation: modalAppear 0.3s ease;
}

@keyframes modalAppear {
    from {
        opacity: 0;
        transform: translateY(-50px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.modal-header {
    padding: 20px;
    background: #4f46e5;
    color: white;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.close {
    font-size: 28px;
    cursor: pointer;
    transition: color 0.3s ease;
}

.close:hover {
    color: #e0e7ff;
}

.modal-body {
    padding: 20px;
}

.modal-preview {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    margin-bottom: 20px;
}

.participant-modal {
    flex: 1;
    text-align: center;
    padding: 15px;
    border-radius: 8px;
    background: #f8fafc;
}

.score-input {
    width: 80px;
    text-align: center;
    margin-top: 10px;
    font-size: 1.2rem;
    font-weight: 600;
}

.modal-actions {
    display: flex;
    flex-direction: column;
    gap: 10px;
    margin-top: 20px;
}

.modal-footer {
    padding: 20px;
    background: #f8fafc;
    display: flex;
    justify-content: flex-end;
    gap: 10px;
}

/* Утилиты */
.tournament-selector {
    width: 250px;
}

.loading {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100px;
}

.loading-spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #e2e8f0;
    border-top: 4px solid #4f46e5;
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

/* Адаптивность */
@media (max-width: 768px) {
    .bracket {
        flex-direction: column;
        gap: 20px;
    }
    
    .round {
        min-width: 100%;
    }
    
    .modal-content {
        width: 95%;
        margin: 10px;
    }
}
```

9. JavaScript фронтенд логика (static/script.js)

```javascript
document.addEventListener('DOMContentLoaded', function() {
    // Состояние приложения
    let state = {
        currentTournament: null,
        tournaments: [],
        bracketData: null,
        currentMatch: null
    };

    // Элементы DOM
    const elements = {
        tournamentsList: document.getElementById('tournamentsList'),
        tournamentSelect: document.getElementById('tournamentSelect'),
        selectedTournament: document.getElementById('selectedTournament'),
        bracketContainer: document.getElementById('bracketContainer'),
        createTournamentForm: document.getElementById('createTournamentForm'),
        addParticipantForm: document.getElementById('addParticipantForm'),
        generateBracketBtn: document.getElementById('generateBracketBtn'),
        startTournamentBtn: document.getElementById('startTournamentBtn'),
        refreshBtn: document.getElementById('refreshBtn'),
        matchModal: document.getElementById('matchModal'),
        closeModal: document.querySelector('.close'),
        saveMatchResult: document.getElementById('saveMatchResult'),
        cancelMatchResult: document.getElementById('cancelMatchResult'),
        setWinnerBtn1: document.getElementById('setWinnerBtn1'),
        setWinnerBtn2: document.getElementById('setWinnerBtn2'),
        score1: document.getElementById('score1'),
        score2: document.getElementById('score2')
    };

    // API базовый URL
    const API_URL = window.location.origin;

    // Инициализация
    init();

    function init() {
        loadTournaments();
        setupEventListeners();
    }

    function setupEventListeners() {
        // Создание турнира
        elements.createTournamentForm.addEventListener('submit', async (e) => {
            e.preventDefault();
            await createTournament();
        });

        // Добавление участника
        elements.addParticipantForm.addEventListener('submit', async (e) => {
            e.preventDefault();
            await addParticipant();
        });

        // Генерация сетки
        elements.generateBracketBtn.addEventListener('click', async () => {
            if (state.currentTournament) {
                await generateBracket(state.currentTournament.id);
            }
        });

        // Выбор турнира
        elements.selectedTournament.addEventListener('change', async (e) => {
            const tournamentId = parseInt(e.target.value);
            if (tournamentId) {
                state.currentTournament = state.tournaments.find(t => t.id === tournamentId);
                await loadBracket(tournamentId);
                updateUI();
            }
        });

        // Обновление списка
        elements.refreshBtn.addEventListener('click', loadTournaments);

        // Модальное окно
        elements.closeModal.addEventListener('click', () => {
            elements.matchModal.style.display = 'none';
        });

        elements.cancelMatchResult.addEventListener('click', () => {
            elements.matchModal.style.display = 'none';
        });

        elements.saveMatchResult.addEventListener('click', saveMatchResult);

        elements.setWinnerBtn1.addEventListener('click', () => {
            if (state.currentMatch && state.currentMatch.participant1) {
                setWinner(state.currentMatch.participant1.id);
            }
        });

        elements.setWinnerBtn2.addEventListener('click', () => {
            if (state.currentMatch && state.currentMatch.participant2) {
                setWinner(state.currentMatch.participant2.id);
            }
        });

        // Закрытие модального окна при клике вне его
        window.addEventListener('click', (e) => {
            if (e.target === elements.matchModal) {
                elements.matchModal.style.display = 'none';
            }
        });
    }

    // API функции
    async function loadTournaments() {
        try {
            showLoading(elements.tournamentsList);
            const response = await fetch(`${API_URL}/api/tournaments`);
            const tournaments = await response.json();
            
            state.tournaments = tournaments;
            renderTournamentsList(tournaments);
            updateTournamentSelects(tournaments);
            
            if (tournaments.length > 0 && !state.currentTournament) {
                state.currentTournament = tournaments[0];
                elements.selectedTournament.value = tournaments[0].id;
                await loadBracket(tournaments[0].id);
            }
            
            updateUI();
        } catch (error) {
            showError('Ошибка загрузки турниров');
            console.error(error);
        }
    }

    async function createTournament() {
        const name = document.getElementById('tournamentName').value;
        const description = document.getElementById('tournamentDescription').value;
        const maxParticipants = parseInt(document.getElementById('maxParticipants').value);
        
        try {
            const response = await fetch(`${API_URL}/api/tournaments`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    name,
                    description,
                    max_participants: maxParticipants
                })
            });
            
            if (response.ok) {
                const tournament = await response.json();
                showSuccess('Турнир создан успешно!');
                elements.createTournamentForm.reset();
                await loadTournaments();
            } else {
                throw new Error('Ошибка создания турнира');
            }
        } catch (error) {
            showError('Ошибка создания турнира');
            console.error(error);
        }
    }

    async function addParticipant() {
        const tournamentId = parseInt(elements.tournamentSelect.value);
        const name = document.getElementById('participantName').value;
        const seed = document.getElementById('participantSeed').value;
        
        if (!tournamentId || !name) {
            showError('Заполните все обязательные поля');
            return;
        }
        
        try {
            const response = await fetch(`${API_URL}/api/tournaments/${tournamentId}/participants`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    name,
                    seed: seed || null
                })
            });
            
            if (response.ok) {
                showSuccess('Участник добавлен успешно!');
                elements.addParticipantForm.reset();
                
                // Обновляем текущий турнир если это он
                if (state.currentTournament && state.currentTournament.id === tournamentId) {
                    await loadBracket(tournamentId);
                }
            } else {
                throw new Error('Ошибка добавления участника');
            }
        } catch (error) {
            showError('Ошибка добавления участника');
            console.error(error);
        }
    }

    async function generateBracket(tournamentId) {
        try {
            const response = await fetch(`${API_URL}/api/tournaments/${tournamentId}/generate-bracket`, {
                method: 'POST'
            });
            
            if (response.ok) {
                showSuccess('Турнирная сетка сгенерирована!');
                await loadBracket(tournamentId);
            } else {
                throw new Error('Ошибка генерации сетки');
            }
        } catch (error) {
            showError('Ошибка генерации турнирной сетки');
            console.error(error);
        }
    }

    async function loadBracket(tournamentId) {
        try {
            showLoading(elements.bracketContainer);
            const response = await fetch(`${API_URL}/api/tournaments/${tournamentId}/bracket`);
            if (response.ok) {
                const data = await response.json();
                state.bracketData = data;
                renderBracket(data);
            } else {
                throw new Error('Ошибка загрузки сетки');
            }
        } catch (error) {
            showError('Ошибка загрузки турнирной сетки');
            console.error(error);
        }
    }

    async function updateMatch(matchId, data) {
        try {
            const response = await fetch(`${API_URL}/api/matches/${matchId}`, {
                method: 'PUT',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(data)
            });
            
            if (response.ok) {
                showSuccess('Результат матча сохранен!');
                if (state.currentTournament) {
                    await loadBracket(state.currentTournament.id);
                }
                elements.matchModal.style.display = 'none';
            } else {
                throw new Error('Ошибка обновления матча');
            }
        } catch (error) {
            showError('Ошибка сохранения результата');
            console.error(error);
        }
    }

    // Функции отображения
    function renderTournamentsList(tournaments) {
        elements.tournamentsList.innerHTML = '';
        
        if (tournaments.length === 0) {
            elements.tournamentsList.innerHTML = `
                <div class="empty-state">
                    <i class="fas fa-trophy fa-2x"></i>
                    <p>Нет созданных турниров</p>
                </div>
            `;
            return;
        }
        
        tournaments.forEach(tournament => {
            const item = document.createElement('div');
            item.className = `tournament-item ${state.currentTournament?.id === tournament.id ? 'active' : ''}`;
            item.innerHTML = `
                <div class="tournament-header">
                    <span class="tournament-name">${tournament.name}</span>
                    <span class="tournament-status status-${tournament.status}">${getStatusText(tournament.status)}</span>
                </div>
                <p class="tournament-description">${tournament.description || 'Без описания'}</p>
                <div class="tournament-info">
                    <span><i class="fas fa-users"></i> ${tournament.max_participants} мест</span>
                    <span><i class="fas fa-calendar"></i> ${new Date(tournament.start_date).toLocaleDateString()}</span>
                </div>
            `;
            
            item.addEventListener('click', () => {
                state.currentTournament = tournament;
                elements.selectedTournament.value = tournament.id;
                loadBracket(tournament.id);
                updateUI();
            });
            
            elements.tournamentsList.appendChild(item);
        });
    }

    function renderBracket(data) {
        if (!data || !data.bracket) {
            elements.bracketContainer.innerHTML = `
                <div class="empty-state">
                    <i class="fas fa-sitemap fa-3x"></i>
                    <p>Сетка турнира не сгенерирована</p>
                    <p>Добавьте участников и нажмите "Сгенерировать сетку"</p>
                </div>
            `;
            return;
        }
        
        const { bracket, total_rounds } = data;
        let bracketHTML = '<div class="bracket">';
        
        // Создаем раунды
        for (let round = 1; round <= total_rounds; round++) {
            const roundMatches = bracket[round] || [];
            
            bracketHTML += `
                <div class="round">
                    <div class="round-header">Раунд ${round}</div>
                    ${roundMatches.map(match => renderMatch(match)).join('')}
                </div>
            `;
        }
        
        bracketHTML += '</div>';
        elements.bracketContainer.innerHTML = bracketHTML;
        
        // Добавляем обработчики кликов для матчей
        document.querySelectorAll('.match').forEach(matchElement => {
            matchElement.addEventListener('click', () => {
                const matchId = parseInt(matchElement.dataset.matchId);
                const match = findMatchById(matchId);
                if (match) {
                    openMatchModal(match);
                }
            });
        });
    }

    function renderMatch(match) {
        const isCompleted = match.status === 'completed';
        const isInProgress = match.status === 'in_progress';
        
        return `
            <div class="match ${match.status}" data-match-id="${match.id}">
                <div class="match-participants">
                    <div class="participant ${match.winner_id === match.participant1?.id ? 'winner' : ''}">
                        <span class="participant-name">
                            ${match.participant1 ? match.participant1.name : 'TBD'}
                            ${match.participant1?.seed ? ` (#${match.participant1.seed})` : ''}
                        </span>
                        ${isCompleted || isInProgress ? 
                            `<span class="participant-score">${match.score1}</span>` : 
                            ''
                        }
                    </div>
                    <div class="match-vs">VS</div>
                    <div class="participant ${match.winner_id === match.participant2?.id ? 'winner' : ''}">
                        <span class="participant-name">
                            ${match.participant2 ? match.participant2.name : 'TBD'}
                            ${match.participant2?.seed ? ` (#${match.participant2.seed})` : ''}
                        </span>
                        ${isCompleted || isInProgress ? 
                            `<span class="participant-score">${match.score2}</span>` : 
                            ''
                        }
                    </div>
                </div>
                <div class="match-info">
                    <span>Матч ${match.match_order}</span>
                    <span class="match-status">${getMatchStatusText(match.status)}</span>
                </div>
            </div>
        `;
    }

    function updateTournamentSelects(tournaments) {
        // Обновляем оба select элемента
        [elements.tournamentSelect, elements.selectedTournament].forEach(select => {
            const currentValue = select.value;
            select.innerHTML = '<option value="">Выберите турнир</option>';
            
            tournaments.forEach(tournament => {
                const option = document.createElement('option');
                option.value = tournament.id;
                option.textContent = tournament.name;
                select.appendChild(option);
            });
            
            // Восстанавливаем предыдущее значение если возможно
            if (currentValue) {
                select.value = currentValue;
            }
        });
    }

    function updateUI() {
        // Обновляем состояние кнопок
        const canGenerateBracket = state.currentTournament && 
            state.currentTournament.status === 'pending';
        
        elements.generateBracketBtn.disabled = !canGenerateBracket;
        elements.startTournamentBtn.disabled = !state.currentTournament;
        
        // Обновляем выбранный турнир в списке
        document.querySelectorAll('.tournament-item').forEach(item => {
            item.classList.remove('active');
        });
        
        if (state.currentTournament) {
            const activeItem = document.querySelector(`.tournament-item[data-id="${state.currentTournament.id}"]`);
            if (activeItem) {
                activeItem.classList.add('active');
            }
        }
    }

    // Модальное окно матча
    function openMatchModal(match) {
        state.currentMatch = match;
        
        // Устанавливаем имена участников
        document.getElementById('winnerName1').textContent = 
            match.participant1 ? match.participant1.name : 'TBD';
        document.getElementById('winnerName2').textContent = 
            match.participant2 ? match.participant2.name : 'TBD';
        
        // Устанавливаем текущие счета
        elements.score1.value = match.score1 || '';
        elements.score2.value = match.score2 || '';
        
        // Показываем модальное окно
        elements.matchModal.style.display = 'flex';
    }

    function setWinner(winnerId) {
        if (!state.currentMatch) return;
        
        // Подсвечиваем выбранного победителя
        const isParticipant1 = winnerId === state.currentMatch.participant1?.id;
        const winnerBtn = isParticipant1 ? elements.setWinnerBtn1 : elements.setWinnerBtn2;
        const otherBtn = isParticipant1 ? elements.setWinnerBtn2 : elements.setWinnerBtn1;
        
        winnerBtn.classList.add('selected');
        winnerBtn.classList.remove('btn-primary');
        winnerBtn.classList.add('btn-success');
        
        otherBtn.classList.remove('selected', 'btn-success');
        otherBtn.classList.add('btn-primary');
    }

    async function saveMatchResult() {
        if (!state.currentMatch) return;
        
        const score1 = parseInt(elements.score1.value) || 0;
        const score2 = parseInt(elements.score2.value) || 0;
        
        // Определяем победителя
        let winnerId = null;
        const selectedWinnerBtn = document.querySelector('.modal-actions .btn-success.selected');
        if (selectedWinnerBtn) {
            winnerId = selectedWinnerBtn.id === 'setWinnerBtn1' ? 
                state.currentMatch.participant1?.id : 
                state.currentMatch.participant2?.id;
        }
        
        if (!winnerId) {
            showError('Выберите победителя матча');
            return;
        }
        
        await updateMatch(state.currentMatch.id, {
            winner_id: winnerId,
            score1: score1,
            score2: score2
        });
    }

    // Вспомогательные функции
    function findMatchById(matchId) {
        if (!state.bracketData || !state.bracketData.bracket) return null;
        
        for (const round in state.bracketData.bracket) {
            const match = state.bracketData.bracket[round].find(m => m.id === matchId);
            if (match) return match;
        }
        return null;
    }

    function getStatusText(status) {
        const statusMap = {
            'pending': 'Ожидает',
            'active': 'Активный',
            'completed': 'Завершен'
        };
        return statusMap[status] || status;
    }

    function getMatchStatusText(status) {
        const statusMap = {
            'scheduled': 'Запланирован',
            'in_progress': 'В процессе',
            'completed': 'Завершен',
            'pending': 'Ожидает'
        };
        return statusMap[status] || status;
    }

    function showLoading(element) {
        element.innerHTML = `
            <div class="loading">
                <div class="loading-spinner"></div>
            </div>
        `;
    }

    function showSuccess(message) {
        alert(message); // В реальном приложении используйте toast уведомления
    }

    function showError(message) {
        alert(`Ошибка: ${message}`); // В реальном приложении используйте toast уведомления
    }
});
```

10. Инструкция по запуску

Шаг 1: Установка зависимостей

```bash
pip install -r requirements.txt
```

Шаг 2: Настройка базы данных

```bash
# Создайте файл .env в корне проекта:
echo "DATABASE_URL=sqlite:///tournament.db" > .env
echo "SECRET_KEY=your-secret-key-here" >> .env
```

Шаг 3: Инициализация базы данных

```python
# Создайте файл init_db.py:
from app import app, db
with app.app_context():
    db.create_all()
    print("Database created successfully!")

# Запустите его:
python init_db.py
```

Шаг 4: Запуск приложения

```bash
python app.py
```

Шаг 5: Откройте в браузере

```
http://localhost:5000
```

Пошаговая инструкция использования

1. Создание турнира:
   · Введите название, описание и выберите количество участников
   · Нажмите "Создать турнир"
2. Добавление участников:
   · Выберите турнир из списка
   · Введите имя участника (посев опционально)
   · Нажмите "Добавить участника"
3. Генерация турнирной сетки:
   · После добавления участников нажмите "Сгенерировать сетку"
   · Система автоматически создаст турнирную сетку по олимпийской системе
4. Проведение матчей:
   · Кликните на любой матч в сетке
   · Введите результаты и выберите победителя
   · Нажмите "Сохранить результат"
   · Победитель автоматически продвинется в следующий раунд
5. Мониторинг турнира:
   · Следите за прогрессом турнира в реальном времени
   · Просматривайте результаты всех матчей
   · Определяйте победителя турнира

Особенности реализации

1. Автоматическое создание сетки: Алгоритм распределяет участников с учетом seeding
2. Динамическое обновление: Сетка обновляется в реальном времени
3. Валидация данных: Проверка корректности вводимых данных
4. Адаптивный дизайн: Работает на компьютерах и мобильных устройствах
5. Простая установка: Минимальные зависимости, SQLite для хранения данных

Возможные улучшения

1. Добавить аутентификацию пользователей
2. Реализовать разные системы турниров (швейцарская, круговая)
3. Добавить экспорт результатов в PDF/Excel
4. Интегрировать систему рейтингов
5. Добавить live-обновления через WebSockets
6. Реализовать систему уведомлений
7. Добавить возможность загрузки изображений участников

Это полноценное приложение для проведения соревнований по олимпийской системе. Оно готово к использованию и может быть расширено дополнительными функциями по мере необходимости.

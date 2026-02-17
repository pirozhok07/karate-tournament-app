Добавим недостающие API-эндпоинты для работы с категориями и участниками.

Обновление app.py – добавляем новые маршруты

```python
# API: категории турнира
@app.route('/api/tournaments/<int:tournament_id>/categories', methods=['GET'])
def get_tournament_categories(tournament_id):
    categories = Category.query.filter_by(tournament_id=tournament_id).all()
    return jsonify([c.to_dict() for c in categories])

# API: создать категорию для турнира
@app.route('/api/categories', methods=['POST'])
def create_category():
    data = request.json
    category = Category(
        tournament_id=data['tournament_id'],
        name=data['name'],
        description=data.get('description', ''),
        weight_min=data.get('weight_min'),
        weight_max=data.get('weight_max'),
        age_min=data.get('age_min'),
        age_max=data.get('age_max'),
        gender=data.get('gender'),
        skill_level=data.get('skill_level')
    )
    db.session.add(category)
    db.session.commit()
    return jsonify(category.to_dict()), 201

# API: обновить категорию
@app.route('/api/categories/<int:category_id>', methods=['PUT'])
def update_category(category_id):
    category = Category.query.get_or_404(category_id)
    data = request.json
    for key, value in data.items():
        if hasattr(category, key):
            setattr(category, key, value)
    db.session.commit()
    return jsonify(category.to_dict())

# API: удалить категорию
@app.route('/api/categories/<int:category_id>', methods=['DELETE'])
def delete_category(category_id):
    category = Category.query.get_or_404(category_id)
    db.session.delete(category)
    db.session.commit()
    return jsonify({'message': 'Category deleted'})

# API: участники категории
@app.route('/api/categories/<int:category_id>/participants', methods=['GET'])
def get_category_participants(category_id):
    participants = Participant.query.filter_by(category_id=category_id).all()
    return jsonify([p.to_dict() for p in participants])

# API: количество участников в категории
@app.route('/api/categories/<int:category_id>/participants/count', methods=['GET'])
def get_category_participants_count(category_id):
    count = Participant.query.filter_by(category_id=category_id).count()
    return jsonify({'count': count})

# API: участники по категориям для турнира
@app.route('/api/tournaments/<int:tournament_id>/participants-by-category', methods=['GET'])
def get_participants_by_category(tournament_id):
    categories = Category.query.filter_by(tournament_id=tournament_id).all()
    result = []
    for cat in categories:
        participants = Participant.query.filter_by(category_id=cat.id).all()
        result.append({
            'category_id': cat.id,
            'category_name': cat.name,
            'participants': [p.to_dict() for p in participants]
        })
    return jsonify(result)

# API: все критерии турнира (для системы оценок)
@app.route('/api/tournaments/<int:tournament_id>/criteria', methods=['GET'])
def get_tournament_criteria(tournament_id):
    # Здесь можно реализовать хранение критериев, для демо вернём статику
    default_criteria = [
        {'name': 'Техника', 'max_value': 10, 'weight': 1.5, 'description': 'Чистота выполнения'},
        {'name': 'Артистизм', 'max_value': 10, 'weight': 1.0, 'description': 'Выразительность и музыкальность'},
        {'name': 'Сложность', 'max_value': 10, 'weight': 1.2, 'description': 'Сложность элементов'}
    ]
    return jsonify(default_criteria)

# API: оценки участника (для системы оценок)
@app.route('/api/participants/<int:participant_id>/scores', methods=['GET'])
def get_participant_scores(participant_id):
    scores = Score.query.filter_by(participant_id=participant_id).all()
    return jsonify([s.to_dict() for s in scores])

# API: детали результата
@app.route('/api/results/<int:result_id>/details', methods=['GET'])
def get_result_details(result_id):
    result = Result.query.get_or_404(result_id)
    participant = result.participant
    scores = Score.query.filter_by(participant_id=participant.id).all()
    match_results = []
    
    # Получить матчи, где участвовал спортсмен (для олимпийской системы)
    matches = Match.query.filter(
        db.or_(
            Match.participant1_id == participant.id,
            Match.participant2_id == participant.id
        )
    ).all()
    
    for match in matches:
        match_results.append({
            'id': match.id,
            'round': match.round,
            'participant1_name': match.p1_participant.athlete.full_name if match.p1_participant else None,
            'participant2_name': match.p2_participant.athlete.full_name if match.p2_participant else None,
            'participant1_id': match.participant1_id,
            'participant2_id': match.participant2_id,
            'score1': match.score1,
            'score2': match.score2,
            'winner_id': match.winner_id,
            'status': match.status
        })
    
    return jsonify({
        'id': result.id,
        'position': result.position,
        'final_score': result.final_score,
        'medal': result.medal,
        'prize_money': result.prize_money,
        'points_awarded': result.points_awarded,
        'participant': participant.to_dict(),
        'scores': [s.to_dict() for s in scores],
        'match_results': match_results
    })

# API: начать матч
@app.route('/api/matches/<int:match_id>/start', methods=['POST'])
def start_match(match_id):
    match = Match.query.get_or_404(match_id)
    match.status = 'in_progress'
    match.start_time = datetime.utcnow()
    db.session.commit()
    return jsonify({'status': 'started'})

# API: завершить матч
@app.route('/api/matches/<int:match_id>/complete', methods=['POST'])
def complete_match(match_id):
    match = Match.query.get_or_404(match_id)
    match.status = 'completed'
    match.end_time = datetime.utcnow()
    db.session.commit()
    # Проверить завершение турнира
    check_tournament_completion(match.tournament_id)
    return jsonify({'status': 'completed'})

# API: турнирная таблица (лидерборд)
@app.route('/api/tournaments/<int:tournament_id>/leaderboard', methods=['GET'])
def get_tournament_leaderboard(tournament_id):
    from scoring_system import generate_leaderboard
    leaderboard = generate_leaderboard(tournament_id)
    return jsonify(leaderboard)

# API: экспорт результатов (заглушка)
@app.route('/api/tournaments/<int:tournament_id>/leaderboard/export', methods=['GET'])
def export_leaderboard(tournament_id):
    format = request.args.get('format', 'pdf')
    # Здесь реальная генерация файла
    return jsonify({'message': f'Export to {format} not implemented'})

# API: пакетное сохранение оценок
@app.route('/api/scores/batch', methods=['POST'])
def save_scores_batch():
    data = request.json
    scores_data = data.get('scores', [])
    judge_name = data.get('judge_name', 'Judge')
    
    # Создать или найти судью (упрощённо)
    judge = Judge.query.filter_by(full_name=judge_name).first()
    if not judge:
        judge = Judge(first_name=judge_name, last_name='', full_name=judge_name)
        db.session.add(judge)
        db.session.commit()
    
    saved = []
    for s_data in scores_data:
        score = Score(
            participant_id=s_data['participant_id'],
            judge_id=judge.id,
            criterion=s_data['criterion'],
            value=s_data['value'],
            max_value=s_data.get('max_value', 10),
            weight=s_data.get('weight', 1.0)
        )
        db.session.add(score)
        saved.append(score)
    
    db.session.commit()
    
    # Пересчитать итоговый балл участника
    participant_id = scores_data[0]['participant_id']
    from scoring_system import calculate_final_scores
    final = calculate_final_scores(participant_id)
    
    participant = Participant.query.get(participant_id)
    participant.final_score = final
    db.session.commit()
    
    return jsonify({'saved': len(saved), 'final_score': final})
```

Дополнительные вспомогательные функции

Добавьте в app.py (или в create_bracket.py) функцию проверки завершения турнира, если её нет:

```python
def check_tournament_completion(tournament_id):
    """Проверяет, все ли матчи турнира завершены, и обновляет статус"""
    tournament = Tournament.query.get(tournament_id)
    if not tournament:
        return
    
    # Проверяем, есть ли незавершённые матчи
    incomplete = Match.query.filter_by(tournament_id=tournament_id).filter(
        Match.status.in_(['scheduled', 'in_progress'])
    ).count()
    
    if incomplete == 0:
        tournament.status = 'completed'
        tournament.end_date = datetime.utcnow()
        db.session.commit()
```

Обновление модели Judge

Если у вас ещё нет модели Judge, добавьте в models.py:

```python
class Judge(db.Model):
    __tablename__ = 'judges'
    
    id = db.Column(db.Integer, primary_key=True)
    first_name = db.Column(db.String(50))
    last_name = db.Column(db.String(50))
    full_name = db.Column(db.String(100))
    license_number = db.Column(db.String(50))
    country = db.Column(db.String(50))
    qualification = db.Column(db.String(100))
    role = db.Column(db.String(50))
    status = db.Column(db.String(20), default='active')
    
    def to_dict(self):
        return {
            'id': self.id,
            'full_name': self.full_name,
            'license_number': self.license_number,
            'country': self.country,
            'qualification': self.qualification,
            'role': self.role
        }
```

Обновление модели Score (если нужно добавить judge_name)

```python
class Score(db.Model):
    __tablename__ = 'scores'
    
    id = db.Column(db.Integer, primary_key=True)
    participant_id = db.Column(db.Integer, db.ForeignKey('participants.id'), nullable=False)
    judge_id = db.Column(db.Integer, db.ForeignKey('judges.id'))
    judge_name = db.Column(db.String(100))  # для обратной совместимости
    criterion = db.Column(db.String(100))
    value = db.Column(db.Float, nullable=False)
    max_value = db.Column(db.Float, default=10.0)
    weight = db.Column(db.Float, default=1.0)
    round_number = db.Column(db.Integer, default=1)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def to_dict(self):
        return {
            'id': self.id,
            'participant_id': self.participant_id,
            'judge_id': self.judge_id,
            'judge_name': self.judge.full_name if self.judge else self.judge_name,
            'criterion': self.criterion,
            'value': self.value,
            'max_value': self.max_value,
            'weight': self.weight,
            'round_number': self.round_number,
            'created_at': self.created_at.isoformat()
        }
```

Проверка API

После добавления всех эндпоинтов перезапустите приложение и проверьте работу страниц:

· Управление категориями
· Система оценок
· Детальная информация о результатах

Все API теперь должны отвечать корректно.

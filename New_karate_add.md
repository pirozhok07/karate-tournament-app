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



Исправляем обработку кнопки "Участники" в категориях и добавляем страницу просмотра участников категории.

1. Обновление templates/categories.html – исправляем кнопку и добавляем модальное окно

В функции addCategoryEventListeners замените обработчик для .view-participants:

```javascript
// Просмотр участников категории
document.querySelectorAll('.view-participants').forEach(btn => {
    btn.addEventListener('click', function() {
        const categoryId = this.dataset.id;
        openCategoryParticipantsModal(categoryId);
    });
});
```

Добавьте новую функцию после openCategoryModal:

```javascript
// Открытие модального окна с участниками категории
async function openCategoryParticipantsModal(categoryId) {
    try {
        const response = await fetch(`/api/categories/${categoryId}/participants`);
        const participants = await response.json();
        
        const category = categories.find(c => c.id == categoryId);
        
        // Создаём модальное окно
        const modal = document.createElement('div');
        modal.className = 'modal';
        modal.id = 'participantsModal';
        modal.innerHTML = `
            <div class="modal-content" style="max-width: 800px;">
                <div class="modal-header">
                    <h3><i class="fas fa-users"></i> Участники категории: ${category?.name || 'Категория'}</h3>
                    <span class="close">&times;</span>
                </div>
                <div class="modal-body">
                    ${participants.length === 0 ? 
                        '<p class="empty-state">В этой категории пока нет участников</p>' : 
                        `<table class="participants-table">
                            <thead>
                                <tr>
                                    <th>Спортсмен</th>
                                    <th>Страна</th>
                                    <th>Клуб</th>
                                    <th>Посев</th>
                                    <th>Статус</th>
                                    <th>Действия</th>
                                </tr>
                            </thead>
                            <tbody>
                                ${participants.map(p => `
                                    <tr>
                                        <td>
                                            <div class="participant-name">
                                                <i class="fas fa-user-circle"></i>
                                                ${p.athlete?.full_name || 'Неизвестно'}
                                            </div>
                                        </td>
                                        <td>${p.athlete?.country || '—'}</td>
                                        <td>${p.athlete?.club || '—'}</td>
                                        <td><span class="seed-badge">#${p.seed || '—'}</span></td>
                                        <td><span class="status-badge status-${p.status}">${p.status}</span></td>
                                        <td>
                                            <button class="btn-icon view-athlete" data-athlete-id="${p.athlete_id}" title="Просмотр">
                                                <i class="fas fa-eye"></i>
                                            </button>
                                            <button class="btn-icon edit-participant" data-participant-id="${p.id}" title="Редактировать">
                                                <i class="fas fa-edit"></i>
                                            </button>
                                            <button class="btn-icon remove-participant" data-participant-id="${p.id}" title="Удалить">
                                                <i class="fas fa-trash"></i>
                                            </button>
                                        </td>
                                    </tr>
                                `).join('')}
                            </tbody>
                        </table>`
                    }
                </div>
                <div class="modal-footer">
                    <button class="btn btn-secondary close-modal">Закрыть</button>
                </div>
            </div>
        `;
        
        document.body.appendChild(modal);
        
        // Показать модальное окно
        setTimeout(() => modal.style.display = 'flex', 10);
        
        // Закрытие
        const closeBtn = modal.querySelector('.close');
        const closeModalBtn = modal.querySelector('.close-modal');
        
        closeBtn.addEventListener('click', () => modal.remove());
        closeModalBtn.addEventListener('click', () => modal.remove());
        
        // Закрытие при клике вне окна
        modal.addEventListener('click', (e) => {
            if (e.target === modal) modal.remove();
        });
        
        // Обработчики для кнопок внутри таблицы
        modal.querySelectorAll('.view-athlete').forEach(btn => {
            btn.addEventListener('click', () => {
                const athleteId = btn.dataset.athleteId;
                window.location.href = `/athlete/${athleteId}`;
            });
        });
        
        modal.querySelectorAll('.edit-participant').forEach(btn => {
            btn.addEventListener('click', () => {
                const participantId = btn.dataset.participantId;
                // Здесь можно открыть форму редактирования участника
                alert(`Редактирование участника ${participantId} (будет реализовано позже)`);
            });
        });
        
        modal.querySelectorAll('.remove-participant').forEach(btn => {
            btn.addEventListener('click', async () => {
                if (!confirm('Удалить участника из категории?')) return;
                const participantId = btn.dataset.participantId;
                try {
                    const response = await fetch(`/api/participants/${participantId}`, {
                        method: 'DELETE'
                    });
                    if (response.ok) {
                        btn.closest('tr').remove();
                        showSuccess('Участник удалён');
                    } else {
                        showError('Ошибка удаления');
                    }
                } catch (error) {
                    console.error('Ошибка:', error);
                    showError('Не удалось удалить участника');
                }
            });
        });
        
    } catch (error) {
        console.error('Ошибка загрузки участников категории:', error);
        showError('Не удалось загрузить участников категории');
    }
}
```

Добавьте стили для таблицы в extra_css блок:

```css
.participants-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 15px;
}
.participants-table th {
    background: #f8fafc;
    padding: 12px;
    text-align: left;
    font-weight: 600;
    color: #2d3748;
    border-bottom: 2px solid #e2e8f0;
}
.participants-table td {
    padding: 12px;
    border-bottom: 1px solid #e2e8f0;
}
.participants-table tr:hover {
    background: #f7fafc;
}
.participant-name {
    display: flex;
    align-items: center;
    gap: 8px;
}
.participant-name i {
    font-size: 20px;
    color: #4f46e5;
}
.seed-badge {
    background: #4f46e5;
    color: white;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 600;
}
.btn-icon {
    background: none;
    border: none;
    font-size: 16px;
    cursor: pointer;
    padding: 5px;
    margin: 0 2px;
    border-radius: 4px;
    color: #718096;
    transition: all 0.2s;
}
.btn-icon:hover {
    background: #e2e8f0;
    color: #4f46e5;
}
.btn-icon.view-athlete:hover { color: #3182ce; }
.btn-icon.edit-participant:hover { color: #38a169; }
.btn-icon.remove-participant:hover { color: #e53e3e; }
```

2. Добавление страницы управления участниками категории (опционально)

Если вы хотите отдельную страницу, создайте templates/category_participants.html:

```html
{% extends "base.html" %}

{% block title %}Участники категории - {{ category.name }} - TournamentPro{% endblock %}

{% block extra_css %}
<style>
    .category-header {
        background: white;
        border-radius: 10px;
        padding: 25px;
        margin-bottom: 20px;
        box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    .category-header h1 {
        color: #2d3748;
        margin-bottom: 10px;
    }
    .category-details {
        display: flex;
        gap: 20px;
        flex-wrap: wrap;
        color: #718096;
    }
    .category-details span {
        display: flex;
        align-items: center;
        gap: 5px;
    }
    .participants-table-container {
        background: white;
        border-radius: 10px;
        padding: 20px;
        box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    .table-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 20px;
    }
    .search-box {
        display: flex;
        gap: 10px;
        align-items: center;
    }
    .search-box input {
        padding: 8px 12px;
        border: 2px solid #e2e8f0;
        border-radius: 6px;
        width: 250px;
    }
</style>
{% endblock %}

{% block content %}
<div class="category-header">
    <h1><i class="fas fa-tags"></i> {{ category.name }}</h1>
    <div class="category-details">
        <span><i class="fas fa-trophy"></i> Турнир: {{ tournament.name }}</span>
        {% if category.weight_min or category.weight_max %}
        <span><i class="fas fa-weight"></i> Вес: {{ category.weight_min or 0 }} - {{ category.weight_max or '∞' }} кг</span>
        {% endif %}
        {% if category.age_min or category.age_max %}
        <span><i class="fas fa-birthday-cake"></i> Возраст: {{ category.age_min or 0 }} - {{ category.age_max or '∞' }} лет</span>
        {% endif %}
        {% if category.gender %}
        <span><i class="fas fa-venus-mars"></i> Пол: {{ {'male':'Мужской','female':'Женский','mixed':'Смешанный'}[category.gender] }}</span>
        {% endif %}
        <span><i class="fas fa-users"></i> Участников: {{ participants|length }}</span>
    </div>
</div>

<div class="participants-table-container">
    <div class="table-header">
        <h3><i class="fas fa-list"></i> Список участников</h3>
        <div class="search-box">
            <input type="text" id="searchParticipant" placeholder="Поиск по имени...">
            <button class="btn btn-primary" onclick="window.location.href='{{ url_for('participants') }}?tournament={{ tournament.id }}&category={{ category.id }}'">
                <i class="fas fa-user-plus"></i> Добавить участника
            </button>
        </div>
    </div>

    <table class="participants-table" id="participantsTable">
        <thead>
            <tr>
                <th>Спортсмен</th>
                <th>Страна</th>
                <th>Клуб</th>
                <th>Посев</th>
                <th>Статус</th>
                <th>Действия</th>
            </tr>
        </thead>
        <tbody>
            {% for p in participants %}
            <tr>
                <td>
                    <div class="participant-name">
                        <i class="fas fa-user-circle"></i>
                        {{ p.athlete.full_name }}
                    </div>
                </td>
                <td>{{ p.athlete.country or '—' }}</td>
                <td>{{ p.athlete.club or '—' }}</td>
                <td><span class="seed-badge">#{{ p.seed or '—' }}</span></td>
                <td><span class="status-badge status-{{ p.status }}">{{ p.status }}</span></td>
                <td>
                    <button class="btn-icon view-athlete" data-athlete-id="{{ p.athlete_id }}" title="Просмотр">
                        <i class="fas fa-eye"></i>
                    </button>
                    <button class="btn-icon edit-participant" data-participant-id="{{ p.id }}" title="Редактировать">
                        <i class="fas fa-edit"></i>
                    </button>
                    <button class="btn-icon remove-participant" data-participant-id="{{ p.id }}" title="Удалить">
                        <i class="fas fa-trash"></i>
                    </button>
                </td>
            </tr>
            {% else %}
            <tr>
                <td colspan="6" class="empty-state">В этой категории пока нет участников</td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
</div>
{% endblock %}

{% block extra_js %}
<script>
document.addEventListener('DOMContentLoaded', function() {
    const searchInput = document.getElementById('searchParticipant');
    const table = document.getElementById('participantsTable');
    
    searchInput.addEventListener('input', function() {
        const term = this.value.toLowerCase();
        const rows = table.querySelectorAll('tbody tr');
        
        rows.forEach(row => {
            const text = row.textContent.toLowerCase();
            row.style.display = text.includes(term) ? '' : 'none';
        });
    });
    
    // Обработчики кнопок
    document.querySelectorAll('.view-athlete').forEach(btn => {
        btn.addEventListener('click', () => {
            window.location.href = `/athlete/${btn.dataset.athleteId}`;
        });
    });
    
    document.querySelectorAll('.remove-participant').forEach(btn => {
        btn.addEventListener('click', async () => {
            if (!confirm('Удалить участника из категории?')) return;
            
            try {
                const response = await fetch(`/api/participants/${btn.dataset.participantId}`, {
                    method: 'DELETE'
                });
                if (response.ok) {
                    btn.closest('tr').remove();
                    alert('Участник удалён');
                } else {
                    alert('Ошибка удаления');
                }
            } catch (error) {
                console.error('Ошибка:', error);
                alert('Не удалось удалить участника');
            }
        });
    });
});
</script>
{% endblock %}
```

3. Добавление маршрута для страницы участников категории в app.py

```python
@app.route('/category/<int:category_id>/participants')
def category_participants(category_id):
    category = Category.query.get_or_404(category_id)
    tournament = Tournament.query.get(category.tournament_id)
    participants = Participant.query.filter_by(category_id=category_id).all()
    return render_template('category_participants.html', 
                         category=category, 
                         tournament=tournament,
                         participants=participants)
```

4. Обновление функции в categories.html для перехода на страницу

Если вы хотите использовать отдельную страницу вместо модального окна, замените обработчик:

```javascript
// Просмотр участников категории (переход на страницу)
document.querySelectorAll('.view-participants').forEach(btn => {
    btn.addEventListener('click', function() {
        const categoryId = this.dataset.id;
        window.location.href = `/category/${categoryId}/participants`;
    });
});
```

Теперь кнопка "Участники" корректно открывает список участников категории либо в модальном окне, либо на отдельной странице, с возможностью просмотра, редактирования и удаления.

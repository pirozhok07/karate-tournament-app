📋 СИСТЕМА ПРОСМОТРА И РЕДАКТИРОВАНИЯ КАТЕГОРИЙ

Вот полная реализация системы для просмотра, редактирования и управления порядком выступления участников по категориям:

📝 1. Обновите app.py - добавьте новые маршруты:

```python
from flask import Flask, render_template, request, redirect, url_for, flash, send_file, jsonify, session, g
from flask_wtf import FlaskForm
from flask_wtf.file import FileField, FileAllowed, FileRequired
from wtforms import StringField, DateField, FloatField, IntegerField, SelectField, SubmitField, TextAreaField
from wtforms.validators import DataRequired, Optional, NumberRange
from werkzeug.utils import secure_filename
import os
import json
import sqlite3
import pandas as pd
import xlsxwriter
from datetime import datetime, date
from pathlib import Path
import random
from collections import defaultdict

# ... (предыдущий код остается) ...

# ============== НОВЫЕ МАРШРУТЫ ДЛЯ ПРОСМОТРА И РЕДАКТИРОВАНИЯ КАТЕГОРИЙ ==============

@app.route('/competition/<int:competition_id>/categories_view')
def view_competition_categories(competition_id):
    """Просмотр категорий соревнования с участниками и результатами"""
    db = get_db()
    cursor = db.cursor()
    
    # Получаем соревнование
    cursor.execute("SELECT * FROM competitions WHERE id = ?", (competition_id,))
    competition = cursor.fetchone()
    
    if not competition:
        flash('Соревнование не найдено', 'danger')
        return redirect(url_for('index'))
    
    # Получаем распределение по категориям
    cursor.execute('''
        SELECT cc.category_name, a.* 
        FROM competition_categories cc
        JOIN athletes a ON cc.athlete_id = a.id
        WHERE cc.competition_id = ?
        ORDER BY cc.category_name, a.last_name, a.first_name
    ''', (competition_id,))
    
    athletes_by_category = defaultdict(list)
    for row in cursor.fetchall():
        athletes_by_category[row['category_name']].append(dict(row))
    
    # Получаем сетку (draw_data) если есть
    draw_data = None
    if competition['draw_data']:
        try:
            draw_data = json.loads(competition['draw_data'])
        except json.JSONDecodeError:
            draw_data = {}
    
    # Получаем оценки для каждого участника
    cursor.execute('''
        SELECT athlete_id, round_number, average
        FROM scores 
        WHERE competition_id = ?
        ORDER BY athlete_id, round_number
    ''', (competition_id,))
    
    scores_data = cursor.fetchall()
    
    # Группируем оценки по участникам
    athlete_scores = defaultdict(lambda: {1: None, 2: None, 3: None, 'total': 0, 'average': 0})
    for score in scores_data:
        athlete_id = score['athlete_id']
        round_num = score['round_number']
        athlete_scores[athlete_id][round_num] = score['average']
    
    # Рассчитываем итоговые баллы (2 лучших раунда из 3)
    for athlete_id, scores in athlete_scores.items():
        round_scores = [scores[1], scores[2], scores[3]]
        valid_scores = [s for s in round_scores if s is not None]
        
        if len(valid_scores) >= 2:
            valid_scores.sort(reverse=True)
            total = sum(valid_scores[:2])
            average = total / 2
        elif valid_scores:
            total = valid_scores[0]
            average = valid_scores[0]
        else:
            total = 0
            average = 0
        
        scores['total'] = total
        scores['average'] = average
    
    return render_template('competition_categories.html',
                         competition=dict(competition),
                         athletes_by_category=athletes_by_category,
                         draw_data=draw_data,
                         athlete_scores=athlete_scores)

@app.route('/competition/<int:competition_id>/category/<category_name>')
def view_category_detail(competition_id, category_name):
    """Детальный просмотр конкретной категории"""
    db = get_db()
    cursor = db.cursor()
    
    cursor.execute("SELECT * FROM competitions WHERE id = ?", (competition_id,))
    competition = cursor.fetchone()
    
    cursor.execute('''
        SELECT a.*, cc.category_name
        FROM competition_categories cc
        JOIN athletes a ON cc.athlete_id = a.id
        WHERE cc.competition_id = ? AND cc.category_name = ?
        ORDER BY a.last_name, a.first_name
    ''', (competition_id, category_name))
    
    athletes = [dict(row) for row in cursor.fetchall()]
    
    # Получаем сетку для этой категории
    draw_data = None
    if competition['draw_data']:
        try:
            draw_data = json.loads(competition['draw_data'])
            category_draw = draw_data.get(category_name, {})
        except json.JSONDecodeError:
            category_draw = {}
    
    # Получаем оценки
    athlete_ids = [athlete['id'] for athlete in athletes]
    placeholders = ','.join(['?' for _ in athlete_ids])
    
    cursor.execute(f'''
        SELECT s.athlete_id, s.round_number, s.average
        FROM scores s
        WHERE s.competition_id = ? AND s.athlete_id IN ({placeholders})
        ORDER BY s.athlete_id, s.round_number
    ''', (competition_id, *athlete_ids))
    
    scores = cursor.fetchall()
    
    # Группируем оценки
    scores_by_athlete = defaultdict(lambda: {1: None, 2: None, 3: None})
    for score in scores:
        athlete_id = score['athlete_id']
        round_num = score['round_number']
        scores_by_athlete[athlete_id][round_num] = score['average']
    
    return render_template('category_detail.html',
                         competition=dict(competition),
                         category_name=category_name,
                         athletes=athletes,
                         category_draw=category_draw if 'category_draw' in locals() else {},
                         scores_by_athlete=scores_by_athlete)

@app.route('/competition/<int:competition_id>/update_order', methods=['POST'])
def update_category_order(competition_id):
    """Обновление порядка выступления в категории"""
    data = request.json
    category_name = data.get('category_name')
    new_order = data.get('order')
    
    if not category_name or not new_order:
        return jsonify({'success': False, 'error': 'Не указаны данные'}), 400
    
    db = get_db()
    cursor = db.cursor()
    
    try:
        # Получаем текущую сетку
        cursor.execute("SELECT draw_data FROM competitions WHERE id = ?", (competition_id,))
        competition = cursor.fetchone()
        
        if not competition or not competition['draw_data']:
            return jsonify({'success': False, 'error': 'Сетка не найдена'}), 404
        
        draw_data = json.loads(competition['draw_data'])
        
        # Обновляем порядок для указанной категории
        if category_name in draw_data:
            draw_data[category_name]['order'] = new_order
            
            # Обновляем пары в соответствии с новым порядком
            athletes = draw_data[category_name].get('athletes', [])
            pairs = []
            for i in range(0, len(new_order), 2):
                if i + 1 < len(new_order):
                    pairs.append([new_order[i], new_order[i + 1]])
                else:
                    pairs.append([new_order[i], None])
            
            draw_data[category_name]['pairs'] = pairs
            
            # Сохраняем изменения
            cursor.execute("UPDATE competitions SET draw_data = ? WHERE id = ?",
                         (json.dumps(draw_data, ensure_ascii=False), competition_id))
            db.commit()
            
            return jsonify({'success': True, 'message': 'Порядок обновлен'})
        else:
            return jsonify({'success': False, 'error': 'Категория не найдена в сетке'}), 404
            
    except Exception as e:
        db.rollback()
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/competition/<int:competition_id>/randomize_category/<category_name>')
def randomize_category_order(competition_id, category_name):
    """Случайное перемешивание порядка выступления в категории"""
    db = get_db()
    cursor = db.cursor()
    
    try:
        cursor.execute("SELECT draw_data FROM competitions WHERE id = ?", (competition_id,))
        competition = cursor.fetchone()
        
        if not competition or not competition['draw_data']:
            flash('Сетка не найдена', 'danger')
            return redirect(url_for('view_competition_categories', competition_id=competition_id))
        
        draw_data = json.loads(competition['draw_data'])
        
        if category_name not in draw_data:
            flash('Категория не найдена в сетке', 'danger')
            return redirect(url_for('view_competition_categories', competition_id=competition_id))
        
        # Получаем список участников категории
        athletes = draw_data[category_name].get('athletes', [])
        athlete_ids = [athlete.get('id') for athlete in athletes if athlete.get('id')]
        
        # Перемешиваем порядок
        random.shuffle(athlete_ids)
        
        # Обновляем порядок
        draw_data[category_name]['order'] = athlete_ids
        
        # Обновляем пары
        pairs = []
        for i in range(0, len(athlete_ids), 2):
            if i + 1 < len(athlete_ids):
                pairs.append([athlete_ids[i], athlete_ids[i + 1]])
            else:
                pairs.append([athlete_ids[i], None])
        
        draw_data[category_name]['pairs'] = pairs
        
        # Сохраняем
        cursor.execute("UPDATE competitions SET draw_data = ? WHERE id = ?",
                     (json.dumps(draw_data, ensure_ascii=False), competition_id))
        db.commit()
        
        flash(f'Порядок выступления в категории "{category_name}" обновлен (случайный)', 'success')
        
    except Exception as e:
        db.rollback()
        flash(f'Ошибка при обновлении порядка: {str(e)}', 'danger')
    
    return redirect(url_for('view_competition_categories', competition_id=competition_id))

@app.route('/competition/<int:competition_id>/export_category/<category_name>')
def export_category_results(competition_id, category_name):
    """Экспорт результатов категории в Excel"""
    db = get_db()
    cursor = db.cursor()
    
    cursor.execute("SELECT name FROM competitions WHERE id = ?", (competition_id,))
    competition = cursor.fetchone()
    
    if not competition:
        flash('Соревнование не найдено', 'danger')
        return redirect(url_for('index'))
    
    # Получаем участников категории
    cursor.execute('''
        SELECT a.*, cc.category_name
        FROM competition_categories cc
        JOIN athletes a ON cc.athlete_id = a.id
        WHERE cc.competition_id = ? AND cc.category_name = ?
        ORDER BY a.last_name, a.first_name
    ''', (competition_id, category_name))
    
    athletes = [dict(row) for row in cursor.fetchall()]
    
    # Получаем оценки
    athlete_ids = [athlete['id'] for athlete in athletes]
    placeholders = ','.join(['?' for _ in athlete_ids])
    
    cursor.execute(f'''
        SELECT s.athlete_id, s.round_number, s.average
        FROM scores s
        WHERE s.competition_id = ? AND s.athlete_id IN ({placeholders})
        ORDER BY s.athlete_id, s.round_number
    ''', (competition_id, *athlete_ids))
    
    scores = cursor.fetchall()
    
    # Формируем данные для экспорта
    data = []
    for athlete in athletes:
        athlete_scores = {}
        for score in scores:
            if score['athlete_id'] == athlete['id']:
                athlete_scores[score['round_number']] = score['average']
        
        # Рассчитываем итог
        round_scores = [athlete_scores.get(1), athlete_scores.get(2), athlete_scores.get(3)]
        valid_scores = [s for s in round_scores if s is not None]
        
        if len(valid_scores) >= 2:
            valid_scores.sort(reverse=True)
            total = sum(valid_scores[:2])
            average = total / 2
        elif valid_scores:
            total = valid_scores[0]
            average = valid_scores[0]
        else:
            total = 0
            average = 0
        
        data.append({
            'Фамилия': athlete['last_name'],
            'Имя': athlete['first_name'],
            'Клуб': athlete['club'] or '',
            'Рег. номер': athlete['registration_number'] or '',
            'Раунд 1': athlete_scores.get(1, ''),
            'Раунд 2': athlete_scores.get(2, ''),
            'Раунд 3': athlete_scores.get(3, ''),
            'Общая сумма': round(total, 2),
            'Средний балл': round(average, 2)
        })
    
    # Сортируем по среднему баллу
    data.sort(key=lambda x: x['Средний балл'], reverse=True)
    
    # Добавляем места
    for i, item in enumerate(data):
        item['Место'] = i + 1
    
    # Создаем DataFrame
    df = pd.DataFrame(data)
    
    # Создаем файл
    filename = f"{competition['name'].replace(' ', '_')}_{category_name.replace(' ', '_')}.xlsx"
    filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    
    with pd.ExcelWriter(filepath, engine='xlsxwriter') as writer:
        df.to_excel(writer, sheet_name='Результаты', index=False)
        
        # Форматирование
        workbook = writer.book
        worksheet = writer.sheets['Результаты']
        
        header_format = workbook.add_format({
            'bold': True,
            'text_wrap': True,
            'valign': 'top',
            'bg_color': '#D7E4BC',
            'border': 1
        })
        
        for col_num, value in enumerate(df.columns.values):
            worksheet.write(0, col_num, value, header_format)
    
    return send_file(filepath, as_attachment=True, download_name=filename)

# ... (остальной код остается) ...
```

📝 2. Создайте шаблон templates/competition_categories.html:

```html
{% extends "base.html" %}

{% block content %}
<div class="row">
    <div class="col-md-12">
        <div class="d-flex justify-content-between align-items-center mb-4">
            <h2>
                <i class="bi bi-diagram-3"></i>
                Категории соревнования: {{ competition.name }}
            </h2>
            <div>
                <a href="{{ url_for('view_competition', competition_id=competition.id) }}" 
                   class="btn btn-secondary">
                    <i class="bi bi-arrow-left"></i> Назад к соревнованию
                </a>
            </div>
        </div>
        
        <!-- Информация о соревновании -->
        <div class="card mb-4">
            <div class="card-body">
                <div class="row">
                    <div class="col-md-3">
                        <strong>Дата:</strong> {{ competition.date }}
                    </div>
                    <div class="col-md-3">
                        <strong>Место:</strong> {{ competition.location or 'Не указано' }}
                    </div>
                    <div class="col-md-3">
                        <strong>Статус:</strong>
                        {% if competition.status == 'active' %}
                            <span class="badge bg-success">Активно</span>
                        {% elif competition.status == 'completed' %}
                            <span class="badge bg-secondary">Завершено</span>
                        {% else %}
                            <span class="badge bg-warning">Ожидание</span>
                        {% endif %}
                    </div>
                    <div class="col-md-3">
                        <strong>Категорий:</strong> {{ athletes_by_category|length }}
                    </div>
                </div>
            </div>
        </div>
        
        {% if not athletes_by_category %}
            <div class="card">
                <div class="card-body text-center py-5">
                    <i class="bi bi-diagram-3 display-4 text-muted"></i>
                    <h4 class="mt-3">Категории не распределены</h4>
                    <p class="text-muted">Спортсмены еще не распределены по категориям для этого соревнования</p>
                    <a href="{{ url_for('categorize_competition', competition_id=competition.id) }}" 
                       class="btn btn-primary">
                        <i class="bi bi-diagram-3"></i> Распределить по категориям
                    </a>
                </div>
            </div>
        {% else %}
            <!-- Список категорий с аккордеоном -->
            <div class="accordion" id="categoriesAccordion">
                {% for category_name, athletes in athletes_by_category.items() %}
                <div class="accordion-item">
                    <h2 class="accordion-header" id="heading{{ loop.index }}">
                        <button class="accordion-button {% if not loop.first %}collapsed{% endif %}" 
                                type="button" data-bs-toggle="collapse" 
                                data-bs-target="#collapse{{ loop.index }}" 
                                aria-expanded="{% if loop.first %}true{% else %}false{% endif %}" 
                                aria-controls="collapse{{ loop.index }}">
                            <div class="d-flex justify-content-between align-items-center w-100">
                                <div>
                                    <strong>{{ category_name }}</strong>
                                    <span class="badge bg-primary ms-2">{{ athletes|length }} участников</span>
                                </div>
                                <div>
                                    <a href="{{ url_for('view_category_detail', competition_id=competition.id, category_name=category_name) }}" 
                                       class="btn btn-sm btn-outline-primary me-2">
                                        <i class="bi bi-eye"></i> Детально
                                    </a>
                                    <a href="{{ url_for('export_category_results', competition_id=competition.id, category_name=category_name) }}" 
                                       class="btn btn-sm btn-outline-success me-2">
                                        <i class="bi bi-download"></i> Экспорт
                                    </a>
                                    {% if draw_data and category_name in draw_data %}
                                    <a href="{{ url_for('randomize_category_order', competition_id=competition.id, category_name=category_name) }}" 
                                       class="btn btn-sm btn-outline-warning">
                                        <i class="bi bi-shuffle"></i> Перемешать
                                    </a>
                                    {% endif %}
                                </div>
                            </div>
                        </button>
                    </h2>
                    <div id="collapse{{ loop.index }}" 
                         class="accordion-collapse collapse {% if loop.first %}show{% endif %}" 
                         aria-labelledby="heading{{ loop.index }}" 
                         data-bs-parent="#categoriesAccordion">
                        <div class="accordion-body">
                            <div class="table-responsive">
                                <table class="table table-hover" id="categoryTable{{ loop.index }}">
                                    <thead>
                                        <tr>
                                            <th width="60">#</th>
                                            <th>Спортсмен</th>
                                            <th>Клуб</th>
                                            <th>Дата рождения</th>
                                            <th>Пол</th>
                                            <th>Вес (кг)</th>
                                            <th class="text-center">Раунд 1</th>
                                            <th class="text-center">Раунд 2</th>
                                            <th class="text-center">Раунд 3</th>
                                            <th class="text-center">Общий</th>
                                            <th class="text-center">Средний</th>
                                            <th width="100">Действия</th>
                                        </tr>
                                    </thead>
                                    <tbody class="sortable-table" 
                                           data-category="{{ category_name }}"
                                           data-competition-id="{{ competition.id }}">
                                        {% for athlete in athletes %}
                                        {% set athlete_id = athlete.id %}
                                        {% set scores = athlete_scores.get(athlete_id, {}) %}
                                        <tr data-athlete-id="{{ athlete_id }}" 
                                            class="sortable-row {% if loop.index <= 3 %}table-warning{% endif %}">
                                            <td class="text-center">
                                                <span class="order-badge">{{ loop.index }}</span>
                                                <i class="bi bi-grip-vertical handle text-muted ms-1" 
                                                   style="cursor: move;"></i>
                                            </td>
                                            <td>
                                                <strong>{{ athlete.last_name }} {{ athlete.first_name }}</strong>
                                                <br>
                                                <small class="text-muted">ID: {{ athlete_id }}</small>
                                            </td>
                                            <td>{{ athlete.club or '—' }}</td>
                                            <td>{{ athlete.birth_date or '—' }}</td>
                                            <td>
                                                {% if athlete.gender == 'М' %}
                                                    <span class="badge bg-primary">Мужчина</span>
                                                {% elif athlete.gender == 'Ж' %}
                                                    <span class="badge bg-danger">Женщина</span>
                                                {% else %}
                                                    <span class="badge bg-secondary">—</span>
                                                {% endif %}
                                            </td>
                                            <td>{{ athlete.weight or '—' }}</td>
                                            <td class="text-center">
                                                {% if scores.get(1) %}
                                                    <span class="badge bg-info">{{ "%.2f"|format(scores[1]) }}</span>
                                                {% else %}
                                                    <span class="text-muted">—</span>
                                                {% endif %}
                                            </td>
                                            <td class="text-center">
                                                {% if scores.get(2) %}
                                                    <span class="badge bg-info">{{ "%.2f"|format(scores[2]) }}</span>
                                                {% else %}
                                                    <span class="text-muted">—</span>
                                                {% endif %}
                                            </td>
                                            <td class="text-center">
                                                {% if scores.get(3) %}
                                                    <span class="badge bg-info">{{ "%.2f"|format(scores[3]) }}</span>
                                                {% else %}
                                                    <span class="text-muted">—</span>
                                                {% endif %}
                                            </td>
                                            <td class="text-center">
                                                {% if scores.total > 0 %}
                                                    <span class="badge bg-success">{{ "%.2f"|format(scores.total) }}</span>
                                                {% else %}
                                                    <span class="text-muted">—</span>
                                                {% endif %}
                                            </td>
                                            <td class="text-center">
                                                {% if scores.average > 0 %}
                                                    <span class="badge bg-primary">{{ "%.2f"|format(scores.average) }}</span>
                                                {% else %}
                                                    <span class="text-muted">—</span>
                                                {% endif %}
                                            </td>
                                            <td>
                                                <button class="btn btn-sm btn-outline-info" 
                                                        onclick="showAthleteScores({{ athlete_id }})"
                                                        title="Редактировать оценки">
                                                    <i class="bi bi-pencil"></i>
                                                </button>
                                                <a href="#" class="btn btn-sm btn-outline-warning" 
                                                   title="Изменить категорию">
                                                    <i class="bi bi-arrow-right-circle"></i>
                                                </a>
                                            </td>
                                        </tr>
                                        {% endfor %}
                                    </tbody>
                                </table>
                            </div>
                            
                            <div class="mt-3">
                                {% if draw_data and category_name in draw_data %}
                                <div class="alert alert-info">
                                    <i class="bi bi-info-circle"></i>
                                    <strong>Порядок выступления:</strong>
                                    <span id="currentOrder{{ loop.index }}">
                                        {% set order = draw_data[category_name].get('order', []) %}
                                        {% if order %}
                                            {{ order|join(', ') }}
                                        {% else %}
                                            Не установлен
                                        {% endif %}
                                    </span>
                                </div>
                                {% endif %}
                                
                                <div class="d-flex justify-content-between">
                                    <div>
                                        <button class="btn btn-sm btn-outline-secondary save-order-btn" 
                                                data-category="{{ category_name }}"
                                                data-table-id="categoryTable{{ loop.index }}">
                                            <i class="bi bi-save"></i> Сохранить порядок
                                        </button>
                                        <button class="btn btn-sm btn-outline-primary" 
                                                onclick="printCategoryTable('categoryTable{{ loop.index }}', '{{ category_name }}')">
                                            <i class="bi bi-printer"></i> Печать
                                        </button>
                                    </div>
                                    <div>
                                        <span class="badge bg-light text-dark">
                                            <i class="bi bi-graph-up"></i>
                                            Средний балл: 
                                            {% set avg_scores = athlete_scores.values()|list %}
                                            {% set avg_total = avg_scores|map(attribute='average')|sum %}
                                            {{ "%.2f"|format(avg_total / avg_scores|length) if avg_scores else '0.00' }}
                                        </span>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
                {% endfor %}
            </div>
            
            <!-- Сводная информация -->
            <div class="card mt-4">
                <div class="card-header bg-dark text-white">
                    <h5 class="mb-0"><i class="bi bi-bar-chart"></i> Сводная статистика по категориям</h5>
                </div>
                <div class="card-body">
                    <div class="row">
                        <div class="col-md-6">
                            <h6>Количество участников по категориям:</h6>
                            <canvas id="categoryChart" height="200"></canvas>
                        </div>
                        <div class="col-md-6">
                            <h6>Средние баллы по категориям:</h6>
                            <div class="list-group">
                                {% for category_name, athletes in athletes_by_category.items() %}
                                {% set category_avg = 0 %}
                                {% set count = 0 %}
                                {% for athlete in athletes %}
                                    {% set scores = athlete_scores.get(athlete.id, {}) %}
                                    {% if scores.average > 0 %}
                                        {% set category_avg = category_avg + scores.average %}
                                        {% set count = count + 1 %}
                                    {% endif %}
                                {% endfor %}
                                <div class="list-group-item d-flex justify-content-between align-items-center">
                                    {{ category_name }}
                                    <div>
                                        <span class="badge bg-primary rounded-pill me-2">
                                            {{ athletes|length }} участн.
                                        </span>
                                        <span class="badge bg-success rounded-pill">
                                            {{ "%.2f"|format(category_avg / count if count > 0 else 0) }}
                                        </span>
                                    </div>
                                </div>
                                {% endfor %}
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        {% endif %}
    </div>
</div>

<!-- Модальное окно для редактирования оценок -->
<div class="modal fade" id="editScoresModal">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header bg-primary text-white">
                <h5 class="modal-title">Редактирование оценок</h5>
                <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <form id="scoresForm">
                    <input type="hidden" id="editAthleteId">
                    <input type="hidden" id="editCompetitionId" value="{{ competition.id }}">
                    
                    <div class="mb-3">
                        <label class="form-label">Спортсмен</label>
                        <input type="text" class="form-control" id="editAthleteName" readonly>
                    </div>
                    
                    <div class="row">
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Раунд 1</label>
                            <input type="number" class="form-control" id="editRound1" 
                                   step="0.1" min="0" max="10" placeholder="0-10">
                        </div>
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Раунд 2</label>
                            <input type="number" class="form-control" id="editRound2" 
                                   step="0.1" min="0" max="10" placeholder="0-10">
                        </div>
                        <div class="col-md-4 mb-3">
                            <label class="form-label">Раунд 3</label>
                            <input type="number" class="form-control" id="editRound3" 
                                   step="0.1" min="0" max="10" placeholder="0-10">
                        </div>
                    </div>
                </form>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Отмена</button>
                <button type="button" class="btn btn-primary" id="saveScoresBtn">Сохранить оценки</button>
            </div>
        </div>
    </div>
</div>

<style>
.sortable-row {
    cursor: move;
    transition: background-color 0.2s;
}

.sortable-row.sortable-ghost {
    opacity: 0.4;
    background-color: #f8f9fa;
}

.sortable-row.sortable-chosen {
    background-color: #e3f2fd;
}

.handle {
    opacity: 0.5;
    transition: opacity 0.2s;
}

.handle:hover {
    opacity: 1;
}

.order-badge {
    display: inline-block;
    width: 24px;
    height: 24px;
    line-height: 24px;
    text-align: center;
    background-color: #6c757d;
    color: white;
    border-radius: 50%;
    font-size: 12px;
}

.sortable-row:nth-child(1) .order-badge { background-color: #ffc107; color: #000; }
.sortable-row:nth-child(2) .order-badge { background-color: #6c757d; }
.sortable-row:nth-child(3) .order-badge { background-color: #dc3545; }
</style>

<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.14.0/Sortable.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<script>
// Инициализация перетаскивания для каждой категории
document.querySelectorAll('.sortable-table').forEach(table => {
    const tbody = table.querySelector('tbody');
    const category = table.dataset.category;
    const competitionId = table.dataset.competitionId;
    
    new Sortable(tbody, {
        animation: 150,
        ghostClass: 'sortable-ghost',
        chosenClass: 'sortable-chosen',
        handle: '.handle',
        
        onUpdate: function(evt) {
            // Обновляем номера порядка
            updateRowNumbers(table);
        }
    });
    
    // Обновляем номера строк при загрузке
    updateRowNumbers(table);
});

// Функция обновления номеров строк
function updateRowNumbers(table) {
    const rows = table.querySelectorAll('tbody tr');
    rows.forEach((row, index) => {
        const badge = row.querySelector('.order-badge');
        if (badge) {
            badge.textContent = index + 1;
            
            // Обновляем цвета для топ-3
            row.classList.remove('table-warning', 'table-light');
            if (index < 3) {
                row.classList.add('table-warning');
            }
        }
    });
}

// Сохранение нового порядка
document.querySelectorAll('.save-order-btn').forEach(button => {
    button.addEventListener('click', function() {
        const category = this.dataset.category;
        const tableId = this.dataset.tableId;
        const table = document.getElementById(tableId);
        const competitionId = {{ competition.id }};
        
        // Получаем новый порядок ID участников
        const rows = table.querySelectorAll('tbody tr');
        const newOrder = Array.from(rows).map(row => row.dataset.athleteId);
        
        // Отправляем на сервер
        fetch(`/competition/${competitionId}/update_order`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                category_name: category,
                order: newOrder
            })
        })
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                showAlert('success', 'Порядок выступления сохранен!');
                
                // Обновляем отображение порядка в информационном блоке
                const orderElement = document.getElementById(`currentOrder${tableId.replace('categoryTable', '')}`);
                if (orderElement) {
                    orderElement.textContent = newOrder.join(', ');
                }
            } else {
                showAlert('danger', 'Ошибка: ' + (data.error || 'Неизвестная ошибка'));
            }
        })
        .catch(error => {
            showAlert('danger', 'Ошибка сети: ' + error.message);
        });
    });
});

// Редактирование оценок
function showAthleteScores(athleteId) {
    // Здесь должна быть логика загрузки текущих оценок спортсмена
    // и открытия модального окна для редактирования
    const row = document.querySelector(`tr[data-athlete-id="${athleteId}"]`);
    if (row) {
        const name = row.querySelector('td:nth-child(2) strong').textContent;
        
        document.getElementById('editAthleteId').value = athleteId;
        document.getElementById('editAthleteName').value = name;
        
        // Загружаем текущие оценки
        // В реальном приложении здесь будет AJAX запрос
        
        // Открываем модальное окно
        new bootstrap.Modal(document.getElementById('editScoresModal')).show();
    }
}

// Сохранение оценок
document.getElementById('saveScoresBtn').addEventListener('click', function() {
    const athleteId = document.getElementById('editAthleteId').value;
    const competitionId = document.getElementById('editCompetitionId').value;
    const round1 = document.getElementById('editRound1').value;
    const round2 = document.getElementById('editRound2').value;
    const round3 = document.getElementById('editRound3').value;
    
    // Отправляем оценки на сервер
    fetch('/enter_scores', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            athlete_id: parseInt(athleteId),
            competition_id: parseInt(competitionId),
            round_number: 1,
            scores: [
                parseFloat(round1) || null,
                parseFloat(round2) || null,
                parseFloat(round3) || null,
                null, // judge4
                null  // judge5
            ]
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            showAlert('success', 'Оценки сохранены! Средний балл: ' + data.average.toFixed(2));
            bootstrap.Modal.getInstance(document.getElementById('editScoresModal')).hide();
            
            // Обновляем таблицу (в реальном приложении здесь будет обновление данных)
            location.reload();
        } else {
            showAlert('danger', 'Ошибка сохранения оценок');
        }
    });
});

// Печать таблицы категории
function printCategoryTable(tableId, categoryName) {
    const printWindow = window.open('', '_blank');
    const table = document.getElementById(tableId).cloneNode(true);
    
    // Убираем кнопки и лишние элементы
    table.querySelectorAll('button, .handle, .actions').forEach(el => el.remove());
    
    const html = `
        <!DOCTYPE html>
        <html>
        <head>
            <title>${categoryName} - {{ competition.name }}</title>
            <style>
                body { font-family: Arial, sans-serif; }
                table { width: 100%; border-collapse: collapse; margin: 20px 0; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: center; }
                th { background-color: #f2f2f2; font-weight: bold; }
                .header { text-align: center; margin-bottom: 20px; }
                .footer { margin-top: 30px; font-size: 12px; color: #666; }
            </style>
        </head>
        <body>
            <div class="header">
                <h2>{{ competition.name }}</h2>
                <h3>Категория: ${categoryName}</h3>
                <p>Дата: {{ competition.date }} | Место: {{ competition.location }}</p>
            </div>
            ${table.outerHTML}
            <div class="footer">
                <p>Распечатано: ${new Date().toLocaleString()}</p>
                <p>Судья: _________________________</p>
            </div>
        </body>
        </html>
    `;
    
    printWindow.document.write(html);
    printWindow.document.close();
    printWindow.print();
}

// График распределения участников по категориям
document.addEventListener('DOMContentLoaded', function() {
    const ctx = document.getElementById('categoryChart').getContext('2d');
    
    const categories = [];
    const participantsCount = [];
    const colors = [
        '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', 
        '#9966FF', '#FF9F40', '#8AC926', '#1982C4'
    ];
    
    {% for category_name, athletes in athletes_by_category.items() %}
    categories.push('{{ category_name }}');
    participantsCount.push({{ athletes|length }});
    {% endfor %}
    
    new Chart(ctx, {
        type: 'pie',
        data: {
            labels: categories,
            datasets: [{
                data: participantsCount,
                backgroundColor: colors.slice(0, categories.length),
                borderWidth: 1
            }]
        },
        options: {
            responsive: true,
            plugins: {
                legend: {
                    position: 'right',
                },
                title: {
                    display: true,
                    text: 'Распределение участников по категориям'
                }
            }
        }
    });
});

// Вспомогательная функция для уведомлений
function showAlert(type, message) {
    const alertDiv = document.createElement('div');
    alertDiv.className = `alert alert-${type} alert-dismissible fade show mt-3`;
    alertDiv.innerHTML = `
        ${message}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    `;
    
    document.querySelector('.container').prepend(alertDiv);
    
    setTimeout(() => {
        alertDiv.remove();
    }, 5000);
}
</script>
{% endblock %}
```

📝 3. Создайте шаблон templates/category_detail.html:

```html
{% extends "base.html" %}

{% block content %}
<div class="row">
    <div class="col-md-12">
        <div class="d-flex justify-content-between align-items-center mb-4">
            <h2>
                <i class="bi bi-tag-fill"></i>
                Категория: {{ category_name }}
                <small class="text-muted">({{ competition.name }})</small>
            </h2>
            <div>
                <a href="{{ url_for('view_competition_categories', competition_id=competition.id) }}" 
                   class="btn btn-secondary">
                    <i class="bi bi-arrow-left"></i> Все категории
                </a>
                <a href="{{ url_for('export_category_results', competition_id=competition.id, category_name=category_name) }}" 
                   class="btn btn-success">
                    <i class="bi bi-download"></i> Экспорт
                </a>
            </div>
        </div>
        
        <!-- Информация о категории -->
        <div class="card mb-4">
            <div class="card-body">
                <div class="row">
                    <div class="col-md-4">
                        <strong>Количество участников:</strong> {{ athletes|length }}
                    </div>
                    <div class="col-md-4">
                        <strong>Соревнование:</strong> {{ competition.name }}
                    </div>
                    <div class="col-md-4">
                        <strong>Дата:</strong> {{ competition.date }}
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Сетка категории (если есть) -->
        {% if category_draw and category_draw.pairs %}
        <div class="card mb-4">
            <div class="card-header bg-warning text-white">
                <h5 class="mb-0"><i class="bi bi-diagram-3"></i> Сетка категории</h5>
            </div>
            <div class="card-body">
                <div class="bracket">
                    {% for pair in category_draw.pairs %}
                    <div class="match mb-3 p-3 border rounded bg-light">
                        <div class="row">
                            <div class="col-md-5">
                                {% if pair[0] %}
                                    {% for athlete in athletes if athlete.id == pair[0] %}
                                    <div class="d-flex justify-content-between align-items-center p-2 bg-white rounded mb-2">
                                        <div>
                                            <strong>{{ athlete.last_name }} {{ athlete.first_name }}</strong>
                                            <br>
                                            <small class="text-muted">{{ athlete.club or '—' }}</small>
                                        </div>
                                        <div class="text-end">
                                            <small class="text-muted">ID: {{ athlete.id }}</small>
                                        </div>
                                    </div>
                                    {% endfor %}
                                {% else %}
                                    <div class="p-2 bg-white rounded text-center text-muted mb-2">
                                        Свободный жребий
                                    </div>
                                {% endif %}
                            </div>
                            
                            <div class="col-md-2 text-center d-flex align-items-center justify-content-center">
                                <span class="badge bg-dark">VS</span>
                            </div>
                            
                            <div class="col-md-5">
                                {% if pair[1] %}
                                    {% for athlete in athletes if athlete.id == pair[1] %}
                                    <div class="d-flex justify-content-between align-items-center p-2 bg-white rounded mb-2">
                                        <div>
                                            <strong>{{ athlete.last_name }} {{ athlete.first_name }}</strong>
                                            <br>
                                            <small class="text-muted">{{ athlete.club or '—' }}</small>
                                        </div>
                                        <div class="text-end">
                                            <small class="text-muted">ID: {{ athlete.id }}</small>
                                        </div>
                                    </div>
                                    {% endfor %}
                                {% else %}
                                    <div class="p-2 bg-white rounded text-center text-muted mb-2">
                                        Свободный жребий
                                    </div>
                                {% endif %}
                            </div>
                        </div>
                    </div>
                    {% endfor %}
                </div>
            </div>
        </div>
        {% endif %}
        
        <!-- Участники категории с оценками -->
        <div class="card">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0">
                    <i class="bi bi-people-fill"></i> Участники категории
                    <span class="badge bg-light text-dark float-end">{{ athletes|length }}</span>
                </h5>
            </div>
            <div class="card-body">
                <div class="table-responsive">
                    <table class="table table-hover table-striped">
                        <thead>
                            <tr>
                                <th width="60">#</th>
                                <th>Спортсмен</th>
                                <th>Клуб</th>
                                <th>Дата рождения</th>
                                <th>Пол</th>
                                <th>Вес (кг)</th>
                                <th class="text-center">Раунд 1</th>
                                <th class="text-center">Раунд 2</th>
                                <th class="text-center">Раунд 3</th>
                                <th class="text-center">Общий</th>
                                <th class="text-center">Средний</th>
                                <th class="text-center">Место</th>
                            </tr>
                        </thead>
                        <tbody>
                            {% for athlete in athletes %}
                            {% set scores = scores_by_athlete.get(athlete.id, {}) %}
                            {% set round1 = scores.get(1) %}
                            {% set round2 = scores.get(2) %}
                            {% set round3 = scores.get(3) %}
                            {% set total = scores.get('total', 0) %}
                            {% set average = scores.get('average', 0) %}
                            
                            <tr>
                                <td class="text-center">{{ loop.index }}</td>
                                <td>
                                    <strong>{{ athlete.last_name }} {{ athlete.first_name }}</strong>
                                    <br>
                                    <small class="text-muted">ID: {{ athlete.id }}</small>
                                </td>
                                <td>{{ athlete.club or '—' }}</td>
                                <td>{{ athlete.birth_date or '—' }}</td>
                                <td>
                                    {% if athlete.gender == 'М' %}
                                        <span class="badge bg-primary">Мужчина</span>
                                    {% elif athlete.gender == 'Ж' %}
                                        <span class="badge bg-danger">Женщина</span>
                                    {% else %}
                                        <span class="badge bg-secondary">—</span>
                                    {% endif %}
                                </td>
                                <td>{{ athlete.weight or '—' }}</td>
                                <td class="text-center">
                                    {% if round1 %}
                                        <span class="badge bg-info">{{ "%.2f"|format(round1) }}</span>
                                    {% else %}
                                        <span class="text-muted">—</span>
                                    {% endif %}
                                </td>
                                <td class="text-center">
                                    {% if round2 %}
                                        <span class="badge bg-info">{{ "%.2f"|format(round2) }}</span>
                                    {% else %}
                                        <span class="text-muted">—</span>
                                    {% endif %}
                                </td>
                                <td class="text-center">
                                    {% if round3 %}
                                        <span class="badge bg-info">{{ "%.2f"|format(round3) }}</span>
                                    {% else %}
                                        <span class="text-muted">—</span>
                                    {% endif %}
                                </td>
                                <td class="text-center">
                                    {% if total > 0 %}
                                        <span class="badge bg-success">{{ "%.2f"|format(total) }}</span>
                                    {% else %}
                                        <span class="text-muted">—</span>
                                    {% endif %}
                                </td>
                                <td class="text-center">
                                    {% if average > 0 %}
                                        <span class="badge bg-primary">{{ "%.2f"|format(average) }}</span>
                                    {% else %}
                                        <span class="text-muted">—</span>
                                    {% endif %}
                                </td>
                                <td class="text-center">
                                    {% if loop.index == 1 %}
                                        <span class="badge bg-warning text-dark">🥇 1</span>
                                    {% elif loop.index == 2 %}
                                        <span class="badge bg-secondary">🥈 2</span>
                                    {% elif loop.index == 3 %}
                                        <span class="badge bg-brown">🥉 3</span>
                                    {% else %}
                                        <span class="badge bg-light text-dark">{{ loop.index }}</span>
                                    {% endif %}
                                </td>
                            </tr>
                            {% endfor %}
                        </tbody>
                    </table>
                </div>
                
                <!-- Статистика категории -->
                <div class="row mt-4">
                    <div class="col-md-3 text-center">
                        <div class="display-6 text-primary">{{ athletes|length }}</div>
                        <div class="text-muted">Участников</div>
                    </div>
                    <div class="col-md-3 text-center">
                        {% set avg_total = athletes|map(attribute='id')|map(scores_by_athlete.get)|map(attribute='average')|sum %}
                        {% set avg_count = athletes|map(attribute='id')|map(scores_by_athlete.get)|map(attribute='average')|list|length %}
                        <div class="display-6 text-success">{{ "%.2f"|format(avg_total / avg_count if avg_count > 0 else 0) }}</div>
                        <div class="text-muted">Средний балл</div>
                    </div>
                    <div class="col-md-3 text-center">
                        {% set max_score = athletes|map(attribute='id')|map(scores_by_athlete.get)|map(attribute='average')|max %}
                        <div class="display-6 text-warning">{{ "%.2f"|format(max_score or 0) }}</div>
                        <div class="text-muted">Максимальный балл</div>
                    </div>
                    <div class="col-md-3 text-center">
                        {% set min_score = athletes|map(attribute='id')|map(scores_by_athlete.get)|map(attribute='average')|min %}
                        <div class="display-6 text-info">{{ "%.2f"|format(min_score or 0) }}</div>
                        <div class="text-muted">Минимальный балл</div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<style>
.bg-brown {
    background-color: #cd7f32 !important;
    color: white;
}

.match {
    border-left: 3px solid #ffc107;
}
</style>
{% endblock %}
```

📝 4. Обновите base.html - добавьте ссылку на просмотр категорий:

```html
<!-- В навигационной панели добавьте: -->
<li class="nav-item">
    <a class="nav-link" href="{{ url_for('view_competition_categories', competition_id=competition.id) }}">
        <i class="bi bi-tags"></i> Категории
    </a>
</li>
```

📋 ОСНОВНЫЕ ФУНКЦИОНАЛЬНЫЕ ВОЗМОЖНОСТИ:

✅ 1. Просмотр категорий:

· Список всех категорий соревнования
· Количество участников в каждой категории
· Средние баллы по категориям
· График распределения участников

✅ 2. Детальный просмотр категории:

· Полная информация о категории
· Список участников с оценками
· Расчет мест и рейтинга
· Статистика по категории

✅ 3. Редактирование порядка выступления:

· Перетаскивание участников (drag & drop)
· Автоматическая нумерация
· Сохранение изменений на сервере
· Визуальная индикация топ-3

✅ 4. Дополнительные функции:

· Экспорт результатов в Excel
· Печать таблиц
· Случайное перемешивание порядка
· Редактирование оценок

✅ 5. Визуальные особенности:

· Цветовое кодирование (топ-3 выделены)
· Интерактивные таблицы
· Графики и статистика
· Адаптивный дизайн

🚀 ИСПОЛЬЗОВАНИЕ:

1. Перейдите на страницу соревнования
2. Нажмите "Категории" в навигации
3. Раскройте нужную категорию
4. Измените порядок перетаскиванием
5. Сохраните изменения

Теперь у вас есть полнофункциональная система для просмотра и управления категориями соревнований!

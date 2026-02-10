Создаем недостающие файлы

1. Файл create_bracket.py - логика генерации турнирной сетки

```python
import math
import random
from datetime import datetime, timedelta
from database import db
from models import Tournament, Participant, Match, Category

def generate_bracket(tournament_id):
    """
    Генерирует турнирную сетку для олимпийской системы
    """
    try:
        tournament = Tournament.query.get(tournament_id)
        if not tournament:
            return False
        
        # Удаляем существующие матчи для этого турнира
        Match.query.filter_by(tournament_id=tournament_id).delete()
        
        # Получаем участников, отсортированных по seed
        participants = Participant.query.filter_by(
            tournament_id=tournament_id,
            status='registered'
        ).order_by(Participant.seed).all()
        
        if len(participants) < 2:
            return False
        
        # Автоматически назначаем seed, если не назначены
        assign_seeds_if_needed(participants)
        
        # Определяем ближайшую степень двойки
        num_participants = len(participants)
        next_power_of_two = 2 ** math.ceil(math.log2(num_participants))
        
        # Создаем список участников для распределения
        participants_list = participants.copy()
        
        # Если есть пустые места, добавляем "bye" участников
        if len(participants_list) < next_power_of_two:
            participants_list.extend([None] * (next_power_of_two - len(participants_list)))
        
        # Распределяем участников по сетке с учетом seeding
        bracket_positions = seed_bracket(participants_list)
        
        # Создаем матчи первого раунда
        round_matches = []
        round_number = 1
        
        for i in range(0, len(bracket_positions), 2):
            participant1 = bracket_positions[i]
            participant2 = bracket_positions[i + 1]
            
            # Пропускаем матчи с двумя None (bye)
            if participant1 is None and participant2 is None:
                continue
            
            # Если один из участников None (bye), он автоматически проходит дальше
            if participant1 is None and participant2:
                # Участник2 автоматически проходит в следующий раунд
                continue
            elif participant2 is None and participant1:
                # Участник1 автоматически проходит в следующий раунд
                continue
            
            match = Match(
                tournament_id=tournament_id,
                round=round_number,
                match_order=len(round_matches) + 1,
                participant1_id=participant1.id if participant1 else None,
                participant2_id=participant2.id if participant2 else None,
                status='scheduled',
                arena=determine_arena(len(round_matches) + 1)
            )
            db.session.add(match)
            round_matches.append(match)
        
        db.session.commit()
        
        # Генерируем последующие раунды
        total_rounds = int(math.log2(next_power_of_two))
        current_round_matches = round_matches.copy()
        
        for current_round in range(2, total_rounds + 1):
            next_round_matches = []
            num_matches_in_round = next_power_of_two // (2 ** current_round)
            
            for i in range(num_matches_in_round):
                match = Match(
                    tournament_id=tournament_id,
                    round=current_round,
                    match_order=i + 1,
                    status='pending',
                    arena=determine_arena(i + 1)
                )
                db.session.add(match)
                next_round_matches.append(match)
            
            db.session.commit()
            
            # Связываем матчи текущего раунда со следующим
            for i in range(0, len(current_round_matches), 2):
                if i + 1 < len(current_round_matches):
                    next_match = next_round_matches[i // 2]
                    current_round_matches[i].next_match_id = next_match.id
                    current_round_matches[i + 1].next_match_id = next_match.id
            
            db.session.commit()
            current_round_matches = next_round_matches
        
        # Обновляем статус турнира
        tournament.status = 'active'
        db.session.commit()
        
        return True
        
    except Exception as e:
        print(f"Ошибка при генерации сетки: {e}")
        db.session.rollback()
        return False

def assign_seeds_if_needed(participants):
    """
    Автоматически назначает seed участникам, если они не заданы
    """
    needs_seeding = any(p.seed is None for p in participants)
    
    if needs_seeding:
        # Сортируем по дате регистрации или случайно
        sorted_participants = sorted(participants, 
                                    key=lambda x: x.id)  # или по другому критерию
        
        for i, participant in enumerate(sorted_participants):
            if participant.seed is None:
                participant.seed = i + 1
        
        db.session.commit()

def seed_bracket(participants):
    """
    Распределяет участников по турнирной сетке с учетом seeding
    Использует стандартный алгоритм seeding для турниров
    """
    n = len(participants)
    if n <= 2:
        return participants
    
    # Сортируем участников по seed (None в конце)
    sorted_participants = sorted(
        [p for p in participants if p is not None],
        key=lambda x: x.seed if x.seed is not None else float('inf')
    )
    
    # Добавляем None в конец, если нужно
    none_count = participants.count(None)
    sorted_participants.extend([None] * none_count)
    
    # Стандартный алгоритм seeding
    bracket = [None] * n
    bracket[0] = sorted_participants[0] if sorted_participants else None
    
    if n > 1:
        bracket[-1] = sorted_participants[1] if len(sorted_participants) > 1 else None
    
    # Рекурсивно заполняем остальные позиции
    fill_bracket_recursive(bracket, sorted_participants[2:], 0, n - 1)
    
    return bracket

def fill_bracket_recursive(bracket, participants, start, end):
    """
    Рекурсивно заполняет турнирную сетку
    """
    if end - start <= 1:
        return
    
    mid = (start + end) // 2
    
    if not bracket[mid]:
        # Находим следующего участника для размещения
        for participant in participants:
            if participant and participant not in bracket:
                bracket[mid] = participant
                break
    
    # Рекурсивно заполняем левую и правую части
    fill_bracket_recursive(bracket, participants, start, mid)
    fill_bracket_recursive(bracket, participants, mid, end)

def determine_arena(match_number):
    """
    Определяет арену для матча
    """
    arenas = ['Центральный корт', 'Корт 1', 'Корт 2', 'Корт 3', 'Корт 4']
    return arenas[(match_number - 1) % len(arenas)]

def update_match_winner(match_id, winner_id, score1=None, score2=None):
    """
    Обновляет результат матча и продвигает победителя в следующий раунд
    """
    try:
        match = Match.query.get(match_id)
        if not match:
            return False
        
        match.winner_id = winner_id
        match.status = 'completed'
        match.end_time = datetime.utcnow()
        
        if score1 is not None:
            match.score1 = score1
        if score2 is not None:
            match.score2 = score2
        
        # Если есть следующий матч, добавляем победителя в него
        if match.next_match_id:
            next_match = Match.query.get(match.next_match_id)
            
            if not next_match.participant1_id:
                next_match.participant1_id = winner_id
            elif not next_match.participant2_id:
                next_match.participant2_id = winner_id
            else:
                # Оба участника уже назначены (может произойти в случае bye)
                pass
            
            # Если оба участника определены, меняем статус матча
            if next_match.participant1_id and next_match.participant2_id:
                next_match.status = 'scheduled'
        
        db.session.commit()
        
        # Проверяем, завершен ли турнир
        check_tournament_completion(match.tournament_id)
        
        return True
        
    except Exception as e:
        print(f"Ошибка при обновлении результата матча: {e}")
        db.session.rollback()
        return False

def check_tournament_completion(tournament_id):
    """
    Проверяет, завершен ли турнир, и обновляет статус
    """
    tournament = Tournament.query.get(tournament_id)
    if not tournament:
        return
    
    # Проверяем, есть ли незавершенные матчи
    incomplete_matches = Match.query.filter_by(
        tournament_id=tournament_id,
        status='scheduled'
    ).count()
    
    if incomplete_matches == 0:
        tournament.status = 'completed'
        tournament.end_date = datetime.utcnow()
        db.session.commit()

def auto_generate_schedule(tournament_id, start_time):
    """
    Автоматически генерирует расписание матчей
    """
    try:
        tournament = Tournament.query.get(tournament_id)
        if not tournament:
            return False
        
        matches = Match.query.filter_by(
            tournament_id=tournament_id
        ).order_by(Match.round, Match.match_order).all()
        
        current_time = datetime.fromisoformat(start_time)
        match_duration = 30  # минут
        
        for match in matches:
            match.start_time = current_time
            current_time += timedelta(minutes=match_duration)
            
            # Добавляем перерыв между раундами
            if match.match_order == 1 and match.round > 1:
                current_time += timedelta(minutes=15)
        
        db.session.commit()
        return True
        
    except Exception as e:
        print(f"Ошибка при генерации расписания: {e}")
        db.session.rollback()
        return False

def get_tournament_standings(tournament_id):
    """
    Возвращает итоговые места участников турнира
    """
    # Получаем всех участников турнира
    participants = Participant.query.filter_by(
        tournament_id=tournament_id
    ).all()
    
    # Для олимпийской системы определяем места по результатам матчей
    standings = []
    
    for participant in participants:
        # Считаем статистику
        matches_won = Match.query.filter_by(
            winner_id=participant.id
        ).count()
        
        matches_played = Match.query.filter(
            db.or_(
                Match.participant1_id == participant.id,
                Match.participant2_id == participant.id
            )
        ).count()
        
        # Определяем место (простейшая логика - по количеству побед)
        standings.append({
            'participant': participant.to_dict(),
            'matches_played': matches_played,
            'matches_won': matches_won,
            'win_rate': matches_won / matches_played if matches_played > 0 else 0
        })
    
    # Сортируем по количеству побед
    standings.sort(key=lambda x: x['win_rate'], reverse=True)
    
    # Присваиваем места
    for i, standing in enumerate(standings):
        standing['position'] = i + 1
    
    return standings

def get_bracket_visualization_data(tournament_id):
    """
    Подготавливает данные для визуализации турнирной сетки
    """
    matches = Match.query.filter_by(
        tournament_id=tournament_id
    ).order_by(Match.round, Match.match_order).all()
    
    # Группируем матчи по раундам
    rounds = {}
    for match in matches:
        if match.round not in rounds:
            rounds[match.round] = []
        rounds[match.round].append(match.to_dict())
    
    # Определяем общее количество раундов
    max_round = max(rounds.keys()) if rounds else 0
    
    return {
        'rounds': rounds,
        'total_rounds': max_round,
        'matches': [m.to_dict() for m in matches]
    }
```

2. Файл scoring_system.py - логика системы оценок

```python
import statistics
from datetime import datetime
from database import db
from models import Participant, Score, Result, Tournament

def calculate_final_scores(participant_id):
    """
    Рассчитывает итоговый балл участника на основе оценок судей
    """
    try:
        scores = Score.query.filter_by(participant_id=participant_id).all()
        
        if not scores:
            return 0.0
        
        # Группируем оценки по судьям
        judge_scores = {}
        for score in scores:
            if score.judge_id not in judge_scores:
                judge_scores[score.judge_id] = []
            judge_scores[score.judge_id].append(score)
        
        # Рассчитываем средний балл для каждого судьи
        judge_averages = []
        for judge_id, judge_scores_list in judge_scores.items():
            total_weighted = 0
            total_weight = 0
            
            for score in judge_scores_list:
                # Нормализуем оценку относительно максимума
                normalized_score = (score.value / score.max_value) * 10
                total_weighted += normalized_score * score.weight
                total_weight += score.weight
            
            if total_weight > 0:
                judge_average = total_weighted / total_weight
                judge_averages.append(judge_average)
        
        if not judge_averages:
            return 0.0
        
        # Исключаем крайние оценки (если судей больше 3)
        if len(judge_averages) > 3:
            judge_averages.sort()
            # Удаляем наименьшую и наибольшую оценки
            judge_averages = judge_averages[1:-1]
        
        # Рассчитываем финальный балл как среднее оставшихся оценок
        final_score = statistics.mean(judge_averages)
        
        return round(final_score, 2)
        
    except Exception as e:
        print(f"Ошибка при расчете финального балла: {e}")
        return 0.0

def generate_leaderboard(tournament_id, category_id=None):
    """
    Генерирует турнирную таблицу для системы оценок
    """
    try:
        # Получаем участников
        query = Participant.query.filter_by(tournament_id=tournament_id)
        if category_id:
            query = query.filter_by(category_id=category_id)
        
        participants = query.all()
        
        # Рассчитываем баллы для каждого участника
        leaderboard = []
        for participant in participants:
            final_score = calculate_final_scores(participant.id)
            
            # Получаем все оценки участника для детализации
            scores = Score.query.filter_by(participant_id=participant.id).all()
            score_details = []
            
            for score in scores:
                score_details.append({
                    'criterion': score.criterion,
                    'value': score.value,
                    'max_value': score.max_value,
                    'weight': score.weight,
                    'judge': score.judge.full_name if score.judge else 'Неизвестный судья'
                })
            
            leaderboard.append({
                'participant': participant.to_dict(),
                'final_score': final_score,
                'score_details': score_details,
                'scores_count': len(scores)
            })
        
        # Сортируем по убыванию баллов
        leaderboard.sort(key=lambda x: x['final_score'], reverse=True)
        
        # Присваиваем места
        for i, entry in enumerate(leaderboard):
            entry['position'] = i + 1
            
            # Определяем медаль
            if i == 0:
                entry['medal'] = 'gold'
            elif i == 1:
                entry['medal'] = 'silver'
            elif i == 2:
                entry['medal'] = 'bronze'
            else:
                entry['medal'] = None
        
        return leaderboard
        
    except Exception as e:
        print(f"Ошибка при генерации турнирной таблицы: {e}")
        return []

def save_final_results(tournament_id, category_id=None):
    """
    Сохраняет окончательные результаты турнира в базу данных
    """
    try:
        leaderboard = generate_leaderboard(tournament_id, category_id)
        
        # Удаляем старые результаты
        Result.query.filter_by(tournament_id=tournament_id).delete()
        
        # Сохраняем новые результаты
        for entry in leaderboard:
            result = Result(
                tournament_id=tournament_id,
                category_id=category_id,
                participant_id=entry['participant']['id'],
                position=entry['position'],
                final_score=entry['final_score'],
                medal=entry['medal']
            )
            db.session.add(result)
        
        db.session.commit()
        return True
        
    except Exception as e:
        print(f"Ошибка при сохранении результатов: {e}")
        db.session.rollback()
        return False

def calculate_statistics(tournament_id):
    """
    Рассчитывает статистику турнира
    """
    try:
        participants = Participant.query.filter_by(tournament_id=tournament_id).all()
        scores = Score.query.join(Participant).filter(Participant.tournament_id == tournament_id).all()
        
        if not scores:
            return {
                'average_score': 0,
                'highest_score': 0,
                'lowest_score': 0,
                'score_distribution': [],
                'judge_consistency': []
            }
        
        # Собираем все финальные баллы
        final_scores = []
        for participant in participants:
            score = calculate_final_scores(participant.id)
            if score > 0:
                final_scores.append(score)
        
        if not final_scores:
            return {
                'average_score': 0,
                'highest_score': 0,
                'lowest_score': 0,
                'score_distribution': [],
                'judge_consistency': []
            }
        
        # Основная статистика
        stats = {
            'average_score': round(statistics.mean(final_scores), 2),
            'highest_score': round(max(final_scores), 2),
            'lowest_score': round(min(final_scores), 2),
            'median_score': round(statistics.median(final_scores), 2),
            'std_deviation': round(statistics.stdev(final_scores), 2) if len(final_scores) > 1 else 0,
            'total_participants': len(participants),
            'participants_with_scores': len(final_scores)
        }
        
        # Распределение оценок
        score_ranges = [(0, 3), (3, 5), (5, 7), (7, 9), (9, 10)]
        distribution = []
        
        for low, high in score_ranges:
            count = sum(1 for score in final_scores if low <= score < high)
            distribution.append({
                'range': f"{low}-{high}",
                'count': count,
                'percentage': round((count / len(final_scores)) * 100, 1) if final_scores else 0
            })
        
        stats['score_distribution'] = distribution
        
        # Консистентность судей
        judges = {}
        for score in scores:
            if score.judge_id not in judges:
                judges[score.judge_id] = []
            judges[score.judge_id].append(score.value)
        
        judge_consistency = []
        for judge_id, judge_scores in judges.items():
            if len(judge_scores) > 1:
                consistency = 1 - (statistics.stdev(judge_scores) / 10)  # Нормализованная консистентность
            else:
                consistency = 1
            
            judge_consistency.append({
                'judge_id': judge_id,
                'consistency': round(consistency, 2),
                'scores_count': len(judge_scores)
            })
        
        stats['judge_consistency'] = judge_consistency
        
        return stats
        
    except Exception as e:
        print(f"Ошибка при расчете статистики: {e}")
        return {}

def export_results_to_csv(tournament_id, filepath):
    """
    Экспортирует результаты турнира в CSV файл
    """
    try:
        import csv
        
        leaderboard = generate_leaderboard(tournament_id)
        
        with open(filepath, 'w', newline='', encoding='utf-8') as csvfile:
            fieldnames = ['Место', 'Участник', 'Страна', 'Клуб', 'Итоговый балл', 'Количество оценок']
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
            
            writer.writeheader()
            for entry in leaderboard:
                participant = entry['participant']['athlete']
                writer.writerow({
                    'Место': entry['position'],
                    'Участник': participant['full_name'],
                    'Страна': participant.get('country', ''),
                    'Клуб': participant.get('club', ''),
                    'Итоговый балл': entry['final_score'],
                    'Количество оценок': entry['scores_count']
                })
        
        return True
        
    except Exception as e:
        print(f"Ошибка при экспорте в CSV: {e}")
        return False

def generate_score_report(tournament_id, participant_id):
    """
    Генерирует детальный отчет по оценкам участника
    """
    try:
        participant = Participant.query.get(participant_id)
        if not participant:
            return None
        
        scores = Score.query.filter_by(participant_id=participant_id).all()
        
        # Группируем оценки по критериям
        criteria = {}
        for score in scores:
            if score.criterion not in criteria:
                criteria[score.criterion] = []
            criteria[score.criterion].append({
                'value': score.value,
                'max_value': score.max_value,
                'weight': score.weight,
                'judge': score.judge.full_name if score.judge else 'Неизвестный судья'
            })
        
        # Рассчитываем статистику по каждому критерию
        criterion_stats = []
        for criterion_name, criterion_scores in criteria.items():
            values = [s['value'] for s in criterion_scores]
            avg_score = sum(values) / len(values) if values else 0
            max_possible = criterion_scores[0]['max_value'] if criterion_scores else 10
            
            criterion_stats.append({
                'criterion': criterion_name,
                'average_score': round(avg_score, 2),
                'max_possible': max_possible,
                'percentage': round((avg_score / max_possible) * 100, 1),
                'scores_count': len(criterion_scores),
                'scores': criterion_scores
            })
        
        # Сортируем критерии по важности (весу)
        criterion_stats.sort(key=lambda x: x['scores'][0]['weight'] if x['scores'] else 0, reverse=True)
        
        final_score = calculate_final_scores(participant_id)
        
        report = {
            'participant': participant.to_dict(),
            'final_score': final_score,
            'criteria': criterion_stats,
            'total_scores': len(scores),
            'unique_judges': len(set(s.judge_id for s in scores if s.judge_id))
        }
        
        return report
        
    except Exception as e:
        print(f"Ошибка при генерации отчета: {e}")
        return None

def validate_scores(tournament_id):
    """
    Проверяет корректность оценок в турнире
    """
    try:
        issues = []
        
        # Проверяем участников без оценок
        participants = Participant.query.filter_by(tournament_id=tournament_id).all()
        for participant in participants:
            scores = Score.query.filter_by(participant_id=participant.id).all()
            if not scores:
                issues.append({
                    'type': 'no_scores',
                    'participant': participant.athlete.full_name,
                    'message': 'У участника нет оценок'
                })
        
        # Проверяем оценки за пределами допустимого диапазона
        scores = Score.query.join(Participant).filter(Participant.tournament_id == tournament_id).all()
        for score in scores:
            if score.value < 0 or score.value > score.max_value:
                issues.append({
                    'type': 'invalid_score',
                    'participant': score.participant.athlete.full_name,
                    'criterion': score.criterion,
                    'value': score.value,
                    'max_value': score.max_value,
                    'message': f'Оценка выходит за допустимый диапазон (0-{score.max_value})'
                })
        
        # Проверяем судей, которые поставили слишком высокие/низкие оценки
        judges = {}
        for score in scores:
            if score.judge_id not in judges:
                judges[score.judge_id] = []
            judges[score.judge_id].append(score.value)
        
        for judge_id, judge_scores in judges.items():
            if len(judge_scores) > 5:  # Только если судья поставил достаточно оценок
                avg_score = statistics.mean(judge_scores)
                std_dev = statistics.stdev(judge_scores) if len(judge_scores) > 1 else 0
                
                if avg_score < 3:
                    issues.append({
                        'type': 'strict_judge',
                        'judge_id': judge_id,
                        'average_score': round(avg_score, 2),
                        'message': 'Судья слишком строгий (средняя оценка < 3)'
                    })
                elif avg_score > 8:
                    issues.append({
                        'type': 'lenient_judge',
                        'judge_id': judge_id,
                        'average_score': round(avg_score, 2),
                        'message': 'Судья слишком мягкий (средняя оценка > 8)'
                    })
        
        return {
            'has_issues': len(issues) > 0,
            'issues': issues,
            'issues_count': len(issues)
        }
        
    except Exception as e:
        print(f"Ошибка при валидации оценок: {e}")
        return {
            'has_issues': True,
            'issues': [{'type': 'validation_error', 'message': str(e)}],
            'issues_count': 1
        }
```

3. Файл static/js/tournament.js - JavaScript для турнирной сетки

```javascript
// Основные функции для работы с турнирной сеткой

class TournamentBracket {
    constructor(containerId, tournamentId) {
        this.container = document.getElementById(containerId);
        this.tournamentId = tournamentId;
        this.zoomLevel = 1;
        this.currentRound = 1;
        this.totalRounds = 1;
        this.matches = [];
    }

    async loadBracket() {
        try {
            const response = await fetch(`/api/tournaments/${this.tournamentId}/bracket`);
            const data = await response.json();
            
            this.matches = data.matches;
            this.totalRounds = data.total_rounds;
            
            this.renderBracket();
            this.setupEventListeners();
            
        } catch (error) {
            console.error('Ошибка загрузки турнирной сетки:', error);
            this.showError('Не удалось загрузить турнирную сетку');
        }
    }

    renderBracket() {
        // Создаем структуру сетки
        this.container.innerHTML = '';
        
        const bracketWrapper = document.createElement('div');
        bracketWrapper.className = 'bracket-wrapper';
        
        // Создаем раунды
        for (let round = 1; round <= this.totalRounds; round++) {
            const roundElement = this.createRoundElement(round);
            bracketWrapper.appendChild(roundElement);
        }
        
        this.container.appendChild(bracketWrapper);
        
        // Применяем текущий масштаб
        this.applyZoom();
    }

    createRoundElement(roundNumber) {
        const roundElement = document.createElement('div');
        roundElement.className = 'round';
        roundElement.dataset.round = roundNumber;
        
        // Заголовок раунда
        const roundHeader = document.createElement('div');
        roundHeader.className = 'round-header';
        roundHeader.textContent = this.getRoundName(roundNumber);
        roundElement.appendChild(roundHeader);
        
        // Контейнер для матчей
        const matchesContainer = document.createElement('div');
        matchesContainer.className = 'matches-container';
        
        // Фильтруем матчи текущего раунда
        const roundMatches = this.matches.filter(match => match.round === roundNumber);
        
        // Создаем элементы матчей
        roundMatches.forEach(match => {
            const matchElement = this.createMatchElement(match);
            matchesContainer.appendChild(matchElement);
        });
        
        roundElement.appendChild(matchesContainer);
        
        return roundElement;
    }

    createMatchElement(matchData) {
        const matchElement = document.createElement('div');
        matchElement.className = `match ${matchData.status}`;
        matchElement.dataset.matchId = matchData.id;
        
        const participant1 = matchData.participant1;
        const participant2 = matchData.participant2;
        
        let html = `
            <div class="match-header">
                <span class="match-info">Матч ${matchData.match_order}</span>
                <span class="match-status status-${matchData.status}">${this.getMatchStatusText(matchData.status)}</span>
            </div>
            
            <div class="match-body">
        `;
        
        // Участник 1
        html += this.createParticipantElement(participant1, matchData.score1, matchData.winner_id === participant1?.id);
        
        // VS
        html += '<div class="match-vs">VS</div>';
        
        // Участник 2
        html += this.createParticipantElement(participant2, matchData.score2, matchData.winner_id === participant2?.id);
        
        html += `
            </div>
            
            <div class="match-footer">
        `;
        
        // Дополнительная информация
        if (matchData.start_time) {
            const startTime = new Date(matchData.start_time);
            html += `<div class="match-time">${startTime.toLocaleTimeString('ru-RU', {hour: '2-digit', minute:'2-digit'})}</div>`;
        }
        
        if (matchData.arena) {
            html += `<div class="match-arena">${matchData.arena}</div>`;
        }
        
        // Кнопки действий
        if (matchData.status === 'scheduled' || matchData.status === 'in_progress') {
            html += `
                <div class="match-actions">
                    <button class="btn-action btn-start" data-match-id="${matchData.id}">Начать</button>
                    <button class="btn-action btn-update" data-match-id="${matchData.id}">Результат</button>
                </div>
            `;
        }
        
        html += '</div></div>';
        
        matchElement.innerHTML = html;
        return matchElement;
    }

    createParticipantElement(participant, score, isWinner) {
        if (!participant) {
            return '<div class="participant empty">TBD</div>';
        }
        
        const winnerClass = isWinner ? 'winner' : '';
        const seedText = participant.seed ? `<span class="seed">#${participant.seed}</span>` : '';
        
        return `
            <div class="participant ${winnerClass}">
                <div class="participant-info">
                    <div class="participant-name">${participant.athlete.full_name}</div>
                    <div class="participant-details">
                        ${seedText}
                        ${participant.athlete.country ? `<span class="country">${participant.athlete.country}</span>` : ''}
                    </div>
                </div>
                ${score !== undefined ? `<div class="participant-score">${score}</div>` : ''}
            </div>
        `;
    }

    getRoundName(roundNumber) {
        if (roundNumber === this.totalRounds) return 'Финал';
        if (roundNumber === this.totalRounds - 1) return 'Полуфинал';
        if (roundNumber === this.totalRounds - 2) return 'Четвертьфинал';
        return `Раунд ${roundNumber}`;
    }

    getMatchStatusText(status) {
        const statusMap = {
            'scheduled': 'Запланирован',
            'in_progress': 'В процессе',
            'completed': 'Завершен',
            'pending': 'Ожидает'
        };
        return statusMap[status] || status;
    }

    setupEventListeners() {
        // Обработчики для кнопок матчей
        this.container.addEventListener('click', (e) => {
            const matchId = e.target.closest('[data-match-id]')?.dataset.matchId;
            if (!matchId) return;
            
            if (e.target.classList.contains('btn-start')) {
                this.startMatch(matchId);
            } else if (e.target.classList.contains('btn-update')) {
                this.openMatchModal(matchId);
            } else if (e.target.closest('.participant')) {
                this.showParticipantInfo(matchId);
            }
        });
        
        // Кнопки управления масштабом
        document.getElementById('zoomInBtn')?.addEventListener('click', () => this.zoomIn());
        document.getElementById('zoomOutBtn')?.addEventListener('click', () => this.zoomOut());
        document.getElementById('resetZoomBtn')?.addEventListener('click', () => this.resetZoom());
        
        // Навигация по раундам
        document.getElementById('prevRoundBtn')?.addEventListener('click', () => this.showPreviousRound());
        document.getElementById('nextRoundBtn')?.addEventListener('click', () => this.showNextRound());
    }

    async startMatch(matchId) {
        try {
            const response = await fetch(`/api/matches/${matchId}/start`, {
                method: 'POST'
            });
            
            if (response.ok) {
                this.showSuccess('Матч начат');
                await this.loadBracket();
            } else {
                throw new Error('Ошибка начала матча');
            }
        } catch (error) {
            console.error('Ошибка начала матча:', error);
            this.showError('Не удалось начать матч');
        }
    }

    async openMatchModal(matchId) {
        try {
            const response = await fetch(`/api/matches/${matchId}`);
            const match = await response.json();
            
            // Здесь можно открыть модальное окно для ввода результатов
            this.showMatchResultModal(match);
            
        } catch (error) {
            console.error('Ошибка загрузки данных матча:', error);
            this.showError('Не удалось загрузить данные матча');
        }
    }

    showMatchResultModal(match) {
        // Создаем модальное окно для ввода результатов
        const modal = document.createElement('div');
        modal.className = 'match-modal';
        modal.innerHTML = `
            <div class="modal-content">
                <div class="modal-header">
                    <h3>Результат матча</h3>
                    <button class="close">&times;</button>
                </div>
                <div class="modal-body">
                    <div class="match-preview">
                        <div class="participant">
                            <div class="name">${match.participant1?.athlete.full_name || 'TBD'}</div>
                            <input type="number" class="score-input" id="score1" value="${match.score1 || 0}" min="0">
                        </div>
                        <div class="vs">VS</div>
                        <div class="participant">
                            <div class="name">${match.participant2?.athlete.full_name || 'TBD'}</div>
                            <input type="number" class="score-input" id="score2" value="${match.score2 || 0}" min="0">
                        </div>
                    </div>
                    <div class="form-group">
                        <label>Победитель:</label>
                        <select id="winnerSelect">
                            <option value="">Выберите победителя</option>
                            <option value="${match.participant1?.id}">${match.participant1?.athlete.full_name || 'Участник 1'}</option>
                            <option value="${match.participant2?.id}">${match.participant2?.athlete.full_name || 'Участник 2'}</option>
                        </select>
                    </div>
                </div>
                <div class="modal-footer">
                    <button class="btn-cancel">Отмена</button>
                    <button class="btn-save">Сохранить</button>
                </div>
            </div>
        `;
        
        document.body.appendChild(modal);
        
        // Обработчики событий для модального окна
        modal.querySelector('.close').addEventListener('click', () => modal.remove());
        modal.querySelector('.btn-cancel').addEventListener('click', () => modal.remove());
        modal.querySelector('.btn-save').addEventListener('click', async () => {
            await this.saveMatchResult(match.id, modal);
            modal.remove();
        });
    }

    async saveMatchResult(matchId, modal) {
        try {
            const score1 = parseInt(modal.querySelector('#score1').value);
            const score2 = parseInt(modal.querySelector('#score2').value);
            const winnerId = modal.querySelector('#winnerSelect').value;
            
            const response = await fetch(`/api/matches/${matchId}`, {
                method: 'PUT',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    score1: score1,
                    score2: score2,
                    winner_id: winnerId || null,
                    status: 'completed'
                })
            });
            
            if (response.ok) {
                this.showSuccess('Результат сохранен');
                await this.loadBracket();
            } else {
                throw new Error('Ошибка сохранения результата');
            }
        } catch (error) {
            console.error('Ошибка сохранения результата:', error);
            this.showError('Не удалось сохранить результат');
        }
    }

    zoomIn() {
        if (this.zoomLevel < 2) {
            this.zoomLevel += 0.1;
            this.applyZoom();
        }
    }

    zoomOut() {
        if (this.zoomLevel > 0.5) {
            this.zoomLevel -= 0.1;
            this.applyZoom();
        }
    }

    resetZoom() {
        this.zoomLevel = 1;
        this.applyZoom();
    }

    applyZoom() {
        const bracketWrapper = this.container.querySelector('.bracket-wrapper');
        if (bracketWrapper) {
            bracketWrapper.style.transform = `scale(${this.zoomLevel})`;
        }
    }

    showPreviousRound() {
        if (this.currentRound > 1) {
            this.currentRound--;
            this.highlightCurrentRound();
        }
    }

    showNextRound() {
        if (this.currentRound < this.totalRounds) {
            this.currentRound++;
            this.highlightCurrentRound();
        }
    }

    highlightCurrentRound() {
        // Убираем подсветку со всех раундов
        this.container.querySelectorAll('.round').forEach(round => {
            round.classList.remove('current');
        });
        
        // Подсвечиваем текущий раунд
        const currentRoundElement = this.container.querySelector(`[data-round="${this.currentRound}"]`);
        if (currentRoundElement) {
            currentRoundElement.classList.add('current');
        }
        
        // Прокручиваем к текущему раунду
        currentRoundElement?.scrollIntoView({
            behavior: 'smooth',
            block: 'nearest',
            inline: 'start'
        });
    }

    showParticipantInfo(matchId) {
        // Здесь можно показать информацию об участнике
        console.log('Показать информацию об участнике матча:', matchId);
    }

    showSuccess(message) {
        this.showNotification(message, 'success');
    }

    showError(message) {
        this.showNotification(message, 'error');
    }

    showNotification(message, type = 'info') {
        const notification = document.createElement('div');
        notification.className = `notification notification-${type}`;
        notification.textContent = message;
        
        document.body.appendChild(notification);
        
        // Автоматическое скрытие через 3 секунды
        setTimeout(() => {
            notification.remove();
        }, 3000);
    }
}

// Инициализация при загрузке страницы
document.addEventListener('DOMContentLoaded', function() {
    const tournamentId = window.tournamentId || document.getElementById('tournamentId')?.value;
    
    if (tournamentId) {
        const bracket = new TournamentBracket('bracketContainer', tournamentId);
        bracket.loadBracket();
        
        // Сохраняем объект в глобальной области видимости для доступа из консоли
        window.tournamentBracket = bracket;
    }
});

// Функции для управления таймером
class MatchTimer {
    constructor(matchId, duration = 300) { // 5 минут по умолчанию
        this.matchId = matchId;
        this.duration = duration; // в секундах
        this.remaining = duration;
        this.timer = null;
        this.isRunning = false;
    }

    start() {
        if (this.isRunning) return;
        
        this.isRunning = true;
        this.timer = setInterval(() => {
            this.remaining--;
            this.updateDisplay();
            
            if (this.remaining <= 0) {
                this.stop();
                this.onTimeUp();
            }
        }, 1000);
    }

    pause() {
        if (!this.isRunning) return;
        
        this.isRunning = false;
        clearInterval(this.timer);
    }

    stop() {
        this.isRunning = false;
        clearInterval(this.timer);
        this.remaining = this.duration;
        this.updateDisplay();
    }

    addTime(seconds) {
        this.remaining += seconds;
        this.updateDisplay();
    }

    updateDisplay() {
        const minutes = Math.floor(this.remaining / 60);
        const seconds = this.remaining % 60;
        const display = `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        
        // Обновляем отображение на странице
        const timerElement = document.getElementById(`timer-${this.matchId}`);
        if (timerElement) {
            timerElement.textContent = display;
            
            // Изменяем цвет при низком времени
            if (this.remaining < 60) {
                timerElement.classList.add('warning');
            } else {
                timerElement.classList.remove('warning');
            }
        }
    }

    onTimeUp() {
        // Вызывается при окончании времени
        console.log('Время матча истекло!');
        this.showNotification('Время матча истекло!', 'warning');
    }

    showNotification(message, type) {
        const notification = document.createElement('div');
        notification.className = `timer-notification ${type}`;
        notification.textContent = message;
        
        document.body.appendChild(notification);
        
        setTimeout(() => {
            notification.remove();
        }, 5000);
    }
}

// Глобальные функции для доступа из HTML
window.startMatchTimer = function(matchId, duration = 300) {
    if (!window.matchTimers) window.matchTimers = {};
    
    if (window.matchTimers[matchId]) {
        window.matchTimers[matchId].start();
    } else {
        window.matchTimers[matchId] = new MatchTimer(matchId, duration);
        window.matchTimers[matchId].start();
    }
};

window.pauseMatchTimer = function(matchId) {
    if (window.matchTimers && window.matchTimers[matchId]) {
        window.matchTimers[matchId].pause();
    }
};

window.stopMatchTimer = function(matchId) {
    if (window.matchTimers && window.matchTimers[matchId]) {
        window.matchTimers[matchId].stop();
    }
};

window.addTimeToTimer = function(matchId, seconds) {
    if (window.matchTimers && window.matchTimers[matchId]) {
        window.matchTimers[matchId].addTime(seconds);
    }
};
```

4. Файл static/css/bracket.css - стили для турнирной сетки

```css
/* Стили для турнирной сетки */

.bracket-container {
    width: 100%;
    overflow-x: auto;
    padding: 20px;
    background: #f8fafc;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.bracket-wrapper {
    display: flex;
    gap: 40px;
    min-width: min-content;
    padding: 20px;
    transition: transform 0.3s ease;
}

.round {
    display: flex;
    flex-direction: column;
    gap: 30px;
    min-width: 280px;
}

.round.current {
    background: linear-gradient(135deg, rgba(79, 70, 229, 0.05) 0%, rgba(124, 58, 237, 0.05) 100%);
    padding: 15px;
    border-radius: 10px;
    border: 2px solid #4f46e5;
}

.round-header {
    text-align: center;
    padding: 12px;
    background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
    color: white;
    border-radius: 8px;
    font-weight: 600;
    font-size: 16px;
    margin-bottom: 10px;
}

.matches-container {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.match {
    background: white;
    border-radius: 10px;
    padding: 15px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    border: 2px solid #e2e8f0;
    transition: all 0.3s ease;
    min-width: 250px;
}

.match:hover {
    border-color: #4f46e5;
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.match.scheduled {
    border-left: 4px solid #e2e8f0;
}

.match.in_progress {
    border-left: 4px solid #f59e0b;
    animation: pulse 2s infinite;
}

.match.completed {
    border-left: 4px solid #10b981;
}

.match-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;
    padding-bottom: 10px;
    border-bottom: 1px solid #e2e8f0;
}

.match-info {
    font-size: 12px;
    color: #6b7280;
    font-weight: 600;
}

.match-status {
    padding: 4px 10px;
    border-radius: 12px;
    font-size: 11px;
    font-weight: 600;
}

.status-scheduled {
    background: #e2e8f0;
    color: #4b5563;
}

.status-in_progress {
    background: #fef3c7;
    color: #92400e;
}

.status-completed {
    background: #d1fae5;
    color: #065f46;
}

.match-body {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.participant {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 12px;
    border-radius: 6px;
    background: #f8fafc;
    border: 2px solid transparent;
    transition: all 0.3s ease;
    cursor: pointer;
}

.participant:hover {
    background: #edf2f7;
    border-color: #cbd5e0;
}

.participant.winner {
    background: linear-gradient(135deg, #d1fae5 0%, #a7f3d0 100%);
    border-color: #10b981;
    font-weight: 600;
}

.participant.empty {
    background: #f1f5f9;
    color: #94a3b8;
    justify-content: center;
    cursor: default;
}

.participant.empty:hover {
    background: #f1f5f9;
    border-color: transparent;
}

.participant-info {
    flex-grow: 1;
}

.participant-name {
    font-weight: 500;
    color: #1f2937;
    margin-bottom: 4px;
}

.participant-details {
    display: flex;
    gap: 8px;
    font-size: 11px;
    color: #6b7280;
}

.seed {
    background: #4f46e5;
    color: white;
    padding: 2px 6px;
    border-radius: 4px;
    font-weight: 600;
}

.country {
    background: #e0e7ff;
    color: #4f46e5;
    padding: 2px 6px;
    border-radius: 4px;
}

.participant-score {
    font-size: 20px;
    font-weight: 700;
    color: #1f2937;
    min-width: 40px;
    text-align: center;
}

.match-vs {
    text-align: center;
    padding: 8px 0;
    color: #9ca3af;
    font-weight: 600;
    font-size: 14px;
}

.match-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 12px;
    padding-top: 10px;
    border-top: 1px solid #e2e8f0;
    font-size: 12px;
    color: #6b7280;
}

.match-time, .match-arena {
    padding: 4px 8px;
    background: #f3f4f6;
    border-radius: 4px;
}

.match-actions {
    display: flex;
    gap: 8px;
}

.btn-action {
    padding: 6px 12px;
    border: none;
    border-radius: 6px;
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
}

.btn-start {
    background: #10b981;
    color: white;
}

.btn-start:hover {
    background: #059669;
}

.btn-update {
    background: #3b82f6;
    color: white;
}

.btn-update:hover {
    background: #2563eb;
}

/* Анимация пульсации для текущих матчей */
@keyframes pulse {
    0% {
        box-shadow: 0 0 0 0 rgba(245, 158, 11, 0.4);
    }
    70% {
        box-shadow: 0 0 0 10px rgba(245, 158, 11, 0);
    }
    100% {
        box-shadow: 0 0 0 0 rgba(245, 158, 11, 0);
    }
}

/* Стили для соединений между матчами */
.bracket-wrapper::before {
    content: '';
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    pointer-events: none;
}

.connector {
    position: absolute;
    background: #cbd5e0;
    z-index: -1;
}

.connector.horizontal {
    height: 2px;
}

.connector.vertical {
    width: 2px;
}

/* Стили для модального окна */
.match-modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
}

.match-modal .modal-content {
    background: white;
    border-radius: 12px;
    width: 90%;
    max-width: 500px;
    animation: modalAppear 0.3s ease;
}

@keyframes modalAppear {
    from {
        opacity: 0;
        transform: translateY(-50px) scale(0.9);
    }
    to {
        opacity: 1;
        transform: translateY(0) scale(1);
    }
}

.match-modal .modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px;
    background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
    color: white;
    border-radius: 12px 12px 0 0;
}

.match-modal .modal-header h3 {
    margin: 0;
    font-size: 18px;
}

.match-modal .close {
    background: none;
    border: none;
    color: white;
    font-size: 24px;
    cursor: pointer;
    padding: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: background-color 0.3s ease;
}

.match-modal .close:hover {
    background: rgba(255, 255, 255, 0.2);
}

.match-modal .modal-body {
    padding: 20px;
}

.match-modal .match-preview {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 20px;
    padding: 20px;
    background: #f8fafc;
    border-radius: 8px;
}

.match-modal .participant {
    flex: 1;
    text-align: center;
    padding: 15px;
}

.match-modal .participant .name {
    font-weight: 600;
    font-size: 16px;
    margin-bottom: 10px;
    color: #1f2937;
}

.match-modal .score-input {
    width: 80px;
    padding: 10px;
    border: 2px solid #e2e8f0;
    border-radius: 6px;
    font-size: 18px;
    font-weight: 700;
    text-align: center;
    transition: border-color 0.3s ease;
}

.match-modal .score-input:focus {
    outline: none;
    border-color: #4f46e5;
}

.match-modal .vs {
    padding: 0 20px;
    font-weight: 600;
    color: #6b7280;
}

.match-modal .form-group {
    margin-bottom: 20px;
}

.match-modal .form-group label {
    display: block;
    margin-bottom: 8px;
    font-weight: 500;
    color: #374151;
}

.match-modal .form-group select {
    width: 100%;
    padding: 10px;
    border: 2px solid #e2e8f0;
    border-radius: 6px;
    font-size: 14px;
    transition: border-color 0.3s ease;
}

.match-modal .form-group select:focus {
    outline: none;
    border-color: #4f46e5;
}

.match-modal .modal-footer {
    padding: 20px;
    background: #f9fafb;
    display: flex;
    justify-content: flex-end;
    gap: 12px;
    border-radius: 0 0 12px 12px;
}

.match-modal .btn-cancel,
.match-modal .btn-save {
    padding: 10px 20px;
    border: none;
    border-radius: 6px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
}

.match-modal .btn-cancel {
    background: #e5e7eb;
    color: #374151;
}

.match-modal .btn-cancel:hover {
    background: #d1d5db;
}

.match-modal .btn-save {
    background: #4f46e5;
    color: white;
}

.match-modal .btn-save:hover {
    background: #4338ca;
}

/* Стили для таймера */
.timer-display {
    font-family: 'Courier New', monospace;
    font-size: 32px;
    font-weight: 700;
    text-align: center;
    padding: 15px;
    background: #1f2937;
    color: #10b981;
    border-radius: 8px;
    margin: 20px 0;
    letter-spacing: 2px;
}

.timer-display.warning {
    color: #ef4444;
    animation: blink 1s infinite;
}

@keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
}

.timer-controls {
    display: flex;
    gap: 10px;
    justify-content: center;
}

.timer-controls button {
    padding: 10px 20px;
    border: none;
    border-radius: 6px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
}

.timer-controls button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

/* Стили для уведомлений */
.notification {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 15px 20px;
    border-radius: 8px;
    color: white;
    font-weight: 500;
    z-index: 1001;
    animation: slideIn 0.3s ease;
    max-width: 300px;
}

@keyframes slideIn {
    from {
        transform: translateX(100%);
        opacity: 0;
    }
    to {
        transform: translateX(0);
        opacity: 1;
    }
}

.notification.success {
    background: #10b981;
}

.notification.error {
    background: #ef4444;
}

.notification.warning {
    background: #f59e0b;
}

.notification.info {
    background: #3b82f6;
}

/* Адаптивность */
@media (max-width: 768px) {
    .bracket-wrapper {
        gap: 20px;
        padding: 10px;
    }
    
    .round {
        min-width: 240px;
    }
    
    .match {
        min-width: 200px;
    }
    
    .participant-name {
        font-size: 14px;
    }
    
    .participant-score {
        font-size: 18px;
    }
    
    .match-modal .match-preview {
        flex-direction: column;
        gap: 15px;
    }
    
    .match-modal .vs {
        padding: 10px 0;
    }
}

@media (max-width: 480px) {
    .bracket-container {
        padding: 10px;
    }
    
    .round {
        min-width: 200px;
    }
    
    .match {
        min-width: 180px;
        padding: 10px;
    }
    
    .participant {
        padding: 8px;
    }
}
```

5. Файл static/js/scoring.js - JavaScript для системы оценок

```javascript
// Функции для системы оценок

class ScoringSystem {
    constructor(tournamentId) {
        this.tournamentId = tournamentId;
        this.participants = [];
        this.criteria = [];
        this.scores = {};
        this.currentParticipantId = null;
    }

    async initialize() {
        await this.loadParticipants();
        await this.loadCriteria();
        await this.loadScores();
        this.renderParticipantsList();
        this.setupEventListeners();
    }

    async loadParticipants() {
        try {
            const response = await fetch(`/api/tournaments/${this.tournamentId}/participants`);
            this.participants = await response.json();
        } catch (error) {
            console.error('Ошибка загрузки участников:', error);
            this.showError('Не удалось загрузить список участников');
        }
    }

    async loadCriteria() {
        try {
            const response = await fetch(`/api/tournaments/${this.tournamentId}/criteria`);
            this.criteria = await response.json();
            this.renderCriteriaForm();
        } catch (error) {
            console.error('Ошибка загрузки критериев:', error);
            this.showError('Не удалось загрузить критерии оценки');
        }
    }

    async loadScores() {
        try {
            const response = await fetch(`/api/tournaments/${this.tournamentId}/scores`);
            const allScores = await response.json();
            
            // Группируем оценки по участникам
            this.scores = {};
            allScores.forEach(score => {
                if (!this.scores[score.participant_id]) {
                    this.scores[score.participant_id] = [];
                }
                this.scores[score.participant_id].push(score);
            });
        } catch (error) {
            console.error('Ошибка загрузки оценок:', error);
        }
    }

    renderParticipantsList() {
        const container = document.getElementById('participantsList');
        if (!container) return;

        container.innerHTML = '';

        // Сортируем участников по оценкам
        const sortedParticipants = [...this.participants].sort((a, b) => {
            const scoreA = this.calculateParticipantScore(a.id);
            const scoreB = this.calculateParticipantScore(b.id);
            return (scoreB?.final_score || 0) - (scoreA?.final_score || 0);
        });

        sortedParticipants.forEach(participant => {
            const participantScore = this.calculateParticipantScore(participant.id);
            const item = document.createElement('div');
            item.className = `participant-item ${participant.id === this.currentParticipantId ? 'active' : ''}`;
            item.dataset.participantId = participant.id;

            item.innerHTML = `
                <div class="participant-info">
                    <div class="participant-name">${participant.athlete.full_name}</div>
                    <div class="participant-details">
                        ${participant.seed ? `<span class="seed">#${participant.seed}</span>` : ''}
                        ${participant.category?.name ? `<span class="category">${participant.category.name}</span>` : ''}
                    </div>
                </div>
                <div class="participant-score">
                    ${participantScore?.final_score?.toFixed(2) || '—'}
                </div>
            `;

            item.addEventListener('click', () => this.selectParticipant(participant.id));
            container.appendChild(item);
        });
    }

    renderCriteriaForm() {
        const container = document.getElementById('criteriaGrid');
        if (!container || !this.currentParticipantId) return;

        container.innerHTML = '';

        this.criteria.forEach(criterion => {
            const participantScores = this.scores[this.currentParticipantId] || [];
            const criterionScore = participantScores.find(s => s.criterion === criterion.name);

            const criterionElement = document.createElement('div');
            criterionElement.className = 'criterion-item';
            criterionElement.innerHTML = `
                <div class="criterion-header">
                    <div class="criterion-name">${criterion.name}</div>
                    <div class="criterion-weight">Вес: ${criterion.weight || 1}</div>
                </div>
                ${criterion.description ? `<div class="criterion-description">${criterion.description}</div>` : ''}
                <input type="number" 
                       class="score-input" 
                       data-criterion="${criterion.name}"
                       min="0" 
                       max="${criterion.max_value || 10}" 
                       step="0.1"
                       value="${criterionScore?.value || ''}"
                       placeholder="0-${criterion.max_value || 10}">
                <div class="score-hint">Максимум: ${criterion.max_value || 10} баллов</div>
            `;

            // Добавляем обработчик изменения оценки
            criterionElement.querySelector('.score-input').addEventListener('input', () => {
                this.updateTotalScore();
            });

            container.appendChild(criterionElement);
        });

        this.updateTotalScore();
    }

    selectParticipant(participantId) {
        this.currentParticipantId = participantId;
        
        // Обновляем выделение в списке
        document.querySelectorAll('.participant-item').forEach(item => {
            item.classList.toggle('active', item.dataset.participantId == participantId);
        });

        // Показываем форму оценки
        document.getElementById('noParticipantSelected')?.style.display = 'none';
        document.getElementById('scoringFormContainer')?.style.display = 'block';

        // Обновляем информацию об участнике
        const participant = this.participants.find(p => p.id == participantId);
        if (participant) {
            document.getElementById('selectedParticipantName').textContent = participant.athlete.full_name;
            document.getElementById('participantSeed').textContent = participant.seed ? `Seed: #${participant.seed}` : '';
            document.getElementById('participantCategory').textContent = participant.category?.name || '';
        }

        this.renderCriteriaForm();
        this.renderJudgeScores();
    }

    calculateParticipantScore(participantId) {
        const participantScores = this.scores[participantId];
        if (!participantScores || participantScores.length === 0) {
            return null;
        }

        // Группируем оценки по критериям
        const criteriaScores = {};
        participantScores.forEach(score => {
            if (!criteriaScores[score.criterion]) {
                criteriaScores[score.criterion] = [];
            }
            criteriaScores[score.criterion].push(score);
        });

        // Рассчитываем средний балл по каждому критерию
        let totalWeightedScore = 0;
        let totalWeight = 0;
        const criterionDetails = [];

        Object.entries(criteriaScores).forEach(([criterion, scores]) => {
            const criterionInfo = this.criteria.find(c => c.name === criterion);
            const maxValue = criterionInfo?.max_value || 10;
            const weight = criterionInfo?.weight || 1;

            // Средняя оценка по этому критерию
            const avgScore = scores.reduce((sum, s) => sum + s.value, 0) / scores.length;
            
            // Нормализуем относительно максимума
            const normalizedScore = (avgScore / maxValue) * 10;
            
            totalWeightedScore += normalizedScore * weight;
            totalWeight += weight;

            criterionDetails.push({
                criterion: criterion,
                average: avgScore.toFixed(2),
                normalized: normalizedScore.toFixed(2),
                weight: weight,
                scores_count: scores.length
            });
        });

        const finalScore = totalWeight > 0 ? totalWeightedScore / totalWeight : 0;

        return {
            final_score: finalScore,
            criteria: criterionDetails,
            total_scores: participantScores.length
        };
    }

    updateTotalScore() {
        if (!this.currentParticipantId) return;

        let totalWeightedScore = 0;
        let totalWeight = 0;
        const scores = [];

        this.criteria.forEach(criterion => {
            const input = document.querySelector(`input[data-criterion="${criterion.name}"]`);
            if (!input) return;

            const value = parseFloat(input.value) || 0;
            const maxValue = criterion.max_value || 10;
            const weight = criterion.weight || 1;

            // Нормализуем оценку
            const normalizedScore = (value / maxValue) * 10;
            
            totalWeightedScore += normalizedScore * weight;
            totalWeight += weight;
            
            scores.push({
                criterion: criterion.name,
                value: value,
                normalized: normalizedScore,
                weight: weight
            });
        });

        const finalScore = totalWeight > 0 ? totalWeightedScore / totalWeight : 0;

        // Обновляем отображение
        const totalScoreElement = document.getElementById('totalScore');
        const breakdownElement = document.getElementById('scoreBreakdown');

        if (totalScoreElement) {
            totalScoreElement.textContent = finalScore.toFixed(2);
        }

        if (breakdownElement) {
            breakdownElement.textContent = `Средний балл: ${finalScore.toFixed(2)} | Максимум: 10.0`;
        }
    }

    renderJudgeScores() {
        if (!this.currentParticipantId) return;

        const container = document.getElementById('judgeScores');
        if (!container) return;

        const participantScores = this.scores[this.currentParticipantId] || [];
        
        // Группируем оценки по судьям
        const judgeScores = {};
        participantScores.forEach(score => {
            const judgeName = score.judge?.full_name || score.judge_name || 'Неизвестный судья';
            if (!judgeScores[judgeName]) {
                judgeScores[judgeName] = [];
            }
            judgeScores[judgeName].push(score);
        });

        container.innerHTML = '';

        Object.entries(judgeScores).forEach(([judgeName, scores]) => {
            const totalScore = scores.reduce((sum, s) => sum + s.value, 0);
            const avgScore = scores.length > 0 ? totalScore / scores.length : 0;

            const judgeElement = document.createElement('div');
            judgeElement.className = 'judge-score-item';
            judgeElement.innerHTML = `
                <div class="judge-name">
                    <i class="fas fa-gavel"></i> ${judgeName}
                </div>
                <div class="judge-average">${avgScore.toFixed(2)}</div>
            `;
            container.appendChild(judgeElement);
        });

        if (Object.keys(judgeScores).length === 0) {
            container.innerHTML = '<p>Оценки судей отсутствуют</p>';
        }
    }

    async saveScores() {
        if (!this.currentParticipantId) {
            this.showError('Пожалуйста, выберите участника');
            return;
        }

        const judgeName = prompt('Введите имя судьи:', 'Судья 1');
        if (!judgeName) {
            this.showError('Имя судьи обязательно');
            return;
        }

        const scoresToSave = [];
        this.criteria.forEach(criterion => {
            const input = document.querySelector(`input[data-criterion="${criterion.name}"]`);
            const value = parseFloat(input.value);
            
            if (!isNaN(value)) {
                scoresToSave.push({
                    participant_id: this.currentParticipantId,
                    criterion: criterion.name,
                    value: value,
                    max_value: criterion.max_value || 10,
                    weight: criterion.weight || 1,
                    judge_name: judgeName
                });
            }
        });

        if (scoresToSave.length === 0) {
            this.showError('Пожалуйста, введите хотя бы одну оценку');
            return;
        }

        try {
            const response = await fetch('/api/scores/batch', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    scores: scoresToSave,
                    judge_name: judgeName
                })
            });

            if (response.ok) {
                this.showSuccess('Оценки успешно сохранены');
                await this.loadScores();
                this.renderParticipantsList();
                this.renderJudgeScores();
            } else {
                throw new Error('Ошибка сохранения оценок');
            }
        } catch (error) {
            console.error('Ошибка сохранения оценок:', error);
            this.showError('Не удалось сохранить оценки');
        }
    }

    setupEventListeners() {
        // Кнопка сохранения оценок
        document.getElementById('saveScoreBtn')?.addEventListener('click', () => this.saveScores());

        // Кнопка сброса оценок
        document.getElementById('resetScoreBtn')?.addEventListener('click', () => this.resetScores());

        // Кнопки сортировки
        document.getElementById('sortByNameBtn')?.addEventListener('click', () => this.sortParticipantsByName());
        document.getElementById('sortByScoreBtn')?.addEventListener('click', () => this.sortParticipantsByScore());

        // Поиск участников
        const searchInput = document.getElementById('participantSearch');
        if (searchInput) {
            searchInput.addEventListener('input', (e) => this.filterParticipants(e.target.value));
        }
    }

    resetScores() {
        if (!this.currentParticipantId) return;

        if (confirm('Сбросить все введенные оценки для этого участника?')) {
            document.querySelectorAll('.score-input').forEach(input => {
                input.value = '';
            });
            this.updateTotalScore();
        }
    }

    sortParticipantsByName() {
        this.participants.sort((a, b) => 
            a.athlete.full_name.localeCompare(b.athlete.full_name)
        );
        this.renderParticipantsList();
    }

    sortParticipantsByScore() {
        this.participants.sort((a, b) => {
            const scoreA = this.calculateParticipantScore(a.id)?.final_score || 0;
            const scoreB = this.calculateParticipantScore(b.id)?.final_score || 0;
            return scoreB - scoreA;
        });
        this.renderParticipantsList();
    }

    filterParticipants(searchTerm) {
        const container = document.getElementById('participantsList');
        if (!container) return;

        const items = container.querySelectorAll('.participant-item');
        const term = searchTerm.toLowerCase();

        items.forEach(item => {
            const text = item.textContent.toLowerCase();
            item.style.display = text.includes(term) ? 'block' : 'none';
        });
    }

    showSuccess(message) {
        this.showNotification(message, 'success');
    }

    showError(message) {
        this.showNotification(message, 'error');
    }

    showNotification(message, type = 'info') {
        const notification = document.createElement('div');
        notification.className = `notification notification-${type}`;
        notification.textContent = message;

        document.body.appendChild(notification);

        setTimeout(() => {
            notification.remove();
        }, 3000);
    }
}

// Инициализация при загрузке страницы
document.addEventListener('DOMContentLoaded', function() {
    const tournamentId = window.tournamentId || document.getElementById('tournamentId')?.value;
    
    if (tournamentId) {
        const scoringSystem = new ScoringSystem(tournamentId);
        scoringSystem.initialize();
        
        // Сохраняем объект в глобальной области видимости
        window.scoringSystem = scoringSystem;
    }
});

// Функции для таблицы результатов
async function loadLeaderboard(tournamentId) {
    try {
        const response = await fetch(`/api/tournaments/${tournamentId}/leaderboard`);
        const leaderboard = await response.json();
        
        renderLeaderboardTable(leaderboard);
    } catch (error) {
        console.error('Ошибка загрузки таблицы результатов:', error);
    }
}

function renderLeaderboardTable(leaderboard) {
    const tbody = document.getElementById('leaderboardBody');
    if (!tbody) return;

    tbody.innerHTML = '';

    leaderboard.forEach((entry, index) => {
        const row = document.createElement('tr');
        
        let medalIcon = '';
        if (index === 0) medalIcon = '🥇';
        else if (index === 1) medalIcon = '🥈';
        else if (index === 2) medalIcon = '🥉';

        row.innerHTML = `
            <td class="position-${index + 1}">
                ${index + 1}
                ${medalIcon}
            </td>
            <td>
                <strong>${entry.participant.athlete.full_name}</strong>
                <div class="participant-details">
                    ${entry.participant.seed ? `<span class="seed">#${entry.participant.seed}</span>` : ''}
                    ${entry.participant.category?.name ? `<span class="category">${entry.participant.category.name}</span>` : ''}
                </div>
            </td>
            <td>${entry.final_score?.toFixed(2) || '—'}</td>
            <td>${getMaxScore(entry.score_details)}</td>
            <td>${getMinScore(entry.score_details)}</td>
            <td>${calculateStdDev(entry.score_details)}</td>
            <td>${medalIcon || '—'}</td>
        `;

        tbody.appendChild(row);
    });
}

function getMaxScore(scoreDetails) {
    if (!scoreDetails || scoreDetails.length === 0) return '—';
    const maxScore = Math.max(...scoreDetails.map(s => s.value));
    return maxScore.toFixed(2);
}

function getMinScore(scoreDetails) {
    if (!scoreDetails || scoreDetails.length === 0) return '—';
    const minScore = Math.min(...scoreDetails.map(s => s.value));
    return minScore.toFixed(2);
}

function calculateStdDev(scoreDetails) {
    if (!scoreDetails || scoreDetails.length < 2) return '—';
    
    const values = scoreDetails.map(s => s.value);
    const mean = values.reduce((a, b) => a + b) / values.length;
    const variance = values.reduce((a, b) => a + Math.pow(b - mean, 2), 0) / values.length;
    const stdDev = Math.sqrt(variance);
    
    return stdDev.toFixed(2);
}

// Функции для экспорта результатов
async function exportLeaderboard(format) {
    const tournamentId = window.tournamentId;
    if (!tournamentId) return;

    try {
        const response = await fetch(`/api/tournaments/${tournamentId}/leaderboard/export?format=${format}`);
        
        if (response.ok) {
            const blob = await response.blob();
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `results_${tournamentId}.${format}`;
            document.body.appendChild(a);
            a.click();
            window.URL.revokeObjectURL(url);
            document.body.removeChild(a);
        } else {
            throw new Error('Ошибка экспорта');
        }
    } catch (error) {
        console.error('Ошибка экспорта результатов:', error);
        alert('Не удалось экспортировать результаты');
    }
}

// Глобальные функции для доступа из HTML
window.exportToPDF = function() {
    exportLeaderboard('pdf');
};

window.exportToExcel = function() {
    exportLeaderboard('xlsx');
};

window.exportToCSV = function() {
    exportLeaderboard('csv');
};

window.printResults = function() {
    window.print();
};
```

6. Файл static/js/main.js - основной JavaScript файл

```javascript
// Основные функции для всего приложения

// Инициализация при загрузке страницы
document.addEventListener('DOMContentLoaded', function() {
    initializeTheme();
    setupNavigation();
    loadActiveTournaments();
    setupNotifications();
    initializeDatePickers();
    setupFormValidation();
});

// Управление темой (светлая/темная)
function initializeTheme() {
    const themeToggle = document.getElementById('themeToggle');
    const savedTheme = localStorage.getItem('theme') || 'light';
    
    document.body.classList.toggle('dark-theme', savedTheme === 'dark');
    
    if (themeToggle) {
        themeToggle.checked = savedTheme === 'dark';
        themeToggle.addEventListener('change', function() {
            const isDark = this.checked;
            document.body.classList.toggle('dark-theme', isDark);
            localStorage.setItem('theme', isDark ? 'dark' : 'light');
        });
    }
}

// Настройка навигации
function setupNavigation() {
    // Активное состояние для текущей страницы
    const currentPath = window.location.pathname;
    document.querySelectorAll('.main-nav a, .sidebar-menu a').forEach(link => {
        if (link.getAttribute('href') === currentPath) {
            link.classList.add('active');
        }
    });
    
    // Мобильное меню
    const mobileMenuBtn = document.getElementById('mobileMenuBtn');
    const sidebar = document.querySelector('.sidebar');
    
    if (mobileMenuBtn && sidebar) {
        mobileMenuBtn.addEventListener('click', function() {
            sidebar.classList.toggle('mobile-open');
        });
    }
    
    // Закрытие мобильного меню при клике вне его
    document.addEventListener('click', function(event) {
        if (sidebar && !sidebar.contains(event.target) && 
            mobileMenuBtn && !mobileMenuBtn.contains(event.target)) {
            sidebar.classList.remove('mobile-open');
        }
    });
}

// Загрузка активных турниров
async function loadActiveTournaments() {
    try {
        const response = await fetch('/api/tournaments?status=active');
        const tournaments = await response.json();
        
        const container = document.getElementById('activeTournaments');
        if (!container) return;
        
        container.innerHTML = '';
        
        tournaments.slice(0, 5).forEach(tournament => {
            const item = document.createElement('div');
            item.className = 'active-tournament-item';
            item.innerHTML = `
                <div class="tournament-name">${tournament.name}</div>
                <div class="tournament-progress">
                    <div class="progress-bar">
                        <div class="progress" style="width: ${calculateProgress(tournament)}%"></div>
                    </div>
                    <span class="progress-text">${calculateProgress(tournament)}%</span>
                </div>
                <a href="/tournament/${tournament.id}" class="btn-view">Просмотр</a>
            `;
            container.appendChild(item);
        });
        
        if (tournaments.length === 0) {
            container.innerHTML = '<p class="no-tournaments">Нет активных турниров</p>';
        }
    } catch (error) {
        console.error('Ошибка загрузки активных турниров:', error);
    }
}

function calculateProgress(tournament) {
    // Простая логика расчета прогресса
    // В реальном приложении можно использовать количество завершенных матчей
    if (tournament.status === 'completed') return 100;
    if (tournament.status === 'pending') return 0;
    
    // Для активных турниров - случайное значение (в реальном приложении нужно считать)
    return Math.floor(Math.random() * 100);
}

// Настройка уведомлений
function setupNotifications() {
    const notificationBtn = document.getElementById('notificationBtn');
    const notificationPanel = document.getElementById('notificationPanel');
    
    if (notificationBtn && notificationPanel) {
        notificationBtn.addEventListener('click', function() {
            notificationPanel.classList.toggle('show');
            markNotificationsAsRead();
        });
        
        // Закрытие панели при клике вне ее
        document.addEventListener('click', function(event) {
            if (!notificationBtn.contains(event.target) && 
                !notificationPanel.contains(event.target)) {
                notificationPanel.classList.remove('show');
            }
        });
    }
    
    // Загрузка уведомлений
    loadNotifications();
}

async function loadNotifications() {
    try {
        const response = await fetch('/api/notifications');
        const notifications = await response.json();
        
        const container = document.getElementById('notificationsList');
        if (!container) return;
        
        container.innerHTML = '';
        
        notifications.forEach(notification => {
            const item = document.createElement('div');
            item.className = `notification-item ${notification.unread ? 'unread' : ''}`;
            item.innerHTML = `
                <div class="notification-icon">
                    <i class="fas ${getNotificationIcon(notification.type)}"></i>
                </div>
                <div class="notification-content">
                    <div class="notification-title">${notification.title}</div>
                    <div class="notification-message">${notification.message}</div>
                    <div class="notification-time">${formatTimeAgo(notification.created_at)}</div>
                </div>
                ${notification.unread ? '<div class="notification-dot"></div>' : ''}
            `;
            container.appendChild(item);
        });
        
        // Обновляем счетчик
        const unreadCount = notifications.filter(n => n.unread).length;
        updateNotificationBadge(unreadCount);
        
    } catch (error) {
        console.error('Ошибка загрузки уведомлений:', error);
    }
}

function getNotificationIcon(type) {
    const icons = {
        'info': 'fa-info-circle',
        'success': 'fa-check-circle',
        'warning': 'fa-exclamation-triangle',
        'error': 'fa-times-circle',
        'match': 'fa-fist-raised',
        'score': 'fa-star',
        'result': 'fa-trophy'
    };
    return icons[type] || 'fa-bell';
}

function formatTimeAgo(dateString) {
    const date = new Date(dateString);
    const now = new Date();
    const diffMs = now - date;
    const diffMins = Math.floor(diffMs / 60000);
    const diffHours = Math.floor(diffMs / 3600000);
    const diffDays = Math.floor(diffMs / 86400000);
    
    if (diffMins < 1) return 'Только что';
    if (diffMins < 60) return `${diffMins} мин назад`;
    if (diffHours < 24) return `${diffHours} ч назад`;
    if (diffDays === 1) return 'Вчера';
    return `${diffDays} дн назад`;
}

function updateNotificationBadge(count) {
    const badge = document.getElementById('notificationBadge');
    if (badge) {
        badge.textContent = count;
        badge.style.display = count > 0 ? 'flex' : 'none';
    }
}

async function markNotificationsAsRead() {
    try {
        await fetch('/api/notifications/mark-read', {
            method: 'POST'
        });
        
        // Обновляем иконки уведомлений
        document.querySelectorAll('.notification-item.unread').forEach(item => {
            item.classList.remove('unread');
            item.querySelector('.notification-dot')?.remove();
        });
        
        updateNotificationBadge(0);
    } catch (error) {
        console.error('Ошибка при отметке уведомлений как прочитанных:', error);
    }
}

// Инициализация date pickers
function initializeDatePickers() {
    // Flatpickr или нативный input[type="date"]
    const dateInputs = document.querySelectorAll('input[type="date"], input[type="datetime-local"]');
    
    dateInputs.forEach(input => {
        // Устанавливаем минимальную дату на сегодня
        if (!input.value) {
            const today = new Date().toISOString().split('T')[0];
            input.min = today;
            
            // Для datetime-local устанавливаем текущее время
            if (input.type === 'datetime-local') {
                const now = new Date();
                const localDateTime = now.toISOString().slice(0, 16);
                input.value = localDateTime;
            }
        }
    });
}

// Настройка валидации форм
function setupFormValidation() {
    const forms = document.querySelectorAll('form[data-validate]');
    
    forms.forEach(form => {
        form.addEventListener('submit', function(event) {
            if (!validateForm(this)) {
                event.preventDefault();
                event.stopPropagation();
            }
        });
        
        // Добавляем обработчики для инпутов
        const inputs = form.querySelectorAll('input[required], select[required], textarea[required]');
        inputs.forEach(input => {
            input.addEventListener('blur', function() {
                validateField(this);
            });
        });
    });
}

function validateForm(form) {
    let isValid = true;
    const inputs = form.querySelectorAll('input[required], select[required], textarea[required]');
    
    inputs.forEach(input => {
        if (!validateField(input)) {
            isValid = false;
        }
    });
    
    return isValid;
}

function validateField(field) {
    const value = field.value.trim();
    const errorElement = field.nextElementSibling?.classList.contains('error-message') 
        ? field.nextElementSibling 
        : null;
    
    let isValid = true;
    let errorMessage = '';
    
    // Проверка на обязательное поле
    if (field.hasAttribute('required') && !value) {
        isValid = false;
        errorMessage = 'Это поле обязательно для заполнения';
    }
    
    // Проверка email
    else if (field.type === 'email' && value) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(value)) {
            isValid = false;
            errorMessage = 'Введите корректный email адрес';
        }
    }
    
    // Проверка числа
    else if (field.type === 'number' && value) {
        const min = field.getAttribute('min');
        const max = field.getAttribute('max');
        
        if (min && parseFloat(value) < parseFloat(min)) {
            isValid = false;
            errorMessage = `Значение должно быть не менее ${min}`;
        } else if (max && parseFloat(value) > parseFloat(max)) {
            isValid = false;
            errorMessage = `Значение должно быть не более ${max}`;
        }
    }
    
    // Проверка даты
    else if ((field.type === 'date' || field.type === 'datetime-local') && value) {
        const selectedDate = new Date(value);
        const today = new Date();
        
        if (field.getAttribute('min') && selectedDate < new Date(field.getAttribute('min'))) {
            isValid = false;
            errorMessage = 'Дата не может быть раньше минимальной';
        }
    }
    
    // Обновляем состояние поля
    field.classList.toggle('invalid', !isValid);
    field.classList.toggle('valid', isValid && value);
    
    // Обновляем сообщение об ошибке
    if (errorElement) {
        errorElement.textContent = errorMessage;
        errorElement.style.display = errorMessage ? 'block' : 'none';
    } else if (errorMessage) {
        // Создаем элемент для сообщения об ошибке
        const newErrorElement = document.createElement('div');
        newErrorElement.className = 'error-message';
        newErrorElement.textContent = errorMessage;
        newErrorElement.style.color = '#ef4444';
        newErrorElement.style.fontSize = '12px';
        newErrorElement.style.marginTop = '4px';
        
        field.parentNode.insertBefore(newErrorElement, field.nextSibling);
    }
    
    return isValid;
}

// Функции для работы с модальными окнами
function openModal(modalId) {
    const modal = document.getElementById(modalId);
    if (modal) {
        modal.style.display = 'flex';
        document.body.style.overflow = 'hidden'; // Блокируем скролл страницы
    }
}

function closeModal(modalId) {
    const modal = document.getElementById(modalId);
    if (modal) {
        modal.style.display = 'none';
        document.body.style.overflow = 'auto'; // Восстанавливаем скролл
    }
}

// Закрытие модальных окон при клике вне их
document.addEventListener('click', function(event) {
    if (event.target.classList.contains('modal')) {
        closeModal(event.target.id);
    }
});

// Закрытие модальных окон по клавише ESC
document.addEventListener('keydown', function(event) {
    if (event.key === 'Escape') {
        const modals = document.querySelectorAll('.modal[style*="display: flex"]');
        modals.forEach(modal => closeModal(modal.id));
    }
});

// Функции для работы с таблицами
function sortTable(tableId, columnIndex, isNumeric = false) {
    const table = document.getElementById(tableId);
    if (!table) return;
    
    const tbody = table.querySelector('tbody');
    const rows = Array.from(tbody.querySelectorAll('tr'));
    const isAscending = table.dataset.sortColumn === columnIndex.toString() 
        ? table.dataset.sortOrder !== 'asc' 
        : true;
    
    rows.sort((a, b) => {
        const aValue = a.cells[columnIndex].textContent.trim();
        const bValue = b.cells[columnIndex].textContent.trim();
        
        let comparison = 0;
        
        if (isNumeric) {
            const aNum = parseFloat(aValue) || 0;
            const bNum = parseFloat(bValue) || 0;
            comparison = aNum - bNum;
        } else {
            comparison = aValue.localeCompare(bValue, 'ru');
        }
        
        return isAscending ? comparison : -comparison;
    });
    
    // Удаляем старые строки
    while (tbody.firstChild) {
        tbody.removeChild(tbody.firstChild);
    }
    
    // Добавляем отсортированные строки
    rows.forEach(row => tbody.appendChild(row));
    
    // Обновляем индикаторы сортировки
    table.querySelectorAll('th').forEach((th, index) => {
        th.classList.remove('sorted-asc', 'sorted-desc');
        if (index === columnIndex) {
            th.classList.add(isAscending ? 'sorted-asc' : 'sorted-desc');
        }
    });
    
    // Сохраняем состояние сортировки
    table.dataset.sortColumn = columnIndex;
    table.dataset.sortOrder = isAscending ? 'asc' : 'desc';
}

function filterTable(tableId, searchTerm) {
    const table = document.getElementById(tableId);
    if (!table) return;
    
    const rows = table.querySelectorAll('tbody tr');
    const term = searchTerm.toLowerCase();
    
    rows.forEach(row => {
        const text = row.textContent.toLowerCase();
        row.style.display = text.includes(term) ? '' : 'none';
    });
}

// Функции для работы с загрузкой файлов
function setupFileUpload(dropZoneId, inputId) {
    const dropZone = document.getElementById(dropZoneId);
    const fileInput = document.getElementById(inputId);
    
    if (!dropZone || !fileInput) return;
    
    // Клик по зоне загрузки
    dropZone.addEventListener('click', () => fileInput.click());
    
    // Drag and drop
    dropZone.addEventListener('dragover', (e) => {
        e.preventDefault();
        dropZone.classList.add('dragover');
    });
    
    dropZone.addEventListener('dragleave', () => {
        dropZone.classList.remove('dragover');
    });
    
    dropZone.addEventListener('drop', (e) => {
        e.preventDefault();
        dropZone.classList.remove('dragover');
        
        if (e.dataTransfer.files.length) {
            fileInput.files = e.dataTransfer.files;
            updateFilePreview(dropZone, e.dataTransfer.files[0]);
        }
    });
    
    // Изменение выбора файла
    fileInput.addEventListener('change', () => {
        if (fileInput.files.length) {
            updateFilePreview(dropZone, fileInput.files[0]);
        }
    });
}

function updateFilePreview(dropZone, file) {
    const fileName = file.name;
    const fileSize = (file.size / 1024 / 1024).toFixed(2); // MB
    
    dropZone.innerHTML = `
        <div class="file-preview">
            <i class="fas fa-file-alt"></i>
            <div class="file-info">
                <div class="file-name">${fileName}</div>
                <div class="file-size">${fileSize} MB</div>
            </div>
            <button type="button" class="btn-remove-file">
                <i class="fas fa-times"></i>
            </button>
        </div>
    `;
    
    // Кнопка удаления файла
    dropZone.querySelector('.btn-remove-file').addEventListener('click', (e) => {
        e.stopPropagation();
        dropZone.innerHTML = `
            <i class="fas fa-cloud-upload-alt"></i>
            <p>Перетащите файл сюда или нажмите для выбора</p>
            <p class="text-small">Поддерживаемые форматы: CSV, Excel, PDF, изображения</p>
        `;
    });
}

// Утилитные функции
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

function throttle(func, limit) {
    let inThrottle;
    return function() {
        const args = arguments;
        const context = this;
        if (!inThrottle) {
            func.apply(context, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// Глобальные хелперы
window.showLoading = function(element) {
    if (element) {
        element.innerHTML = `
            <div class="loading-spinner">
                <div class="spinner"></div>
                <p>Загрузка...</p>
            </div>
        `;
    }
};

window.hideLoading = function(element, originalContent) {
    if (element && originalContent) {
        element.innerHTML = originalContent;
    }
};

window.showToast = function(message, type = 'info') {
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.innerHTML = `
        <div class="toast-content">
            <i class="fas ${getToastIcon(type)}"></i>
            <span>${message}</span>
        </div>
        <button class="toast-close">&times;</button>
    `;
    
    document.body.appendChild(toast);
    
    // Анимация появления
    setTimeout(() => toast.classList.add('show'), 10);
    
    // Кнопка закрытия
    toast.querySelector('.toast-close').addEventListener('click', () => {
        toast.classList.remove('show');
        setTimeout(() => toast.remove(), 300);
    });
    
    // Автоматическое скрытие
    setTimeout(() => {
        if (toast.parentNode) {
            toast.classList.remove('show');
            setTimeout(() => toast.remove(), 300);
        }
    }, 5000);
};

function getToastIcon(type) {
    const icons = {
        'success': 'fa-check-circle',
        'error': 'fa-times-circle',
        'warning': 'fa-exclamation-triangle',
        'info': 'fa-info-circle'
    };
    return icons[type] || 'fa-bell';
}

// Глобальные обработчики ошибок
window.addEventListener('error', function(event) {
    console.error('Произошла ошибка:', event.error);
    
    // Показываем пользователю сообщение об ошибке
    showToast('Произошла непредвиденная ошибка. Пожалуйста, обновите страницу.', 'error');
});

// Обработка обещаний без catch
window.addEventListener('unhandledrejection', function(event) {
    console.error('Необработанное обещание:', event.reason);
    showToast('Произошла ошибка при выполнении операции.', 'error');
});

// Экспорт глобальных функций
window.openModal = openModal;
window.closeModal = closeModal;
window.sortTable = sortTable;
window.filterTable = filterTable;
window.showToast = showToast;
```

Теперь у вас есть все недостающие файлы для полноценной работы системы проведения соревнований!

Краткое описание созданных файлов:

1. create_bracket.py - логика генерации и управления турнирной сеткой олимпийской системы
2. scoring_system.py - логика работы с системой оценок (расчет баллов, статистика)
3. static/js/tournament.js - JavaScript для работы с турнирной сеткой (отображение, управление матчами)
4. static/css/bracket.css - стили для красивого отображения турнирной сетки
5. static/js/scoring.js - JavaScript для системы оценок (ввод оценок, таблица результатов)
6. static/js/main.js - общие функции для всего приложения

Все файлы готовы к использованию и интегрированы с остальными компонентами системы.

Проблема с match.participant1 = undefined возникает из-за того, что при сериализации матча в JSON связанные объекты (участники, спортсмены) не подгружаются. Нужно явно указать SQLAlchemy загрузить эти отношения с помощью joinedload. Вот исправления для API эндпоинтов в app.py:

```python
from sqlalchemy.orm import joinedload

@app.route('/api/tournaments/<int:tournament_id>/bracket', methods=['GET'])
def tournament_bracket_api(tournament_id):
    # Загружаем матчи вместе с участниками и спортсменами
    matches = Match.query.options(
        joinedload(Match.participant1).joinedload(Participant.athlete),
        joinedload(Match.participant2).joinedload(Participant.athlete),
        joinedload(Match.winner)
    ).filter_by(tournament_id=tournament_id).order_by(Match.round, Match.match_order).all()

    bracket = {}
    for match in matches:
        if match.round not in bracket:
            bracket[match.round] = []
        bracket[match.round].append(match.to_dict())

    tournament = Tournament.query.get(tournament_id)
    participants_count = Participant.query.filter_by(tournament_id=tournament_id).count()
    import math
    total_rounds = int(math.ceil(math.log2(participants_count))) if participants_count > 0 else 0

    return jsonify({
        'bracket': bracket,
        'total_rounds': total_rounds
    })

@app.route('/api/tournaments/<int:tournament_id>/matches/live', methods=['GET'])
def get_live_matches(tournament_id):
    matches = Match.query.options(
        joinedload(Match.participant1).joinedload(Participant.athlete),
        joinedload(Match.participant2).joinedload(Participant.athlete)
    ).filter_by(tournament_id=tournament_id, status='in_progress').all()
    return jsonify([m.to_dict() for m in matches])
```

После этих изменений в JSON будут включены полные данные об участниках и спортсменах, и фронтенд сможет корректно отобразить имена.

Также убедитесь, что в модели Participant определён метод to_dict(), который включает данные спортсмена:

```python
def to_dict(self):
    athlete_data = self.athlete.to_dict() if self.athlete else {}
    # ... остальные поля
    return {
        # ...
        'athlete': athlete_data
    }
```

Теперь данные должны подтягиваться без ошибок.

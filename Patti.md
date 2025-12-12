Вот пример обработчика нажатия на кнопку активации участника с использованием JavaScript и Fetch API:

```javascript
// Обработчик для кнопок активации
document.addEventListener('DOMContentLoaded', function() {
  // Используем делегирование событий для динамически добавленных кнопок
  document.addEventListener('click', function(event) {
    // Проверяем, что клик был по кнопке активации
    if (event.target.classList.contains('activate-btn')) {
      const button = event.target;
      const participantId = button.getAttribute('data-id');
      
      // Блокируем кнопку на время запроса
      button.disabled = true;
      button.textContent = 'Активирую...';
      
      // Отправляем запрос на сервер
      activateParticipant(participantId, button);
    }
  });
});

// Функция активации участника
async function activateParticipant(participantId, button) {
  try {
    const response = await fetch('/api/participants/activate', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').getAttribute('content') // Для Laravel
      },
      body: JSON.stringify({
        id: participantId
      })
    });

    if (response.ok) {
      const result = await response.json();
      
      // Успешная активация
      button.textContent = 'Активен';
      button.classList.remove('btn-primary');
      button.classList.add('btn-success');
      
      // Можно обновить статус в таблице
      updateTableRow(participantId);
      
      // Показываем сообщение об успехе
      showNotification('Участник успешно активирован', 'success');
    } else {
      throw new Error('Ошибка активации');
    }
  } catch (error) {
    // В случае ошибки
    button.disabled = false;
    button.textContent = 'Активировать';
    
    console.error('Ошибка:', error);
    showNotification('Ошибка активации участника', 'error');
  }
}

// Функция обновления строки в таблице
function updateTableRow(participantId) {
  const row = document.querySelector(`tr[data-id="${participantId}"]`);
  if (row) {
    // Обновляем статус в таблице
    const statusCell = row.querySelector('.status-cell');
    if (statusCell) {
      statusCell.textContent = 'Активен';
      statusCell.classList.add('text-success');
    }
    
    // Скрываем кнопку активации
    const activateBtn = row.querySelector('.activate-btn');
    if (activateBtn) {
      activateBtn.style.display = 'none';
    }
  }
}

// Функция показа уведомлений
function showNotification(message, type) {
  // Реализация зависит от вашей системы уведомлений
  // Например, используя Bootstrap:
  const alertDiv = document.createElement('div');
  alertDiv.className = `alert alert-${type === 'success' ? 'success' : 'danger'} alert-dismissible fade show`;
  alertDiv.innerHTML = `
    ${message}
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  `;
  
  document.querySelector('.notifications').appendChild(alertDiv);
  
  // Автоматическое скрытие через 5 секунд
  setTimeout(() => {
    alertDiv.remove();
  }, 5000);
}
```

Серверная часть (пример на PHP/Laravel):

```php
// routes/api.php
Route::post('/api/participants/activate', [ParticipantController::class, 'activate']);

// app/Http/Controllers/ParticipantController.php
public function activate(Request $request)
{
    $validated = $request->validate([
        'id' => 'required|integer|exists:participants,id'
    ]);

    try {
        $participant = Participant::find($validated['id']);
        $participant->is_active = true;
        $participant->activated_at = now();
        $participant->save();

        return response()->json([
            'success' => true,
            'message' => 'Participant activated successfully'
        ]);
    } catch (\Exception $e) {
        return response()->json([
            'success' => false,
            'message' => 'Activation failed'
        ], 500);
    }
}
```

HTML структура таблицы:

```html
<table class="table">
  <thead>
    <tr>
      <th>ID</th>
      <th>Имя</th>
      <th>Статус</th>
      <th>Действия</th>
    </tr>
  </thead>
  <tbody>
    <?php foreach ($participants as $participant): ?>
    <tr data-id="<?= $participant->id ?>">
      <td><?= $participant->id ?></td>
      <td><?= htmlspecialchars($participant->name) ?></td>
      <td class="status-cell">
        <?= $participant->is_active ? 'Активен' : 'Не активен' ?>
      </td>
      <td>
        <?php if (!$participant->is_active): ?>
        <button class="btn btn-sm btn-primary activate-btn" 
                data-id="<?= $participant->id ?>">
          Активировать
        </button>
        <?php endif; ?>
      </td>
    </tr>
    <?php endforeach; ?>
  </tbody>
</table>

<!-- Контейнер для уведомлений -->
<div class="notifications"></div>

<!-- CSRF токен для Laravel -->
<meta name="csrf-token" content="<?= csrf_token() ?>">
```

Ключевые моменты:

1. Делегирование событий - обработчик вешается на документ, чтобы работать с динамически добавленными кнопками
2. Блокировка кнопки - предотвращаем повторные нажатия во время запроса
3. Визуальная обратная связь - меняем текст и стиль кнопки после активации
4. Безопасность - используем CSRF-токен для защиты от межсайтовых запросов
5. Обработка ошибок - предусмотрены сценарии успеха и ошибки
6. Обновление интерфейса - автоматическое обновление статуса в таблице после успешной активации

Альтернативный вариант с использованием AJAX (jQuery):

```javascript
$(document).on('click', '.activate-btn', function() {
  const button = $(this);
  const participantId = button.data('id');
  
  button.prop('disabled', true).text('Активирую...');
  
  $.ajax({
    url: '/api/participants/activate',
    method: 'POST',
    data: {
      id: participantId,
      _token: $('meta[name="csrf-token"]').attr('content')
    },
    success: function(response) {
      button.text('Активен')
           .removeClass('btn-primary')
           .addClass('btn-success');
      
      // Обновляем строку
      const row = button.closest('tr');
      row.find('.status-cell').text('Активен').addClass('text-success');
    },
    error: function() {
      button.prop('disabled', false).text('Активировать');
      alert('Ошибка активации');
    }
  });
});
```

Выберите вариант в зависимости от используемых в проекте технологий и предпочтений.

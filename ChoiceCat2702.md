Вот готовое решение для выбора категорий с изменением цвета и отправкой в базу данных:

HTML часть

```html
<div class="categories-container">
    <div class="category-card" data-category-id="1" data-category-name="Категория 1">
        Категория 1
    </div>
    <div class="category-card" data-category-id="2" data-category-name="Категория 2">
        Категория 2
    </div>
    <div class="category-card" data-category-id="3" data-category-name="Категория 3">
        Категория 3
    </div>
</div>

<button id="saveCategoriesBtn">Сохранить выбранные категории</button>
```

CSS стили

```css
.category-card {
    padding: 15px 25px;
    margin: 10px;
    border: 2px solid #ddd;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.3s ease;
    display: inline-block;
    background-color: #fff;
    color: #333;
}

.category-card.selected {
    background-color: #4CAF50;
    color: white;
    border-color: #45a049;
    transform: scale(1.05);
    box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}
```

JavaScript код

```javascript
class CategorySelector {
    constructor() {
        this.selectedCategories = new Set();
        this.init();
    }

    init() {
        // Находим все карточки категорий
        this.categoryCards = document.querySelectorAll('.category-card');
        
        // Добавляем обработчики событий
        this.categoryCards.forEach(card => {
            card.addEventListener('click', (e) => this.toggleCategory(e));
        });

        // Добавляем обработчик для кнопки сохранения
        const saveBtn = document.getElementById('saveCategoriesBtn');
        if (saveBtn) {
            saveBtn.addEventListener('click', () => this.saveCategories());
        }
    }

    toggleCategory(event) {
        const card = event.currentTarget;
        const categoryId = card.dataset.categoryId;
        
        // Переключаем класс selected
        card.classList.toggle('selected');
        
        // Добавляем или удаляем из набора выбранных
        if (card.classList.contains('selected')) {
            this.selectedCategories.add({
                id: categoryId,
                name: card.dataset.categoryName
            });
        } else {
            // Удаляем из набора по id
            this.selectedCategories.forEach(item => {
                if (item.id === categoryId) {
                    this.selectedCategories.delete(item);
                }
            });
        }
        
        console.log('Выбранные категории:', Array.from(this.selectedCategories));
    }

    async saveCategories() {
        const selectedArray = Array.from(this.selectedCategories);
        
        if (selectedArray.length === 0) {
            alert('Выберите хотя бы одну категорию');
            return;
        }

        // Блокируем кнопку во время отправки
        const saveBtn = document.getElementById('saveCategoriesBtn');
        saveBtn.disabled = true;
        saveBtn.textContent = 'Сохранение...';

        try {
            // Отправляем данные на сервер
            const response = await this.sendToServer(selectedArray);
            
            if (response.ok) {
                alert('Категории успешно сохранены!');
                console.log('Отправленные данные:', selectedArray);
            } else {
                throw new Error('Ошибка при сохранении');
            }
        } catch (error) {
            alert('Произошла ошибка при сохранении: ' + error.message);
            console.error('Ошибка:', error);
        } finally {
            // Разблокируем кнопку
            saveBtn.disabled = false;
            saveBtn.textContent = 'Сохранить выбранные категории';
        }
    }

    async sendToServer(categories) {
        // Пример отправки на сервер (замените URL на ваш)
        return await fetch('/api/save-categories', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                categories: categories,
                userId: 1, // Если нужно, добавьте ID пользователя
                timestamp: new Date().toISOString()
            })
        });
    }

    // Метод для сброса выбора
    resetSelection() {
        this.categoryCards.forEach(card => {
            card.classList.remove('selected');
        });
        this.selectedCategories.clear();
    }

    // Метод для предварительной загрузки выбранных категорий
    loadSelectedCategories(categoryIds) {
        categoryIds.forEach(id => {
            const card = document.querySelector(`[data-category-id="${id}"]`);
            if (card && !card.classList.contains('selected')) {
                card.classList.add('selected');
                this.selectedCategories.add({
                    id: card.dataset.categoryId,
                    name: card.dataset.categoryName
                });
            }
        });
    }
}

// Инициализация после загрузки DOM
document.addEventListener('DOMContentLoaded', () => {
    const selector = new CategorySelector();
    
    // Пример загрузки предварительно выбранных категорий
    // selector.loadSelectedCategories([1, 3]);
});
```

Пример PHP бэкенда (save-categories.php)

```php
<?php
header('Content-Type: application/json');

// Получаем данные из запроса
$input = json_decode(file_get_contents('php://input'), true);

if (!$input || !isset($input['categories'])) {
    http_response_code(400);
    echo json_encode(['error' => 'No categories provided']);
    exit;
}

try {
    $pdo = new PDO('mysql:host=localhost;dbname=your_db', 'username', 'password');
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    
    // Начинаем транзакцию
    $pdo->beginTransaction();
    
    // Очищаем старые категории пользователя (если нужно)
    // $stmt = $pdo->prepare("DELETE FROM user_categories WHERE user_id = ?");
    // $stmt->execute([$input['userId']]);
    
    // Сохраняем новые категории
    $stmt = $pdo->prepare("INSERT INTO user_categories (user_id, category_id, created_at) VALUES (?, ?, NOW())");
    
    foreach ($input['categories'] as $category) {
        $stmt->execute([$input['userId'], $category['id']]);
    }
    
    // Подтверждаем транзакцию
    $pdo->commit();
    
    echo json_encode(['success' => true, 'message' => 'Categories saved successfully']);
    
} catch (PDOException $e) {
    // Откатываем транзакцию в случае ошибки
    $pdo->rollBack();
    http_response_code(500);
    echo json_encode(['error' => 'Database error: ' . $e->getMessage()]);
}
?>
```

Особенности реализации:

1. Визуальная обратная связь: При клике карточка меняет цвет, масштабируется и получает тень
2. Хранение состояния: Используется Set для хранения выбранных категорий без дубликатов
3. Асинхронная отправка: Данные отправляются через fetch без перезагрузки страницы
4. Обработка ошибок: Добавлена обработка ошибок и блокировка кнопки во время отправки
5. Data-атрибуты: Используются data-category-id и data-category-name для хранения данных

Для использования:

1. Добавьте HTML разметку с категориями
2. Подключите CSS стили
3. Добавьте JavaScript код перед закрывающим тегом </body>
4. Настройте PHP скрипт под вашу структуру БД
5. Замените URL в fetch на ваш реальный эндпоинт

Категории будут выделяться при клике и отправляться на сервер при нажатии кнопки "Сохранить".

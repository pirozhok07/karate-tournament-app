Вот максимально простое решение:

HTML

```html
<div class="categories">
    <div class="category" data-id="1">Категория 1</div>
    <div class="category" data-id="2">Категория 2</div>
    <div class="category" data-id="3">Категория 3</div>
</div>

<button id="saveBtn">Сохранить</button>
```

CSS

```css
.category {
    padding: 10px 20px;
    margin: 5px;
    border: 1px solid #ccc;
    cursor: pointer;
    display: inline-block;
}

.category.active {
    background-color: #4CAF50;
    color: white;
    border-color: #4CAF50;
}
```

JavaScript

```javascript
// Массив для хранения выбранных категорий
let selectedCategories = [];

// Добавляем обработчик на все категории
document.querySelectorAll('.category').forEach(category => {
    category.addEventListener('click', function() {
        // Переключаем класс active
        this.classList.toggle('active');
        
        const categoryId = this.dataset.id;
        const categoryName = this.textContent;
        
        // Если добавили класс active - добавляем в массив
        if (this.classList.contains('active')) {
            selectedCategories.push({
                id: categoryId,
                name: categoryName
            });
        } else {
            // Если убрали класс - удаляем из массива
            selectedCategories = selectedCategories.filter(c => c.id !== categoryId);
        }
        
        console.log('Выбрано:', selectedCategories);
    });
});

// Сохранение в БД
document.getElementById('saveBtn').addEventListener('click', function() {
    if (selectedCategories.length === 0) {
        alert('Выберите категории');
        return;
    }
    
    // Отправка на сервер
    fetch('save-categories.php', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(selectedCategories)
    })
    .then(response => response.json())
    .then(data => {
        alert('Сохранено!');
        console.log('Ответ сервера:', data);
    })
    .catch(error => {
        alert('Ошибка: ' + error);
    });
});
```

PHP (save-categories.php)

```php
<?php
$data = json_decode(file_get_contents('php://input'), true);

// Здесь ваш код сохранения в БД
// Например:
// foreach ($data as $category) {
//     $id = $category['id'];
//     $name = $category['name'];
//     // Сохраняем в базу
// }

echo json_encode(['success' => true]);
?>
```

Всё! Просто и работает.

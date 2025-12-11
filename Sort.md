Вот пример реализации таблицы с сортировкой и фильтрами на чистом HTML/CSS/JavaScript:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Таблица с сортировкой и фильтрами</title>
    <style>
        * {
            box-sizing: border-box;
        }
        
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f5f5f5;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        
        h1 {
            text-align: center;
            color: #333;
        }
        
        .controls {
            display: flex;
            flex-wrap: wrap;
            gap: 15px;
            margin-bottom: 20px;
            padding: 15px;
            background: #f8f9fa;
            border-radius: 5px;
        }
        
        .filter-group {
            display: flex;
            flex-direction: column;
            min-width: 200px;
        }
        
        .filter-group label {
            margin-bottom: 5px;
            font-weight: bold;
            color: #555;
        }
        
        .filter-group input,
        .filter-group select {
            padding: 8px 12px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-size: 14px;
        }
        
        .reset-btn {
            padding: 8px 20px;
            background: #6c757d;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            align-self: flex-end;
            margin-left: auto;
        }
        
        .reset-btn:hover {
            background: #5a6268;
        }
        
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        
        th {
            background-color: #4a6fa5;
            color: white;
            padding: 12px 15px;
            text-align: left;
            cursor: pointer;
            user-select: none;
            position: relative;
        }
        
        th:hover {
            background-color: #3a5a8a;
        }
        
        th.sorted-asc::after {
            content: " ▲";
            font-size: 12px;
        }
        
        th.sorted-desc::after {
            content: " ▼";
            font-size: 12px;
        }
        
        td {
            padding: 12px 15px;
            border-bottom: 1px solid #ddd;
        }
        
        tr:hover {
            background-color: #f5f9ff;
        }
        
        tr:nth-child(even) {
            background-color: #f8f9fa;
        }
        
        tr:nth-child(even):hover {
            background-color: #e9f0ff;
        }
        
        .status-active {
            color: #28a745;
            font-weight: bold;
        }
        
        .status-inactive {
            color: #dc3545;
            font-weight: bold;
        }
        
        .priority-high {
            background-color: #ffe6e6;
            font-weight: bold;
        }
        
        .priority-medium {
            background-color: #fff3cd;
        }
        
        .no-data {
            text-align: center;
            padding: 40px;
            color: #666;
            font-style: italic;
        }
        
        .stats {
            margin-top: 15px;
            color: #666;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Список задач</h1>
        
        <div class="controls">
            <div class="filter-group">
                <label for="search">Поиск:</label>
                <input type="text" id="search" placeholder="Введите текст для поиска...">
            </div>
            
            <div class="filter-group">
                <label for="status-filter">Статус:</label>
                <select id="status-filter">
                    <option value="">Все статусы</option>
                    <option value="active">Активный</option>
                    <option value="inactive">Неактивный</option>
                </select>
            </div>
            
            <div class="filter-group">
                <label for="priority-filter">Приоритет:</label>
                <select id="priority-filter">
                    <option value="">Все приоритеты</option>
                    <option value="high">Высокий</option>
                    <option value="medium">Средний</option>
                    <option value="low">Низкий</option>
                </select>
            </div>
            
            <button class="reset-btn" id="reset-filters">Сбросить фильтры</button>
        </div>
        
        <table id="data-table">
            <thead>
                <tr>
                    <th data-column="id">ID</th>
                    <th data-column="name">Название</th>
                    <th data-column="assignee">Исполнитель</th>
                    <th data-column="status">Статус</th>
                    <th data-column="priority">Приоритет</th>
                    <th data-column="date">Дата</th>
                    <th data-column="progress">Прогресс (%)</th>
                </tr>
            </thead>
            <tbody id="table-body">
                <!-- Данные будут вставлены через JavaScript -->
            </tbody>
        </table>
        
        <div class="stats" id="stats">
            Показано 0 из 0 записей
        </div>
    </div>

    <script>
        // Исходные данные таблицы
        const tableData = [
            { id: 1, name: "Разработка интерфейса", assignee: "Иван Иванов", status: "active", priority: "high", date: "2024-01-15", progress: 75 },
            { id: 2, name: "Тестирование модуля", assignee: "Петр Петров", status: "active", priority: "medium", date: "2024-01-10", progress: 100 },
            { id: 3, name: "Документация API", assignee: "Мария Сидорова", status: "inactive", priority: "low", date: "2024-01-05", progress: 30 },
            { id: 4, name: "Оптимизация базы данных", assignee: "Алексей Смирнов", status: "active", priority: "high", date: "2024-01-18", progress: 60 },
            { id: 5, name: "Редизайн логотипа", assignee: "Елена Кузнецова", status: "active", priority: "medium", date: "2024-01-12", progress: 90 },
            { id: 6, name: "Настройка сервера", assignee: "Дмитрий Попов", status: "inactive", priority: "high", date: "2024-01-03", progress: 100 },
            { id: 7, name: "Интеграция платежной системы", assignee: "Анна Васильева", status: "active", priority: "high", date: "2024-01-20", progress: 45 },
            { id: 8, name: "Анализ пользовательских данных", assignee: "Сергей Михайлов", status: "active", priority: "medium", date: "2024-01-14", progress: 80 },
            { id: 9, name: "Подготовка отчета", assignee: "Ольга Новикова", status: "inactive", priority: "low", date: "2023-12-28", progress: 20 },
            { id: 10, name: "Обновление контента", assignee: "Николай Федоров", status: "active", priority: "low", date: "2024-01-16", progress: 100 }
        ];

        let currentData = [...tableData];
        let currentSortColumn = null;
        let currentSortDirection = 'asc';

        // Инициализация таблицы
        document.addEventListener('DOMContentLoaded', function() {
            renderTable();
            setupEventListeners();
            updateStats();
        });

        // Рендеринг таблицы
        function renderTable() {
            const tbody = document.getElementById('table-body');
            tbody.innerHTML = '';

            if (currentData.length === 0) {
                tbody.innerHTML = '<tr><td colspan="7" class="no-data">Нет данных, соответствующих фильтрам</td></tr>';
                return;
            }

            currentData.forEach(item => {
                const row = document.createElement('tr');
                
                // Определяем классы для статуса и приоритета
                const statusClass = item.status === 'active' ? 'status-active' : 'status-inactive';
                const priorityClass = `priority-${item.priority}`;
                
                // Форматируем дату
                const dateObj = new Date(item.date);
                const formattedDate = dateObj.toLocaleDateString('ru-RU');
                
                // Создаем ячейки
                row.innerHTML = `
                    <td>${item.id}</td>
                    <td>${item.name}</td>
                    <td>${item.assignee}</td>
                    <td class="${statusClass}">${item.status === 'active' ? 'Активный' : 'Неактивный'}</td>
                    <td class="${priorityClass}">${
                        item.priority === 'high' ? 'Высокий' : 
                        item.priority === 'medium' ? 'Средний' : 'Низкий'
                    }</td>
                    <td>${formattedDate}</td>
                    <td>${item.progress}</td>
                `;
                
                tbody.appendChild(row);
            });
        }

        // Настройка обработчиков событий
        function setupEventListeners() {
            // Сортировка по клику на заголовок
            document.querySelectorAll('th[data-column]').forEach(th => {
                th.addEventListener('click', () => {
                    const column = th.getAttribute('data-column');
                    sortTable(column);
                });
            });

            // Поиск по всем полям
            document.getElementById('search').addEventListener('input', applyFilters);
            
            // Фильтр по статусу
            document.getElementById('status-filter').addEventListener('change', applyFilters);
            
            // Фильтр по приоритету
            document.getElementById('priority-filter').addEventListener('change', applyFilters);
            
            // Сброс фильтров
            document.getElementById('reset-filters').addEventListener('click', resetFilters);
        }

        // Сортировка таблицы
        function sortTable(column) {
            // Сбрасываем предыдущие индикаторы сортировки
            document.querySelectorAll('th').forEach(th => {
                th.classList.remove('sorted-asc', 'sorted-desc');
            });

            // Определяем направление сортировки
            if (currentSortColumn === column) {
                currentSortDirection = currentSortDirection === 'asc' ? 'desc' : 'asc';
            } else {
                currentSortColumn = column;
                currentSortDirection = 'asc';
            }

            // Добавляем индикатор сортировки
            const currentTh = document.querySelector(`th[data-column="${column}"]`);
            currentTh.classList.add(`sorted-${currentSortDirection}`);

            // Сортируем данные
            currentData.sort((a, b) => {
                let aValue = a[column];
                let bValue = b[column];

                // Для числовых полей
                if (column === 'id' || column === 'progress') {
                    aValue = Number(aValue);
                    bValue = Number(bValue);
                }

                // Для дат
                if (column === 'date') {
                    aValue = new Date(aValue);
                    bValue = new Date(bValue);
                }

                // Для строк (регистронезависимо)
                if (typeof aValue === 'string') {
                    aValue = aValue.toLowerCase();
                    bValue = bValue.toLowerCase();
                }

                // Сравнение
                if (aValue < bValue) {
                    return currentSortDirection === 'asc' ? -1 : 1;
                }
                if (aValue > bValue) {
                    return currentSortDirection === 'asc' ? 1 : -1;
                }
                return 0;
            });

            renderTable();
            updateStats();
        }

        // Применение фильтров
        function applyFilters() {
            const searchText = document.getElementById('search').value.toLowerCase();
            const statusFilter = document.getElementById('status-filter').value;
            const priorityFilter = document.getElementById('priority-filter').value;

            currentData = tableData.filter(item => {
                // Поиск по всем текстовым полям
                const matchesSearch = !searchText || 
                    item.name.toLowerCase().includes(searchText) ||
                    item.assignee.toLowerCase().includes(searchText) ||
                    item.id.toString().includes(searchText);

                // Фильтр по статусу
                const matchesStatus = !statusFilter || item.status === statusFilter;

                // Фильтр по приоритету
                const matchesPriority = !priorityFilter || item.priority === priorityFilter;

                return matchesSearch && matchesStatus && matchesPriority;
            });

            // Применяем текущую сортировку, если она есть
            if (currentSortColumn) {
                sortTable(currentSortColumn);
            } else {
                renderTable();
            }
            
            updateStats();
        }

        // Сброс всех фильтров
        function resetFilters() {
            document.getElementById('search').value = '';
            document.getElementById('status-filter').value = '';
            document.getElementById('priority-filter').value = '';
            
            currentSortColumn = null;
            currentSortDirection = 'asc';
            
            // Сбрасываем индикаторы сортировки
            document.querySelectorAll('th').forEach(th => {
                th.classList.remove('sorted-asc', 'sorted-desc');
            });
            
            currentData = [...tableData];
            renderTable();
            updateStats();
        }

        // Обновление статистики
        function updateStats() {
            const totalItems = tableData.length;
            const shownItems = currentData.length;
            document.getElementById('stats').textContent = `Показано ${shownItems} из ${totalItems} записей`;
        }
    </script>
</body>
</html>
```

Возможности этой реализации:

1. Сортировка по столбцам:
   · Кликните на заголовок столбца для сортировки
   · Повторный клик меняет направление сортировки
   · Индикаторы (▲/▼) показывают направление сортировки
2. Фильтрация:
   · Поиск по всем полям (текстовый поиск)
   · Фильтр по статусу (выпадающий список)
   · Фильтр по приоритету (выпадающий список)
3. Дополнительные функции:
   · Кнопка сброса всех фильтров
   · Статистика (сколько записей показано)
   · Визуальное оформление (цвета для статусов и приоритетов)
   · Адаптивный дизайн
4. Особенности кода:
   · Чистый JavaScript без зависимостей
   · Модульная структура
   · Комментарии на русском языке
   · Простая адаптация под ваши данные

Как адаптировать под свои данные:

1. Замените массив tableData своими данными
2. Измените заголовки столбцов в HTML (теги <th>)
3. Настройте рендеринг строк в функции renderTable()
4. При необходимости добавьте новые фильтры

Вы можете сохранить этот код как HTML-файл и открыть в браузере. Все функции работают без серверной части.

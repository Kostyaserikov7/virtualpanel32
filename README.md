<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Управление ключами</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #121212;
            color: white;
        }
        h1, h2 {
            text-align: center;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
        }
        .form-group input {
            width: 100%;
            padding: 8px;
            box-sizing: border-box;
            background-color: #333;
            color: white;
            border: 1px solid #444;
            border-radius: 5px;
        }
        .form-group button {
            width: 100%;
            padding: 10px;
            background-color: #4CAF50;
            color: white;
            border: none;
            cursor: pointer;
            border-radius: 10px;
            font-size: 16px;
        }
        .form-group button:hover {
            background-color: #45a049;
        }
        .keys-list {
            margin-top: 20px;
        }
        .key-item {
            padding: 15px;
            border: 1px solid #444;
            margin-bottom: 10px;
            background-color: #222;
            border-radius: 8px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .key-info {
            flex-grow: 1;
        }
        .key-actions {
            display: flex;
            gap: 10px;
        }
        .action-btn {
            padding: 6px 12px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 14px;
        }
        .delete-btn {
            background-color: #ff4444;
            color: white;
        }
        .edit-btn {
            background-color: #2196F3;
            color: white;
        }
        .modal {
            display: none;
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: #333;
            padding: 25px;
            border-radius: 10px;
            z-index: 1000;
            width: 300px;
        }
        .modal-overlay {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.5);
            z-index: 999;
        }
        .status-active {
            color: #4CAF50;
            font-weight: bold;
        }
        .status-inactive {
            color: #ff4444;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Управление ключами доступа</h1>

        <!-- Форма создания ключа -->
        <div class="form-section">
            <h2>Создать новый ключ</h2>
            <div class="form-group">
                <label for="key">Ключ:</label>
                <input type="text" id="key" placeholder="Введите уникальный ключ">
            </div>
            <div class="form-group">
                <label for="valid_until">Срок действия:</label>
                <input type="datetime-local" id="valid_until">
            </div>
            <div class="form-group">
                <label for="device_limit">Лимит устройств:</label>
                <input type="number" id="device_limit" value="1" min="1">
            </div>
            <div class="form-group">
                <button onclick="createKey()">Создать ключ</button>
            </div>
        </div>

        <!-- Список ключей -->
        <div class="keys-list">
            <h2>Активные ключи</h2>
            <div id="keys-container"></div>
        </div>
    </div>

    <!-- Модальное окно редактирования -->
    <div class="modal-overlay" id="modalOverlay"></div>
    <div class="modal" id="editModal">
        <h3>Редактирование ключа</h3>
        <div class="form-group">
            <label for="edit_limit">Лимит устройств:</label>
            <input type="number" id="edit_limit" min="1">
        </div>
        <div class="form-group">
            <label for="edit_active">
                <input type="checkbox" id="edit_active"> Активен
            </label>
        </div>
        <div class="form-group">
            <button class="edit-btn action-btn" onclick="saveChanges()">Сохранить</button>
            <button class="delete-btn action-btn" onclick="closeModal()">Отмена</button>
        </div>
    </div>

    <script>
        let currentEditingKey = null;

        // Создание ключа
        async function createKey() {
            const keyData = {
                key: document.getElementById('key').value,
                valid_until: document.getElementById('valid_until').value + ':00',
                device_limit: parseInt(document.getElementById('device_limit').value)
            };

            if (!validateKeyData(keyData)) return;

            try {
                const response = await fetch('http://localhost:5000/keys', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify(keyData)
                });

                if (response.ok) {
                    alert('Ключ успешно создан!');
                    loadKeys();
                    clearForm();
                } else {
                    const error = await response.json();
                    alert(`Ошибка: ${error.detail}`);
                }
            } catch (error) {
                alert('Ошибка соединения с сервером');
            }
        }

        // Загрузка списка ключей
        async function loadKeys() {
            try {
                const response = await fetch('http://localhost:5000/keys');
                const keys = await response.json();
                renderKeys(keys);
            } catch (error) {
                console.error('Ошибка загрузки ключей:', error);
            }
        }

        // Отображение ключей
        function renderKeys(keys) {
            const container = document.getElementById('keys-container');
            container.innerHTML = '';

            keys.forEach(key => {
                const keyElement = document.createElement('div');
                keyElement.className = 'key-item';
                keyElement.innerHTML = `
                    <div class="key-info">
                        <div><strong>${key.key}</strong></div>
                        <div>Срок: ${formatDate(key.valid_until)}</div>
                        <div>Устройств: ${key.devices_used}/${key.device_limit}</div>
                        <div class="${key.is_active ? 'status-active' : 'status-inactive'}">
                            ${key.is_active ? 'АКТИВЕН' : 'НЕАКТИВЕН'}
                        </div>
                    </div>
                    <div class="key-actions">
                        <button class="edit-btn action-btn" onclick="openEditModal(${key.id}, ${key.device_limit}, ${key.is_active})">
                            Изменить
                        </button>
                        <button class="delete-btn action-btn" onclick="deleteKey(${key.id})">
                            Удалить
                        </button>
                    </div>
                `;
                container.appendChild(keyElement);
            });
        }

        // Открытие модального окна редактирования
        function openEditModal(keyId, limit, isActive) {
            currentEditingKey = keyId;
            document.getElementById('edit_limit').value = limit;
            document.getElementById('edit_active').checked = isActive;
            document.getElementById('editModal').style.display = 'block';
            document.getElementById('modalOverlay').style.display = 'block';
        }

        // Сохранение изменений
        async function saveChanges() {
            const updateData = {
                device_limit: parseInt(document.getElementById('edit_limit').value),
                is_active: document.getElementById('edit_active').checked
            };

            try {
                const response = await fetch(`http://localhost:5000/keys/${currentEditingKey}`, {
                    method: 'PATCH',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify(updateData)
                });

                if (response.ok) {
                    closeModal();
                    loadKeys();
                } else {
                    const error = await response.json();
                    alert(`Ошибка: ${error.detail}`);
                }
            } catch (error) {
                alert('Ошибка соединения с сервером');
            }
        }

        // Удаление ключа
        async function deleteKey(keyId) {
            if (!confirm('Вы уверены, что хотите удалить ключ?')) return;

            try {
                const response = await fetch(`http://localhost:5000/keys/${keyId}`, {
                    method: 'DELETE'
                });

                if (response.ok) {
                    loadKeys();
                } else {
                    const error = await response.json();
                    alert(`Ошибка: ${error.detail}`);
                }
            } catch (error) {
                alert('Ошибка соединения с сервером');
            }
        }

        // Вспомогательные функции
        function validateKeyData(data) {
            if (!data.key || !data.valid_until || !data.device_limit) {
                alert('Все поля обязательны для заполнения!');
                return false;
            }
            if (data.device_limit < 1) {
                alert('Лимит устройств должен быть не менее 1');
                return false;
            }
            return true;
        }

        function formatDate(isoString) {
            const date = new Date(isoString);
            return date.toLocaleDateString() + ' ' + date.toLocaleTimeString();
        }

        function clearForm() {
            document.getElementById('key').value = '';
            document.getElementById('valid_until').value = '';
            document.getElementById('device_limit').value = '1';
        }

        function closeModal() {
            document.getElementById('editModal').style.display = 'none';
            document.getElementById('modalOverlay').style.display = 'none';
            currentEditingKey = null;
        }

        // Инициализация
        window.onload = loadKeys;
    </script>
</body>
</html>

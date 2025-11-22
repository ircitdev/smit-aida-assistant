# MAX-FreeScout Integration Bridge

Интеграция мессенджера MAX с системой FreeScout для автоматической обработки обращений клиентов.

## Описание

Модуль синхронизирует сообщения между MAX мессенджером и FreeScout:
- ✅ Входящие сообщения из MAX → создание/обновление тикетов в FreeScout
- ✅ Ответы агентов в FreeScout → отправка в MAX
- ✅ Защита от дублирования сообщений
- ✅ Автоматическое создание клиентов
- ✅ Связывание диалогов с тикетами

## Требования

### Регистрация и доступ

1. **Юридическое лицо РФ**
   - MAX требует верифицированное юрлицо РФ для создания ботов
   - Подготовьте документы организации

2. **Регистрация в MAX**
   - Зарегистрируйтесь на https://dev.max.ru
   - Пройдите верификацию организации

3. **Создание бота**
   - Перейдите в раздел "Боты" на https://dev.max.ru
   - Создайте нового бота
   - Получите Bot Token и Bot ID

## Установка

### 1. Файлы уже созданы

```bash
/var/www/max-freescout/
├── max_freescout_bridge.py  # Основной модуль
├── .env.example              # Пример конфигурации
├── requirements.txt          # Зависимости Python
├── venv/                     # Виртуальное окружение
└── README.md                 # Эта документация
```

### 2. Настройка конфигурации

Создайте файл `.env` на основе `.env.example`:

```bash
cp /var/www/max-freescout/.env.example /var/www/max-freescout/.env
nano /var/www/max-freescout/.env
```

Заполните переменные окружения:

```env
# FreeScout Configuration
FREESCOUT_URL=https://support.smit34.ru
FREESCOUT_API_KEY=ваш_freescout_api_key
FREESCOUT_MAILBOX_ID=2

# MAX Bot Configuration
MAX_BOT_TOKEN=ваш_max_bot_token_от_platform-api.max.ru
MAX_BOT_ID=ваш_max_bot_id
```

### 3. Получение MAX Bot Token

1. Зайдите на https://dev.max.ru/bots
2. Создайте нового бота или выберите существующего
3. Скопируйте:
   - **Bot Token** (для Authorization: Bearer)
   - **Bot ID** (идентификатор бота)

### 4. Настройка Webhook в MAX

После запуска сервиса (см. ниже), настройте webhook:

**Webhook URL:**
```
https://ваш_домен/max/webhook
```

**Метод настройки через API:**

```bash
curl -X POST https://platform-api.max.ru/subscriptions \
  -H "Authorization: Bearer ВАШ_MAX_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"url\": \"https://ваш_домен/max/webhook\"}"
```

**Важно:** MAX поддерживает только HTTPS! HTTP не поддерживается.

### 5. Настройка Webhook в FreeScout

1. Войдите в FreeScout как администратор
2. Перейдите в **Manage → Settings → API**
3. Включите **WebHooks**
4. Добавьте webhook:
   - URL: `https://ваш_домен/freescout/webhook`
   - Events: выберите нужные события (рекомендуется "conversation.created", "thread.created")
   - Mailbox: выберите mailbox ID 2 (или тот, что указали в .env)

## Запуск

### Ручной запуск (для тестирования)

```bash
cd /var/www/max-freescout
source venv/bin/activate
python max_freescout_bridge.py
```

Сервис запустится на порту **8902**.

### Создание systemd службы

Создайте файл `/etc/systemd/system/max-freescout.service`:

```ini
[Unit]
Description=MAX-FreeScout Integration Bridge
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/max-freescout
Environment="PATH=/var/www/max-freescout/venv/bin"
ExecStart=/var/www/max-freescout/venv/bin/python /var/www/max-freescout/max_freescout_bridge.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Запустите службу:

```bash
systemctl daemon-reload
systemctl enable max-freescout
systemctl start max-freescout
systemctl status max-freescout
```

### Настройка Nginx (если нужен HTTPS)

Добавьте в конфигурацию Nginx:

```nginx
location /max/webhook {
    proxy_pass http://localhost:8902/max/webhook;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

location /freescout/webhook {
    proxy_pass http://localhost:8902/freescout/webhook;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Перезапустите Nginx:

```bash
nginx -t
systemctl reload nginx
```

## Проверка работы

### 1. Health Check

```bash
curl http://localhost:8902/
```

Ожидаемый ответ:
```json
{
  "service": "MAX-FreeScout Bridge",
  "status": "running",
  "freescout_url": "https://support.smit34.ru",
  "max_bot_id": "ваш_bot_id"
}
```

### 2. Тест Webhook

```bash
# Проверка MAX webhook
curl http://localhost:8902/max/webhook

# Проверка FreeScout webhook
curl http://localhost:8902/freescout/webhook
```

### 3. Логи

```bash
# Если запущен через systemd
journalctl -u max-freescout -f

# Если запущен вручную - смотрите консоль
```

## Принцип работы

### Входящие сообщения (MAX → FreeScout)

1. Пользователь пишет боту в MAX
2. MAX отправляет webhook на `/max/webhook`
3. Модуль:
   - Проверяет дубликаты
   - Ищет существующий тикет для пользователя
   - Если нет - создает клиента и новый тикет
   - Если есть - добавляет сообщение в тикет

### Исходящие сообщения (FreeScout → MAX)

1. Агент отвечает в FreeScout
2. FreeScout отправляет webhook на `/freescout/webhook`
3. Модуль:
   - Проверяет дубликаты
   - Находит MAX user_id по тикету
   - Отправляет сообщение в MAX

### Защита от дубликатов

- Для MAX→FreeScout: проверка по `message_id`
- Для FreeScout→MAX: проверка по `thread_id`
- Все проверенные сообщения сохраняются в БД

## База данных

SQLite база: `/var/www/max-freescout/mapping.db`

### Таблицы

**max_freescout_mapping** - связь пользователей с тикетами
- max_user_id (PRIMARY KEY)
- freescout_conversation_id
- freescout_customer_id
- last_message_id
- created_at, updated_at

**message_sync** - история синхронизации
- id (AUTO INCREMENT)
- max_message_id
- freescout_thread_id
- direction (max_to_freescout / freescout_to_max)
- synced_at

### Проверка БД

```bash
cd /var/www/max-freescout
python3 -c "
import sqlite3
conn = sqlite3.connect('mapping.db')
cursor = conn.cursor()

# Количество связей
cursor.execute('SELECT COUNT(*) FROM max_freescout_mapping')
print(f'Связей MAX↔FreeScout: {cursor.fetchone()[0]}')

# Количество синхронизированных сообщений
cursor.execute('SELECT COUNT(*) FROM message_sync')
print(f'Синхронизировано сообщений: {cursor.fetchone()[0]}')

conn.close()
"
```

## Отладка

### Включить подробные логи

В `max_freescout_bridge.py` измените:

```python
uvicorn.run(
    app,
    host="0.0.0.0",
    port=8902,
    log_level="debug"  # было "info"
)
```

### Типичные проблемы

**1. Ошибка "MAX API Error: 401"**
- Проверьте правильность MAX_BOT_TOKEN
- Токен должен быть актуальным

**2. Ошибка "FREESCOUT_API_KEY не установлен"**
- Создайте .env файл
- Заполните все обязательные поля

**3. Webhook не срабатывает**
- Проверьте, что URL доступен извне
- MAX требует HTTPS
- Проверьте firewall и nginx конфигурацию

**4. Дублируются сообщения**
- Проверьте, что функции is_message_processed() и is_thread_sent_to_max() работают
- Проверьте таблицу message_sync в БД

## Полезные ссылки

- [MAX API Документация](https://dev.max.ru/docs-api)
- [MAX для разработчиков](https://dev.max.ru/docs/chatbots/bots-coding/prepare)
- [GitHub: max-botapi-python](https://github.com/max-messenger/max-botapi-python)
- [FreeScout API](https://github.com/freescout-helpdesk/freescout/wiki/API)

## Поддержка

При возникновении проблем:
1. Проверьте логи: `journalctl -u max-freescout -f`
2. Проверьте health check: `curl http://localhost:8902/`
3. Проверьте базу данных
4. Проверьте настройки .env

## Лицензия

MIT

---

Дата создания: 2025-11-22
Версия: 1.0.0

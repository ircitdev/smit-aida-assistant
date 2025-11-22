# Быстрый старт MAX-FreeScout Integration

## Шаг 1: Получите MAX Bot Token

1. Перейдите на https://dev.max.ru/bots
2. Создайте бота (требуется верифицированное юрлицо РФ)
3. Получите:
   - Bot Token (длинная строка для API)
   - Bot ID (числовой идентификатор)

## Шаг 2: Настройте .env

```bash
cd /var/www/max-freescout
cp .env.example .env
nano .env
```

Заполните:
```env
FREESCOUT_URL=https://support.smit34.ru
FREESCOUT_API_KEY=dd2f001033f31ab74caa0db5c6f75553
FREESCOUT_MAILBOX_ID=2

MAX_BOT_TOKEN=ваш_токен_бота_от_max
MAX_BOT_ID=ваш_id_бота
```

## Шаг 3: Запустите сервис

### Вариант A: Systemd служба (рекомендуется)

```bash
cat > /etc/systemd/system/max-freescout.service << 'EOF'
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
EOF

systemctl daemon-reload
systemctl enable max-freescout
systemctl start max-freescout
systemctl status max-freescout
```

### Вариант B: Ручной запуск (для теста)

```bash
cd /var/www/max-freescout
source venv/bin/activate
python max_freescout_bridge.py
```

## Шаг 4: Настройте Nginx для HTTPS

Добавьте в конфигурацию Nginx:

```nginx
location /max/webhook {
    proxy_pass http://localhost:8902/max/webhook;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

location /freescout/webhook {
    proxy_pass http://localhost:8902/freescout/webhook;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

Перезагрузите Nginx:
```bash
nginx -t && systemctl reload nginx
```

## Шаг 5: Настройте Webhook в MAX

```bash
curl -X POST https://platform-api.max.ru/subscriptions \
  -H "Authorization: Bearer ВАШ_MAX_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://ваш_домен.ru/max/webhook"}'
```

**ВАЖНО:** URL должен быть HTTPS! MAX не поддерживает HTTP.

## Шаг 6: Настройте Webhook в FreeScout

1. FreeScout Admin → Manage → Settings → API
2. Enable WebHooks
3. Add webhook:
   - URL: `https://ваш_домен.ru/freescout/webhook`
   - Events: conversation.created, thread.created
   - Mailbox ID: 2

## Шаг 7: Проверьте работу

```bash
# Health check
curl http://localhost:8902/

# Логи
journalctl -u max-freescout -f
```

## Готово!

Теперь:
- Сообщения из MAX → создают тикеты в FreeScout
- Ответы агентов из FreeScout → отправляются в MAX
- Дубликаты фильтруются автоматически

## Troubleshooting

**Проблема:** Сервис не стартует
```bash
# Проверьте логи
journalctl -u max-freescout -n 50

# Проверьте .env
cat /var/www/max-freescout/.env
```

**Проблема:** Webhook не срабатывает
```bash
# Проверьте доступность URL
curl https://ваш_домен.ru/max/webhook

# Проверьте nginx
nginx -t
systemctl status nginx
```

**Проблема:** MAX API Error 401
- Проверьте правильность MAX_BOT_TOKEN в .env
- Токен должен быть актуальным и без пробелов

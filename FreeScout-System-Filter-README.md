# FreeScout System Emails Filter

Автоматическое удаление системных и служебных тикетов в FreeScout.

## Что фильтруется

### Заблокированные email адреса:
- `mailer-daemon@*` - ошибки доставки
- `postmaster@*` - системные уведомления
- `no-reply@*` / `noreply@*` - автоматические письма
- `noreply-dmarc-support@google.com` - DMARC отчеты

### Заблокированные имена клиентов:
- "Mail Delivery System"
- "Mail Delivery Failed"

### Заблокированные темы писем:
- "Mail delivery failed"
- "Delivery status notification"
- "Undelivered mail returned to sender"
- "Report domain:" (DMARC отчеты)
- "DMARC report"
- "Returned mail"

## Использование

### Ручной запуск

**Режим тестирования (не удаляет, только показывает):**
```bash
cd /var/www/freescout
php scripts/filter_system_emails.php dry-run
```

**Удалить найденные тикеты:**
```bash
cd /var/www/freescout
php scripts/filter_system_emails.php delete
```

**Закрыть найденные тикеты (не удалять):**
```bash
cd /var/www/freescout
php scripts/filter_system_emails.php close
```

### Автоматический запуск

Настроен cron job, который запускается **каждый час**:
- Файл: `/etc/cron.d/freescout-filter-system-emails`
- Лог: `/var/log/freescout-filter.log`

**Проверить логи:**
```bash
tail -f /var/log/freescout-filter.log
```

**Отключить автоматическое удаление:**
```bash
rm /etc/cron.d/freescout-filter-system-emails
```

**Включить снова:**
```bash
cat > /etc/cron.d/freescout-filter-system-emails << 'EOF'
0 * * * * www-data cd /var/www/freescout && /usr/bin/php scripts/filter_system_emails.php delete >> /var/log/freescout-filter.log 2>&1
EOF
chmod 644 /etc/cron.d/freescout-filter-system-emails
```

## Настройка фильтров

Отредактируйте файл `/var/www/freescout/scripts/filter_system_emails.php`:

### Добавить email в черный список:
```php
$blocked_emails = [
    'mailer-daemon@',
    'your-blocked-email@example.com', // Добавьте здесь
];
```

### Добавить тему в черный список:
```php
$blocked_subjects = [
    'mail delivery failed',
    'ваша новая тема', // Добавьте здесь
];
```

### Изменить период проверки:
По умолчанию проверяются тикеты за последние 30 дней:
```php
->where('created_at', '>', now()->subDays(30))  // Измените 30 на нужное число
```

## Проверка работы

**Посмотреть последние удаления:**
```bash
tail -30 /var/log/freescout-filter.log
```

**Протестировать без удаления:**
```bash
cd /var/www/freescout
php scripts/filter_system_emails.php dry-run
```

## Статистика

**Результат работы 22.11.2024:**
- Удалено тикетов: 13
  - Mail Delivery System: 2
  - DMARC отчеты (Google, Mail.Ru): 11

## Безопасность

- Скрипт работает только с тикетами младше 30 дней
- Режим dry-run позволяет протестировать перед реальным удалением
- Все операции логируются в `/var/log/freescout-filter.log`

## Troubleshooting

**Проблема: Скрипт не работает**
```bash
# Проверьте права
ls -la /var/www/freescout/scripts/filter_system_emails.php
# Должно быть: -rwxr-xr-x www-data www-data

# Запустите вручную для диагностики
cd /var/www/freescout
php scripts/filter_system_emails.php dry-run
```

**Проблема: Cron не запускается**
```bash
# Проверьте cron job
cat /etc/cron.d/freescout-filter-system-emails

# Проверьте логи cron
grep filter /var/log/syslog
```

**Проблема: Удаляются нужные тикеты**
1. Остановите cron: `rm /etc/cron.d/freescout-filter-system-emails`
2. Отредактируйте фильтры в скрипте
3. Протестируйте: `php scripts/filter_system_emails.php dry-run`
4. Если все ОК - включите cron снова

---
Дата создания: 2025-11-22

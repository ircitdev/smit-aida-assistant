# Интеграция биллинга с FreeScout - Завершено

**Дата:** 08 ноября 2025
**Время:** 02:18 - 02:30

---

## ✅ Реализованные функции

### 1. Backend (AIDA GPT server.py на порту 8900)

#### Функция `update_freescout_customer_from_billing(customer_id, phone)`

**Назначение:** Получает данные из биллинга и обновляет профиль клиента в FreeScout

**Алгоритм:**
1. Запрос в биллинг API: `https://billing.smit34.ru/api/phone.php?phone={phone}`
2. Получение ответа:
   ```json
   {
     "client": {
       "fullname": "Новичков Тарас Викторович",
       "contract_number": "0138",
       "address": "поселок Соляной Набережная 8/1",
       "tariff": "2024_SMiT",
       "ballance": "256.52"
     }
   }
   ```
3. Парсинг ФИО:
   - `"Новичков Тарас Викторович"` → `first_name="Тарас"`, `last_name="Новичков"`
   - Отчество отбрасывается

4. Парсинг адреса:
   - `"поселок Соляной Набережная 8/1"` → `zip="Соляной"` (до 12 символов)
   - Удаляются префиксы: "поселок", "город", "село", "пос.", "г."

5. Обновление FreeScout customer через API:
   ```json
   {
     "first_name": "Тарас",
     "last_name": "Новичков",
     "company": "0138",
     "address": {
       "zip": "Соляной"
     }
   }
   ```

6. Возврат баланса и данных клиента

**Расположение:** `/var/www/aida-gpt/server.py` строка ~1096

---

#### Endpoint `POST /get_balance`

**URL:** `http://31.44.7.144:8900/get_balance`

**Метод:** POST

**Payload:**
```json
{
  "customer_id": 31,
  "phone": "+79004445566"
}
```

**Response (успех):**
```json
{
  "success": true,
  "balance": "256.52",
  "fullname": "Новичков Тарас Викторович",
  "contract": "0138",
  "first_name": "Тарас",
  "last_name": "Новичков",
  "zip": "Соляной"
}
```

**Response (ошибка - клиент не найден):**
```json
{
  "success": false,
  "message": "Клиент с телефоном +79001234567 не найден в биллинге"
}
```

**Расположение:** `/var/www/aida-gpt/server.py` строка ~2265

---

### 2. Frontend (FreeScout Module)

#### Модуль AidaBilling

**Расположение:** `/var/www/freescout/Modules/AidaBilling/`

**Структура:**
```
AidaBilling/
├── module.json                        # Конфигурация модуля
├── Providers/
│   └── AidaBillingServiceProvider.php # Service Provider с хуками
├── Resources/
│   └── views/
│       └── customer_profile_extra.blade.php  # View шаблон
└── Public/
    ├── js/
    │   └── module.js                  # JavaScript логика
    └── css/
        └── module.css                 # Стили
```

---

#### Добавляемые элементы в профиль клиента

**1. Ссылка "Профиль в AMOCRM"**

Отображается под адресом клиента, если в поле Website сохранена ссылка на AmoCRM:

```html
<div class="aida-amocrm-link">
    <i class="glyphicon glyphicon-link"></i>
    <a href="https://pavelsmit34ru.amocrm.ru/contacts/detail/75005811" target="_blank">
        Профиль в AMOCRM
    </a>
</div>
```

**Условие отображения:** Website содержит "amocrm.ru"

---

**2. Кнопка "Запросить баланс"**

```html
<button
    id="aida-request-balance-btn"
    class="btn btn-default btn-sm"
    data-customer-id="31"
    data-phone="+79004445566">
    <i class="glyphicon glyphicon-ruble"></i>
    Запросить баланс
</button>
```

**При клике:**
1. Отправляется AJAX запрос к `http://31.44.7.144:8900/get_balance`
2. Показывается спиннер загрузки
3. При успехе:
   - Отображается баланс в зеленом алерте
   - Профиль обновляется данными из биллинга
   - Страница перезагружается через 2 секунды
4. При ошибке:
   - Отображается сообщение об ошибке в красном алерте

---

**3. Блок отображения результата**

```html
<div id="aida-balance-result" style="display: none;">
    <div class="alert alert-success">
        <strong>Баланс:</strong> 256.52 ₽<br>
        <strong>ФИО:</strong> Новичков Тарас Викторович<br>
        <strong>Договор:</strong> 0138<br>
        <small class="text-muted">Профиль обновлен</small>
    </div>
</div>
```

---

## 🔧 Технические детали

### Хуки FreeScout

Модуль использует хук `customer.profile.extra` для добавления контента в профиль:

```php
\Eventy::addAction('customer.profile.extra', function($customer, $conversation) {
    echo \View::make('aidabilling::customer_profile_extra', [
        'customer' => $customer,
        'conversation' => $conversation
    ])->render();
}, 10, 2);
```

**Где срабатывает:** После заметок (notes) в блоке профиля клиента

---

### AJAX запрос (JavaScript)

```javascript
$.ajax({
    url: 'http://31.44.7.144:8900/get_balance',
    method: 'POST',
    contentType: 'application/json',
    data: JSON.stringify({
        customer_id: customerId,
        phone: phone
    }),
    success: function(response) {
        if (response.success) {
            showBalanceSuccess(response);
            setTimeout(function() {
                location.reload();
            }, 2000);
        } else {
            showBalanceError(response.message);
        }
    },
    error: function(xhr, status, error) {
        showBalanceError('Ошибка сервера');
    }
});
```

---

### Обновляемые поля FreeScout

| Поле API | Источник | Пример |
|----------|----------|--------|
| `first_name` | `fullname.split()[1]` | "Тарас" |
| `last_name` | `fullname.split()[0]` | "Новичков" |
| `company` | `contract_number` | "0138" |
| `address.zip` | Парсинг `address` | "Соляной" |

---

## 📝 Изменения в коде

### `/var/www/aida-gpt/server.py`

**Импорты:**
```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
```

**Добавленные функции:**
1. `async def update_freescout_customer_from_billing(customer_id: int, phone: str)` - строка ~1096
2. Endpoint `@app.post("/get_balance")` - строка ~2265

**Изменения:**
- Проверка статуса ответа FreeScout: `if resp.status_code in [200, 204]` (204 = успех без тела)

---

### Скрипты модификации

1. `add_billing_balance_v2.py` - Добавление функции и endpoint
2. `fix_request_import.py` - Импорт Request и JSONResponse
3. `fix_status_check.py` - Поддержка статуса 204
4. `create_freescout_module.sh` - Создание модуля FreeScout

---

## 🧪 Тестирование

### Тест 1: Проверка endpoint

```bash
curl -X POST http://31.44.7.144:8900/get_balance \
  -H "Content-Type: application/json" \
  -d '{"customer_id": 31, "phone": "+79004445566"}'
```

**Ожидаемый результат:**
```json
{
  "success": true,
  "balance": "256.52",
  "fullname": "Новичков Тарас Викторович",
  "contract": "0138",
  "first_name": "Тарас",
  "last_name": "Новичков",
  "zip": "Соляной"
}
```

---

### Тест 2: Проверка UI в FreeScout

**Шаги:**
1. Открыть профиль клиента: `https://support.smit34.ru/customers/31/edit`
2. Проверить наличие:
   - ✅ Ссылки "Профиль в AMOCRM" (если Website заполнен)
   - ✅ Кнопки "Запросить баланс"
3. Нажать "Запросить баланс"
4. Проверить:
   - ✅ Появился спиннер загрузки
   - ✅ Отобразился баланс
   - ✅ Профиль обновился (Имя, Фамилия, Компания, Индекс)
   - ✅ Страница перезагрузилась

---

### Тест 3: Проверка обновления полей

**До запроса баланса:**
- First Name: (пусто или старое значение)
- Last Name: (пусто или старое значение)
- Company: (пусто)
- Zip: (пусто)

**После запроса баланса:**
- First Name: "Тарас"
- Last Name: "Новичков"
- Company: "0138"
- Zip: "Соляной"

---

## 🚀 Запуск и перезапуск

### Перезапуск AIDA GPT server.py

```bash
pkill -f 'python.*server.py'
cd /var/www/aida-gpt
nohup /var/www/aida-gpt/venv/bin/python /var/www/aida-gpt/server.py >> /var/www/aida-gpt/logs/server.log 2>&1 &
```

### Очистка кеша FreeScout

```bash
cd /var/www/freescout
php artisan cache:clear
php artisan view:clear
php artisan config:clear
```

---

## 📊 Архитектура

```
┌─────────────────┐
│  FreeScout UI   │
│  (браузер)      │
└────────┬────────┘
         │ Click "Запросить баланс"
         │
         ▼
┌─────────────────────────┐
│  AidaBilling Module     │
│  (JavaScript)           │
│  module.js              │
└────────┬────────────────┘
         │ AJAX POST
         │ /get_balance
         │ {customer_id, phone}
         ▼
┌─────────────────────────┐
│  AIDA GPT Server        │
│  server.py :8900        │
│                         │
│  ┌─────────────────┐    │
│  │ /get_balance    │    │
│  └────────┬────────┘    │
│           │             │
│           ▼             │
│  ┌─────────────────┐    │
│  │ fetch_billing   │    │ ◄──── Billing API
│  │ _by_phone()     │────┼────── :phone.php?phone=...
│  └────────┬────────┘    │
│           │             │
│           ▼             │
│  ┌─────────────────┐    │
│  │ update_         │    │
│  │ freescout_      │────┼────── FreeScout API
│  │ customer()      │────┼────── PUT /api/customers/{id}
│  └────────┬────────┘    │
│           │             │
│           ▼             │
│  ┌─────────────────┐    │
│  │ Return balance  │    │
│  │ and data        │    │
│  └─────────────────┘    │
└────────┬────────────────┘
         │ JSON response
         │ {success, balance, ...}
         ▼
┌─────────────────────────┐
│  FreeScout UI           │
│  - Show balance         │
│  - Reload page          │
│  - Updated profile      │
└─────────────────────────┘
```

---

## 🔗 Ссылки

- **AIDA GPT:** http://31.44.7.144:8900
- **Endpoint:** http://31.44.7.144:8900/get_balance
- **Billing API:** https://billing.smit34.ru/api/phone.php
- **FreeScout:** https://support.smit34.ru
- **AmoCRM:** https://pavelsmit34ru.amocrm.ru

---

## ✅ Статус

**Версия:** AIDA GPT v2.2 с биллинг интеграцией
**Дата:** 08.11.2025 02:30
**Статус:** ✅ Готово к использованию

**Модули:**
- ✅ Backend endpoint /get_balance
- ✅ FreeScout модуль AidaBilling
- ✅ Ссылка на AmoCRM в профиле
- ✅ Кнопка "Запросить баланс"
- ✅ Обновление профиля из биллинга
- ✅ Отображение баланса

---

**Конец документа**

# SMIT AIDA Assistant

AI-powered customer service assistant for SMIT internet provider, providing automated support for new client onboarding, service inquiries, and lead generation.

## Overview

AIDA (AI Digital Assistant) is an intelligent chat assistant that:
- **Handles New Client Onboarding** - Guides customers through tariff selection and connection scheduling
- **Integrates with AmoCRM** - Automatically creates leads and contacts with complete customer data
- **Provides Real-time Support** - Answers questions about services, tariffs, and coverage
- **Tracks Marketing Attribution** - Captures and stores UTM parameters for analytics
- **Works Across Platforms** - Available as web widget and standalone assistant

## Architecture

### Components

1. **FastAPI Backend** (`server.py`)
   - REST API for chat interactions
   - OpenAI GPT-4 integration with function calling
   - AmoCRM API v4 integration
   - Session management
   - UTM tracking

2. **Web Widget** (`widget.js`, `aida_widget.html`)
   - Embeddable chat widget for websites
   - Shadow DOM isolation
   - Responsive design
   - Automatic deployment to aida.smit34.ru

3. **Standalone Assistant** (`aida_chat.html`)
   - Full-page chat interface
   - Direct access at assistant.smit34.ru
   - UTM parameter extraction
   - Session persistence

## Installation

### Prerequisites

- Python 3.8+
- AmoCRM account with API access
- OpenAI API key (GPT-4 access)
- Nginx for reverse proxy
- SSL certificate

### Setup

1. **Clone the repository:**
```bash
git clone https://github.com/ircitdev/smit-aida-assistant.git
cd smit-aida-assistant
```

2. **Install dependencies:**
```bash
pip install fastapi uvicorn openai httpx python-dotenv pydantic
```

3. **Configure environment variables:**
```bash
# OpenAI Configuration
export OPENAI_API_KEY="sk-..."

# AmoCRM Configuration
export AMO_SUBDOMAIN="your-subdomain"
export AMO_ACCESS_TOKEN="your-access-token"
export AMO_REFRESH_TOKEN="your-refresh-token"
export AMO_CLIENT_ID="your-client-id"
export AMO_CLIENT_SECRET="your-client-secret"
export AMO_REDIRECT_URI="https://your-domain.com"

# AmoCRM Lead Custom Fields (New Group)
export AMO_CF_LEAD_CONNECTION_DATE="2578411"
export AMO_CF_LEAD_CONNECTION_TIME="2578413"
export AMO_CF_LEAD_ROUTER="2578885"
export AMO_CF_LEAD_CCTV="2578889"
export AMO_CF_LEAD_STATIC_IP="2578891"
export AMO_CF_LEAD_TARIFF="2578883"
export AMO_CF_LEAD_ADDRESS="2578887"

# UTM Tracking Fields
export AMO_CF_LEAD_UTM_SOURCE="2375289"
export AMO_CF_LEAD_UTM_MEDIUM="2375285"
export AMO_CF_LEAD_UTM_CAMPAIGN="2375287"
export AMO_CF_LEAD_UTM_CONTENT="2375283"
export AMO_CF_LEAD_UTM_TERM="2375291"
```

4. **Run the server:**
```bash
uvicorn server:app --host 0.0.0.0 --port 8900
```

## API Endpoints

### Chat Endpoint

```http
POST /chat
Content-Type: application/json

{
  "session_id": "session-123",
  "message": "Здравствуйте, хочу подключить интернет",
  "utm_source": "google",
  "utm_medium": "cpc",
  "utm_campaign": "winter2025"
}
```

**Response:**
```json
{
  "reply": "Здравствуйте! Помогу подобрать подходящий тариф...",
  "suggestions": [
    "Тарифы для дома",
    "Тарифы для офиса",
    "Проверить адрес подключения"
  ]
}
```

### Available Tools (Function Calling)

AIDA uses OpenAI function calling to execute actions:

1. **`get_tariffs`** - Retrieve available internet tariffs
2. **`check_coverage`** - Verify service availability at address
3. **`create_lead`** - Create lead in AmoCRM with all collected data
4. **`update_lead_referrer`** - Update referral source (recommendation field)

## Features

### 1. Intelligent Onboarding Flow

**Step-by-step process:**
1. Greet customer and identify intent
2. Present tariff options based on requirements
3. Offer additional services (router, CCTV, static IP)
4. Collect contact information
5. Schedule installation
6. Create lead in AmoCRM

### 2. Smart Date Parsing

**Russian month names:**
- "25 ноября 2025" → 25.11.2025

**Relative dates:**
- "завтра утром" → next day, 09:00
- "послезавтра вечером" → day after tomorrow, 18:00
- "через 3 дня" → 3 days from now

**Implementation:**
```python
def parse_relative_date(text: str) -> tuple:
    # Handles: сегодня, завтра, послезавтра, через N дней
    # Time keywords: утром (09:00), вечером (18:00), or explicit time
    return (date_str, time_str)
```

### 3. Suggestion Buttons

**Dynamic suggestions:**
- Automatically parsed from GPT-4 response
- Format: `💡 «Button 1», «Button 2», «Button 3»`
- Displayed as clickable buttons in UI
- Fallback to manual input

### 4. UTM Tracking

**Complete tracking flow:**

**Frontend (aida_chat.html):**
```javascript
// Extract UTM from URL
const urlParams = new URLSearchParams(window.location.search);
const utmData = {
    utm_source: urlParams.get('utm_source') || '',
    utm_medium: urlParams.get('utm_medium') || ''
    // ... other UTM params
};

// Store in sessionStorage
sessionStorage.setItem('utm_data', JSON.stringify(utmData));

// Send with every message
fetch('/chat', {
    body: JSON.stringify({
        session_id: sessionId,
        message: message,
        ...utmData
    })
});
```

**Backend (server.py):**
```python
# Store UTM per session
session_utm: Dict[str, Dict[str, str]] = {}

@app.post("/chat")
async def chat(msg: ChatMessage):
    # Save UTM from message
    if msg.utm_source or msg.utm_medium:
        session_utm[session_id] = {
            "utm_source": msg.utm_source,
            "utm_medium": msg.utm_medium,
            # ...
        }

    # Auto-inject into create_lead
    if function_name == "create_lead":
        arguments["utm_source"] = session_utm[session_id]["utm_source"]
        # ...
```

### 5. AmoCRM Integration

**Lead Creation Structure:**

**Contact (basic fields only):**
- Name
- Phone (WORK)
- Email (WORK)

**Lead (all custom fields):**
- Тариф (ID: 2578883) - Tariff name
- Адрес (ID: 2578887) - Installation address
- Дата подключения (ID: 2578411) - Installation date (timestamp)
- Время подключения (ID: 2578413) - Installation time (text)
- Роутер (ID: 2578885) - Router option
- Видеонаблюдение (ID: 2578889) - CCTV service
- Постоянный IP (ID: 2578891) - Static IP (checkbox)
- UTM метки (IDs: 2375283-2375291) - Marketing attribution

**Code example:**
```python
async def create_amocrm_lead(
    name: str,
    phone: str,
    email: str,
    address: str,
    tariff_name: str,
    tariff_price: int,
    preferred_date: str,
    preferred_time: str,
    router_option: str,
    cctv_option: str,
    static_ip: bool,
    utm_source: str = "",
    # ...
) -> Dict[str, Any]:
    # Step 1: Create contact with basic fields
    contact_data = {
        "name": name,
        "custom_fields_values": [
            {"field_code": "PHONE", "values": [{"value": phone}]},
            {"field_code": "EMAIL", "values": [{"value": email}]}
        ]
    }

    # Step 2: Create lead with all custom fields
    lead_custom_fields = [
        {"field_id": AMO_CF_LEAD_TARIFF, "values": [{"value": tariff_name}]},
        {"field_id": AMO_CF_LEAD_ADDRESS, "values": [{"value": address}]},
        # ... all other fields
    ]
```

### 6. Conversation Flow Enforcement

**Strict workflow rules in SYSTEM_PROMPT:**

```
🚨 КРИТИЧЕСКИ ВАЖНО - СТРОГИЙ ПОРЯДОК ШАГОВ:

1️⃣ Приветствие и выбор тарифа
2️⃣ ПОСЛЕ тарифа → предложить ВСЕ доп. услуги:
   - Роутер (если не входит в тариф)
   - Видеонаблюдение (ВСЕГДА!)
   - Постоянный IP (ВСЕГДА!)

❌ ЗАПРЕЩЕНО спрашивать адрес до завершения шага 2!
3️⃣ Только после доп. услуг → запросить адрес
4️⃣ Контактные данные (имя, телефон, email)
5️⃣ Удобное время подключения
6️⃣ Подтверждение и создание заявки
```

## Widget Integration

### Basic Embedding

Add to your website:

```html
<script src="https://aida.smit34.ru/widget.js"></script>
<script>
    AidaWidget.init({
        apiUrl: 'https://aida.smit34.ru',
        position: 'bottom-right'
    });
</script>
```

### Customization

```javascript
AidaWidget.init({
    apiUrl: 'https://aida.smit34.ru',
    position: 'bottom-right',
    primaryColor: '#007bff',
    buttonText: 'Задать вопрос',
    welcomeMessage: 'Здравствуйте! Я помогу подобрать интернет-тариф.'
});
```

## Deployment

### Systemd Service

Create `/etc/systemd/system/aida-assistant.service`:

```ini
[Unit]
Description=AIDA AI Assistant
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/aida-gpt
Environment="PATH=/usr/bin"
EnvironmentFile=/var/www/aida-gpt/.env
ExecStart=/usr/bin/uvicorn server:app --host 0.0.0.0 --port 8900
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable aida-assistant
sudo systemctl start aida-assistant
sudo systemctl status aida-assistant
```

### Nginx Configuration

**Widget endpoint (aida.smit34.ru):**
```nginx
server {
    listen 80;
    server_name aida.smit34.ru;

    location / {
        root /var/www/aida-widget;
        try_files $uri $uri/ =404;
    }

    location /chat {
        proxy_pass http://127.0.0.1:8900/chat;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Assistant endpoint (assistant.smit34.ru):**
```nginx
server {
    listen 80;
    server_name assistant.smit34.ru;

    location / {
        root /var/www/assistant;
        try_files $uri $uri/ /index.html;
    }

    location /aida/ {
        proxy_pass http://127.0.0.1:8900/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Available Tariffs

| Тариф | Скорость | Цена | Роутер |
|-------|----------|------|--------|
| Пакет Домашний | 70 Мбит/с | 1090 ₽/мес | В подарок |
| Пакет Оптимальный | 100 Мбит/с | 1290 ₽/мес | В подарок |
| Пакет Премиум | 250 Мбит/с | 1490 ₽/мес | В подарок |
| Безлимитище | 500 Мбит/с | 1690 ₽/мес | В подарок |
| Пакет Офисный СТАРТ | 100 Мбит/с | 1850 ₽/мес | Аренда (200₽) |
| Пакет Офисный СТАНДАРТ | 200 Мбит/с | 2300 ₽/мес | Аренда (200₽) |
| Пакет Офисный ОПТИМУМ | 500 Мбит/с | 3500 ₽/мес | Аренда (200₽) |

## Additional Services

- **Видеонаблюдение:**
  - 1 камера + 1 месяц хранения: 1 500 ₽
  - 2 камеры + 1 месяц хранения: 2 900 ₽
  - 4 камеры + 1 месяц хранения: 5 700 ₽

- **Постоянный IP адрес:** 300 ₽/мес

- **Роутер (если не включен):**
  - Аренда: 200 ₽/мес
  - Покупка: Tenda, Xiaomi, D-Link (цена по запросу)

## Monitoring

### Logs

```bash
# View real-time logs
sudo journalctl -u aida-assistant -f

# Search for errors
sudo journalctl -u aida-assistant | grep ERROR

# View session activity
sudo journalctl -u aida-assistant | grep "session-"
```

### Metrics

Track key metrics:
- Sessions created
- Leads generated
- Conversion rate
- Average handling time
- UTM attribution

## Security

- HTTPS enforced on all endpoints
- API keys in environment variables
- Session isolation
- Rate limiting
- Input sanitization

## Troubleshooting

### Common Issues

**1. No leads created in AmoCRM**
- Verify AmoCRM tokens are valid
- Check custom field IDs match AmoCRM configuration
- Review logs for API errors
- Ensure contact creation succeeds before lead creation

**2. Date parsing errors**
- Check for typos in month names
- Verify date format matches expectations
- Test relative date parsing with debug output

**3. Widget not loading**
- Verify CORS configuration
- Check API endpoint accessibility
- Review browser console for errors
- Ensure widget.js is accessible

**4. UTM parameters not captured**
- Verify URL contains UTM parameters
- Check sessionStorage in browser dev tools
- Review backend logs for UTM storage
- Ensure frontend sends UTM with messages

## Development

### Local Development

```bash
# Install dependencies
pip install -r requirements.txt

# Run with hot reload
uvicorn server:app --reload --host 0.0.0.0 --port 8900

# Test frontend locally
python -m http.server 8000
# Open http://localhost:8000/aida_chat.html
```

### Testing

```bash
# Test AmoCRM integration
curl -X POST http://localhost:8900/chat   -H "Content-Type: application/json"   -d '{
    "session_id": "test-123",
    "message": "Хочу подключить интернет"
  }'

# Test UTM tracking
curl -X POST http://localhost:8900/chat   -H "Content-Type: application/json"   -d '{
    "session_id": "test-utm",
    "message": "Здравствуйте",
    "utm_source": "google",
    "utm_medium": "cpc"
  }'
```

## Architecture Diagram

```
┌─────────────────┐
│   Customer      │
│   Browser       │
└────────┬────────┘
         │
    ┌────▼─────┐
    │  Widget  │ (aida.smit34.ru)
    │   HTML   │ or assistant.smit34.ru
    └────┬─────┘
         │ POST /chat
         │
    ┌────▼─────────────┐
    │  FastAPI         │
    │  Backend         │ (port 8900)
    │  - OpenAI GPT-4  │
    │  - Session Mgmt  │
    │  - UTM Tracking  │
    └────┬──────┬──────┘
         │      │
    ┌────▼──┐  └──────────────┐
    │OpenAI │              ┌──▼──────┐
    │ API   │              │ AmoCRM  │
    │GPT-4  │              │ API v4  │
    └───────┘              └─────────┘
```

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

## License

Proprietary - SMIT Internet Provider

## Support

For issues and questions:
- Email: support@smit34.ru
- Telegram: @smit_support
- GitHub Issues: https://github.com/ircitdev/smit-aida-assistant/issues

## Credits

Developed by SMIT Technology Team
- AI Model: OpenAI GPT-4
- CRM: AmoCRM
- Framework: FastAPI

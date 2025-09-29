# API v1 Documentation

## Обзор

API v1 предоставляет интерфейс для работы с платежным агрегатором. Включает операции с транзакциями и спорами.

## Базовые URL

**Production**:
- Base URL: `https://platform.mmgatepay.net/monkey`
- Merchant API: `/api/v1/merchant`
- Dispute API: `/api/v1/dispute`

**Примеры полных URL**:
- Создание транзакции: `https://platform.mmgatepay.net/monkey/api/v1/merchant/create/transaction`
- Создание диспута: `https://platform.mmgatepay.net/monkey/api/v1/dispute/create`

## Аутентификация и подписи

### Заголовки запроса

Все аутентифицированные запросы должны содержать следующие заголовки:

```https
Content-Type: application/json
X-API-KEY: your_public_key
X-SIGNATURE: calculated_signature
X-TIMESTAMP: unix_timestamp
```

### Алгоритм создания подписи

#### Для JSON запросов (application/json)

1. **Подготовка данных**:
   - Сериализация JSON без пробелов: `json.dumps(data, separators=(',', ':'))`
   - Получение текущего timestamp: `time.time()`

2. **Создание сообщения**:
   ```
   message = f"{json_body}:{timestamp}++"
   ```

3. **Кодирование секретного ключа**:
   ```python
   secret_key_bytes = base64.b64encode(secret_key.encode())
   ```

4. **Вычисление подписи**:
   ```python
   signature = hmac.new(
       secret_key_bytes,
       message.encode('utf-8'),
       hashlib.sha256
   ).hexdigest()
   ```

#### Для multipart/form-data запросов

**⚠️ ВАЖНО**: Для multipart запросов подпись создается на основе JSON представления только основных полей формы (исключая файлы).

1. **Подготовка данных**:
   - Извлекаем только обычные поля из формы (не файлы)
   - Сериализуем их в JSON: `json.dumps(form_fields, separators=(',', ':'))`
   - Получение текущего timestamp: `time.time()`

2. **Создание сообщения**:
   ```
   message = f"{json_body}:{timestamp}++"
   ```

3. **Остальные шаги аналогичны JSON запросам**

**Пример для dispute с файлами**:
```python
# Данные для подписи (только основные поля, БЕЗ файлов)
form_data = {
    "transaction_id": "123e4567-e89b-12d3-a456-426614174000"
}

# Создаем подпись как обычно
signature = create_signature(form_data, secret_key)

# Отправляем multipart запрос
```

### Пример кода для создания подписи (Python)

```python
import hmac
import hashlib
import base64
import json
import time

def create_signature(data: dict, secret_key: str) -> tuple[str, str]:
    timestamp = str(time.time())
    body = json.dumps(data, separators=(',', ':'))
    message = f"{body}:{timestamp}++"

    secret_key_bytes = base64.b64encode(secret_key.encode())
    signature = hmac.new(
        secret_key_bytes,
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return signature, timestamp
```

### Валидация времени

- Timestamp не должен быть старше 3 минут (180 секунд)
- Timestamp не должен быть в будущем

---

## Merchant Router (`/api/v1/merchant`)

### Права доступа

- `IsMerchant` - полная аутентификация с проверкой подписи
- `IsHasMerchantKey` - только проверка публичного ключа

### Эндпоинты

#### 1. Создание транзакции

**POST** `/api/v1/merchant/create/transaction`

**Права доступа**: `IsMerchant`

**Описание**: Создает новую транзакцию для платежа

**Схема запроса** (`TransactionRequest`):
```json
{
  "amount": 1000,
  "currency": "RUB",
  "order_type": 1,
  "bank": "sberbank",
  "user_id": "user123",
  "back_url": "https://yoursite.com/callback"
}
```

**Поля запроса**:
- `amount` (int required): Сумма сделки, минимум 1000
- `currency` (str required): Валюта сделки, максимум 5 символов
- `order_type` (int required): Тип сделки (1 - карта, 2 - СБП, 3 - трансгран, 4 - alphpa-alpha, 5 - sber-sber, 6 - tbank-tbank, 7 - ozon-ozon)
- `bank` (str | ""): Банк, максимум 10 символов
- `user_id` (str optional): ID пользователя
- `back_url` (str optional): Ссылка на возврат

**Схема ответа** (`TransactionResponse`):
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "amount": 1000,
  "rate": 78.12,
  "usdt_amount": 12.56,
  "status": "pending",
  "currency": "RUB",
  "payment_method": "card",
  "payment_name": "Олег М.",
  "payment_bank": "Сбербанк",
  "payment_card": "4111111111111111",
  "payment_phone": "+79999999999",
  "link": "https://pay_page"
}
```



#### 2. Получение информации мерчанта

**GET** `/api/v1/merchant/get/info`

**Права доступа**: `IsMerchant`

**Описание**: Возвращает сводную информацию о мерчанте и показатели кошелька.

**Параметры запроса**: отсутствуют

**Схема ответа** (`MerchantDashboard`):
```json
{
  "info": {
    "name": "MerchantName",
    "webhook": "https://your.webhook/endpoint",
    "conversion_rate": 1.23,
    "extradition_rate": 0.95,
    "email": "user@example.com"
  },
  "wallet": {
    "settle_limit": 1000,
    "all_settles": 50000,
    "balance": 1234.56
  }
}
```


#### 3. Получение транзакции по ID

**GET** `/api/v1/merchant/get/transaction?transaction_id=uuid`

**Права доступа**: `IsMerchant`

**Описание**: Получает информацию о конкретной транзакции

**Параметры запроса**:
- `transaction_id` (str): ID транзакции

**Схема ответа**: `TransactionResponse` (см. выше)

#### 4. Получение списка транзакций

**GET** `/api/v1/merchant/get/transactions`

**Права доступа**: `IsMerchant`

**Описание**: Получает список транзакций с пагинацией и фильтрацией

**Параметры запроса**:
- `items_per_page` (str): Количество транзакций на страницу
- `page_number` (str): Номер страницы
- `status` (str, optional): Статус транзакции (`cancelled|expired|disputed|completed|pending`)

**Схема ответа** (`TransactionsResponse`):
```json
{
  "transactions": [
    {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "amount": 1000,
      "rate": 78.12,
      "usdt_amount": 12.56,
      "status": "pending",
      "currency": "RUB",
      "payment_method": "card",
      "payment_name": "Олег М.",
      "payment_bank": "Сбербанк",
      "payment_card": "4111111111111111",
      "payment_phone": "+79999999999",
      "link": "https://pay_page"
    }
  ],
  "pagination": {
    "current_page": 1,
    "page_size": 10,
    "total_count": 100,
    "total_pages": 10,
    "has_next": true,
    "has_prev": false
  }
}
```

#### 5. Получение списка банков

**GET** `/api/v1/merchant/banks`

**Права доступа**: `IsMerchant`

**Описание**: Получает список доступных банков для создания транзакций

**Схема ответа**:
```json
[
  {
    "id": 1,
    "name": "Сбербанк"
  },
  {
    "id": 2,
    "name": "Тинькофф"
  }
]
```

#### 6. Получение timestamp

**GET** `/api/v1/merchant/timestamp`

**Права доступа**: `IsHasMerchantKey`

**Описание**: Получает текущий timestamp сервера для синхронизации

**Схема ответа**:
```json
{
  "timestamp": 1642518123.456
}
```

---

## Test Router

Раздел удален. Тестирование проводится через мерчанта со статусом `test`.

## Dispute Router (`/api/v1/dispute`)

### Эндпоинты

#### 1. Создание спора

**POST** `/api/v1/dispute/create`

**Права доступа**: `IsMerchant`

**Описание**: Создает спор по транзакции с возможностью прикрепления файлов

**Content-Type**: `multipart/form-data`

**Поля запроса**:
- `transaction_id` (string): ID транзакции для создания диспута
- `files` (file[], optional): Прикрепляемые файлы (скриншоты, документы и т.д.)

**⚠️ ВАЖНО: Подпись для multipart запросов**

Для multipart/form-data запросов подпись создается только на основе обычных полей формы (НЕ включая файлы). Подпись всегда вычисляется как JSON, даже если запрос отправляется как multipart.

**Данные для подписи**:
```json
{
  "transaction_id": "123e4567-e89b-12d3-a456-426614174000"
}
```

**Пример запроса**:
```bash
# Сначала создаем подпись на основе JSON данных
DATA='{"transaction_id":"123e4567-e89b-12d3-a456-426614174000"}'
TIMESTAMP=$(date +%s.%3N)
MESSAGE="${DATA}:${TIMESTAMP}++"
SECRET_ENCODED=$(echo -n "$SECRET_KEY" | base64)
SIGNATURE=$(echo -n "$MESSAGE" | openssl dgst -sha256 -hmac "$(echo -n "$SECRET_ENCODED" | base64 -d)" | cut -d' ' -f2)

# Затем отправляем multipart запрос (НЕ указываем Content-Type)
curl -X POST "https://platform.mmgatepay.net/monkey/api/v1/dispute/create/dispute" \
  -H "X-API-KEY: $PUBLIC_KEY" \
  -H "X-SIGNATURE: $SIGNATURE" \
  -H "X-TIMESTAMP: $TIMESTAMP" \
  -F "transaction_id=123e4567-e89b-12d3-a456-426614174000" \
  -F "files=@screenshot.png" \
  -F "files=@receipt.pdf"
```

**Схема ответа** (`CreateDisputeResponse`):
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "created_at": "2024-01-01T12:00:00Z",
  "transaction_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "created"
}
```

**Ограничения файлов**:
- Максимальный размер одного файла: 50 МБ
- Максимальный общий размер всех файлов: 100 МБ
- Максимальное количество файлов: 10
- Поддерживаемые форматы:
  - **Документы**: PDF, DOC, DOCX, TXT
  - **Изображения**: PNG, JPG, JPEG
  - **Видео**: MP4, AVI, MOV, WMV, FLV, WebM, MKV
  - **Аудио**: MP3, WAV, M4A

---

## Статусы транзакций

- `pending` - Ожидает оплаты
- `completed` - Успешно завершена
- `cancelled` - Отменена
- `expired` - Истекла
- `disputed` - Спорная

## Типы оплаты

- `card` - Банковская карта (order_type = 1)
- `sbp` - Система быстрых платежей (order_type = 2)
- `transgran` - Трансгран (order_type = 3)
- `apay` - APay (order_type = 4)
- `spay` - SPay (order_type = 5)
- `tpay` - TPay (order_type = 6)
- `opay` - OPay (order_type = 7)

## Обработка ошибок

### Коды ошибок HTTP

- `400` - Неверный запрос или параметры
- `401` - Неавторизован (неверная подпись или ключи)
- `404` - Ресурс не найден
- `500` - Внутренняя ошибка сервера

### Формат ошибки

```json
{
  "detail": "Описание ошибки"
}
```

### Типичные ошибки

1. **Неверная подпись**:
   ```json
   {
     "detail": "Unauthorized"
   }
   ```

2. **Истекший timestamp**:
   ```json
   {
     "detail": "Unauthorized"
   }
   ```

3. **Транзакция не найдена**:
   ```json
   {
     "detail": "Нет транзакции с заданным ID"
   }
   ```

4. **Недостаточно средств**:
   ```json
   {
     "detail": "Нет доступных реквизитов для создания транзакции"
   }
   ```

---

## Примеры использования

### Создание транзакции (cURL)

```bash
#!/bin/bash

# Данные
PUBLIC_KEY="your_public_key"
SECRET_KEY="your_secret_key"
TIMESTAMP=$(date +%s.%3N)
DATA='{"amount":1000,"currency":"RUB","order_type":1,"bank":"sberbank","user_id":"user123","back_url":"https://yoursite.com/callback"}'

# Создание подписи
MESSAGE="${DATA}:${TIMESTAMP}++"
SECRET_ENCODED=$(echo -n "$SECRET_KEY" | base64)
SIGNATURE=$(echo -n "$MESSAGE" | openssl dgst -sha256 -hmac "$(echo -n "$SECRET_ENCODED" | base64 -d)" | cut -d' ' -f2)

# Запрос
curl -X POST "https://your-api.com/api/v1/merchant/create/transaction" \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: $PUBLIC_KEY" \
  -H "X-SIGNATURE: $SIGNATURE" \
  -H "X-TIMESTAMP: $TIMESTAMP" \
  -d "$DATA"
```

### Создание транзакции (Python)

```python
import requests
import json
import time
import hmac
import hashlib
import base64

def create_transaction():
    # Конфигурация
    PUBLIC_KEY = "your_public_key"
    SECRET_KEY = "your_secret_key"
    BASE_URL = "https://platform.mmgatepay.net"

    # Данные транзакции
    data = {
        "amount": 1000,
        "currency": "RUB",
        "order_type": 1,
        "bank": "sberbank",
        "user_id": "user123",
        "back_url": "https://yoursite.com/callback"
    }

    # Создание подписи
    timestamp = str(time.time())
    body = json.dumps(data, separators=(',', ':'))
    message = f"{body}:{timestamp}++"

    secret_key_bytes = base64.b64encode(SECRET_KEY.encode())
    signature = hmac.new(
        secret_key_bytes,
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    # Заголовки
    headers = {
        'Content-Type': 'application/json',
        'X-API-KEY': PUBLIC_KEY,
        'X-SIGNATURE': signature,
        'X-TIMESTAMP': timestamp
    }

    # Отправка запроса
    response = requests.post(
        f"{BASE_URL}/monkey/api/v1/merchant/create/transaction",
        headers=headers,
        json=data
    )

    return response.json()

# Использование
result = create_transaction()
print(json.dumps(result, indent=2))
```

### Создание диспута с файлами (Python)

```python
import requests
import json
import time
import hmac
import hashlib
import base64
import os

class APIAuth:
    def __init__(self, public_key: str, secret_key: str):
        self.public_key = public_key
        self.secret_key = secret_key
    
    def create_signature(self, data: dict) -> tuple[str, str]:
        timestamp = str(time.time())
        body = json.dumps(data, separators=(',', ':'))
        message = f"{body}:{timestamp}++"
        
        secret_key_bytes = base64.b64encode(self.secret_key.encode())
        signature = hmac.new(
            secret_key_bytes,
            message.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()
        
        return signature, timestamp
    
    def create_headers(self, data: dict) -> dict[str, str]:
        signature, timestamp = self.create_signature(data)
        return {
            'Content-Type': 'application/json',
            'X-API-KEY': self.public_key,
            'X-SIGNATURE': signature,
            'X-TIMESTAMP': timestamp
        }

def create_dispute_with_files(transaction_id: str, files: list[str] = None):
    # Конфигурация  
    PUBLIC_KEY = "your_public_key"
    SECRET_KEY = "your_secret_key"
    BASE_URL = "https://platform.mmgatepay.net"
    
    auth = APIAuth(PUBLIC_KEY, SECRET_KEY)
    
    # Данные для подписи (только основные поля, БЕЗ файлов)
    signature_data = {"transaction_id": transaction_id}
    
    # Создаем заголовки
    headers = auth.create_headers(signature_data)
    # Удаляем Content-Type для multipart запроса
    headers.pop('Content-Type', None)
    
    # Подготавливаем данные формы
    form_data = {"transaction_id": transaction_id}
    
    # Подготавливаем файлы
    files_data = []
    if files:
        for file_path in files:
            if os.path.exists(file_path):
                files_data.append(('files', (os.path.basename(file_path), 
                                            open(file_path, 'rb'), 
                                            'application/octet-stream')))
    
    try:
        # Отправка multipart запроса
        response = requests.post(
            f"{BASE_URL}/monkey/api/v1/dispute/create/dispute",
            headers=headers,
            data=form_data,
            files=files_data
        )
        
        # Закрываем файлы
        for _, file_tuple in files_data:
            file_tuple[1].close()
        
        return response.json()
        
    except Exception as e:
        # Закрываем файлы в случае ошибки
        for _, file_tuple in files_data:
            file_tuple[1].close()
        raise e

# Использование
result = create_dispute_with_files(
    transaction_id="123e4567-e89b-12d3-a456-426614174000",
    files=["screenshot.png", "receipt.pdf"]
)
print(json.dumps(result, indent=2))
```

---

## Webhook уведомления

После изменения статуса транзакции система отправляет webhook на URL мерчанта:

### Формат webhook

```json
{
  "id": "transaction_id",
  "status": "completed",
  "timestamp": 1642518123
}
```

### Заголовки webhook

```http
    "Content-Type": "application/json",
    "X-API-KEY": public_key,
    "X-SIGNATURE": signature,
    "X-TIMESTAMP": timestamp,
    "User-Agent": "MonkeysMoney-Callback/1.0"

```

### Проверка подписи webhook

```python
def verify_webhook_signature(data: dict, signature: str, secret_key: str) -> bool:
    timestamp = str(data['timestamp'])
    body = json.dumps(data, separators=(',', ':'))
    message = f"{body}:{timestamp}++"

    secret_key_bytes = base64.b64encode(secret_key.encode())
    expected_signature = hmac.new(
        secret_key_bytes,
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return expected_signature == signature
```

---

## Утилиты и вспомогательные функции

### Полная библиотека для работы с API

```python
"""
Комплексная библиотека для работы с API платежного агрегатора
"""

import hmac
import hashlib
import base64
import json
import time
import requests
import os
from typing import Dict, Tuple, Any, List, Optional


class APIAuth:
    """Класс для работы с аутентификацией API"""
    
    def __init__(self, public_key: str, secret_key: str):
        self.public_key = public_key
        self.secret_key = secret_key
    
    def create_signature(self, data: Dict[str, Any]) -> Tuple[str, str]:
        """Создает подпись для запроса"""
        timestamp = str(time.time())
        body = json.dumps(data, separators=(',', ':'))
        message = f"{body}:{timestamp}++"
        
        secret_key_bytes = base64.b64encode(self.secret_key.encode())
        signature = hmac.new(
            secret_key_bytes,
            message.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()
        
        return signature, timestamp
    
    def create_headers(self, data: Dict[str, Any]) -> Dict[str, str]:
        """Создает заголовки для аутентифицированного запроса"""
        signature, timestamp = self.create_signature(data)
        
        return {
            'Content-Type': 'application/json',
            'X-API-KEY': self.public_key,
            'X-SIGNATURE': signature,
            'X-TIMESTAMP': timestamp
        }
    
    def verify_webhook_signature(self, data: Dict[str, Any], signature: str) -> bool:
        """Проверяет подпись webhook'а"""
        timestamp = str(data['timestamp'])
        body = json.dumps(data, separators=(',', ':'))
        message = f"{body}:{timestamp}++"
        
        secret_key_bytes = base64.b64encode(self.secret_key.encode())
        expected_signature = hmac.new(
            secret_key_bytes,
            message.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()
        
        return expected_signature == signature


class PaymentAPI:
    """Основной класс для работы с API платежного агрегатора"""
    
    def __init__(self, public_key: str, secret_key: str, base_url: str = "https://platform.mmgatepay.net/monkey"):
        self.auth = APIAuth(public_key, secret_key)
        self.base_url = base_url
    
    def create_transaction(self, amount: int, currency: str = "RUB", 
                          order_type: int = 1, bank: str = "sberbank",
                          user_id: str = "", back_url: str = "") -> Dict[str, Any]:
        """Создает новую транзакцию"""
        
        data = {
            "amount": amount,
            "currency": currency,
            "order_type": order_type,
            "bank": bank,
            "user_id": user_id,
            "back_url": back_url
        }
        
        headers = self.auth.create_headers(data)
        
        response = requests.post(
            f"{self.base_url}/api/v1/merchant/create/transaction",
            headers=headers,
            json=data
        )
        
        return response.json()
    
    def get_transaction(self, transaction_id: str) -> Dict[str, Any]:
        """Получает информацию о транзакции"""
        
        headers = self.auth.create_headers({})
        
        response = requests.get(
            f"{self.base_url}/api/v1/merchant/get/transaction",
            headers=headers,
            params={"transaction_id": transaction_id}
        )
        
        return response.json()
    
    def create_dispute(self, transaction_id: str, files: Optional[List[str]] = None) -> Dict[str, Any]:
        """Создает диспут для транзакции с возможностью прикрепления файлов"""
        
        # Данные для подписи (только основные поля, БЕЗ файлов)
        signature_data = {"transaction_id": transaction_id}
        
        # Создаем заголовки
        headers = self.auth.create_headers(signature_data)
        # Удаляем Content-Type для multipart запроса
        headers.pop('Content-Type', None)
        
        # Подготавливаем данные формы
        form_data = {"transaction_id": transaction_id}
        
        # Подготавливаем файлы
        files_data = []
        if files:
            for file_path in files:
                if os.path.exists(file_path):
                    files_data.append(('files', (os.path.basename(file_path), 
                                                open(file_path, 'rb'), 
                                                'application/octet-stream')))
        
        try:
            # Отправка multipart запроса
            response = requests.post(
                f"{self.base_url}/api/v1/dispute/create/dispute",
                headers=headers,
                data=form_data,
                files=files_data
            )
            
            # Закрываем файлы
            for _, file_tuple in files_data:
                file_tuple[1].close()
            
            return response.json()
            
        except Exception as e:
            # Закрываем файлы в случае ошибки
            for _, file_tuple in files_data:
                file_tuple[1].close()
            raise e


# Пример использования
if __name__ == "__main__":
    # Ваши учетные данные
    PUBLIC_KEY = "your_public_key"
    SECRET_KEY = "your_secret_key"
    
    # Создаем API клиент
    api = PaymentAPI(PUBLIC_KEY, SECRET_KEY)
    
    # Создаем транзакцию
    transaction = api.create_transaction(
        amount=1000,
        currency="RUB",
        order_type=1,
        bank="sberbank",
        user_id="user123",
        back_url="https://yoursite.com/callback"
    )
    
    print("Транзакция создана:")
    print(json.dumps(transaction, indent=2, ensure_ascii=False))
    
    # Создаем диспут с файлами
    if 'id' in transaction:
        dispute = api.create_dispute(
            transaction_id=transaction['id'],
            files=["screenshot.png", "receipt.pdf"]
        )
        
        print("Диспут создан:")
        print(json.dumps(dispute, indent=2, ensure_ascii=False))
```

---

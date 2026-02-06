# Airmiles – Rabita Bank 2-Way API Integration Specification

**Version:** 1.0
**Date:** 2026-02-02
**Status:** Final
**Classification:** Confidential – For Rabita Bank and Airmiles only

---

## Mündəricat

1. [Ümumi Baxış](#1-ümumi-baxış)
2. [2-Way Arxitektura](#2-2-way-arxitektura)
3. [Əlaqə və Təhlükəsizlik](#3-əlaqə-və-təhlükəsizlik)
4. [HMAC Signature Hesablama](#4-hmac-signature-hesablama)
5. [API Endpoints](#5-api-endpoints)
6. [Webhook/Callback (Airmiles → Bank)](#6-webhookcallback-airmiles--bank)
7. [Data Formatları və Validasiya](#7-data-formatları-və-validasiya)
8. [Status və Lifecycle](#8-status-və-lifecycle)
9. [Rate Limiting](#9-rate-limiting)
10. [SLA və Performance](#10-sla-və-performance)
11. [Error Handling](#11-error-handling)
12. [Retry Policy](#12-retry-policy)
13. [Idempotency](#13-idempotency)
14. [Reconciliation / Mutabakat](#14-reconciliation--mutabakat)
15. [Test Mühiti](#15-test-mühiti)
16. [Əlavələr](#16-əlavələr)
17. [Appendix: cURL Nümunələri](#17-appendix-curl-nümunələri)

---

## 1. Ümumi Baxış

### 1.1 Məqsəd

Bu sənəd Rabitə Bank və Airmiles arasında real-time 2-Way API inteqrasiyasını təsvir edir.

### 1.2 "2-Way" Nə Deməkdir?

| İstiqamət | Təsvir | Metod |
|-----------|--------|-------|
| **Bank → Airmiles** | Transaction göndərmə, status sorğulama | POST, GET |
| **Airmiles → Bank** | Status bildirişləri, miles nəticəsi | Webhook (opsional) |

```
┌─────────────┐                              ┌─────────────┐
│             │  POST /transactions          │             │
│   Rabita    │ ───────────────────────────▶ │  Airmiles   │
│    Bank     │                              │             │
│  (Azericard │  GET /transactions/{id}      │             │
│   WAY4)     │ ───────────────────────────▶ │             │
│             │                              │             │
│             │ ◀─────────────────────────── │             │
│             │  Webhook (opsional callback) │             │
└─────────────┘                              └─────────────┘
```

### 1.3 Əsas Flow

1. Bank sistemində kart əməliyyatı baş verir
2. Bank Airmiles API-yə transaction payload göndərir
3. Airmiles müştərini identifikasiya edir (`airmiles_client_code` və ya `fin_code` ilə)
4. Airmiles məlumatları validate edir və **dərhal 201 Created** cavabı qaytarır
5. Əməliyyat asinxron olaraq **pending** statusunda qeyd olunur
6. Gözləmə müddəti (30 gün) bitdikdən sonra miles hesablanır
7. Webhook və ya polling ilə nəticə alınır

### 1.4 Asinxron İşləmə Modeli

Airmiles API sinxron olaraq sürətli ACK (201) qaytarır. Əməliyyat asinxron işlənir.

```
Bank Request → API → Fast ACK (201) → Async Processing → Miles Engine
                │
                └── < 500ms response time
```

---

## 2. 2-Way Arxitektura

### 2.1 Issuer Lifecycle Reallıqları

| Hal | Təsvir | Airmiles Davranışı |
|-----|--------|-------------------|
| **Late Reversal** | Auth/clearing-dən günlər sonra reversal gəlir | Miles verilmişsə → reversal edilir |
| **Late Refund** | Completed transaction-a sonradan refund | Miles geri alınır, status `refunded` olur |
| **Adjustment** | Chargeback/clearing sonrası məbləğ düzəlişi | Proporsional miles adjustment |

---

## 3. Əlaqə və Təhlükəsizlik

### 3.1 Base URL-lər

| Mühit | URL |
|-------|-----|
| **Production** | `https://{{prod-url}}/api/v1` |
| **Staging** | `https://{{staging-url}}/api/v1` |

### 3.2 Transport Təhlükəsizliyi

| Tələb | Dəyər |
|-------|-------|
| Protokol | HTTPS (HTTP qəbul edilmir) |
| TLS Versiya | TLS 1.2 minimum, TLS 1.3 tövsiyə |
| Sertifikat | Etibarlı CA-dan SSL sertifikatı |

### 3.3 IP Allowlist

| Parametr | Dəyər |
|----------|-------|
| Maksimum IP sayı | 10 |
| Dəyişiklik lead time | 48 saat |
| Əlavə üsulu | E-mail: integration@airmiles.az |


Bank tərəfindən göndəriləcək:
- Statik IP adreslər siyahısı
- Dəyişiklik üçün texniki əlaqə şəxsi


### 3.4 Authentication Headers

| Header | Təsvir | Məcburi |
|--------|--------|---------|
| `X-Api-Key` | Rabita Bank üçün unikal API açarı | ✅ |
| `X-Timestamp` | Unix timestamp (saniyə) | ✅ |
| `X-Signature` | HMAC-SHA256 imza | ✅ |
| `X-Request-Id` | Unikal request ID (UUID v4) | ✅ |
| `Content-Type` | `application/json` | ✅ |

### 3.5 Credential Rotation

| Parametr | Dəyər |
|----------|-------|
| Transition period | 24 saat |
| Bildiriş müddəti | 72 saat əvvəl |
| Prosedur | E-mail ilə yeni credentials göndərilir |

---

## 4. HMAC Signature Hesablama

### 4.1 String-to-Sign Formatı

```
string_to_sign = HTTP_METHOD + "\n" +
                 REQUEST_PATH + "\n" +
                 X-Timestamp + "\n" +
                 SHA256(request_body)
```

| Komponent | Nümunə |
|-----------|--------|
| HTTP_METHOD | `POST` |
| REQUEST_PATH | `/partner/v1/transactions` |
| X-Timestamp | `1706284200` |
| SHA256(body) | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |

### 4.2 Signature Hesablama

```
signature = HMAC-SHA256(api_secret, string_to_sign)
```

### 4.3 Nümunə Kod (Python)

```python
import hmac
import hashlib
import time
import json
import uuid

# Credentials (Airmiles tərəfindən verilir)
API_KEY = "rbk_live_xxxxxxxxxxxxxxxx"
API_SECRET = "sk_live_yyyyyyyyyyyyyyyy"

def create_signature(method, path, body, timestamp):
    """HMAC-SHA256 signature yaradır"""

    # Request body-nin SHA256 hash-i
    body_json = json.dumps(body, separators=(',', ':'), ensure_ascii=False)
    body_hash = hashlib.sha256(body_json.encode('utf-8')).hexdigest()

    # String-to-sign
    string_to_sign = f"{method}\n{path}\n{timestamp}\n{body_hash}"

    # HMAC-SHA256
    signature = hmac.new(
        API_SECRET.encode('utf-8'),
        string_to_sign.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return signature

def send_transaction(transaction_data):
    """Transaction göndərir"""
    import requests

    method = "POST"
    path = "/api/v1/transactions"
    timestamp = str(int(time.time()))
    request_id = str(uuid.uuid4())

    signature = create_signature(method, path, transaction_data, timestamp)

    headers = {
        "Content-Type": "application/json",
        "X-Api-Key": API_KEY,
        "X-Timestamp": timestamp,
        "X-Signature": signature,
        "X-Request-Id": request_id
    }

    response = requests.post(
        f"https://{{prod-url}}{path}",
        json=transaction_data,
        headers=headers,
        timeout=30
    )

    return response.json()

# Nümunə istifadə
transaction = {
    "transaction_id": "RBK-2026012615301234567",
    "airmiles_client_code": "1234567890",
    "client_name": "V**** A****v",
    "card_number": "419255******521",
    "transaction_amount": 15000,
    "transaction_currency": "AZN",
    "transaction_amount_azn": 15000,
    "points": None,
    "country_code": "AZE",
    "merchant_id": "MRC123456",
    "mcc": "5411",
    "transaction_type": "Retail",
    "transaction_details": "BRAVO SUPERMARKET BAKI",
    "rrn": "412345678901",
    "arn": "74512345678901234567890",
    "auth_code": "A12345",
    "bank_transaction_date": "2026-01-26",
    "terminal_id": "TERM001",
    "status": "confirm"
}

result = send_transaction(transaction)
print(result)
```

### 4.4 Nümunə Kod (Node.js)

```javascript
const crypto = require('crypto');
const axios = require('axios');
const { v4: uuidv4 } = require('uuid');

// Credentials
const API_KEY = 'rbk_live_xxxxxxxxxxxxxxxx';
const API_SECRET = 'sk_live_yyyyyyyyyyyyyyyy';
const BASE_URL = 'https://{{prod-url}}';

function createSignature(method, path, body, timestamp) {
    // Body JSON string
    const bodyJson = JSON.stringify(body);

    // SHA256 hash of body
    const bodyHash = crypto
        .createHash('sha256')
        .update(bodyJson, 'utf8')
        .digest('hex');

    // String to sign
    const stringToSign = `${method}\n${path}\n${timestamp}\n${bodyHash}`;

    // HMAC-SHA256
    const signature = crypto
        .createHmac('sha256', API_SECRET)
        .update(stringToSign, 'utf8')
        .digest('hex');

    return signature;
}

async function sendTransaction(transactionData) {
    const method = 'POST';
    const path = '/api/v1/transactions';
    const timestamp = Math.floor(Date.now() / 1000).toString();
    const requestId = uuidv4();

    const signature = createSignature(method, path, transactionData, timestamp);

    const headers = {
        'Content-Type': 'application/json',
        'X-Api-Key': API_KEY,
        'X-Timestamp': timestamp,
        'X-Signature': signature,
        'X-Request-Id': requestId
    };

    const response = await axios.post(
        `${BASE_URL}${path}`,
        transactionData,
        { headers, timeout: 30000 }
    );

    return response.data;
}

// Nümunə istifadə
const transaction = {
    transaction_id: 'RBK-2026012615301234567',
    airmiles_client_code: '1234567890',
    client_name: 'V**** A****v',
    card_number: '419255******521',
    transaction_amount: 15000,
    transaction_currency: 'AZN',
    transaction_amount_azn: 15000,
    points: null,
    country_code: 'AZE',
    merchant_id: 'MRC123456',
    mcc: '5411',
    transaction_type: 'Retail',
    transaction_details: 'BRAVO SUPERMARKET BAKI',
    rrn: '412345678901',
    arn: '74512345678901234567890',
    auth_code: 'A12345',
    bank_transaction_date: '2026-01-26',
    terminal_id: 'TERM001',
    status: 'confirm'
};

sendTransaction(transaction)
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

### 4.5 Timestamp Validasiyası

| Parametr | Dəyər |
|----------|-------|
| Format | Unix timestamp (saniyə) |
| Tolerans | ±5 dəqiqə (±300 saniyə) |
| Xəta kodu | `401 TIMESTAMP_EXPIRED` |

---

## 5. API Endpoints

### 5.1 Endpoint Xülasəsi

| Metod | Endpoint | Təsvir |
|-------|----------|--------|
| `POST` | `/transactions` | Yeni transaction yaratma |
| `PATCH` | `/transactions/{transaction_id}` | Status yeniləmə (cancel/refund) |
| `GET` | `/transactions/{transaction_id}` | Tək transaction sorğulama **(Source of Truth)** |
| `GET` | `/transactions` | Toplu transaction sorğulama |
| `GET` | `/reconciliation/daily` | Gündəlik mutabakat |
| `GET` | `/health` | Servis sağlamlıq yoxlaması |

---

### 5.2 POST /transactions – Yeni Transaction

**Request:**
```http
POST /partner/v1/transactions
Content-Type: application/json
X-Api-Key: rbk_live_xxxxxxxx
X-Timestamp: 1706284200
X-Signature: a1b2c3d4e5f6...
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
```

**Request Body:**
```json
{
  "transaction_id": "RBK-2026012615301234567",
  "airmiles_client_code": "1234567890",
  "client_name": "V**** A****v",
  "card_number": "419255******521",
  "transaction_amount": 15000,
  "transaction_currency": "AZN",
  "transaction_amount_azn": 15000,
  "points": null,
  "country_code": "AZE",
  "merchant_id": "MRC123456",
  "mcc": "5411",
  "transaction_type": "Retail",
  "transaction_details": "BRAVO SUPERMARKET BAKI",
  "rrn": "412345678901",
  "arn": "74512345678901234567890",
  "auth_code": "A12345",
  "bank_transaction_date": "2026-01-26",
  "terminal_id": "TERM001",
  "status": "confirm"
}
```

**Response (201 Created):**
```json
{
  "status": "success",
  "data": {
    "id": "txn_abc123xyz",
    "transaction_id": "RBK-2026012615301234567",
    "airmiles_status": "pending",
    "holding_period_ends": "2026-02-25T15:30:00Z",
    "received_at": "2026-01-26T15:30:00Z"
  }
}
```

> **Qeyd:** 201 cavabı transaction-ın qəbul edildiyini bildirir. Miles hesablanması asinxron baş verir.

---

### 5.3 PATCH /transactions/{transaction_id} – Status Yeniləmə

Cancel, refund və ya late reversal üçün istifadə olunur.

**Request:**
```http
PATCH /partner/v1/transactions/RBK-2026012615301234567
Content-Type: application/json
X-Api-Key: rbk_live_xxxxxxxx
X-Timestamp: 1706370600
X-Signature: x1y2z3...
X-Request-Id: 660e8400-e29b-41d4-a716-446655440001
```

**Cancel üçün Body:**
```json
{
  "status": "cancel",
  "reason": "Customer request"
}
```

**Refund üçün Body:**
```json
{
  "status": "refund",
  "refund_amount": 15000,
  "refund_reason": "Product return",
  "refund_transaction_id": "RBK-2026012715301234568"
}
```

**Late Reversal üçün Body:**
```json
{
  "status": "reversal",
  "reason": "Late reversal from clearing",
  "original_rrn": "412345678901"
}
```

> **Qeyd:** `refund_transaction_id` opsionaldır. Göndərilməsə, orijinal `transaction_id` ilə refund qeyd olunur. Status update / refund işlemləri `transaction_id` ilə edilir; RRN/ARN log məqsədilə saxlanılır.

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "txn_abc123xyz",
    "transaction_id": "RBK-2026012615301234567",
    "previous_status": "completed",
    "current_status": "refunded",
    "miles_reversed": 150,
    "updated_at": "2026-01-27T10:00:00Z"
  }
}
```

---

### 5.4 GET /transactions/{transaction_id} – Status Sorğulama (Source of Truth)

> **Vacib:** Bu endpoint transaction statusunun **mötəbər mənbəyidir (source of truth)**. Webhook down olsa belə, bank bu endpoint ilə statusu sorğulaya bilər.

**Request:**
```http
GET /partner/v1/transactions/RBK-2026012615301234567
X-Api-Key: rbk_live_xxxxxxxx
X-Timestamp: 1706370600
X-Signature: x1y2z3...
X-Request-Id: 770e8400-e29b-41d4-a716-446655440002
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "transaction_id": "RBK-2026012615301234567",
    "airmiles_status": "completed",
    "bank_status": "confirm",
    "transaction_amount": 15000,
    "transaction_currency": "AZN",
    "transaction_amount_azn": 15000,
    "rrn": "412345678901",
    "arn": "74512345678901234567890",
    "auth_code": "A12345",
    "mcc": "5411",
    "miles_calculated": 150,
    "miles_posted": true,
    "miles_posted_at": "2026-02-25T15:30:00Z",
    "holding_period_ends": "2026-02-25T15:30:00Z",
    "received_at": "2026-01-26T15:30:00Z",
    "updated_at": "2026-02-25T15:30:00Z"
  }
}
```

**Response (404 Not Found):**
```json
{
  "status": "error",
  "error_code": "NOT_FOUND",
  "message": "Transaction not found"
}
```

---

### 5.5 GET /transactions – Toplu Sorğulama

**Request:**
```http
GET /partner/v1/transactions?start_date=2026-01-01&end_date=2026-01-31&page=1&per_page=100
X-Api-Key: rbk_live_xxxxxxxx
X-Timestamp: 1706370600
X-Signature: x1y2z3...
X-Request-Id: 880e8400-e29b-41d4-a716-446655440003
```

**Query Parameters:**

| Parametr | Tip | Məcburi | Təsvir |
|----------|-----|---------|--------|
| `start_date` | string | ✅ | Başlanğıc tarixi (YYYY-MM-DD) |
| `end_date` | string | ✅ | Bitmə tarixi (YYYY-MM-DD) |
| `status` | string | ❌ | Status filter (pending, completed, etc.) |
| `page` | integer | ❌ | Səhifə nömrəsi (default: 1) |
| `per_page` | integer | ❌ | Səhifədəki sayı (default: 100, max: 1000) |

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "transactions": [
      {
        "transaction_id": "RBK-2026012615301234567",
        "airmiles_status": "completed",
        "transaction_amount_azn": 15000,
        "rrn": "412345678901",
        "miles_calculated": 150,
        "received_at": "2026-01-26T15:30:00Z"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 100,
      "total_pages": 5,
      "total_count": 487
    }
  }
}
```

---

### 5.6 GET /health – Sağlamlıq Yoxlaması

**Request:**
```http
GET /partner/v1/health
```

> **Qeyd:** Health endpoint authentication tələb etmir.

**Response (200 OK):**
```json
{
  "status": "healthy",
  "version": "3.1",
  "timestamp": "2026-01-26T15:30:00Z"
}
```

**Response (503 Service Unavailable):**
```json
{
  "status": "unhealthy",
  "message": "Database connection failed",
  "timestamp": "2026-01-26T15:30:00Z"
}
```

---

## 6. Webhook/Callback (Airmiles → Bank)

### 6.1 Webhook Opsionallığı

> **Vacib:** Webhook **tövsiyə olunur, lakin məcburi deyil**.
> **Source of Truth:** `GET /transactions/{transaction_id}` endpoint-i həmişə mötəbər mənbədir.

### 6.2 Event Növləri

| Event | Təsvir |
|-------|--------|
| `transaction.received` | Transaction qəbul edildi |
| `transaction.completed` | Miles hesablandı və balansa əlavə edildi |
| `transaction.cancelled` | Transaction ləğv edildi |
| `transaction.refunded` | Refund edildi, miles geri alındı |
| `transaction.reversed` | Late reversal, miles geri alındı |
| `transaction.error` | Xəta baş verdi |

### 6.3 Webhook Payload

```json
{
  "event": "transaction.completed",
  "webhook_id": "wh_xyz789",
  "timestamp": "2026-02-25T15:30:00Z",
  "data": {
    "transaction_id": "RBK-2026012615301234567",
    "airmiles_client_code": "1234567890",
    "rrn": "412345678901",
    "miles_calculated": 150,
    "miles_posted": true,
    "customer_new_balance": 12500
  }
}
```

### 6.4 Webhook Retry Policy

| Cəhd | Gecikmə |
|------|---------|
| 1 | Dərhal |
| 2 | 1 dəqiqə |
| 3 | 5 dəqiqə |
| 4 | 30 dəqiqə |
| 5 | 2 saat |

---

## 7. Data Formatları və Validasiya

### 7.1 Field Spesifikasiyası

| Field | Tip | Məcburi | Max Uzunluq | Təsvir |
|-------|-----|---------|-------------|--------|
| `transaction_id` | string | ✅ | 50 | Bankın unikal transaction ID-si |
| `airmiles_client_code` | string | 🔶 Şərti | 20 | Airmiles müştəri kodu (fin_code olmadıqda məcburi) |
| `fin_code` | string | 🔶 Şərti | 7 | FIN kodu (airmiles_client_code olmadıqda məcburi) |
| `phone_number` | string | 🔶 Şərti | 12 | (airmiles_client_code olmadıqda məcburi) |
| `client_name` | string | ✅ | 50 | Maskalanmış müştəri adı |
| `card_number` | string | ✅ | 19 | Maskalanmış kart nömrəsi |
| `transaction_amount` | integer | ✅ | - | Məbləğ (minor units / qəpik) |
| `transaction_currency` | string | ✅ | 3 | ISO 4217 valyuta kodu |
| `transaction_amount_azn` | integer | ✅ | - | AZN məbləği (minor units) |
| `points` | integer | ❌ | - | Bank kampaniya xalları |
| `country_code` | string | ✅ | 3 | ISO 3166-1 alpha-3 |
| `merchant_id` | string | ✅ | 20 | Merchant identifikatoru |
| `mcc` | string | 🔶 Tövsiyə | 4 | Merchant Category Code (kampaniya matching üçün) |
| `transaction_type` | string | ✅ | 20 | Əməliyyat növü |
| `transaction_details` | string | ❌ | 100 | Əməliyyat təsviri |
| `rrn` | string | 🔶 Tövsiyə | 12 | Retrieval Reference Number |
| `arn` | string | ❌ | 30 | Acquirer Reference Number |
| `auth_code` | string | ❌ | 6 | Authorization Code |
| `bank_transaction_date` | string | ✅ | 10 | Tarix (YYYY-MM-DD) |
| `terminal_id` | string | 🔴 Biznes-məcburi | 20 | Terminal ID (kampaniya üçün **məcburidir**) |
| `status` | string | ✅ | 10 | confirm/cancel/refund/reversal |

### 7.2 terminal_id Qaydası

> ⚠️ **Biznes-Məcburi:** `terminal_id` texniki olaraq nullable olsa da, kampaniya əsaslı miles hesablaması üçün **mütləq göndərilməlidir**.

| Hal | terminal_id | Nəticə |
|-----|-------------|--------|
| Göndərildi | `TERM001` | ✅ Kampaniya matching aktiv |
| Göndərilmədi | `null` | ⚠️ Yalnız base miles, kampaniya yox |

### 7.3 Customer Identification & Airmiles Client Code

#### 7.3.1 airmiles_client_code Nədir?

`airmiles_client_code` Airmiles sistemində müştərinin **unikal identifikatorudur**:
- Miles balansı bu koda bağlıdır
- Mobil tətbiqdə bu kodla görünür
- Bütün transaction-lar bu kodla əlaqələndirilir

#### 7.3.2 Müştəri İdentifikasiya Qaydaları (Precedence Rules)

```
┌─────────────────────────────────────────────────────────────────┐
│                  Customer Identification Flow                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  airmiles_client_code mövcuddur?                                │
│           │                                                      │
│     ┌─────┴─────┐                                               │
│     │           │                                               │
│    YES         NO                                                │
│     │           │                                               │
│     ▼           ▼                                               │
│  ┌──────────┐  fin_code mövcuddur?                              │
│  │ PRIMARY  │       │                                           │
│  │ SOURCE   │  ┌────┴────┐                                      │
│  │ OF TRUTH │  │         │                                      │
│  └──────────┘ YES        NO                                      │
│               │         │                                       │
│               ▼         ▼                                       │
│          ┌──────────┐  ┌──────────┐                             │
│          │ FIN ilə  │  │  ERROR   │                             │
│          │ resolve  │  │  400     │                             │
│          │ və ya    │  └──────────┘                             │
│          │ auto-    │                                           │
│          │ create   │                                           │
│          └──────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Ssenari | airmiles_client_code | fin_code | Davranış |
|---------|---------------------|----------|----------|
| **Normal** | `1234567890` | `null` | ✅ airmiles_client_code istifadə edilir |
| **FIN Fallback** | `null` | `ABC1234` | ✅ FIN ilə müştəri tapılır və ya yaradılır |
| **Hər ikisi** | `1234567890` | `ABC1234` | ⚠️ Consistency check - uyğunsuzluqda 409 |
| **Heç biri** | `null` | `null` | ❌ 400 VALIDATION_ERROR |

#### 7.3.3 FIN ilə Müştəri Tapılmadıqda (Auto-Create)

Əgər `fin_code` göndərilib amma Airmiles sistemində müştəri tapılmırsa:

1. Airmiles yeni müştəri yaradır(phone_number ile)
2. Yeni `airmiles_client_code` generasiya edilir
3. Response-da `airmiles_client_code_created: true` qaytarılır

**Auto-Create Response:**
```json
{
  "status": "success",
  "data": {
    "id": "txn_abc123xyz",
    "transaction_id": "RBK-2026012615301234567",
    "airmiles_client_code": "9876543210",
    "airmiles_client_code_created": true,
    "fin_code": "ABC1234",
    "airmiles_status": "pending"
  }
}
```

#### 7.3.4 FIN Conflict (409)

Əgər hər iki field göndərilibsə və uyğunsuzluq varsa:

```json
{
  "status": "error",
  "error_code": "FIN_CONFLICT",
  "message": "airmiles_client_code and fin_code do not match the same customer",
  "details": {
    "provided_airmiles_client_code": "1234567890",
    "provided_fin_code": "ABC1234",
    "expected_fin_code_for_customer": "XYZ9999"
  }
}
```

### 7.4 Amount Formatı (VACIB)

Bütün məbləğlər **minor currency units** (qəpik) şəklindədir:

| Real Məbləğ | Göndəriləcək Dəyər |
|-------------|-------------------|
| 150.00 AZN | `15000` |
| 25.50 AZN | `2550` |
| 1,234.99 AZN | `123499` |

> ⚠️ **Decimal point istifadə etməyin!** `150.00` deyil, `15000` göndərin.

### 7.5 transaction_id Formatı

| Tələb | Dəyər |
|-------|-------|
| Unikallıq | Bank daxilində qlobal unikal |
| Max uzunluq | 50 simvol |
| İcazə verilən simvollar | A-Z, a-z, 0-9, -, _ |
| Tövsiyə olunan format | `RBK-{timestamp}{sequence}` |

Nümunə: `RBK-2026012615301234567`

### 7.6 RRN (Retrieval Reference Number) Formatı

| Tələb | Dəyər |
|-------|-------|
| Uzunluq | 12 simvol (standart) |
| Format | Adətən rəqəmsal |
| İstifadə | Issuer dünyasında transaction tapma |

Nümunə: `412345678901`

### 7.7 Data Maskalama Qaydaları

**Client Name:**
```
Format: {Ad ilk hərf}**** {Soyad ilk hərf}****{Soyad son hərf}
Nümunə: Vahid Amanov → V**** A****v
```

**Card Number (PAN):**
```
Format: {İlk 6 rəqəm}******{Son 3 rəqəm}
Nümunə: 4192551234567521 → 419255******521
```

### 7.8 MCC (Merchant Category Code)

MCC kampaniya uyğunlaşdırması üçün **çox tövsiyə olunur**:

| MCC | Kateqoriya |
|-----|------------|
| 5411 | Grocery stores, supermarkets |
| 5812 | Restaurants |
| 5541 | Gas stations |
| 5311 | Department stores |
| 5912 | Pharmacies |
| 7011 | Hotels |
| 4111 | Transportation |

> **Qeyd:** MCC göndərilməsə, kampaniya matching məhdud olacaq. Əgər bank MCC-dən xəbərsizdirsə, Airmiles merchant_id əsasında MCC təyin etməyə çalışacaq.

### 7.9 Transaction Type Dəyərləri

| Dəyər | Təsvir |
|-------|--------|
| `Retail` | POS alış-verişi |
| `E-commerce` | Onlayn alış-veriş |
| `ATM` | Bankomat əməliyyatı |
| `Transfer` | Pul köçürməsi |
| `Payment` | Ödəniş (kommunal və s.) |

### 7.10 Status Dəyərləri

| Dəyər | Təsvir | Miles Təsiri |
|-------|--------|--------------|
| `confirm` | Əməliyyat təsdiqləndi | Miles hesablanacaq |
| `cancel` | Əməliyyat ləğv edildi | Miles hesablanmayacaq / geri alınacaq |
| `refund` | Məbləğ geri qaytarıldı | Proporsional miles geri alınacaq |
| `reversal` | Late reversal (clearing sonrası) | Miles tam geri alınacaq |

---

## 8. Status və Lifecycle

### 8.1 Transaction Status Axını

```
PENDING → HOLDING → PROCESSING → COMPLETED
              │                      │
              ├─→ CANCELLED          ├─→ REFUNDED
              └─→ REFUNDED           └─→ REVERSED
```

### 8.2 Status Təsvirləri

| Status | Təsvir |
|--------|--------|
| `pending` | Transaction qəbul edildi |
| `holding` | Gözləmə dövrü (30 gün) |
| `processing` | Miles hesablanır |
| `completed` | Miles balansa əlavə edildi |
| `cancelled` | Ləğv edildi |
| `refunded` | Geri qaytarıldı, miles reversed |
| `reversed` | Late reversal, miles reversed |
| `error` | Xəta |

### 8.3 Late Refund/Reversal

Miles verilmişsə və sonradan refund/reversal gəlsə, miles geri alınır.

---

## 9. Rate Limiting

| Limit Növü | Dəyər |
|------------|-------|
| Request per second | **50** |
| Request per minute | **1,000** |
| Burst limit | **100** |

**Headers:**
```
X-RateLimit-Limit: 50
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1706284260
```

---

## 10. SLA və Performance

| Metrik | Hədəf |
|--------|-------|
| Aylıq uptime | **99.9%** |
| API response (p99) | < 500ms |
| Miles hesablama | < 5 dəqiqə (holding sonrası) |

---

## 11. Error Handling

### 11.1 Error Response Formatı

```json
{
  "status": "error",
  "error_code": "ERROR_CODE",
  "message": "Human readable message",
  "details": [
    {
      "field": "transaction_amount",
      "message": "must be greater than 0"
    }
  ],
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-01-26T15:30:00Z"
}
```

### 11.2 Error Kodları

| HTTP Status | Error Code | Təsvir | Retry? |
|-------------|------------|--------|--------|
| 400 | `INVALID_REQUEST` | Request parametrləri yanlış | ❌ |
| 400 | `VALIDATION_ERROR` | Field validasiyası uğursuz | ❌ |
| 400 | `INVALID_AMOUNT` | Məbləğ ≤ 0 | ❌ |
| 400 | `INVALID_DATE_FORMAT` | Tarix formatı yanlış | ❌ |
| 400 | `INVALID_STATUS` | Status dəyəri tanınmır | ❌ |
| 400 | `UNSUPPORTED_CURRENCY` | Valyuta dəstəklənmir | ❌ |
| 401 | `UNAUTHORIZED` | API Key yanlış/yoxdur | ❌ |
| 401 | `INVALID_SIGNATURE` | HMAC signature yanlış | ❌ |
| 401 | `TIMESTAMP_EXPIRED` | Timestamp ±5 dəqiqə xaricində | ✅* |
| 403 | `IP_NOT_ALLOWED` | IP allowlist-də deyil | ❌ |
| 404 | `NOT_FOUND` | Transaction tapılmadı | ❌ |
| 409 | `CONFLICT` | Transaction artıq mövcud | ❌ |
| 409 | `INVALID_STATUS_TRANSITION` | Status keçidi mümkün deyil | ❌ |
| 422 | `CUSTOMER_NOT_FOUND` | Airmiles müştəri tapılmadı | ❌ |
| 422 | `CARD_NOT_REGISTERED` | Kart müştəriyə bağlı deyil | ❌ |
| 429 | `RATE_LIMIT_EXCEEDED` | Rate limit aşıldı | ✅ |
| 500 | `INTERNAL_ERROR` | Server xətası | ✅ |
| 502 | `BAD_GATEWAY` | Upstream xətası | ✅ |
| 503 | `SERVICE_UNAVAILABLE` | Servis müvəqqəti əlçatmazdır | ✅ |

*Timestamp düzəldildikdən sonra

### 11.3 Çoxlu Validasiya Xətaları

```json
{
  "status": "error",
  "error_code": "VALIDATION_ERROR",
  "message": "Multiple validation errors",
  "details": [
    {"field": "transaction_amount", "message": "must be greater than 0"},
    {"field": "airmiles_client_code", "message": "is required"},
    {"field": "bank_transaction_date", "message": "must be YYYY-MM-DD format"}
  ]
}
```

---

## 12. Retry Policy

### 12.1 Retry Qərarı

| HTTP Status | Retry? | Qeyd |
|-------------|--------|------|
| 2xx | ❌ | Uğurlu |
| 400 | ❌ | Request-i düzəldin |
| 401 | ❌ | Credentials yoxlayın |
| 403 | ❌ | IP allowlist yoxlayın |
| 404 | ❌ | transaction_id yoxlayın |
| 409 | ❌ | Duplicate handling |
| 422 | ❌ | Business logic xətası |
| 429 | ✅ | Retry-After header-ə əsasən |
| 500 | ✅ | Server xətası |
| 502 | ✅ | Gateway xətası |
| 503 | ✅ | Müvəqqəti əlçatmazlıq |

### 12.2 Tövsiyə Olunan Retry Strategiyası

```
Retry count: 4
Backoff: Exponential with jitter

Cəhd 1: Dərhal
Cəhd 2: 2 saniyə + random(0-500ms)
Cəhd 3: 4 saniyə + random(0-500ms)
Cəhd 4: 8 saniyə + random(0-500ms)
Cəhd 5: 16 saniyə + random(0-500ms)
```

### 12.3 Circuit Breaker Tövsiyəsi

Bank tərəfində circuit breaker tətbiq edilməlidir:

| Parametr | Dəyər |
|----------|-------|
| Açılma threshold | 5 ardıcıl uğursuzluq |
| Half-open sonra | 30 saniyə |
| Bağlanma threshold | 2 ardıcıl uğur |

---

## 13. Idempotency

### 13.1 Idempotency Açarı

`transaction_id` idempotency key kimi xidmət edir.

### 13.2 Duplicate Request Davranışı

| Ssenari | Response | HTTP Status |
|---------|----------|-------------|
| İlk request | Normal processing | 201 Created |
| Eyni payload ilə duplicate | Mövcud record qaytarılır | 200 OK |
| Fərqli payload ilə duplicate | Reject | 409 Conflict |

### 13.3 Duplicate Response Nümunəsi

```json
{
  "status": "success",
  "data": {
    "id": "txn_abc123xyz",
    "transaction_id": "RBK-2026012615301234567",
    "airmiles_status": "pending",
    "duplicate": true,
    "original_received_at": "2026-01-26T15:30:00Z"
  }
}
```

### 13.4 Timeout Ssenarisi

Bank request göndərdi amma cavab almadı:

1. **Əvvəlcə status yoxlayın:**
   ```http
   GET /transactions/RBK-2026012615301234567
   ```

2. **404 alsanız:** Request çatmayıb, yenidən göndərin
3. **200 alsanız:** Transaction artıq mövcuddur, duplicate göndərməyin

---

## 14. Reconciliation / Mutabakat

### 14.1 Ümumi Baxış

> **Vacib:** Aylıq reconciliation hesabatı Rabita Bank və Airmiles arasında **maliyyə mötəbər mənbəyidir (financial source of truth)**.

### 14.2 GET /reconciliation/daily

```json
{
  "status": "success",
  "data": {
    "date": "2026-01-26",
    "summary": {
      "total_transactions": 1523,
      "total_amount_azn": 456780000,
      "miles_posted": 4200,
      "miles_reversed": 80
    },
    "report_url": "https://{{prod-url}}/reports/recon_2026-01-26_rabita.csv"
  }
}
```

---

## 15. Test Mühiti

### 15.1 Staging

| Parametr | Dəyər |
|----------|-------|
| Base URL | `https://{{staging-url}}/api/v1` |

### 15.2 Test Credentials Alma

1. integration@airmiles.az-a müraciət edin
2. Test API Key və Secret alacaqsınız
3. Staging IP allowlist-ə əlavə olunacaqsınız

### 15.3 Test Müştəri Kodları

| Kod | Ssenari |
|-----|---------|
| `TEST000001` | Normal flow |
| `TEST000002` | Customer not found |
| `FIN_TEST01` | FIN auto-create test |

### 15.3 Simulyasiya Header-ləri

Staging mühitində bu header-lərlə xüsusi ssenariləri test edə bilərsiniz:

| Header | Dəyər | Effekt |
|--------|-------|--------|
| `X-Simulate-Delay` | `5000` | 5 saniyə gecikmə |
| `X-Simulate-Error` | `500` | 500 Internal Error qaytarır |
| `X-Simulate-Error` | `429` | 429 Rate Limit qaytarır |
| `X-Simulate-Timeout` | `true` | Timeout simulyasiyası |
| `X-Simulate-Webhook-Fail` | `true` | Webhook delivery uğursuz |
| `X-Simulate-Late-Refund` | `true` | Late refund ssenarisi |

---

## 16. Əlavələr

### 16.1 Checklist - Canlıya Keçmədən Əvvəl

- [ ] HMAC signature düzgün hesablanır
- [ ] terminal_id göndərilir (kampaniya üçün)
- [ ] RRN/MCC göndərilir (tövsiyə)
- [ ] Error handling implement edildi
- [ ] Retry logic implement edildi
- [ ] Webhook və ya polling quruldu

### 16.2 Əlaqə

| Rol | Əlaqə |
|-----|-------|
| İnteqrasiya | integration@airmiles.az |
| Təcili Xətt | +994 XX XXX XX XX |


---

**Sənədin Sonu**

Version 1.0 | 2026-02-02 | Confidential

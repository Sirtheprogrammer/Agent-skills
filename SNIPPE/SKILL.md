# Snippe Payment Gateway Integration Skill

## Overview

This document provides a **framework-agnostic** guide for integrating **Snippe Payment Gateway** for processing mobile money payments. Snippe is a payment gateway that supports mobile payments via USSD, SMS, and other channels for Tanzania and East Africa.

**Compatible With:** Laravel, Node.js/Express, Python/Django, PHP/Vanilla, Go, Java, Ruby on Rails, ASP.NET, and any HTTP-capable framework.

---

## 1. Architecture Overview

```
Client Application (Frontend - any framework)
    ↓
HTTP API Endpoints (Backend - any language/framework)
    ↓
PaymentController/Handler (Create Order, Check Status)
    ↓
Snippe REST API (https://api.snippe.sh/v1)
    ↓
Database (PaymentGateway, Transaction records)
    ↓
Webhook Handler (Optional - payment notifications)
```

---

## 2. Data Models & Schema

### 2.1 PaymentGateway Record
Stores Snippe credentials and configuration.

**Fields:**
| Field | Type | Example | Required |
|-------|------|---------|----------|
| `name` | String | "snippe" | Yes |
| `display_name` | String | "Snippe Payment Gateway" | Yes |
| `api_key` | String | Bearer token from Snippe | Yes |
| `webhook_url` | String | "https://yourapp.com/webhooks/snippe" | No |
| `base_url` | String | "https://api.snippe.sh/v1" | Yes |
| `is_active` | Boolean | true/false | Yes |
| `description` | String | "Mobile money gateway for Tanzania" | No |

**Database Schema (SQL):**
```sql
CREATE TABLE payment_gateways (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) UNIQUE,
    display_name VARCHAR(255),
    api_key VARCHAR(500),
    webhook_url VARCHAR(500),
    base_url VARCHAR(500),
    is_active BOOLEAN DEFAULT true,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 2.2 Transaction Record
Tracks all payment attempts and status updates.

**Fields:**
| Field | Type | Purpose |
|-------|------|---------|
| `id` | Integer | Unique transaction ID |
| `page_id` | Integer | Reference to product/page being purchased |
| `order_id` | String | Internal order ID (ORD-xxxxx) |
| `reference` | String | Snippe payment reference (pay_xxxxx) |
| `buyer_email` | String | Customer email address |
| `buyer_name` | String | Customer full name |
| `buyer_phone` | String | Customer phone (255XXXXXXXXX format) |
| `amount` | Decimal | Payment amount in TZS |
| `currency` | String | "TZS" |
| `gateway` | String | "snippe" |
| `payment_status` | String | "pending", "completed", "canceled" |
| `transaction_id` | String | Snippe transaction ID |
| `channel` | String | "Vodacom", "Airtel", etc. |
| `msisdn` | String | Phone number used for payment |
| `response_data` | JSON | Full Snippe API response |
| `completed_at` | Timestamp | When payment completed |
| `created_at` | Timestamp | When transaction created |
| `updated_at` | Timestamp | Last update time |

**Database Schema (SQL):**
```sql
CREATE TABLE transactions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    page_id INT NOT NULL,
    order_id VARCHAR(100) UNIQUE,
    reference VARCHAR(100),
    buyer_email VARCHAR(255),
    buyer_name VARCHAR(255),
    buyer_phone VARCHAR(20),
    amount DECIMAL(10, 2),
    currency VARCHAR(3),
    gateway VARCHAR(50),
    payment_status VARCHAR(50),
    transaction_id VARCHAR(100),
    channel VARCHAR(50),
    msisdn VARCHAR(20),
    response_data JSON,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (page_id) REFERENCES pages(id)
);
```

### 2.3 Environment Configuration
Store sensitive credentials in environment variables:

```env
SNIPPE_API_KEY=your_bearer_token_here
SNIPPE_WEBHOOK_URL=https://yourapp.com/webhooks/snippe
SNIPPE_BASE_URL=https://api.snippe.sh/v1
```

---

## 3. Core Implementation Logic

### 3.1 Create Payment Order Flow

**Pseudo-Code (Framework-Independent):**

```
FUNCTION createPaymentOrder(pageId, buyerPhone, buyerName, buyerEmail):
    
    // Step 1: Validate inputs
    IF NOT isValidPhone(buyerPhone):
        RETURN error("Invalid phone number")
    
    // Step 2: Fetch gateway configuration from database
    gatewayConfig = database.query("SELECT * FROM payment_gateways WHERE name='snippe' AND is_active=true")
    IF NOT gatewayConfig:
        RETURN error("Snippe gateway not configured")
    
    // Step 3: Normalize phone number to Tanzania format
    normalizedPhone = normalizePhoneNumber(buyerPhone)  // Result: 255XXXXXXXXX
    
    // Step 4: Generate unique order ID
    orderId = "ORD-" + unixTimestamp() + "-" + randomId()
    
    // Step 5: Create transaction record in database
    transaction = database.insert("transactions", {
        page_id: pageId,
        order_id: orderId,
        buyer_phone: normalizedPhone,
        buyer_name: buyerName,
        buyer_email: buyerEmail,
        amount: getPagePrice(pageId),
        currency: "TZS",
        gateway: "snippe",
        payment_status: "pending"
    })
    
    // Step 6: Prepare Snippe API request
    snippeRequest = {
        payment_type: "mobile",
        details: {
            amount: getPagePrice(pageId),
            currency: "TZS"
        },
        phone_number: normalizedPhone,
        customer: {
            firstname: firstName(buyerName),
            lastname: lastName(buyerName),
            email: buyerEmail
        },
        webhook_url: gatewayConfig.webhook_url,
        metadata: {
            order_id: orderId
        }
    }
    
    // Step 7: Call Snippe API
    httpHeaders = {
        "Authorization": "Bearer " + gatewayConfig.api_key,
        "Content-Type": "application/json"
    }
    
    response = httpPost("https://api.snippe.sh/v1/payments", snippeRequest, httpHeaders)
    
    // Step 8: Handle API response
    IF response.status != "success":
        database.update("transactions", transaction.id, {payment_status: "failed"})
        RETURN error(response.message)
    
    // Step 9: Store Snippe response in transaction
    database.update("transactions", transaction.id, {
        reference: response.data.reference,
        payment_status: response.data.status,
        response_data: JSON.stringify(response)
    })
    
    // Step 10: Return transaction details to client
    RETURN {
        status: "success",
        transaction_id: transaction.id,
        reference: response.data.reference,
        amount: response.data.amount,
        currency: response.data.currency
    }
END FUNCTION
```

### 3.2 Check Payment Status Flow

```
FUNCTION checkPaymentStatus(transactionId):
    
    // Step 1: Fetch transaction from database
    transaction = database.query("SELECT * FROM transactions WHERE id = ?", transactionId)
    IF NOT transaction:
        RETURN error("Transaction not found")
    
    // Step 2: Fetch gateway configuration
    gatewayConfig = database.query("SELECT * FROM payment_gateways WHERE name='snippe'")
    
    // Step 3: Call Snippe API with payment reference
    httpHeaders = {
        "Authorization": "Bearer " + gatewayConfig.api_key
    }
    
    reference = transaction.reference
    response = httpGet("https://api.snippe.sh/v1/payments/" + reference, httpHeaders)
    
    // Step 4: Handle API response
    IF response.status != "success":
        RETURN error("Failed to check status")
    
    // Step 5: Normalize status (Snippe returns lowercase)
    paymentStatus = response.data.status.toUpperCase()  // "PENDING", "COMPLETED", "CANCELED"
    
    // Step 6: Map and validate status
    mappedStatus = SWITCH paymentStatus:
        CASE "COMPLETED": "completed"
        CASE "PENDING": "pending"
        CASE "CANCELED": "canceled"
        DEFAULT: "pending"
    
    // Step 7: Update transaction with latest data
    database.update("transactions", transactionId, {
        payment_status: mappedStatus,
        transaction_id: response.data.external_reference OR transaction.transaction_id,
        channel: response.data.channel.provider OR transaction.channel,
        msisdn: response.data.customer.phone OR transaction.msisdn,
        response_data: JSON.stringify(response),
        completed_at: IF mappedStatus == "completed" THEN response.data.completed_at ELSE NULL
    })
    
    // Step 8: Return status to client
    RETURN {
        status: "success",
        payment_status: mappedStatus,
        data: response.data
    }
END FUNCTION
```

### 3.3 Phone Number Normalization

```
FUNCTION normalizePhoneNumber(phone):
    // Remove all non-digit characters except leading +
    cleaned = removeNonDigits(phone)  // Input: "0712345678" → "0712345678"
    
    // Handle different Tanzania phone formats
    IF cleaned.startsWith("+255"):
        cleaned = cleaned.substring(1)  // "+255712345678" → "255712345678"
    
    IF cleaned.startsWith("255"):
        RETURN cleaned  // Already normalized: "255712345678"
    
    IF cleaned.startsWith("0"):
        // Add country code: "0712345678" → "255712345678"
        RETURN "255" + cleaned.substring(1)
    
    // Invalid format
    RETURN NULL
END FUNCTION
```

### 3.4 Implementation Examples by Framework

#### Node.js / Express Example:

```javascript
const express = require('express');
const axios = require('axios');
const app = express();

app.post('/api/payments/create-order', async (req, res) => {
    try {
        const { page_id, buyer_phone, buyer_name, buyer_email } = req.body;
        
        // Fetch gateway config
        const gateway = await db.query(
            'SELECT * FROM payment_gateways WHERE name = ? AND is_active = ?',
            ['snippe', true]
        );
        
        if (!gateway) {
            return res.status(400).json({ status: 'error', message: 'Gateway not configured' });
        }
        
        // Normalize phone
        const phone = normalizePhoneNumber(buyer_phone);
        
        // Create transaction
        const transaction = await db.query(
            'INSERT INTO transactions (page_id, buyer_phone, ...) VALUES (?, ?, ...)',
            [page_id, phone, ...]
        );
        
        // Call Snippe API
        const response = await axios.post(
            'https://api.snippe.sh/v1/payments',
            {
                payment_type: 'mobile',
                details: { amount: 1000, currency: 'TZS' },
                phone_number: phone,
                customer: { firstname: 'John', lastname: 'Doe', email: buyer_email },
                webhook_url: gateway.webhook_url
            },
            { headers: { 'Authorization': `Bearer ${gateway.api_key}` } }
        );
        
        // Update transaction with response
        await db.query(
            'UPDATE transactions SET reference = ?, response_data = ? WHERE id = ?',
            [response.data.data.reference, JSON.stringify(response.data), transaction.insertId]
        );
        
        res.json({ status: 'success', data: { transaction_id: transaction.insertId, reference: response.data.data.reference } });
    } catch (error) {
        res.status(500).json({ status: 'error', message: error.message });
    }
});
```

#### Python / Django Example:

```python
import requests
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods
from .models import PaymentGateway, Transaction

@require_http_methods(["POST"])
def create_payment_order(request):
    try:
        page_id = request.POST.get('page_id')
        buyer_phone = request.POST.get('buyer_phone')
        buyer_name = request.POST.get('buyer_name')
        buyer_email = request.POST.get('buyer_email')
        
        # Get gateway config
        gateway = PaymentGateway.objects.filter(name='snippe', is_active=True).first()
        if not gateway:
            return JsonResponse({'status': 'error', 'message': 'Gateway not configured'}, status=400)
        
        # Normalize phone
        phone = normalize_phone_number(buyer_phone)
        
        # Create transaction
        transaction = Transaction.objects.create(
            page_id=page_id,
            buyer_phone=phone,
            buyer_name=buyer_name,
            buyer_email=buyer_email,
            amount=1000,
            currency='TZS',
            gateway='snippe',
            payment_status='pending'
        )
        
        # Call Snippe API
        headers = {'Authorization': f'Bearer {gateway.api_key}'}
        payload = {
            'payment_type': 'mobile',
            'details': {'amount': 1000, 'currency': 'TZS'},
            'phone_number': phone,
            'customer': {'firstname': 'John', 'lastname': 'Doe', 'email': buyer_email},
            'webhook_url': gateway.webhook_url
        }
        
        response = requests.post('https://api.snippe.sh/v1/payments', json=payload, headers=headers)
        response_data = response.json()
        
        # Update transaction
        transaction.reference = response_data['data']['reference']
        transaction.response_data = response_data
        transaction.save()
        
        return JsonResponse({'status': 'success', 'data': {'transaction_id': transaction.id, 'reference': response_data['data']['reference']}})
    except Exception as e:
        return JsonResponse({'status': 'error', 'message': str(e)}, status=500)
```

#### PHP / Vanilla Example:

```php
<?php
// Fetch gateway config
$stmt = $pdo->prepare("SELECT * FROM payment_gateways WHERE name = ? AND is_active = ?");
$stmt->execute(['snippe', true]);
$gateway = $stmt->fetch();

if (!$gateway) {
    http_response_code(400);
    echo json_encode(['status' => 'error', 'message' => 'Gateway not configured']);
    exit;
}

// Normalize phone
$phone = normalizePhoneNumber($_POST['buyer_phone']);

// Create transaction
$stmt = $pdo->prepare("
    INSERT INTO transactions (page_id, buyer_phone, buyer_name, buyer_email, amount, currency, gateway, payment_status)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
");
$stmt->execute([$_POST['page_id'], $phone, $_POST['buyer_name'], $_POST['buyer_email'], 1000, 'TZS', 'snippe', 'pending']);
$transactionId = $pdo->lastInsertId();

// Call Snippe API
$payload = [
    'payment_type' => 'mobile',
    'details' => ['amount' => 1000, 'currency' => 'TZS'],
    'phone_number' => $phone,
    'customer' => ['firstname' => 'John', 'lastname' => 'Doe', 'email' => $_POST['buyer_email']],
    'webhook_url' => $gateway['webhook_url']
];

$ch = curl_init('https://api.snippe.sh/v1/payments');
curl_setopt($ch, CURLOPT_HTTPHEADER, ['Authorization: Bearer ' . $gateway['api_key'], 'Content-Type: application/json']);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
$response = curl_exec($ch);

$responseData = json_decode($response, true);

// Update transaction
$stmt = $pdo->prepare("UPDATE transactions SET reference = ?, response_data = ? WHERE id = ?");
$stmt->execute([$responseData['data']['reference'], $response, $transactionId]);

echo json_encode(['status' => 'success', 'data' => ['transaction_id' => $transactionId]]);
?>
```

---

## 4. REST API Endpoints

These endpoints should be implemented in any backend framework and called via HTTP/AJAX from the frontend.

### 4.1 Create Payment Order
**Method:** POST  
**Endpoint:** `/api/payments/create-order`

**Request Payload:**
```json
{
    "page_id": 1,
    "buyer_phone": "0712345678",
    "buyer_name": "John Doe",
    "buyer_email": "john@example.com"
}
```

**Success Response (HTTP 200):**
```json
{
    "status": "success",
    "data": {
        "transaction_id": 5,
        "reference": "pay_abc123def456",
        "amount": 1000,
        "currency": "TZS"
    }
}
```

**Error Response (HTTP 400):**
```json
{
    "status": "error",
    "message": "Snippe gateway is not configured or inactive."
}
```

### 4.2 Check Payment Status
**Method:** POST  
**Endpoint:** `/api/payments/check-status`

**Request Payload:**
```json
{
    "transaction_id": 5
}
```

**Success Response (HTTP 200):**
```json
{
    "status": "success",
    "payment_status": "completed",
    "data": {
        "status": "completed",
        "reference": "pay_abc123def456",
        "amount": 1000,
        "channel": {
            "provider": "Vodacom"
        },
        "customer": {
            "phone": "255712345678"
        },
        "completed_at": "2026-03-05T10:30:00Z"
    }
}
```

**Error Response (HTTP 400):**
```json
{
    "status": "error",
    "message": "Transaction not found"
}
```

---

## 5. Frontend Payment Flow

The frontend can be built with **any JavaScript framework** (React, Vue, Angular, vanilla JS, etc.) or even native mobile (iOS, Android).

### 5.1 Payment Flow Steps

```
1. User initiates payment (clicks button, fills form)
   ↓
2. Frontend calls POST /api/payments/create-order
   ↓
3. Backend creates transaction, calls Snippe API
   ↓
4. User receives USSD prompt on phone
   ↓
5. Frontend starts polling POST /api/payments/check-status
   ↓
6. User completes payment on phone
   ↓
7. Status check returns "completed"
   ↓
8. Frontend redirects to success page
```

### 5.2 JavaScript Implementation (Framework-Independent)

```javascript
// Constants
const API_BASE = '/api';
const POLL_INTERVAL = 4000;  // 4 seconds
const MAX_POLLS = 30;        // 2 minutes max

// Step 1: Create Payment Order
async function createPaymentOrder(pageId, phoneNumber) {
    try {
        const response = await fetch(`${API_BASE}/payments/create-order`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]')?.content || ''
            },
            body: JSON.stringify({
                page_id: pageId,
                buyer_phone: phoneNumber,
                buyer_name: 'Customer',
                buyer_email: 'customer@example.com'
            })
        });

        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message || 'Failed to create order');
        }

        const data = await response.json();
        return data.data.transaction_id;  // Return transaction ID for polling
    } catch (error) {
        console.error('Payment creation error:', error);
        throw error;
    }
}

// Step 2: Poll Payment Status
async function pollPaymentStatus(transactionId) {
    let pollCount = 0;
    
    return new Promise((resolve, reject) => {
        const interval = setInterval(async () => {
            pollCount++;
            
            try {
                const response = await fetch(`${API_BASE}/payments/check-status`, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]')?.content || ''
                    },
                    body: JSON.stringify({ transaction_id: transactionId })
                });

                if (!response.ok) {
                    if (pollCount >= MAX_POLLS) {
                        clearInterval(interval);
                        reject(new Error('Payment verification timeout'));
                    }
                    return;
                }

                const data = await response.json();
                
                // CRITICAL: Standardize status comparison
                const status = (data.payment_status || '').toUpperCase();
                
                if (status === 'COMPLETED') {
                    clearInterval(interval);
                    resolve(data);
                } else if (status === 'CANCELED' || status === 'REJECTED') {
                    clearInterval(interval);
                    reject(new Error('Payment was cancelled or rejected'));
                }
            } catch (error) {
                console.error('Status check error:', error);
            }

            // Stop polling after max attempts
            if (pollCount >= MAX_POLLS) {
                clearInterval(interval);
                reject(new Error('Payment verification timeout'));
            }
        }, POLL_INTERVAL);
    });
}

// Step 3: Complete Payment Handler
async function handlePayment(pageId, phoneNumber) {
    try {
        // Show loading state
        showLoader(true);
        showMessage('Creating payment order...', 'info');
        
        // Create order
        const transactionId = await createPaymentOrder(pageId, phoneNumber);
        showMessage('Check your phone for USSD prompt...', 'info');
        
        // Poll status
        await pollPaymentStatus(transactionId);
        
        // Success
        showMessage('Payment successful! Redirecting...', 'success');
        setTimeout(() => {
            window.location.href = 'https://example.com/success';
        }, 1500);
    } catch (error) {
        showMessage(`Error: ${error.message}`, 'error');
        showLoader(false);
    }
}

// UI Helper Functions
function showLoader(show) {
    const loader = document.getElementById('paymentLoader');
    if (loader) loader.style.display = show ? 'block' : 'none';
}

function showMessage(text, type = 'info') {
    const container = document.getElementById('messageContainer');
    if (!container) return;
    
    const message = document.createElement('div');
    message.className = `message message-${type}`;
    message.textContent = text;
    container.appendChild(message);
    
    setTimeout(() => message.remove(), 4000);
}
```

### 5.3 React Example

```jsx
import { useState } from 'react';

export function PaymentModal({ pageId, onSuccess }) {
    const [loading, setLoading] = useState(false);
    const [phone, setPhone] = useState('');
    const [message, setMessage] = useState('');

    async function handlePayment() {
        setLoading(true);
        try {
            // Create order
            const createRes = await fetch('/api/payments/create-order', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ page_id: pageId, buyer_phone: phone })
            });
            const createData = await createRes.json();
            setMessage('Check your phone for payment prompt...');
            
            // Poll status
            let pollCount = 0;
            const interval = setInterval(async () => {
                pollCount++;
                const statusRes = await fetch('/api/payments/check-status', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ transaction_id: createData.data.transaction_id })
                });
                const statusData = await statusRes.json();
                
                if ((statusData.payment_status || '').toUpperCase() === 'COMPLETED') {
                    clearInterval(interval);
                    setMessage('Payment successful!');
                    onSuccess();
                } else if (pollCount >= 30) {
                    clearInterval(interval);
                    setMessage('Payment timeout');
                    setLoading(false);
                }
            }, 4000);
        } catch (error) {
            setMessage(`Error: ${error.message}`);
            setLoading(false);
        }
    }

    return (
        <div className="payment-modal">
            <input 
                type="tel" 
                value={phone} 
                onChange={(e) => setPhone(e.target.value)} 
                placeholder="Enter phone number"
            />
            <button onClick={handlePayment} disabled={loading}>
                {loading ? 'Processing...' : 'Pay Now'}
            </button>
            {message && <p>{message}</p>}
        </div>
    );
}
```

---

## 6. Snippe API Response Formats

### 6.1 Create Payment Response (from Snippe API)

```json
{
    "status": "success",
    "data": {
        "reference": "pay_abc123def456",
        "status": "pending",
        "amount": {
            "value": 1000,
            "currency": "TZS"
        },
        "phone_number": "255712345678",
        "payment_type": "mobile"
    }
}
```

### 6.2 Status Check Response (from Snippe API)

```json
{
    "status": "success",
    "data": {
        "reference": "pay_abc123def456",
        "status": "completed",
        "external_reference": "trans_xyz789abc",
        "amount": 1000,
        "currency": "TZS",
        "channel": {
            "provider": "Vodacom",
            "name": "Cell Phone"
        },
        "customer": {
            "firstname": "John",
            "lastname": "Doe",
            "email": "john@example.com",
            "phone": "255712345678"
        },
        "completed_at": "2026-03-05T10:30:00Z",
        "created_at": "2026-03-05T10:25:00Z"
    }
}
```

### 6.3 Important Field Mappings

| Snippe Response Field | Your DB Field | Notes |
|----------------------|---------------|-------|
| `data.reference` | `reference` | Use to poll status |
| `data.status` | `payment_status` | ⚠️ Lowercase: "pending", "completed", "canceled" |
| `data.external_reference` | `transaction_id` | Snippe transaction ID |
| `data.channel.provider` | `channel` | e.g., "Vodacom", "Airtel" |
| `data.customer.phone` | `msisdn` | Customer phone number |
| `data.completed_at` | `completed_at` | Parse as timestamp |

---

## 7. Error Handling & Common Issues

### 7.1 Error Scenarios

| Scenario | HTTP Status | Response | Solution |
|----------|------------|----------|----------|
| Gateway not configured | 400 | `{status: "error", message: "...not configured"}` | Create PaymentGateway record in DB |
| Invalid phone number | 400 | `{status: "error", message: "Invalid phone"}` | Validate/normalize phone format |
| Snippe API failure | 400 | `{status: "error", message: "Failed to create order"}` | Check API key, network, Snippe status |
| Transaction not found | 404 | `{status: "error", message: "Transaction not found"}` | Verify transaction_id is correct |
| Network timeout | 500 | `{status: "error", message: "Connection timeout"}` | Retry request, check internet |

### 7.2 Status Comparison Gotcha

⚠️ **CRITICAL:** Snippe returns statuses in **lowercase**, but your code may expect **uppercase**.

```
✅ CORRECT:
if ((statusData.payment_status || '').toUpperCase() === 'COMPLETED') { ... }

❌ WRONG:
if (statusData.payment_status === 'COMPLETED') { ... }  // Never matches "completed"
```

### 7.3 Phone Number Validation

Accepted formats for Tanzania:
- `0712345678` (local format)
- `+255712345678` (international)
- `255712345678` (international without +)

All must normalize to: `255712345678`

```
0712345678  → Remove first 0, prepend 255 → 255712345678
+255712345... → Remove +, keep → 255712345678
255712345678 → Already correct
```

### 7.4 Debugging Checklist

- [ ] Is `SNIPPE_API_KEY` set in environment?
- [ ] Is PaymentGateway record created with `is_active = true`?
- [ ] Is webhook URL accessible from internet?
- [ ] Is phone number normalized to 255XXXXXXXXX format?
- [ ] Is status comparison case-insensitive (.toUpperCase())?
- [ ] Is transaction_id stored and passed to status checks?
- [ ] Are HTTP headers set correctly (Authorization: Bearer)?
- [ ] Is response parsed as JSON safely?

---

## 8. Universal Integration Checklist

This checklist applies to **any framework or language**.

### Database Setup
- [ ] Create `payment_gateways` table
- [ ] Create `transactions` table
- [ ] Insert Snippe gateway record with `is_active = true`
- [ ] Set environment variables (SNIPPE_API_KEY, SNIPPE_WEBHOOK_URL)

### Backend Implementation
- [ ] Create `/api/payments/create-order` endpoint
- [ ] Create `/api/payments/check-status` endpoint
- [ ] Implement phone number normalization function
- [ ] Add error handling for missing/invalid gateway
- [ ] Validate incoming phone numbers
- [ ] Make HTTP requests to Snippe API
- [ ] Store Snippe responses in database
- [ ] Make status comparison case-insensitive

### Frontend Implementation
- [ ] Create payment form/modal UI
- [ ] Call create-order endpoint on form submit
- [ ] Handle response and extract transaction_id
- [ ] Implement polling loop (every 4 seconds)
- [ ] Call check-status endpoint in loop
- [ ] Handle status responses (completed, pending, canceled)
- [ ] Clear interval when status is final
- [ ] Redirect on successful payment
- [ ] Show error messages on failure
- [ ] Add CSRF protection if needed

### Testing
- [ ] Test with valid phone numbers (0712345678, +255712345678, 255712345678)
- [ ] Test payment creation and status polling
- [ ] Test timeout after 2 minutes
- [ ] Test cancellation handling
- [ ] Test network error recovery
- [ ] Verify transaction data stored in database
- [ ] Check response formats match examples

### Deployment
- [ ] Set SNIPPE_API_KEY in production environment
- [ ] Set SNIPPE_WEBHOOK_URL to production URL
- [ ] Verify PaymentGateway is_active = true
- [ ] Test payment flow end-to-end
- [ ] Monitor error logs

---

## 9. Quick Reference: Framework Implementation Guide

| Framework | HTTP Library | DB Library | Setup Time |
|-----------|--------------|-----------|----------|
| **Laravel** | `Http::post()` | Eloquent ORM | 2-3 hours |
| **Node.js/Express** | `axios`, `fetch` | `mysql2`, `mongodb` | 2-3 hours |
| **Python/Django** | `requests` | Django ORM | 2-3 hours |
| **PHP (Vanilla)** | `curl` | MySQLi, PDO | 3-4 hours |
| **Go** | `net/http` | `database/sql` | 3-4 hours |
| **Java/Spring** | `RestTemplate` | Spring Data JPA | 4-5 hours |
| **Ruby on Rails** | `Net::HTTP`, `HTTParty` | ActiveRecord | 2-3 hours |
| **ASP.NET** | `HttpClient` | Entity Framework | 3-4 hours |

---

## 10. Key Takeaways

1. **Snippe uses Bearer Token authentication**
   ```
   Header: Authorization: Bearer {api_key}
   ```

2. **Phone numbers must be normalized to 255XXXXXXXXX format**
   - Required before API calls
   - Stored in database for audit trail

3. **Status comparison must be case-insensitive**
   - Snippe returns: "pending", "completed", "canceled" (lowercase)
   - Always convert to uppercase before comparing

4. **Payment reference is key to status checks**
   - Store `reference` from create response
   - Use `reference` to poll status, not `transaction_id`

5. **Poll status every 4 seconds maximum**
   - Too frequent = wasted API calls
   - Too slow = poor user experience
   - 30 polls × 4 seconds = 2 minutes timeout

6. **Handle three payment statuses**
   ```
   "completed"  → Redirect to success page
   "pending"    → Keep polling
   "canceled"   → Show error, allow retry
   ```

7. **Store complete API responses**
   - Keep JSON in database for audit/debugging
   - Extract needed fields into separate columns
   - Useful for investigating payment issues

8. **Implement webhooks for production (optional)**
   - Snippe can notify your app on completion
   - Reduces polling needs
   - More reliable than polling alone

---

## 11. Comparison: Snippe vs Other Gateways

| Feature | Snippe | SonicPesa | M-Pesa | Stripe |
|---------|--------|-----------|--------|--------|
| **Mobile Money** | ✅ | ✅ | ✅ | ❌ |
| **Tanzania Focus** | ✅ | ✅ | ❌ | ❌ |
| **Auth Type** | Bearer Token | Custom key | OAuth | API keys |
| **Status Check** | GET request | POST request | GET request | GET request |
| **Webhook Support** | ✅ | ✅ | ✅ | ✅ |
| **Status Format** | Lowercase | Uppercase | Varies | Lowercase |

---

## 12. Resources

- **Snippe API Docs:** https://docs.snippe.sh/
- **Snippe Dashboard:** https://dashboard.snippe.sh/
- **Supported Channels:** Vodacom, Airtel, Halotel, Tigo, Zantel
- **Test Mode:** Available in Snippe dashboard
- **Support Email:** support@snippe.sh

---

## 13. Troubleshooting Guide

### Payment Order Won't Create
**Checklist:**
1. Is SNIPPE_API_KEY set correctly?
2. Is PaymentGateway record active?
3. Is phone number normalized correctly?
4. Can you curl the Snippe API directly?
5. Are headers set (Authorization, Content-Type)?

### Status Check Returns Wrong Status
**Checklist:**
1. Is status comparison case-insensitive?
2. Is reference stored correctly?
3. Are you using correct endpoint format?
4. Is Authorization header correct?
5. Try viewing raw API response in development console

### Payment Never Completes
**Checklist:**
1. Is status polling running?
2. Is polling interval 4 seconds?
3. Is max polling set to 30 (2 minutes)?
4. Is response validation working?
5. Check browser console for JavaScript errors
6. Verify phone number format is correct

### Timeout Issues
**Solution:**
```
Current: Poll 30 times × 4 seconds = 2 minutes max
If users need more time: Increase MAX_POLLS constant
If API is slow: Increase POLL_INTERVAL to 5-6 seconds
```

---

## 14. Future Enhancements

- [ ] Implement webhook handler for instant payment notifications
- [ ] Add retry logic for failed API requests
- [ ] Cache gateway config to reduce database hits
- [ ] Add logging/monitoring for payment events
- [ ] Implement multi-gateway selector (Snippe + SonicPesa)
- [ ] Add payment refund functionality
- [ ] Create admin dashboard for transaction reporting
- [ ] Implement email receipts for customers


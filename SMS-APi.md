# BDApps SMS API ডকুমেন্টেশন


## Base URL
```
https://developer.bdapps.com
```

## Authentication
- **Application ID**: আপনার BDApps অ্যাপ্লিকেশন ID
- **Password**: আপনার BDApps অ্যাপ্লিকেশন পাসওয়ার্ড

---

## 1. SMS Send API

### Endpoint
```
POST /sms/send
```

### বিবরণ
নির্দিষ্ট মোবাইল নম্বরে SMS পাঠানোর জন্য ব্যবহৃত হয়।

### Headers
```
Content-Type: application/json
```

### Request Body
```json
{
  "version": "1.0",
  "applicationId": "APP_999999",
  "password": "your_password",
  "message": "আপনার SMS বার্তা এখানে লিখুন",
  "destinationAddresses": [
    "tel:8801812345678",
    "tel:8801987654321"
  ],
  "sourceAddress": "21213",
  "deliveryStatusRequest": "1",
  "encoding": "8",
}
```

### Parameters
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| version | string | Yes | API version (সাধারণত "1.0") |
| applicationId | string | Yes | BDApps অ্যাপ্লিকেশন ID |
| password | string | Yes | BDApps পাসওয়ার্ড |
| message | string | Yes | SMS বার্তা (বাংলা/ইংরেজি) |
| destinationAddresses | array | Yes | গন্তব্য মোবাইল নম্বরের তালিকা |
| sourceAddress | string | Yes | প্রেরক ঠিকানা (সাধারণত "77000") |
| deliveryStatusRequest | string | No | ডেলিভারি স্ট্যাটাস চাইলে "1" |
| encoding | string | No | SMS encoding (0 = Text, 240 = Flash SMS, 8 = Unicode) |
| binaryHeader | string | No | বাইনারি হেডার (hexadecimal) |

### Response

#### সফল Response
```json
{
  "version": "1.0",
  "requestId": "101901031657410007",
  "destinationResponses": [
    {
      "timeStamp": "20190103165801",
      "address": "tel:8801812345678",
      "messageId": "101901031657410007",
      "statusCode": "S1000",
      "statusDetail": "Success"
    }
  ],
  "statusCode": "S1000",
  "statusDetail": "Success."
}
```

#### ব্যর্থ Response
```json
{
  "version": "1.0",
  "requestId": "101901031657410008",
  "destinationResponses": [
    {
      "timeStamp": "20190103165801",
      "address": "tel:8801812345678",
      "messageId": "101901031657410008",
      "statusCode": "E1000",
      "statusDetail": "Insufficient Balance"
    }
  ],
  "statusCode": "E1000",
  "statusDetail": "Failed."
}
```

### Status Codes
| Code | Description | বাংলা বার্তা |
|------|-------------|-------------|
| S1000 | Success | SMS সফলভাবে পাঠানো হয়েছে |
| E1000 | Insufficient Balance | অপর্যাপ্ত ব্যালেন্স |
| E1001 | Invalid Application ID | ভুল অ্যাপ্লিকেশন ID |
| E1002 | Invalid Password | ভুল পাসওয়ার্ড |
| E1003 | Invalid Destination | ভুল গন্তব্য নম্বর |
| E1004 | Message Too Long | বার্তা অতিরিক্ত দীর্ঘ |

---

## 2. SMS Receive (Webhook)

### বিবরণ
যখন ব্যবহারকারী আপনার keyword সহ 21213 নম্বরে SMS পাঠায়, তখন BDApps আপনার configured webhook URL-এ POST request পাঠায়।

### Webhook Format
```
<Your-Keyword> <Space> <Message>
```

**উদাহরণ:**
```
QUIZ What is the capital of Bangladesh?
NEWS Get latest news
HELP Show help menu
```

### Webhook Request Body
```json
{
  "version": "1.0",
  "applicationId": "APP_000029",
  "sourceAddress": "tel:8801812345678",
  "message": "QUIZ What is the capital of Bangladesh?",
  "requestId": "APP_000001",
  "encoding": "0"
}
```

### Webhook Parameters
| Field | Type | Description |
|-------|------|-------------|
| version | string | API version |
| applicationId | string | আপনার অ্যাপ্লিকেশন ID |
| sourceAddress | string | প্রেরকের মোবাইল নম্বর |
| message | string | সম্পূর্ণ SMS বার্তা |
| requestId | string | Unique request ID |
| encoding | string | Message encoding (0 = Plain text) |

### Webhook Response
আপনার webhook endpoint থেকে HTTP 200 status code return করতে হবে।

```json
{
  "status": "received",
  "message": "SMS processed successfully"
}
```

---
## 3. SMS Delivery Report (Webhook)

### বিবরণ
যখন আপনি SMS পাঠানোর সময় `"deliveryStatusRequest": "1"` সেট করেন, তখন SMS delivery status BDApps আপনার configured delivery report webhook URL-এ পাঠায়।

### Delivery Report Webhook Request Body
```json
{
  "destinationAddress": "tel:8801812345678",
  "timeStamp": "20120113082110",
  "requestId": "MSG_000111",
  "deliveryStatus": "DELIVERED"
}
```

### Delivery Report Parameters
| Field | Type | Description |
|-------|------|-------------|
| destinationAddress | string | যে নম্বরে SMS পাঠানো হয়েছিল |
| timeStamp | string | Delivery report এর timestamp (YYYYMMDDHHMMSS) |
| requestId | string | Original SMS request এর ID |
| deliveryStatus | string | SMS delivery status |

### Delivery Status Values
| Status | Description | বাংলা বার্তা |
|--------|-------------|-------------|
| DELIVERED | SMS successfully delivered | SMS সফলভাবে পৌঁছেছে |
| FAILED | SMS delivery failed | SMS পৌঁছাতে ব্যর্থ |
| EXPIRED | SMS expired before delivery | SMS এর মেয়াদ শেষ |
| REJECTED | SMS rejected by operator | অপারেটর SMS প্রত্যাখ্যান করেছে |
| UNKNOWN | Status unknown | অজানা অবস্থা |

### Delivery Report Webhook Response
আপনার delivery report webhook endpoint থেকে HTTP 200 status code return করতে হবে।

```json
{
  "status": "received",
  "message": "Delivery report processed successfully"
}
```

---

## Implementation Examples

### Node.js/Express

#### SMS Send
```javascript
const express = require('express');
const request = require('request');
const app = express();

class BDAppsSMS {
  constructor(appId, password) {
    this.appId = appId;
    this.password = password;
    this.baseUrl = 'https://developer.bdapps.com';
  }

  async sendSMS(phoneNumbers, message, sourceAddress = '77000') {
    // Phone numbers array তৈরি করা
    const destinationAddresses = phoneNumbers.map(phone => {
      let msisdn = phone.toString();
      
      if (msisdn.startsWith('+880')) {
        msisdn = msisdn.slice(4);
      }
      if (msisdn.startsWith('01')) {
        msisdn = '88' + msisdn;
      }
      if (msisdn.length === 10) {
        msisdn = '880' + msisdn;
      }
      
      return `tel:${msisdn}`;
    });

    const requestBody = {
      version: "1.0",
      applicationId: this.appId,
      password: this.password,
      message: message,
      destinationAddresses: destinationAddresses,
      sourceAddress: sourceAddress,
      deliveryStatusRequest: "1",
      encoding: "245" // Unicode for Bangla support
    };

    const options = {
      method: 'POST',
      url: `${this.baseUrl}/sms/send`,
      headers: {
        'Content-Type': 'application/json',
      },
      body: requestBody,
      json: true
    };

    return new Promise((resolve, reject) => {
      request(options, (error, response, body) => {
        if (error) {
          reject(error);
        } else {
          resolve(body);
        }
      });
    });
  }
}

// SMS Send Route
app.post('/send-sms', async (req, res) => {
  try {
    const { phoneNumbers, message } = req.body;
    
    const smsService = new BDAppsSMS('APP_999999', 'your_password');
    const result = await smsService.sendSMS(phoneNumbers, message);
    
    res.json({ success: true, data: result });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// SMS Receive Webhook
app.use(express.json());

app.post('/sms-webhook', (req, res) => {
  const { applicationId, sourceAddress, message, requestId } = req.body;
  
  console.log('Received SMS:', {
    from: sourceAddress,
    message: message,
    requestId: requestId
  });

  // Message parse করা
  const messageParts = message.split(' ');
  const keyword = messageParts[0].toUpperCase();
  const userMessage = messageParts.slice(1).join(' ');

  // Keyword based response
  let responseMessage = '';
  
  switch (keyword) {
    case 'QUIZ':
      responseMessage = handleQuizCommand(userMessage, sourceAddress);
      break;
    case 'NEWS':
      responseMessage = handleNewsCommand(userMessage, sourceAddress);
      break;
    case 'HELP':
      responseMessage = 'Available commands: QUIZ, NEWS, HELP';
      break;
    default:
      responseMessage = 'Invalid command. Send HELP for available commands.';
  }

  // Auto-reply SMS পাঠানো
  if (responseMessage) {
    const smsService = new BDAppsSMS('APP_999999', 'your_password');
    const phoneNumber = sourceAddress.replace('tel:', '');
    
    smsService.sendSMS([phoneNumber], responseMessage)
      .then(result => {
        console.log('Auto-reply sent:', result);
      })
      .catch(error => {
        console.error('Auto-reply failed:', error);
      });
  }

  // Webhook response
  res.status(200).json({
    status: 'received',
    message: 'SMS processed successfully'
  });
});

// SMS Delivery Report Webhook
app.post('/sms-delivery-report', (req, res) => {
  const { destinationAddress, timeStamp, requestId, deliveryStatus } = req.body;
  
  console.log('SMS Delivery Report:', {
    to: destinationAddress,
    status: deliveryStatus,
    timestamp: timeStamp,
    requestId: requestId
  });

  // Database এ delivery status update করা
  updateSMSDeliveryStatus(requestId, deliveryStatus, timeStamp);

  // Delivery status based actions
  switch (deliveryStatus) {
    case 'DELIVERED':
      console.log(`SMS successfully delivered to ${destinationAddress}`);
      // Success tracking logic
      break;
    case 'FAILED':
      console.log(`SMS delivery failed to ${destinationAddress}`);
      // Retry logic or error handling
      handleDeliveryFailure(destinationAddress, requestId);
      break;
    case 'EXPIRED':
      console.log(`SMS expired for ${destinationAddress}`);
      // Handle expired SMS
      break;
    case 'REJECTED':
      console.log(`SMS rejected for ${destinationAddress}`);
      // Handle rejection
      break;
    default:
      console.log(`Unknown delivery status: ${deliveryStatus}`);
  }

  // Webhook response
  res.status(200).json({
    status: 'received',
    message: 'Delivery report processed successfully'
  });
});

function updateSMSDeliveryStatus(requestId, status, timestamp) {
  // Database update logic
  // যেমন: MongoDB, MySQL, PostgreSQL
  console.log(`Updating SMS ${requestId} status to ${status}`);
}

function handleDeliveryFailure(phoneNumber, requestId) {
  // Failed delivery handling logic
  // যেমন: retry mechanism, alternative notification
  console.log(`Handling delivery failure for ${phoneNumber}`);
}

function handleQuizCommand(userMessage, sourceAddress) {
  // Quiz logic এখানে implement করুন
  return `Quiz answer: Dhaka is the capital of Bangladesh. Thanks for playing!`;
}

function handleNewsCommand(userMessage, sourceAddress) {
  // News logic এখানে implement করুন
  return `Latest News: Today's weather is sunny. Visit our website for more news.`;
}

app.listen(3000, () => {
  console.log('SMS API server running on port 3000');
});
```

### PHP

#### SMS Send
```php
<?php
class BDAppsSMS {
    private $appId;
    private $password;
    private $baseUrl = 'https://developer.bdapps.com';
    
    public function __construct($appId, $password) {
        $this->appId = $appId;
        $this->password = $password;
    }
    
    public function sendSMS($phoneNumbers, $message, $sourceAddress = '77000') {
        // Phone numbers format করা
        $destinationAddresses = array_map(function($phone) {
            $msisdn = (string)$phone;
            
            if (strpos($msisdn, '+880') === 0) {
                $msisdn = substr($msisdn, 4);
            }
            if (strpos($msisdn, '01') === 0) {
                $msisdn = '88' . $msisdn;
            }
            if (strlen($msisdn) === 10) {
                $msisdn = '880' . $msisdn;
            }
            
            return 'tel:' . $msisdn;
        }, $phoneNumbers);
        
        $data = [
            'version' => '1.0',
            'applicationId' => $this->appId,
            'password' => $this->password,
            'message' => $message,
            'destinationAddresses' => $destinationAddresses,
            'sourceAddress' => $sourceAddress,
            'deliveryStatusRequest' => '1',
            'encoding' => '8'
        ];
        
        return $this->makeRequest('/sms/send', $data);
    }
    
    private function makeRequest($endpoint, $data) {
        $curl = curl_init();
        
        curl_setopt_array($curl, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json'
            ]
        ]);
        
        $response = curl_exec($curl);
        $httpCode = curl_getinfo($curl, CURLINFO_HTTP_CODE);
        curl_close($curl);
        
        if ($httpCode !== 200) {
            throw new Exception('HTTP Error: ' . $httpCode);
        }
        
        return json_decode($response, true);
    }
}

// SMS Send
$sms = new BDAppsSMS('APP_999999', 'your_password');

try {
    $result = $sms->sendSMS(['01812345678'], 'Hello from BDApps SMS API!');
    echo json_encode($result);
} catch (Exception $e) {
    echo "Error: " . $e->getMessage();
}

// SMS Receive Webhook
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $input = json_decode(file_get_contents('php://input'), true);
    
    $applicationId = $input['applicationId'];
    $sourceAddress = $input['sourceAddress'];
    $message = $input['message'];
    $requestId = $input['requestId'];
    
    // Log the received SMS
    error_log("Received SMS from: " . $sourceAddress . ", Message: " . $message);
    
    // Parse message
    $messageParts = explode(' ', $message);
    $keyword = strtoupper($messageParts[0]);
    $userMessage = implode(' ', array_slice($messageParts, 1));
    
    $responseMessage = '';
    
    switch ($keyword) {
        case 'QUIZ':
            $responseMessage = handleQuizCommand($userMessage, $sourceAddress);
            break;
        case 'NEWS':
            $responseMessage = handleNewsCommand($userMessage, $sourceAddress);
            break;
        case 'HELP':
            $responseMessage = 'Available commands: QUIZ, NEWS, HELP';
            break;
        default:
            $responseMessage = 'Invalid command. Send HELP for available commands.';
    }
    
    // Auto-reply
    if ($responseMessage) {
        $phoneNumber = str_replace('tel:', '', $sourceAddress);
        try {
            $sms->sendSMS([$phoneNumber], $responseMessage);
        } catch (Exception $e) {
            error_log("Auto-reply failed: " . $e->getMessage());
        }
    }
    
    // Webhook response
    http_response_code(200);
    echo json_encode(['status' => 'received', 'message' => 'SMS processed successfully']);
}

function handleQuizCommand($userMessage, $sourceAddress) {
    return "Quiz answer: Dhaka is the capital of Bangladesh. Thanks for playing!";
}

function handleNewsCommand($userMessage, $sourceAddress) {
    return "Latest News: Today's weather is sunny. Visit our website for more news.";
}
?>
```

### Python

#### SMS Send এবং Webhook
```python
from flask import Flask, request, jsonify
import requests
import json

app = Flask(__name__)

class BDAppsSMS:
    def __init__(self, app_id, password):
        self.app_id = app_id
        self.password = password
        self.base_url = 'https://developer.bdapps.com'
    
    def format_phone_numbers(self, phone_numbers):
        formatted = []
        for phone in phone_numbers:
            msisdn = str(phone)
            
            if msisdn.startswith('+880'):
                msisdn = msisdn[4:]
            if msisdn.startswith('01'):
                msisdn = '88' + msisdn
            if len(msisdn) == 10:
                msisdn = '880' + msisdn
                
            formatted.append(f'tel:{msisdn}')
        
        return formatted
    
    def send_sms(self, phone_numbers, message, source_address='77000'):
        destination_addresses = self.format_phone_numbers(phone_numbers)
        
        data = {
            'version': '1.0',
            'applicationId': self.app_id,
            'password': self.password,
            'message': message,
            'destinationAddresses': destination_addresses,
            'sourceAddress': source_address,
            'deliveryStatusRequest': '1',
            'encoding': '8'
        }
        
        response = requests.post(
            f'{self.base_url}/sms/send',
            headers={'Content-Type': 'application/json'},
            json=data
        )
        
        return response.json()

# SMS Send Route
@app.route('/send-sms', methods=['POST'])
def send_sms():
    try:
        data = request.json
        phone_numbers = data['phoneNumbers']
        message = data['message']
        
        sms_service = BDAppsSMS('APP_999999', 'your_password')
        result = sms_service.send_sms(phone_numbers, message)
        
        return jsonify({'success': True, 'data': result})
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

# SMS Receive Webhook
@app.route('/sms-webhook', methods=['POST'])
def sms_webhook():
    try:
        data = request.json
        
        application_id = data['applicationId']
        source_address = data['sourceAddress']
        message = data['message']
        request_id = data['requestId']
        
        print(f"Received SMS from: {source_address}, Message: {message}")
        
        # Parse message
        message_parts = message.split(' ')
        keyword = message_parts[0].upper()
        user_message = ' '.join(message_parts[1:])
        
        response_message = ''
        
        if keyword == 'QUIZ':
            response_message = handle_quiz_command(user_message, source_address)
        elif keyword == 'NEWS':
            response_message = handle_news_command(user_message, source_address)
        elif keyword == 'HELP':
            response_message = 'Available commands: QUIZ, NEWS, HELP'
        else:
            response_message = 'Invalid command. Send HELP for available commands.'
        
        # Auto-reply
        if response_message:
            phone_number = source_address.replace('tel:', '')
            sms_service = BDAppsSMS('APP_999999', 'your_password')
            
            try:
                sms_service.send_sms([phone_number], response_message)
                print(f"Auto-reply sent to: {phone_number}")
            except Exception as e:
                print(f"Auto-reply failed: {str(e)}")
        
        return jsonify({'status': 'received', 'message': 'SMS processed successfully'})
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

def handle_quiz_command(user_message, source_address):
    return "Quiz answer: Dhaka is the capital of Bangladesh. Thanks for playing!"

def handle_news_command(user_message, source_address):
    return "Latest News: Today's weather is sunny. Visit our website for more news."

if __name__ == '__main__':
    app.run(debug=True, port=3000)
```

---

## BDApps Portal Configuration

### 1. Webhook URL Setup
1. BDApps Developer Portal-এ login করুন
2. আপনার Application-এ যান
3. **SMS Settings** → **Webhook URL** সেট করুন:
   - **Incoming SMS Webhook**: `https://yourserver.com/sms-webhook`
   - **Delivery Report Webhook**: `https://yourserver.com/sms-delivery-report`

### 2. Keyword Registration
1. **SMS Settings** → **Keywords** 
2. আপনার keyword register করুন (যেমন: QUIZ, NEWS, HELP)
3. Short code: **21213** (BDApps default)

### 3. Testing
- ব্যবহারকারী পাঠাবে: `QUIZ What is 2+2?` to **21213**
- আপনার webhook এ request আসবে
- Auto-reply পাঠাতে পারবেন

---

## Best Practices

### 1. Message Encoding
- **"0"** - Plain Text (ইংরেজি text-এর জন্য)
- **"240"** - Flash SMS (জরুরি বার্তার জন্য - phone screen-এ instant popup)
- **"8"** - Unicode (বাংলা text-এর জন্য)

### 2. Phone Number Validation
- সব phone number **tel:** prefix সহ পাঠান
- Format: `tel:8801XXXXXXXXX`

### 3. Error Handling
```javascript
// Retry mechanism
async function sendSMSWithRetry(phoneNumbers, message, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const result = await smsService.sendSMS(phoneNumbers, message);
      if (result.statusCode === 'S1000') {
        return result;
      }
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, i)));
    }
  }
}
```

### 4. Webhook Security
```javascript
// IP Whitelist
const allowedIPs = ['bdapps.server.ip.address'];

app.use('/sms-webhook', (req, res, next) => {
  const clientIP = req.ip || req.connection.remoteAddress;
  
  if (!allowedIPs.includes(clientIP)) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  next();
});
```

---

## Troubleshooting

### Common Issues

1. **SMS Not Sending**
   - Check balance in BDApps account
   - Verify phone number format
   - Check application credentials

2. **Webhook Not Receiving**
   - Verify webhook URLs are publicly accessible
   - Check if URLs are configured in BDApps portal
   - Ensure server is returning HTTP 200
   - Test both incoming SMS and delivery report webhooks

3. **Delivery Reports Not Coming**
   - Ensure `"deliveryStatusRequest": "1"` is set in SMS send request
   - Verify delivery report webhook URL is configured
   - Check delivery report webhook endpoint

4. **Bangla Text Not Working**
   - Use encoding: "8" for Unicode
   - Test with simple Bangla text first

5. **Flash SMS Not Showing**
   - Use encoding: "240" for Flash SMS
   - Flash SMS shows as popup on recipient's phone

4. **Keyword Not Working**
   - Verify keyword is registered in BDApps portal
   - Check if keyword is case-sensitive
   - Users must send to **21213**

### Debug Tips
```javascript
// Webhook logging
app.post('/sms-webhook', (req, res) => {
  console.log('Headers:', req.headers);
  console.log('Body:', req.body);
  console.log('IP:', req.ip);
  
  // Your webhook logic here
});
```

---

## Rate Limits এবং Configuration

### Rate Limits
- SMS Send: BDApps কর্তৃক configured TPS (Transactions Per Second) অনুযায়ী
- Webhook: No limit
- TPS limit আপনার application configuration এবং BDApps agreement অনুযায়ী varies করে

### Pricing
- BDApps SMS API সাধারণত subscription-based service
- আলাদা কোনো per-SMS cost নেই
- Pricing আপনার BDApps contract অনুযায়ী

### Monitoring
- BDApps portal-এ SMS delivery reports check করুন
- Delivery report webhook থেকে real-time status পান
- TPS usage monitor করুন
- Webhook logs maintain করুন

### Delivery Tracking Implementation
```javascript
// SMS tracking database schema example
const smsSchema = {
  requestId: String,
  phoneNumber: String,
  message: String,
  sentAt: Date,
  deliveryStatus: String, // PENDING, DELIVERED, FAILED, EXPIRED, REJECTED
  deliveredAt: Date,
  retryCount: Number
};

// SMS send with tracking
async function sendTrackedSMS(phoneNumbers, message) {
  const smsService = new BDAppsSMS('APP_999999', 'your_password');
  const result = await smsService.sendSMS(phoneNumbers, message);
  
  // Save to database for tracking
  for (const response of result.destinationResponses) {
    await saveSMSRecord({
      requestId: response.requestId,
      phoneNumber: response.address,
      message: message,
      sentAt: new Date(),
      deliveryStatus: 'PENDING'
    });
  }
  
  return result;
}
```

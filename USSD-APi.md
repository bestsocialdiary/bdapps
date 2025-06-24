# BDApps USSD API ডকুমেন্টেশন

## সংক্ষিপ্ত বিবরণ
BDApps USSD API বাংলাদেশে USSD (Unstructured Supplementary Service Data) সেবা প্রদানের জন্য ব্যবহৃত হয়। এই API দিয়ে আপনি interactive USSD menu তৈরি করতে পারবেন যা ব্যবহারকারীরা *213 *XXX# ডায়াল করে access করতে পারবে।

## Base URL
```
https://developer.bdapps.com
```

## Authentication
- **Application ID**: আপনার BDApps অ্যাপ্লিকেশন ID
- **Password**: আপনার BDApps অ্যাপ্লিকেশন পাসওয়ার্ড

---

## 1. USSD Receive (Webhook)

### বিবরণ
যখন ব্যবহারকারী USSD code dial করে বা USSD menu তে navigate করে, তখন BDApps আপনার configured webhook URL-এ POST request পাঠায়।

### Webhook Request Body
```json
{
  "version": "1.0",
  "applicationId": "APP_000029",
  "message": "*213*111#",
  "requestId": "1330933229901",
  "sessionId": "1330929317043",
  "ussdOperation": "mt-init",
  "sourceAddress": "tel:8801812345678",
  "vlrAddress": "tel:8801812345678",
  "encoding": "440"
}
```

### Webhook Parameters
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| version | string | Yes | API version (1.0, 2.0 etc.) |
| applicationId | string | Yes | BDApps অ্যাপ্লিকেশন ID |
| message | string | Yes | ব্যবহারকারীর পাঠানো message/input |
| requestId | string | Yes | TAP থেকে unique request identifier |
| sessionId | string | Yes | USSD session এর unique ID |
| ussdOperation | string | Yes | USSD operation type |
| sourceAddress | string | Yes | ব্যবহারকারীর মোবাইল নম্বর |
| vlrAddress | string | No | VLR address |
| encoding | string | Yes | Message encoding (440 = Plain ASCII) |

### USSD Operations
| Operation | Description | ব্যবহার |
|-----------|-------------|---------|
| mo-init | Mobile Originated Initial | ব্যবহারকারী প্রথমবার USSD dial করেছে |
| mo-cont | Mobile Originated Continue | ব্যবহারকারী menu তে input দিয়েছে |
| mt-init | Mobile Terminated Initial | অ্যাপ থেকে USSD session শুরু |
| mt-cont | Mobile Terminated Continue | অ্যাপ থেকে menu continue |
| mt-fin | Mobile Terminated Final | অ্যাপ থেকে session শেষ |

---

## 2. USSD Send API

### Endpoint
```
POST /ussd/send
```

### বিবরণ
USSD session-এ ব্যবহারকারীর কাছে menu বা response পাঠানোর জন্য ব্যবহৃত হয়।

### Headers
```
Content-Type: application/json
```

### Request Body
```json
{
  "version": "1.0",
  "applicationId": "APP_000001",
  "message": "1. Press One\n2. Press two\n3. Press three\n4. Exit\n",
  "password": "your_password",
  "sessionId": "1330929317043",
  "ussdOperation": "mt-cont",
  "destinationAddress": "tel:8801812345678",
  "encoding": "440"
}
```

### Parameters
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| version | string | Yes | API version |
| applicationId | string | Yes | BDApps অ্যাপ্লিকেশন ID |
| message | string | Yes | USSD menu বা response message |
| password | string | Yes | BDApps পাসওয়ার্ড |
| sessionId | string | Yes | Webhook থেকে প্রাপ্ত session ID |
| ussdOperation | string | Yes | USSD operation type |
| destinationAddress | string | Yes | ব্যবহারকারীর মোবাইল নম্বর |
| encoding | string | Yes | Message encoding (440 = ASCII) |

### Response

#### সফল Response
```json
{
  "version": "1.0",
  "requestId": "101901031657410007",
  "timeStamp": "20190103165801",
  "statusCode": "S1000",
  "statusDetail": "Success."
}
```

#### ব্যর্থ Response
```json
{
  "version": "1.0",
  "requestId": "101901031657410008",
  "timeStamp": "20190103165801",
  "statusCode": "E1000",
  "statusDetail": "Invalid Session."
}
```

### Status Codes
- **S1000**: Success - USSD সফলভাবে পাঠানো হয়েছে
- **Other codes**: Failed - USSD পাঠাতে ব্যর্থ

---

## Implementation Examples

### Node.js/Express

```javascript
const express = require('express');
const request = require('request');
const app = express();

class BDAppsUSSD {
  constructor(appId, password) {
    this.appId = appId;
    this.password = password;
    this.baseUrl = 'https://developer.bdapps.com';
  }

  async sendUSSD(sessionId, destinationAddress, message, operation = 'mt-cont') {
    const requestBody = {
      version: "1.0",
      applicationId: this.appId,
      message: message,
      password: this.password,
      sessionId: sessionId,
      ussdOperation: operation,
      destinationAddress: destinationAddress,
      encoding: "440"
    };

    const options = {
      method: 'POST',
      url: `${this.baseUrl}/ussd/send`,
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

// USSD Session Storage (In production, use database)
const ussdSessions = new Map();

// USSD Webhook Handler
app.use(express.json());

app.post('/ussd-webhook', async (req, res) => {
  try {
    const {
      applicationId,
      message,
      requestId,
      sessionId,
      ussdOperation,
      sourceAddress,
      encoding
    } = req.body;

    console.log('USSD Request:', {
      from: sourceAddress,
      message: message,
      operation: ussdOperation,
      sessionId: sessionId
    });

    const ussdService = new BDAppsUSSD('APP_000001', 'your_password');
    let responseMessage = '';
    let responseOperation = 'mt-cont';

    // Session state management
    let sessionData = ussdSessions.get(sessionId) || { 
      level: 0, 
      phoneNumber: sourceAddress,
      data: {} 
    };

    if (ussdOperation === 'mo-init') {
      // Initial USSD request (*213*111#)
      responseMessage = `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                       `1. ব্যালেন্স দেখুন\n` +
                       `2. টাকা পাঠান\n` +
                       `3. রিচার্জ করুন\n` +
                       `4. বিল পেমেন্ট\n` +
                       `5. লোন তথ্য\n` +
                       `6. PIN পরিবর্তন\n` +
                       `0. বের হোন\n\n` +
                       `পছন্দ নির্বাচন করুন:`;
      
      sessionData.level = 1;
      ussdSessions.set(sessionId, sessionData);

    } else if (ussdOperation === 'mo-cont') {
      // User input handling
      const userInput = message.trim();
      
      switch (sessionData.level) {
        case 1: // Main menu
          responseMessage = await handleMainMenu(userInput, sessionData);
          break;
        case 2: // Sub menu
          responseMessage = await handleSubMenu(userInput, sessionData);
          break;
        case 3: // Final level
          responseMessage = await handleFinalLevel(userInput, sessionData);
          responseOperation = 'mt-fin'; // End session
          ussdSessions.delete(sessionId);
          break;
        default:
          responseMessage = 'অবৈধ অনুরোধ। ধন্যবাদ।';
          responseOperation = 'mt-fin';
          ussdSessions.delete(sessionId);
      }
    }

    // Send USSD response
    const result = await ussdService.sendUSSD(
      sessionId,
      sourceAddress,
      responseMessage,
      responseOperation
    );

    console.log('USSD Response sent:', result);

    // Webhook response to BDApps
    res.status(200).json({
      status: 'received',
      message: 'USSD request processed successfully'
    });

  } catch (error) {
    console.error('USSD Error:', error);
    res.status(500).json({
      error: error.message
    });
  }
});

// Main Menu Handler
async function handleMainMenu(userInput, sessionData) {
  let response = '';
  
  switch (userInput) {
    case '1':
      response = `আপনার একাউন্ট ব্যালেন্স:\n\n` +
                `সঞ্চয় একাউন্ট: ৫০,০০০ টাকা\n` +
                `চলতি একাউন্ট: ২৫,০০০ টাকা\n` +
                `মোবাইল একাউন্ট: ৩,৫০০ টাকা\n\n` +
                `সর্বশেষ লেনদেন:\n` +
                `${new Date().toLocaleDateString('bn-BD')}\n` +
                `রিচার্জ: -৫০ টাকা\n\n` +
                `1. বিস্তারিত দেখুন\n` +
                `0. মূল মেনু\n\n` +
                `পছন্দ নির্বাচন করুন:`;
      sessionData.level = 2;
      sessionData.data.service = 'balance';
      break;
      
    case '2':
      response = `টাকা পাঠান:\n\n` +
                `1. মোবাইল নম্বরে\n` +
                `2. ব্যাংক একাউন্টে\n` +
                `3. এজেন্ট নম্বরে\n` +
                `4. কার্ড নম্বরে\n` +
                `0. মূল মেনু\n\n` +
                `পছন্দ নির্বাচন করুন:`;
      sessionData.level = 2;
      sessionData.data.service = 'transfer';
      break;
      
    case '3':
      response = `মোবাইল রিচার্জ:\n\n` +
                `1. নিজের নম্বর\n` +
                `2. অন্য নম্বর\n` +
                `3. প্রিপেইড প্যাকেজ\n` +
                `4. ডেটা প্যাকেজ\n` +
                `0. মূল মেনু\n\n` +
                `পছন্দ নির্বাচন করুন:`;
      sessionData.level = 2;
      sessionData.data.service = 'recharge';
      break;
      
    case '4':
      response = `বিল পেমেন্ট:\n\n` +
                `1. বিদ্যুৎ বিল\n` +
                `2. গ্যাস বিল\n` +
                `3. পানির বিল\n` +
                `4. ইন্টারনেট বিল\n` +
                `5. ইনস্যুরেন্স\n` +
                `0. মূল মেনু\n\n` +
                `পছন্দ নির্বাচন করুন:`;
      sessionData.level = 2;
      sessionData.data.service = 'bills';
      break;
      
    case '5':
      response = `লোন তথ্য:\n\n` +
                `ব্যক্তিগত লোন: ২,০০,০০০ টাকা\n` +
                `বকেয়া: ১,৫০,০০০ টাকা\n` +
                `পরবর্তী কিস্তি: ১৫,০০০ টাকা\n` +
                `নির্ধারিত তারিখ: ১৫/১২/২০২৫\n\n` +
                `1. কিস্তি পরিশোধ\n` +
                `2. বিস্তারিত স্টেটমেন্ট\n` +
                `3. নতুন লোনের আবেদন\n` +
                `0. মূল মেনু\n\n` +
                `পছন্দ নির্বাচন করুন:`;
      sessionData.level = 2;
      sessionData.data.service = 'loan';
      break;
      
    case '6':
      // PIN change requires additional security
      sessionData.data.pinAttempts = 0;
      response = `PIN পরিবর্তন:\n\n` +
                `নিরাপত্তার জন্য আপনার\n` +
                `বর্তমান ৪ ডিজিটের PIN\n` +
                `প্রবেশ করান:\n\n` +
                `(উদাহরণ: 1234)`;
      sessionData.level = 2;
      sessionData.data.service = 'pin_change';
      break;
      
    case '0':
      response = 'ধন্যবাদ ABC Bank মোবাইল ব্যাংকিং ব্যবহারের জন্য!';
      sessionData.level = 3;
      break;
      
    default:
      response = `ভুল নির্বাচন! অনুগ্রহ করে\n` +
                `সঠিক নম্বর দিন।\n\n` +
                `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                `1. ব্যালেন্স দেখুন\n` +
                `2. টাকা পাঠান\n` +
                `3. রিচার্জ করুন\n` +
                `4. বিল পেমেন্ট\n` +
                `5. লোন তথ্য\n` +
                `6. PIN পরিবর্তন\n` +
                `0. বের হোন\n\n` +
                `পছন্দ নির্বাচন করুন:`;
  }
  
  return response;
}

// Sub Menu Handler
async function handleSubMenu(userInput, sessionData) {
  let response = '';
  
  if (sessionData.data.service === 'balance') {
    switch (userInput) {
      case '1':
        response = `একাউন্ট স্টেটমেন্ট:\n\n` +
                  `গত ৭ দিনের লেনদেন:\n` +
                  `২২/১২ - রিচার্জ -৫০\n` +
                  `২১/১২ - গ্রহণ +২,০০০\n` +
                  `২০/১২ - বিল -১,২০০\n` +
                  `১৯/১২ - ATM -৩,০০০\n\n` +
                  `বিস্তারিত SMS পেতে:\n` +
                  `*২২২*১# ডায়াল করুন\n\n` +
                  `0. মূল মেনু`;
        sessionData.level = 3;
        break;
        
      case '0':
        response = `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                  `1. ব্যালেন্স দেখুন\n` +
                  `2. টাকা পাঠান\n` +
                  `3. রিচার্জ করুন\n` +
                  `4. বিল পেমেন্ট\n` +
                  `5. লোন তথ্য\n` +
                  `6. PIN পরিবর্তন\n` +
                  `0. বের হোন\n\n` +
                  `পছন্দ নির্বাচন করুন:`;
        sessionData.level = 1;
        break;
        
      default:
        response = `ভুল নির্বাচন! পুনরায় চেষ্টা করুন।`;
    }
  } else if (sessionData.data.service === 'transfer') {
    switch (userInput) {
      case '1':
        response = `মোবাইল নম্বরে টাকা পাঠান:\n\n` +
                  `গন্তব্য নম্বর লিখুন:\n` +
                  `(যেমন: 01812345678)\n\n` +
                  `অথবা 0 চেপে মূল মেনুতে যান`;
        sessionData.level = 3;
        sessionData.data.transferType = 'mobile';
        break;
        
      case '2':
        response = `ব্যাংক একাউন্টে টাকা পাঠান:\n\n` +
                  `একাউন্ট নম্বর লিখুন:\n` +
                  `(১০-১৬ ডিজিট)\n\n` +
                  `অথবা 0 চেপে মূল মেনুতে যান`;
        sessionData.level = 3;
        sessionData.data.transferType = 'bank';
        break;
        
      case '3':
        response = `এজেন্ট নম্বরে টাকা পাঠান:\n\n` +
                  `এজেন্ট নম্বর লিখুন:\n` +
                  `(যেমন: 01712345678)\n\n` +
                  `অথবা 0 চেপে মূল মেনুতে যান`;
        sessionData.level = 3;
        sessionData.data.transferType = 'agent';
        break;
        
      case '0':
        response = `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                  `1. ব্যালেন্স দেখুন\n` +
                  `2. টাকা পাঠান\n` +
                  `3. রিচার্জ করুন\n` +
                  `4. বিল পেমেন্ট\n` +
                  `5. লোন তথ্য\n` +
                  `6. PIN পরিবর্তন\n` +
                  `0. বের হোন\n\n` +
                  `পছন্দ নির্বাচন করুন:`;
        sessionData.level = 1;
        break;
        
      default:
        response = `ভুল নির্বাচন! পুনরায় চেষ্টা করুন।`;
    }
  } else if (sessionData.data.service === 'recharge') {
    switch (userInput) {
      case '1':
        response = `নিজের নম্বর রিচার্জ:\n\n` +
                  `পরিমাণ নির্বাচন করুন:\n` +
                  `1. ২০ টাকা\n` +
                  `2. ৫০ টাকা\n` +
                  `3. ১০০ টাকা\n` +
                  `4. ২০০ টাকা\n` +
                  `5. অন্য পরিমাণ\n` +
                  `0. মূল মেনু`;
        sessionData.level = 3;
        sessionData.data.rechargeType = 'self';
        break;
        
      case '2':
        response = `অন্য নম্বর রিচার্জ:\n\n` +
                  `মোবাইল নম্বর লিখুন:\n` +
                  `(যেমন: 01812345678)\n\n` +
                  `অথবা 0 চেপে মূল মেনুতে যান`;
        sessionData.level = 3;
        sessionData.data.rechargeType = 'other';
        break;
        
      case '0':
        response = `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                  `1. ব্যালেন্স দেখুন\n` +
                  `2. টাকা পাঠান\n` +
                  `3. রিচার্জ করুন\n` +
                  `4. বিল পেমেন্ট\n` +
                  `5. লোন তথ্য\n` +
                  `6. PIN পরিবর্তন\n` +
                  `0. বের হোন\n\n` +
                  `পছন্দ নির্বাচন করুন:`;
        sessionData.level = 1;
        break;
        
      default:
        response = `ভুল নির্বাচন! পুনরায় চেষ্টা করুন।`;
    }
  } else if (sessionData.data.service === 'bills') {
    switch (userInput) {
      case '1':
        response = `বিদ্যুৎ বিল পেমেন্ট:\n\n` +
                  `কাস্টমার নম্বর লিখুন:\n` +
                  `(১২ ডিজিট)\n\n` +
                  `অথবা 0 চেপে মূল মেনুতে যান`;
        sessionData.level = 3;
        sessionData.data.billType = 'electricity';
        break;
        
      case '2':
        response = `গ্যাস বিল পেমেন্ট:\n\n` +
                  `কাস্টমার নম্বর লিখুন:\n` +
                  `(১০ ডিজিট)\n\n` +
                  `অথবা 0 চেপে মূল মেনুতে যান`;
        sessionData.level = 3;
        sessionData.data.billType = 'gas';
        break;
        
      case '0':
        response = `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                  `1. ব্যালেন্স দেখুন\n` +
                  `2. টাকা পাঠান\n` +
                  `3. রিচার্জ করুন\n` +
                  `4. বিল পেমেন্ট\n` +
                  `5. লোন তথ্য\n` +
                  `6. PIN পরিবর্তন\n` +
                  `0. বের হোন\n\n` +
                  `পছন্দ নির্বাচন করুন:`;
        sessionData.level = 1;
        break;
        
      default:
        response = `ভুল নির্বাচন! পুনরায় চেষ্টা করুন।`;
    }
  } else if (sessionData.data.service === 'loan') {
    switch (userInput) {
      case '1':
        response = `লোন কিস্তি পরিশোধ:\n\n` +
                  `পরিমাণ: ১৫,০০০ টাকা\n` +
                  `নির্ধারিত তারিখ: ১৫/১২/২০২৫\n\n` +
                  `1. এখনই পরিশোধ করুন\n` +
                  `2. রিমাইন্ডার সেট করুন\n` +
                  `0. মূল মেনু\n\n` +
                  `পছন্দ নির্বাচন করুন:`;
        sessionData.level = 3;
        break;
        
      case '2':
        response = `লোন স্টেটমেন্ট:\n\n` +
                  `মূল লোন: ২,০০,০০০ টাকা\n` +
                  `পরিশোধিত: ৫০,০০০ টাকা\n` +
                  `বকেয়া: ১,৫০,০০০ টাকা\n` +
                  `মাসিক কিস্তি: ১৫,০০০ টাকা\n` +
                  `বাকি কিস্তি: ১০টি\n\n` +
                  `বিস্তারিত SMS পেতে:\n` +
                  `*৩৩৩*১# ডায়াল করুন\n\n` +
                  `0. মূল মেনু`;
        sessionData.level = 3;
        break;
        
      case '0':
        response = `স্বাগতম ABC Bank মোবাইল ব্যাংকিং!\n\n` +
                  `1. ব্যালেন্স দেখুন\n` +
                  `2. টাকা পাঠান\n` +
                  `3. রিচার্জ করুন\n` +
                  `4. বিল পেমেন্ট\n` +
                  `5. লোন তথ্য\n` +
                  `6. PIN পরিবর্তন\n` +
                  `0. বের হোন\n\n` +
                  `পছন্দ নির্বাচন করুন:`;
        sessionData.level = 1;
        break;
        
      default:
        response = `ভুল নির্বাচন! পুনরায় চেষ্টা করুন।`;
    }
  } else if (sessionData.data.service === 'pin_change') {
    // PIN validation logic (demo - in production, verify with backend)
    if (userInput === '1234') {
      response = `PIN যাচাই সফল!\n\n` +
                `নতুন ৪ ডিজিটের PIN\n` +
                `প্রবেশ করান:\n\n` +
                `(১২৩৪ থেকে ৯৮৭৬)`;
      sessionData.level = 3;
      sessionData.data.step = 'new_pin';
    } else {
      sessionData.data.pinAttempts = (sessionData.data.pinAttempts || 0) + 1;
      if (sessionData.data.pinAttempts >= 3) {
        response = `PIN ভুল! নিরাপত্তার জন্য\n` +
                  `আপনার একাউন্ট সাময়িক\n` +
                  `বন্ধ করা হয়েছে।\n\n` +
                  `১৬২৬৩ নম্বরে কল করুন।`;
        sessionData.level = 3;
      } else {
        response = `PIN ভুল! আবার চেষ্টা করুন\n` +
                  `(${3 - sessionData.data.pinAttempts} বার বাকি)\n\n` +
                  `বর্তমান ৪ ডিজিটের PIN\n` +
                  `প্রবেশ করান:`;
      }
    }
  }
  
  return response;
}

// Final Level Handler
async function handleFinalLevel(userInput, sessionData) {
  let response = 'ধন্যবাদ ABC Bank মোবাইল ব্যাংকিং ব্যবহারের জন্য!';
  
  // Handle specific final level interactions
  if (sessionData.data.service === 'transfer' && sessionData.data.transferType) {
    // Validate mobile number format for transfer
    if (userInput === '0') {
      response = 'ধন্যবাদ ABC Bank মোবাইল ব্যাংকিং ব্যবহারের জন্য!';
    } else if (sessionData.data.transferType === 'mobile' && /^01[3-9]\d{8}$/.test(userInput)) {
      response = `টাকা পাঠানোর তথ্য:\n\n` +
                `গন্তব্য: ${userInput}\n` +
                `পরিমাণ লিখুন (১০-৫০,০০০):\n\n` +
                `অথবা 0 চেপে বাতিল করুন`;
      // In real implementation, move to amount input stage
    } else {
      response = `ভুল নম্বর! সঠিক মোবাইল\n` +
                `নম্বর লিখুন\n` +
                `(যেমন: 01812345678)\n\n` +
                `অথবা 0 চেপে মূল মেনুতে যান`;
    }
  } else if (sessionData.data.service === 'recharge' && sessionData.data.rechargeType === 'self') {
    switch (userInput) {
      case '1':
        response = `রিচার্জ নিশ্চিতকরণ:\n\n` +
                  `পরিমাণ: ২০ টাকা\n` +
                  `নম্বর: ${sessionData.phoneNumber}\n` +
                  `চার্জ: ২ টাকা\n\n` +
                  `1. নিশ্চিত করুন\n` +
                  `0. বাতিল`;
        break;
      case '2':
        response = `রিচার্জ নিশ্চিতকরণ:\n\n` +
                  `পরিমাণ: ৫০ টাকা\n` +
                  `নম্বর: ${sessionData.phoneNumber}\n` +
                  `চার্জ: ৩ টাকা\n\n` +
                  `1. নিশ্চিত করুন\n` +
                  `0. বাতিল`;
        break;
      default:
        response = 'ধন্যবাদ ABC Bank মোবাইল ব্যাংকিং ব্যবহারের জন্য!';
    }
  } else if (sessionData.data.service === 'pin_change' && sessionData.data.step === 'new_pin') {
    if (/^\d{4}$/.test(userInput) && userInput !== '1234') {
      response = `PIN সফলভাবে পরিবর্তন হয়েছে!\n\n` +
                `নিরাপত্তার জন্য নতুন PIN\n` +
                `গোপন রাখুন।\n\n` +
                `নিশ্চিতকরণ SMS পাঠানো হয়েছে।\n\n` +
                `ধন্যবাদ!`;
    } else {
      response = `PIN গ্রহণযোগ্য নয়!\n\n` +
                `• ৪ ডিজিট হতে হবে\n` +
                `• পুরাতন PIN এর মত হতে পারবে না\n` +
                `• ১১১১, ১২৩৪ ব্যবহার করবেন না\n\n` +
                `নতুন PIN লিখুন:`;
    }
  }
  
  return response;
}

app.listen(3000, () => {
  console.log('USSD API server running on port 3000');
});
```

### PHP

```php
<?php
class BDAppsUSSD {
    private $appId;
    private $password;
    private $baseUrl = 'https://developer.bdapps.com';
    
    public function __construct($appId, $password) {
        $this->appId = $appId;
        $this->password = $password;
    }
    
    public function sendUSSD($sessionId, $destinationAddress, $message, $operation = 'mt-cont') {
        $data = [
            'version' => '1.0',
            'applicationId' => $this->appId,
            'message' => $message,
            'password' => $this->password,
            'sessionId' => $sessionId,
            'ussdOperation' => $operation,
            'destinationAddress' => $destinationAddress,
            'encoding' => '440'
        ];
        
        return $this->makeRequest('/ussd/send', $data);
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

// Session storage (Use database in production)
session_start();

// USSD Webhook Handler
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $input = json_decode(file_get_contents('php://input'), true);
    
    $applicationId = $input['applicationId'];
    $message = $input['message'];
    $requestId = $input['requestId'];
    $sessionId = $input['sessionId'];
    $ussdOperation = $input['ussdOperation'];
    $sourceAddress = $input['sourceAddress'];
    
    error_log("USSD Request: " . json_encode($input));
    
    $ussd = new BDAppsUSSD('APP_000001', 'your_password');
    
    // Session management
    if (!isset($_SESSION['ussd_sessions'])) {
        $_SESSION['ussd_sessions'] = [];
    }
    
    $sessionData = $_SESSION['ussd_sessions'][$sessionId] ?? [
        'level' => 0,
        'phoneNumber' => $sourceAddress,
        'data' => []
    ];
    
    $responseMessage = '';
    $responseOperation = 'mt-cont';
    
    try {
        if ($ussdOperation === 'mo-init') {
            $responseMessage = "স্বাগতম BDApps USSD সেবায়!\n\n" .
                             "1. ব্যালেন্স চেক\n" .
                             "2. প্যাকেজ তথ্য\n" .
                             "3. সাহায্য\n" .
                             "0. বের হোন\n\n" .
                             "আপনার পছন্দ লিখুন:";
            
            $sessionData['level'] = 1;
            $_SESSION['ussd_sessions'][$sessionId] = $sessionData;
            
        } elseif ($ussdOperation === 'mo-cont') {
            $userInput = trim($message);
            
            switch ($sessionData['level']) {
                case 1:
                    $responseMessage = handleMainMenu($userInput, $sessionData);
                    break;
                case 2:
                    $responseMessage = handleSubMenu($userInput, $sessionData);
                    break;
                case 3:
                    $responseMessage = 'ধন্যবাদ BDApps USSD সেবা ব্যবহারের জন্য!';
                    $responseOperation = 'mt-fin';
                    unset($_SESSION['ussd_sessions'][$sessionId]);
                    break;
                default:
                    $responseMessage = 'অবৈধ অনুরোধ। ধন্যবাদ।';
                    $responseOperation = 'mt-fin';
                    unset($_SESSION['ussd_sessions'][$sessionId]);
            }
            
            $_SESSION['ussd_sessions'][$sessionId] = $sessionData;
        }
        
        // Send USSD response
        $result = $ussd->sendUSSD($sessionId, $sourceAddress, $responseMessage, $responseOperation);
        
        error_log("USSD Response sent: " . json_encode($result));
        
        http_response_code(200);
        echo json_encode(['status' => 'received', 'message' => 'USSD processed successfully']);
        
    } catch (Exception $e) {
        error_log("USSD Error: " . $e->getMessage());
        http_response_code(500);
        echo json_encode(['error' => $e->getMessage()]);
    }
}

function handleMainMenu($userInput, &$sessionData) {
    switch ($userInput) {
        case '1':
            $sessionData['level'] = 2;
            $sessionData['data']['service'] = 'balance';
            return "আপনার ব্যালেন্স: ১০০ টাকা\n\n" .
                   "1. রিচার্জ করুন\n" .
                   "2. ব্যালেন্স ট্রান্সফার\n" .
                   "0. পূর্বের মেনু\n\n" .
                   "আপনার পছন্দ:";
                   
        case '2':
            $sessionData['level'] = 2;
            $sessionData['data']['service'] = 'packages';
            return "প্যাকেজ তথ্য:\n\n" .
                   "1. ইন্টারনেট প্যাকেজ\n" .
                   "2. মিনিট প্যাকেজ\n" .
                   "3. SMS প্যাকেজ\n" .
                   "0. পূর্বের মেনু\n\n" .
                   "আপনার পছন্দ:";
                   
        case '3':
            $sessionData['level'] = 3;
            return "সাহায্য:\n\n" .
                   "কাস্টমার কেয়ার: ১২১\n" .
                   "ইমেইল: support@bdapps.com\n" .
                   "ওয়েবসাইট: www.bdapps.com\n\n" .
                   "ধন্যবাদ!";
                   
        case '0':
            $sessionData['level'] = 3;
            return 'ধন্যবাদ BDApps USSD সেবা ব্যবহারের জন্য!';
            
        default:
            return "অবৈধ পছন্দ। অনুগ্রহ করে সঠিক নম্বর লিখুন।\n\n" .
                   "1. ব্যালেন্স চেক\n" .
                   "2. প্যাকেজ তথ্য\n" .
                   "3. সাহায্য\n" .
                   "0. বের হোন\n\n" .
                   "আপনার পছন্দ:";
    }
}

function handleSubMenu($userInput, &$sessionData) {
    if ($sessionData['data']['service'] === 'balance') {
        switch ($userInput) {
            case '1':
                $sessionData['level'] = 3;
                return "রিচার্জ পদ্ধতি:\n\n" .
                       "*১২১*রিচার্জ_কার্ড_নম্বর#\n" .
                       "অথবা\n" .
                       "মোবাইল ব্যাংকিং ব্যবহার করুন\n\n" .
                       "ধন্যবাদ!";
                       
            case '2':
                $sessionData['level'] = 3;
                return "ব্যালেন্স ট্রান্সফার:\n\n" .
                       "*১২৩*গন্তব্য_নম্বর*পরিমাণ#\n" .
                       "চার্জ: ৫ টাকা\n\n" .
                       "ধন্যবাদ!";
                       
            case '0':
                $sessionData['level'] = 1;
                return "স্বাগতম BDApps USSD সেবায়!\n\n" .
                       "1. ব্যালেন্স চেক\n" .
                       "2. প্যাকেজ তথ্য\n" .
                       "3. সাহায্য\n" .
                       "0. বের হোন\n\n" .
                       "আপনার পছন্দ লিখুন:";
                       
            default:
                return "অবৈধ পছন্দ। পুনরায় চেষ্টা করুন।";
        }
    }
    // Handle other services...
    return "ধন্যবাদ!";
}
?>
```

### Python/Flask

```python
from flask import Flask, request, jsonify
import requests
import json

app = Flask(__name__)

class BDAppsUSSD:
    def __init__(self, app_id, password):
        self.app_id = app_id
        self.password = password
        self.base_url = 'https://developer.bdapps.com'
    
    def send_ussd(self, session_id, destination_address, message, operation='mt-cont'):
        data = {
            'version': '1.0',
            'applicationId': self.app_id,
            'message': message,
            'password': self.password,
            'sessionId': session_id,
            'ussdOperation': operation,
            'destinationAddress': destination_address,
            'encoding': '440'
        }
        
        response = requests.post(
            f'{self.base_url}/ussd/send',
            headers={'Content-Type': 'application/json'},
            json=data
        )
        
        return response.json()

# Session storage (Use database in production)
ussd_sessions = {}

@app.route('/ussd-webhook', methods=['POST'])
def ussd_webhook():
    try:
        data = request.json
        
        application_id = data['applicationId']
        message = data['message']
        request_id = data['requestId']
        session_id = data['sessionId']
        ussd_operation = data['ussdOperation']
        source_address = data['sourceAddress']
        
        print(f"USSD Request: {json.dumps(data, indent=2)}")
        
        ussd_service = BDAppsUSSD('APP_000001', 'your_password')
        
        # Session management
        session_data = ussd_sessions.get(session_id, {
            'level': 0,
            'phoneNumber': source_address,
            'data': {}
        })
        
        response_message = ''
        response_operation = 'mt-cont'
        
        if ussd_operation == 'mo-init':
            response_message = ("স্বাগতম BDApps USSD সেবায়!\n\n"
                              "1. ব্যালেন্স চেক\n"
                              "2. প্যাকেজ তথ্য\n"
                              "3. সাহায্য\n"
                              "0. বের হোন\n\n"
                              "আপনার পছন্দ লিখুন:")
            
            session_data['level'] = 1
            ussd_sessions[session_id] = session_data
            
        elif ussd_operation == 'mo-cont':
            user_input = message.strip()
            
            if session_data['level'] == 1:
                response_message = handle_main_menu(user_input, session_data)
            elif session_data['level'] == 2:
                response_message = handle_sub_menu(user_input, session_data)
            elif session_data['level'] == 3:
                response_message = 'ধন্যবাদ BDApps USSD সেবা ব্যবহারের জন্য!'
                response_operation = 'mt-fin'
                if session_id in ussd_sessions:
                    del ussd_sessions[session_id]
            else:
                response_message = 'অবৈধ অনুরোধ। ধন্যবাদ।'
                response_operation = 'mt-fin'
                if session_id in ussd_sessions:
                    del ussd_sessions[session_id]
            
            ussd_sessions[session_id] = session_data
        
        # Send USSD response
        result = ussd_service.send_ussd(
            session_id, 
            source_address, 
            response_message, 
            response_operation
        )
        
        print(f"USSD Response sent: {json.dumps(result, indent=2)}")
        
        return jsonify({
            'status': 'received',
            'message': 'USSD request processed successfully'
        })
        
    except Exception as e:
        print(f"USSD Error: {str(e)}")
        return jsonify({'error': str(e)}), 500

def handle_main_menu(user_input, session_data):
    if user_input == '1':
        session_data['level'] = 2
        session_data['data']['service'] = 'balance'
        return ("আপনার ব্যালেন্স: ১০০ টাকা\n\n"
               "1. রিচার্জ করুন\n"
               "2. ব্যালেন্স ট্রান্সফার\n"
               "0. পূর্বের মেনু\n\n"
               "আপনার পছন্দ:")
    elif user_input == '2':
        session_data['level'] = 2
        session_data['data']['service'] = 'packages'
        return ("প্যাকেজ তথ্য:\n\n"
               "1. ইন্টারনেট প্যাকেজ\n"
               "2. মিনিট প্যাকেজ\n"
               "3. SMS প্যাকেজ\n"
               "0. পূর্বের মেনু\n\n"
               "আপনার পছন্দ:")
    elif user_input == '3':
        session_data['level'] = 3
        return ("সাহায্য:\n\n"
               "কাস্টমার কেয়ার: ১২১\n"
               "ইমেইল: support@bdapps.com\n"
               "ওয়েবসাইট: www.bdapps.com\n\n"
               "ধন্যবাদ!")
    elif user_input == '0':
        session_data['level'] = 3
        return 'ধন্যবাদ BDApps USSD সেবা ব্যবহারের জন্য!'
    else:
        return ("অবৈধ পছন্দ। অনুগ্রহ করে সঠিক নম্বর লিখুন।\n\n"
               "1. ব্যালেন্স চেক\n"
               "2. প্যাকেজ তথ্য\n"
               "3. সাহায্য\n"
               "0. বের হোন\n\n"
               "আপনার পছন্দ:")

def handle_sub_menu(user_input, session_data):
    if session_data['data']['service'] == 'balance':
        if user_input == '1':
            session_data['level'] = 3
            return ("রিচার্জ পদ্ধতি:\n\n"
                   "*১২১*রিচার্জ_কার্ড_নম্বর#\n"
                   "অথবা\n"
                   "মোবাইল ব্যাংকিং ব্যবহার করুন\n\n"
                   "ধন্যবাদ!")
        elif user_input == '2':
            session_data['level'] = 3
            return ("ব্যালেন্স ট্রান্সফার:\n\n"
                   "*১২৩*গন্তব্য_নম্বর*পরিমাণ#\n"
                   "চার্জ: ৫ টাকা\n\n"
                   "ধন্যবাদ!")
        elif user_input == '0':
            session_data['level'] = 1
            return ("স্বাগতম BDApps USSD সেবায়!\n\n"
                   "1. ব্যালেন্স চেক\n"
                   "2. প্যাকেজ তথ্য\n"
                   "3. সাহায্য\n"
                   "0. বের হোন\n\n"
                   "আপনার পছন্দ লিখুন:")
        else:
            return "অবৈধ পছন্দ। পুনরায় চেষ্টা করুন।"
    
    # Handle other services...
    return "ধন্যবাদ!"

if __name__ == '__main__':
    app.run(debug=True, port=3000)
```

---

## BDApps Portal Configuration

### 1. USSD Code Setup
1. BDApps Developer Portal-এ login করুন
2. আপনার Application-এ যান
3. **USSD Settings** → **USSD Code** assign করুন (যেমন: *213*111#)

### 2. Webhook URL Setup
1. **USSD Settings** → **Webhook URL** সেট করুন
2. URL: `https://yourserver.com/ussd-webhook`
3. Ensure URL is publicly accessible

### 3. Testing
- ব্যবহারকারী dial করবে: **\*213\*111#**
- Banking service এর interactive menu দেখতে পাবে
- বিভিন্ন banking operations করতে পারবে

---

## Best Practices

### 1. Session Management
- Production-এ database ব্যবহার করুন session store করার জন্য
- Session timeout handle করুন
- Memory leaks এড়াতে expired sessions clean করুন

### 2. Menu Design
- Banking service এর জন্য realistic menu structure
- Security features (PIN verification)
- Transaction confirmations
- Clear error messages বাংলায়

### 3. Input Validation
```javascript
// Mobile number validation
function validateMobileNumber(number) {
  return /^01[3-9]\d{8}$/.test(number);
}

// PIN validation
function validatePIN(pin) {
  return /^\d{4}$/.test(pin) && pin !== '1234' && pin !== '1111';
}

// Amount validation
function validateAmount(amount) {
  const num = parseInt(amount);
  return num >= 10 && num <= 50000;
}
```

### 3. Error Handling
```javascript
// Session timeout handling
const SESSION_TIMEOUT = 120000; // 2 minutes

setInterval(() => {
  const now = Date.now();
  for (const [sessionId, sessionData] of ussdSessions.entries()) {
    if (now - sessionData.lastActivity > SESSION_TIMEOUT) {
      ussdSessions.delete(sessionId);
    }
  }
}, 30000); // Check every 30 seconds
```

### 4. Logging
```javascript
// Comprehensive logging
app.post('/ussd-webhook', (req, res) => {
  const logData = {
    timestamp: new Date().toISOString(),
    sessionId: req.body.sessionId,
    phoneNumber: req.body.sourceAddress,
    operation: req.body.ussdOperation,
    userInput: req.body.message,
    response: responseMessage
  };
  
  console.log('USSD Log:', JSON.stringify(logData));
  // Save to database for analytics
});
```

---

## Troubleshooting

### Common Issues

1. **USSD Code Not Working**
   - Verify USSD code is registered in BDApps portal
   - Check if code format is correct (e.g., *213*111#)
   - Test with registered mobile numbers

2. **Session Not Maintaining**
   - Check sessionId consistency
   - Verify session storage implementation
   - Handle session timeout properly

3. **Menu Not Displaying Properly**
   - Check message length (max 160 chars)
   - Verify encoding (440 = ASCII)
   - Test menu formatting

4. **Webhook Not Receiving Requests**
   - Verify webhook URL is publicly accessible
   - Check SSL certificate if using HTTPS
   - Ensure server returns HTTP 200

### Debug Tips
```javascript
// Debug middleware
app.use('/ussd-webhook', (req, res, next) => {
  console.log('USSD Headers:', req.headers);
  console.log('USSD Body:', req.body);
  console.log('Session Storage:', ussdSessions);
  next();
});
```

---

## Security Considerations

### 1. Input Validation
```javascript
function validateUSSDInput(input) {
  // Remove special characters
  const sanitized = input.replace(/[^0-9a-zA-Z\s]/g, '');
  
  // Limit length
  return sanitized.substring(0, 160);
}
```

### 2. Rate Limiting
```javascript
const rateLimiter = new Map();

app.post('/ussd-webhook', (req, res, next) => {
  const phoneNumber = req.body.sourceAddress;
  const now = Date.now();
  const requests = rateLimiter.get(phoneNumber) || [];
  
  // Allow max 10 requests per minute
  const recentRequests = requests.filter(time => now - time < 60000);
  
  if (recentRequests.length >= 10) {
    return res.status(429).json({ error: 'Rate limit exceeded' });
  }
  
  recentRequests.push(now);
  rateLimiter.set(phoneNumber, recentRequests);
  
  next();
});
```

### 3. Session Security
- Session IDs encrypt করুন sensitive data store করার সময়
- User data sanitize করুন
- SQL injection prevention করুন database queries-এ

---

## Analytics এবং Monitoring

### 1. USSD Usage Analytics
```javascript
// Analytics data collection
function trackUSSDUsage(sessionData, action) {
  const analytics = {
    sessionId: sessionData.sessionId,
    phoneNumber: sessionData.phoneNumber,
    action: action,
    timestamp: new Date(),
    menuLevel: sessionData.level,
    service: sessionData.data.service
  };
  
  // Save to analytics database
  saveAnalytics(analytics);
}
```

### 2. Performance Monitoring
- Response time track করুন
- Session duration monitor করুন
- Error rates check করুন
- Popular menu options identify করুন

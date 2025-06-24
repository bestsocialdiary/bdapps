# BDApps OTP API ডকুমেন্টেশন

## Base URL
```
https://developer.bdapps.com
```

## Authentication
- **Application ID**: আপনার BDApps অ্যাপ্লিকেশন ID
- **Password**: আপনার BDApps অ্যাপ্লিকেশন API Key

---

## 1. OTP Request API

### Endpoint
```
POST /subscription/otp/request
```

### বিবরণ
নির্দিষ্ট মোবাইল নম্বরে OTP পাঠানোর জন্য রিকোয়েস্ট করে।

### Headers
```
Content-Type: application/json
```

### Request Body
```json
{
  "applicationId": "your_app_id",
  "password": "your_app_password",
  "subscriberId": "tel:8801XXXXXXXXX",
  "applicationHash": "your_app_domain.com",
  "applicationMetaData": {
    "client": "MOBILEAPP",
    "device": "App",
    "os": "android",
    "appCode": "your_app_domain.com"
  }
}
```

### Parameters
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| applicationId | string | Yes | BDApps থেকে প্রাপ্ত অ্যাপ্লিকেশন ID |
| password | string | Yes | BDApps থেকে প্রাপ্ত পাসওয়ার্ড |
| subscriberId | string | Yes | `tel:` প্রিফিক্স সহ 13 ডিজিটের নম্বর |
| applicationHash | string | Yes | আপনার ডোমেইন নাম |
| applicationMetaData | object | Yes | অ্যাপ্লিকেশনের মেটাডেটা |

### Response

#### সফল Response
```json
{
  "statusCode": "S1000",
  "statusDetail": "Success",
  "referenceNo": "REF123456789",
  "subscriberId": "tel:8801XXXXXXXXX"
}
```

#### ব্যর্থ Response
```json
{
  "statusCode": "E1301",
  "statusDetail": "Invalid Operator",
  "subscriberId": "tel:8801XXXXXXXXX"
}
```

### Status Codes
| Code | Description | বাংলা বার্তা |
|------|-------------|-------------|
| S1000 | Success | সফলভাবে OTP পাঠানো হয়েছে |
| E1301 | Invalid Operator | রবি/এয়ারটেল নাম্বার দিন |
| E1342 | Blacklisted Number | নম্বরটি ব্ল্যাকলিস্ট করা |
| E1351 | Already Subscribed | ইতিমধ্যে সাবস্ক্রাইব করা |
| E1852 | OTP Attempt Limit | OTP প্রবেশের সীমা শেষ |
| E1853 | OTP Request Limit | OTP অনুরোধের সীমা শেষ |

---

## 2. OTP Verification API

### Endpoint
```
POST /subscription/otp/verify
```

### বিবরণ
প্রদত্ত OTP কোড যাচাই করে এবং সাবস্ক্রিপশন সক্রিয় করে।

### Headers
```
Content-Type: application/json
```

### Request Body
```json
{
  "applicationId": "your_app_id",
  "password": "your_app_password",
  "referenceNo": "REF123456789",
  "otp": "123456"
}
```

### Parameters
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| applicationId | string | Yes | BDApps অ্যাপ্লিকেশন ID |
| password | string | Yes | BDApps পাসওয়ার্ড |
| referenceNo | string | Yes | OTP রিকোয়েস্ট থেকে প্রাপ্ত রেফারেন্স নম্বর |
| otp | string | Yes | 6 ডিজিটের OTP কোড |

### Response

#### সফল Response
```json
{
  "statusCode": "S1000",
  "statusDetail": "Success",
  "subscriberId": "tel:8801XXXXXXXXX",
  "referenceNo": "REF123456789"
}
```

#### ব্যর্থ Response
```json
{
  "statusCode": "E1850",
  "statusDetail": "Wrong OTP",
  "subscriberId": "tel:8801XXXXXXXXX"
}
```

### Status Codes
| Code | Description | বাংলা বার্তা |
|------|-------------|-------------|
| S1000 | Success | সাবস্ক্রিপশন সফল |
| E1850 | Wrong OTP | ভুল OTP |
| E1854 | Invalid OTP | অবৈধ OTP |
| E1852 | OTP Attempt Limit | OTP প্রচেষ্টার সীমা শেষ |
| E1853 | OTP Request Limit | OTP অনুরোধের সীমা শেষ |
| E1601 | Internal Error | অভ্যন্তরীণ ত্রুটি |

---

## Phone Number Format

### Supported Formats
- `+8801XXXXXXXXX` → `8801XXXXXXXXX`
- `8801XXXXXXXXX` → `8801XXXXXXXXX`
- `01XXXXXXXXX` → `8801XXXXXXXXX`
- `1XXXXXXXXX` → `8801XXXXXXXXX`

### Validation Rules
- চূড়ান্ত ফরম্যাট: 13 ডিজিট
- অবশ্যই `8801` দিয়ে শুরু হতে হবে
- শুধুমাত্র রবি (013, 014, 018) এবং এয়ারটেল (016, 019) সাপোর্ট করে

---

## Implementation Examples

### Node.js/JavaScript
```javascript
const request = require('request');

class BDAppsOTP {
  constructor(appId, appPassword, appDomain) {
    this.appId = appId;
    this.appPassword = appPassword;
    this.appDomain = appDomain;
    this.baseUrl = 'https://developer.bdapps.com';
  }

  // Phone number ফরম্যাট করা
  formatPhoneNumber(phoneNumber) {
    let msisdn = phoneNumber.toString();
    
    if (msisdn.startsWith('+880')) {
      msisdn = msisdn.slice(4);
    }
    if (msisdn.startsWith('01')) {
      msisdn = '88' + msisdn;
    }
    if (msisdn.length === 10) {
      msisdn = '880' + msisdn;
    }
    
    return msisdn;
  }

  // OTP পাঠানো
  async sendOTP(phoneNumber) {
    const msisdn = this.formatPhoneNumber(phoneNumber);
    
    if (msisdn.length !== 13 || !msisdn.startsWith('8801')) {
      throw new Error('Invalid phone number format');
    }

    const options = {
      method: 'POST',
      url: `${this.baseUrl}/subscription/otp/request`,
      headers: {
        'Content-Type': 'application/json',
      },
      body: {
        applicationId: this.appId,
        password: this.appPassword,
        subscriberId: `tel:${msisdn}`,
        applicationHash: this.appDomain,
        applicationMetaData: {
          client: 'MOBILEAPP',
          device: 'App',
          os: 'android',
          appCode: this.appDomain,
        }
      },
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

  // OTP যাচাই করা
  async verifyOTP(referenceNo, otp) {
    const options = {
      method: 'POST',
      url: `${this.baseUrl}/subscription/otp/verify`,
      headers: {
        'Content-Type': 'application/json',
      },
      body: {
        applicationId: this.appId,
        password: this.appPassword,
        referenceNo: referenceNo,
        otp: otp
      },
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

// ব্যবহারের উদাহরণ
const bdapps = new BDAppsOTP('your_app_id', 'your_password', 'yourdomain.com');

// OTP পাঠানো
bdapps.sendOTP('01XXXXXXXXX')
  .then(result => {
    console.log('OTP পাঠানো হয়েছে:', result);
    if (result.statusCode === 'S1000') {
      // OTP যাচাই করা
      return bdapps.verifyOTP(result.referenceNo, '123456');
    }
  })
  .then(result => {
    console.log('OTP যাচাই:', result);
  })
  .catch(error => {
    console.error('ত্রুটি:', error);
  });
```

### PHP
```php
<?php
class BDAppsOTP {
    private $appId;
    private $appPassword;
    private $appDomain;
    private $baseUrl = 'https://developer.bdapps.com';
    
    public function __construct($appId, $appPassword, $appDomain) {
        $this->appId = $appId;
        $this->appPassword = $appPassword;
        $this->appDomain = $appDomain;
    }
    
    public function formatPhoneNumber($phoneNumber) {
        $msisdn = (string)$phoneNumber;
        
        if (strpos($msisdn, '+880') === 0) {
            $msisdn = substr($msisdn, 4);
        }
        if (strpos($msisdn, '01') === 0) {
            $msisdn = '88' . $msisdn;
        }
        if (strlen($msisdn) === 10) {
            $msisdn = '880' . $msisdn;
        }
        
        return $msisdn;
    }
    
    public function sendOTP($phoneNumber) {
        $msisdn = $this->formatPhoneNumber($phoneNumber);
        
        if (strlen($msisdn) !== 13 || strpos($msisdn, '8801') !== 0) {
            throw new Exception('Invalid phone number format');
        }
        
        $data = [
            'applicationId' => $this->appId,
            'password' => $this->appPassword,
            'subscriberId' => 'tel:' . $msisdn,
            'applicationHash' => $this->appDomain,
            'applicationMetaData' => [
                'client' => 'MOBILEAPP',
                'device' => 'App',
                'os' => 'android',
                'appCode' => $this->appDomain
            ]
        ];
        
        return $this->makeRequest('/subscription/otp/request', $data);
    }
    
    public function verifyOTP($referenceNo, $otp) {
        $data = [
            'applicationId' => $this->appId,
            'password' => $this->appPassword,
            'referenceNo' => $referenceNo,
            'otp' => $otp
        ];
        
        return $this->makeRequest('/subscription/otp/verify', $data);
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

// ব্যবহারের উদাহরণ
$bdapps = new BDAppsOTP('your_app_id', 'your_password', 'yourdomain.com');

try {
    // OTP পাঠানো
    $result = $bdapps->sendOTP('01XXXXXXXXX');
    echo "OTP পাঠানো: " . json_encode($result) . "\n";
    
    if ($result['statusCode'] === 'S1000') {
        // OTP যাচাই করা
        $verifyResult = $bdapps->verifyOTP($result['referenceNo'], '123456');
        echo "OTP যাচাই: " . json_encode($verifyResult) . "\n";
    }
} catch (Exception $e) {
    echo "ত্রুটি: " . $e->getMessage() . "\n";
}
?>
```

### Python
```python
import requests
import json

class BDAppsOTP:
    def __init__(self, app_id, app_password, app_domain):
        self.app_id = app_id
        self.app_password = app_password
        self.app_domain = app_domain
        self.base_url = 'https://developer.bdapps.com'
    
    def format_phone_number(self, phone_number):
        msisdn = str(phone_number)
        
        if msisdn.startswith('+880'):
            msisdn = msisdn[4:]
        if msisdn.startswith('01'):
            msisdn = '88' + msisdn
        if len(msisdn) == 10:
            msisdn = '880' + msisdn
            
        return msisdn
    
    def send_otp(self, phone_number):
        msisdn = self.format_phone_number(phone_number)
        
        if len(msisdn) != 13 or not msisdn.startswith('8801'):
            raise ValueError('Invalid phone number format')
        
        data = {
            'applicationId': self.app_id,
            'password': self.app_password,
            'subscriberId': f'tel:{msisdn}',
            'applicationHash': self.app_domain,
            'applicationMetaData': {
                'client': 'MOBILEAPP',
                'device': 'App',
                'os': 'android',
                'appCode': self.app_domain
            }
        }
        
        response = requests.post(
            f'{self.base_url}/subscription/otp/request',
            headers={'Content-Type': 'application/json'},
            json=data
        )
        
        return response.json()
    
    def verify_otp(self, reference_no, otp):
        data = {
            'applicationId': self.app_id,
            'password': self.app_password,
            'referenceNo': reference_no,
            'otp': otp
        }
        
        response = requests.post(
            f'{self.base_url}/subscription/otp/verify',
            headers={'Content-Type': 'application/json'},
            json=data
        )
        
        return response.json()

# ব্যবহারের উদাহরণ
bdapps = BDAppsOTP('your_app_id', 'your_password', 'yourdomain.com')

try:
    # OTP পাঠানো
    result = bdapps.send_otp('01XXXXXXXXX')
    print(f"OTP পাঠানো: {json.dumps(result, indent=2)}")
    
    if result['statusCode'] == 'S1000':
        # OTP যাচাই করা
        verify_result = bdapps.verify_otp(result['referenceNo'], '123456')
        print(f"OTP যাচাই: {json.dumps(verify_result, indent=2)}")
        
except Exception as e:
    print(f"ত্রুটি: {str(e)}")
```

---

## Best Practices

### 1. Error Handling
- সব API কল-এ proper error handling যোগ করুন
- Network timeout সেট করুন
- Retry mechanism implement করুন

### 2. Rate Limiting
- একই নম্বরের জন্য OTP রিকোয়েস্ট ক্যাশ করুন
- Minimum interval maintain করুন requests-এর মধ্যে

### 3. Security
- API credentials secure করে রাখুন
- Environment variables ব্যবহার করুন
- Request/Response log করুন debugging-এর জন্য

### 4. User Experience
- Clear error messages দিন বাংলায়
- Loading states দেখান
- OTP input validation যোগ করুন

---

## Troubleshooting

### Common Issues
1. **E1301 Error**: শুধুমাত্র রবি/এয়ারটেল নাম্বার সাপোর্ট করে
2. **E1853 Error**: OTP request limit exceeded - কিছুক্ষণ অপেক্ষা করুন
3. **Network Timeout**: Internet connection চেক করুন
4. **Wrong OTP**: 6 ডিজিট সংখ্যা নিশ্চিত করুন

### Testing
- সবসময় valid রবি/এয়ারটেল নম্বর দিয়ে টেস্ট করুন
- Development environment-এ proper logging রাখুন
- Production-এ rate limiting implement করুন

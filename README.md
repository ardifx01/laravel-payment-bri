
# Laravel BRI Payments (QRIS MPM Dynamic + BRIVA Virtual Account)

**Namespace:** `ESolution\BriPayments`  
**License:** Apache-2.0

This package provides a pragmatic Laravel integration for **Bank BRI** payments:

- **QRIS MPM Dynamic (SNAP)**: B2B access token (RSA-SHA256), generate QR, inquiry, and a ready-to-wire webhook controller.
- **BRIVA (Virtual Account, Non‑SNAP)**: OAuth token, create/get/update/delete VA, get payment status, reports, and a push-notification verifier.

It is designed to be production-friendly, with clear signatures, headers, and timestamps handled for you.

> ⚠️ Always verify the latest BRI docs for any contract changes. This package follows BRI’s official docs for the endpoints and headers.

---

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Environment Variables](#environment-variables)
- [QRIS (SNAP) Usage](#qris-snap-usage)
- [BRIVA (VA, Non-SNAP) Usage](#briva-va-non-snap-usage)
- [Webhooks](#webhooks)
  - [QRIS Notification](#qris-notification)
  - [BRIVA Push Notification](#briva-push-notification)
- [Examples](#examples)
- [Testing Tips (Postman/cURL)](#testing-tips-postmancurl)
- [Production Notes & Security](#production-notes--security)
- [Versioning](#versioning)
- [Support & Hiring](#support--hiring)
- [Donations](#donations)
  - [Easiest Ways to Receive Donations](#easiest-ways-to-receive-donations)
- [License](#license)

---

## Requirements

- PHP **8.2+**
- Laravel **10.x or 11.x**
- `ext-openssl`
- Network access to BRI sandbox/production endpoints

---

## Installation

```bash
composer require elgibor-solution/laravel-payment-bri

php artisan vendor:publish   --provider="ESolution\BriPayments\BriPaymentsServiceProvider"   --tag=bri-payments-config
```

This publishes a config file at `config/bri.php`.

---

## Configuration

`config/bri.php`

```php
return [
    'base_url' => env('BRI_BASE_URL', 'https://sandbox.partner.api.bri.co.id'),

    'common' => [
        'client_id'     => env('BRI_CLIENT_ID'),
        'client_secret' => env('BRI_CLIENT_SECRET'),
    ],

    'qris' => [
        'partner_id'  => env('BRI_SNAP_PARTNER_ID'),
        'channel_id'  => env('BRI_SNAP_CHANNEL_ID', '95221'),
        'merchant_id' => env('BRI_SNAP_MERCHANT_ID'),
        'terminal_id' => env('BRI_SNAP_TERMINAL_ID'),
        'private_key_path' => env('BRI_SNAP_PRIVATE_KEY_PATH'),
        'public_key_path'  => env('BRI_SNAP_PUBLIC_KEY_PATH'), // optional: verify incoming signatures
        'timeout'     => env('BRI_SNAP_TIMEOUT', 30),

        'notify' => [
            'enabled'    => true,
            'uri'        => 'bri/qris/notify',
            'middleware' => ['api'],
        ],
    ],

    'briva' => [
        'institution_code' => env('BRI_BRIVA_INSTITUTION_CODE'),
        'briva_no'         => env('BRI_BRIVA_NUMBER'),
        'timeout'          => env('BRI_BRIVA_TIMEOUT', 30),

        'notify' => [
            'enabled'    => true,
            'uri'        => 'bri/briva/notify',
            'middleware' => ['api'],
        ],
    ],
];
```

---

## Environment Variables

```env
# Common
BRI_BASE_URL=https://sandbox.partner.api.bri.co.id
BRI_CLIENT_ID=your_client_id
BRI_CLIENT_SECRET=your_client_secret

# SNAP (QRIS)
BRI_SNAP_PARTNER_ID=your_partner_id
BRI_SNAP_CHANNEL_ID=95221
BRI_SNAP_MERCHANT_ID=00007100010926
BRI_SNAP_TERMINAL_ID=213141251124
BRI_SNAP_PRIVATE_KEY_PATH=storage/keys/bri-snap-private.pem
BRI_SNAP_PUBLIC_KEY_PATH=storage/keys/bri-snap-public.pem
BRI_SNAP_TIMEOUT=30

# BRIVA (Non-SNAP)
BRI_BRIVA_INSTITUTION_CODE=J104408
BRI_BRIVA_NUMBER=77777
BRI_BRIVA_TIMEOUT=30
```

> Keep keys outside your repository; do not commit secrets. Consider using a secret manager or encrypted storage.

---

## QRIS (SNAP) Usage

**Namespaces**  
`ESolution\BriPayments\Qris\QrisClient`  
`ESolution\BriPayments\Support\SnapSignature` (internal)

### 1) Get Token (B2B, RSA-SHA256)
```php
use ESolution\BriPayments\Qris\QrisClient;

/** @var QrisClient $qris */
$qris = app(QrisClient::class);
$token = $qris->getToken();
```

### 2) Generate Dynamic QR
```php
$qr = $qris->generateQr(
    partnerReferenceNo: 'INV-2025-0001',
    amount: '10000.00',
    currency: 'IDR'
);
// $qr->qrContent, $qr->referenceNo
```

### 3) Inquiry Payment
```php
$status = $qris->inquiryPayment(
    originalReferenceNo: $qr->referenceNo,
    terminalId: config('bri.qris.terminal_id')
);
// Use latestTransactionStatus (e.g., "00" for success) according to BRI docs
```

> SNAP business requests sign with **HMAC-SHA512** over canonical string; headers include `Authorization: Bearer <accessToken>`, `X-TIMESTAMP`, `X-SIGNATURE`, `X-PARTNER-ID`, `CHANNEL-ID`, and `X-EXTERNAL-ID` (package also sends `X-EXTRENAL-ID` for compatibility).

---

## BRIVA (VA, Non-SNAP) Usage

**Namespace**  
`ESolution\BriPayments\Briva\BrivaClient`

### 1) Get OAuth Token (Non‑SNAP)
```php
use ESolution\BriPayments\Briva\BrivaClient;

/** @var BrivaClient $briva */
$briva = app(BrivaClient::class);
$token = $briva->getToken();
```

### 2) Create a VA
```php
$res = $briva->createVa([
  'institutionCode' => config('bri.briva.institution_code'),
  'brivaNo'         => config('bri.briva.briva_no'),
  'custCode'        => 'CUST001',
  'nama'            => 'John Doe',
  'amount'          => '25000',                     // string, numbers only
  'keterangan'      => 'Invoice INV-001',
  'expiredDate'     => '2025-12-31 23:59:59',      // YYYY-MM-DD HH:mm:ss
]);
```

### 3) Get VA / Get Payment Status
```php
$va = $briva->getVa(config('bri.briva.institution_code'), config('bri.briva.briva_no'), 'CUST001');

$status = $briva->getStatus(config('bri.briva.institution_code'), config('bri.briva.briva_no'), 'CUST001');
```

### 4) Update VA or Mark as Paid/Unpaid
```php
$update = $briva->updateVa([/* ...payload per BRI docs... */]);

$mark   = $briva->updateStatus(
  config('bri.briva.institution_code'),
  config('bri.briva.briva_no'),
  'CUST001',
  'Y' // Y = paid, N = unpaid (check docs)
);
```

### 5) Delete VA
```php
$del = $briva->deleteVa(
  config('bri.briva.institution_code'),
  config('bri.briva.briva_no'),
  'CUST001'
);
```

### 6) Reports
```php
$report = $briva->getReport(
  config('bri.briva.institution_code'),
  config('bri.briva.briva_no'),
  '2025-01-01', '2025-01-31'
);

$reportTime = $briva->getReportTime(
  config('bri.briva.institution_code'),
  config('bri.briva.briva_no'),
  '2025-01-01', '00:00',
  '2025-01-02', '23:59'
);
```

> Non‑SNAP requests use **`BRI-Signature`** = `base64(HMAC_SHA256(payload, client_secret))` and **`BRI-Timestamp`** (UTC). For `DELETE /v1/briva`, BRI expects `Content-Type: text/plain` with `institutionCode=&brivaNo=&custCode=` body — the package handles this and signs **exactly** that body.

---

## Webhooks

### QRIS Notification
- Route: `POST /bri/qris/notify` (can be changed in `config/bri.php`)
- Controller: `ESolution\BriPayments\Http\Controllers\QrisNotificationController@handle`
- Event: `ESolution\BriPayments\Events\QrisPaymentNotified`

**Usage:**
```php
use ESolution\BriPayments\Events\QrisPaymentNotified;
use Illuminate\Support\Facades\Event;

Event::listen(QrisPaymentNotified::class, function ($event) {
    // $event->payload, $event->headers, $event->validSignature
    // Update your order/payment here
});
```

### BRIVA Push Notification
- Route: `POST /bri/briva/notify` (can be changed in `config/bri.php`)
- Controller: `ESolution\BriPayments\Http\Controllers\BrivaNotificationController@handle`
- Event: `ESolution\BriPayments\Events\BrivaPaymentNotified`

The controller verifies `BRI-Signature`. BRI’s push docs often sign using the **absolute partner URL** as `path`. Some integrations sign only the **path** (without scheme/host). The package verifies both for compatibility.

**Usage:**
```php
use ESolution\BriPayments\Events\BrivaPaymentNotified;
use Illuminate\Support\Facades\Event;

Event::listen(BrivaPaymentNotified::class, function ($event) {
    // $event->payload, $event->headers, $event->validSignature
    // Update your VA/payment here
});
```

---

## Examples

### Minimal Controller to Create QR (SNAP)
```php
use ESolution\BriPayments\Qris\QrisClient;

class PaymentController
{
    public function createQris(QrisClient $qris)
    {
        $qr = $qris->generateQr('INV-2025-0001', '10000.00');
        return response()->json([
            'referenceNo' => $qr->referenceNo ?? null,
            'qrContent'   => $qr->qrContent ?? null,
        ]);
    }
}
```

### Minimal Controller to Create BRIVA
```php
use ESolution\BriPayments\Briva\BrivaClient;

class VaController
{
    public function create(BrivaClient $briva)
    {
        $res = $briva->createVa([
            'institutionCode' => config('bri.briva.institution_code'),
            'brivaNo'         => config('bri.briva.briva_no'),
            'custCode'        => 'CUST-ABC-001',
            'nama'            => 'Jane Doe',
            'amount'          => '150000',
            'keterangan'      => 'Order #12345',
            'expiredDate'     => now()->addDay()->format('Y-m-d H:i:s'),
        ]);

        return response()->json($res);
    }
}
```

---

## Testing Tips (Postman/cURL)

- Use **BRI Sandbox** credentials.
- For SNAP token, ensure your **RSA private key** matches the **client key**.
- For BRIVA, verify `BRI-Timestamp` is **UTC (ISO 8601)**.
- To test BRIVA DELETE, send `text/plain` body exactly as BRI expects.
- For webhooks, expose your local URL using `ngrok` and register it with BRI.

---

## Production Notes & Security

- Rotate secrets regularly and do not log sensitive headers or bodies.
- Constrain webhook routes with IP allowlist or additional shared secrets if possible.
- Implement idempotency for webhook processing to avoid double credits.
- Add retries/backoff for intermittent gateway errors.
- Monitor and alert on non-`00` statuses and signature mismatches.

---

## Versioning

Semantic versioning (MAJOR.MINOR.PATCH). Breaking changes will bump MAJOR.

---

## Support & Hiring

Need professional help or want to move faster? **Hire the E-Solution / Elgibor team** for integration, audits, or custom features.  
📧 **info@elgibor-solution.com**

---

## Donations

If this package saves you time, consider supporting development ❤️

- **Ko‑fi**: [![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/U7U21L7D5J)

---

## License

Apache-2.0

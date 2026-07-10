# PRD-21: In-App Purchase (IAP) Validation

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** In-App Purchase Validation  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification  
> **Reference:** [Reference Engine IAP Validation Documentation](https://heroiclabs.com/docs/game-server/concepts/iap-validation/)

---

## 1. Purpose & Scope

Define the requirements for a built-in server-side in-app purchase (IAP) validation system. This system validates purchase receipts from Apple App Store, Google Play, Huawei AppGallery, and Facebook Instant Games, preventing fraud through replay attack detection and receipt verification. This is a built-in feature of the game server, not a custom implementation.

---

Refer to [BRD-21](../BRD/21_iap_validation.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

IAP validation requests are submitted via REST or gRPC. The server interacts with external platform APIs (Apple, Google, Huawei, Facebook) asynchronously to verify receipt validity, and registers incoming server-to-server webhook endpoints to receive platform notifications (such as refunds or subscription state updates).

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Validate Apple Purchase Receipt
* **Endpoint**: `POST /v2/iap/purchase/apple`
* **Request Body**:
  ```json
  {
    "receipt": "MIIS9gIBBaNDAW...[base64 receipt data]...",
    "persist": true
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "validated_purchases": [
      {
        "product_id": "gems_pack_100",
        "transaction_id": "1000000812345678",
        "purchase_time": "2026-07-01T22:00:00Z",
        "store": "AppleAppStore",
        "seen_before": false,
        "environment": "Sandbox"
      }
    ]
  }
  ```

#### 2. Validate Google Purchase Token
* **Endpoint**: `POST /v2/iap/purchase/google`
* **Request Body**:
  ```json
  {
    "product_id": "gems_pack_100",
    "purchase_token": "gplay-purchase-token-string",
    "persist": true
  }
  ```
* **Response Body (200 OK)**: Same structure as Apple validation.

#### 3. Validate Huawei Purchase
* **Endpoint**: `POST /v2/iap/purchase/huawei`
* **Request Body**:
  ```json
  {
    "purchase_data": "{\"orderId\":\"...\",\"productId\":\"gems_pack_100\",...}",
    "signature": "huawei-signature-string",
    "persist": true
  }
  ```
* **Response Body (200 OK)**: Same structure as Apple validation.

#### 4. Validate Facebook Instant Games Purchase
* **Endpoint**: `POST /v2/iap/purchase/facebook_instant`
* **Request Body**:
  ```json
  {
    "signed_request": "fb-signed-request-payload-string",
    "persist": true
  }
  ```
* **Response Body (200 OK)**: Same structure as Apple validation.

#### 5. Platform Server-to-Server Webhook Notifications (Unauthenticated REST)
* **Endpoints**:
  * `POST /v2/iap/notification/apple`
  * `POST /v2/iap/notification/google`
* **Response Status**: `200 OK` (no body)

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

service IAPService {
  rpc ValidatePurchaseApple(ValidatePurchaseAppleRequest) returns (ValidatePurchaseResponse);
  rpc ValidatePurchaseGoogle(ValidatePurchaseGoogleRequest) returns (ValidatePurchaseResponse);
  rpc ValidatePurchaseHuawei(ValidatePurchaseHuaweiRequest) returns (ValidatePurchaseResponse);
  rpc ValidatePurchaseFacebookInstant(ValidatePurchaseFBInstantRequest) returns (ValidatePurchaseResponse);
  rpc ListPurchases(ListPurchasesRequest) returns (PurchaseList);
}

message ValidatePurchaseAppleRequest {
  string receipt = 1;
  bool persist = 2;
}

message ValidatePurchaseGoogleRequest {
  string product_id = 1;
  string purchase_token = 2;
  bool persist = 3;
}

message ValidatePurchaseHuaweiRequest {
  string purchase_data = 1;
  string signature = 2;
  bool persist = 3;
}

message ValidatePurchaseFBInstantRequest {
  string signed_request = 1;
  bool persist = 2;
}

message ValidatedPurchase {
  string product_id = 1;
  string transaction_id = 2;
  string purchase_time = 3;
  string store = 4;
  bool seen_before = 5;
  string environment = 6;
}

message ValidatePurchaseResponse {
  repeated ValidatedPurchase validated_purchases = 1;
}

message ListPurchasesRequest {
  string user_id = 1;
  int32 limit = 2;
  string cursor = 3;
}

message PurchaseList {
  repeated ValidatedPurchase purchases = 1;
  string next_cursor = 2;
}
```

---

### Server Runtime API

```lua
nk.purchase_validate_apple(user_id, receipt, persist)
nk.purchase_validate_google(user_id, product_id, purchase_token, persist)
nk.purchase_validate_huawei(user_id, purchase_data, signature, persist)
nk.purchase_validate_facebook_instant(user_id, signed_request, persist)
nk.purchases_list(user_id, limit, cursor)
```

---

## 4. Product Configurations

```yaml
iap:
  apple:
    shared_password: "apple-shared-secret-key"
  google:
    service_account: "/path/to/gcp-service-account.json"
    package_name: "com.example.game"
  huawei:
    public_key: "huawei-iap-public-key"
  facebook_instant:
    app_secret: "fb-app-secret-key"
```

---

## 5. Linked Documents
- [BRD-21](../BRD/21_iap_validation.md) (Business Requirements Document)
- [TDD-21](../TDD/21_iap_validation.md) (Technical Design Document)

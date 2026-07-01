# PRD-01: User Authentication

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** User Authentication  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a comprehensive user authentication system that supports multiple identity providers, session management, and account linking. This system serves as the foundational identity layer for all other game server features.

---

Refer to [BRD-01](../BRD/01_user_authentication.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

All authentication requests run over HTTP/1.1 or HTTP/2 (REST) and HTTP/2 (gRPC) using TLS 1.3 by default. 

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>` (required for account details, updating, linking, and unlinking endpoints).

### Common Error Schema
All error responses from REST endpoints will return the following JSON structure with a HTTP status code corresponding to the error:
```json
{
  "error": "UNAUTHORIZED",
  "message": "Invalid credentials provided.",
  "code": 3
}
```

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Authenticate with Email
* **Endpoint**: `POST /v2/account/authenticate/email`
* **Query Parameters**: 
  * `create` (boolean, optional, default: `true`): Automatically create account if it does not exist.
  * `username` (string, optional): Desired username if account is created.
* **Request Body**:
  ```json
  {
    "email": "player@example.com",
    "password": "supersecretpassword"
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "def502008cb0a...",
    "created": true
  }
  ```

#### 2. Authenticate with Device ID
* **Endpoint**: `POST /v2/account/authenticate/device`
* **Query Parameters**:
  * `create` (boolean, optional, default: `true`)
  * `username` (string, optional)
* **Request Body**:
  ```json
  {
    "id": "device-uuid-or-hardware-id-string"
  }
  ```
* **Response Body (200 OK)**: Same as Email Authentication.

#### 3. Authenticate with Platform/Social Provider (Apple, Google, Facebook, Steam, Game Center, Custom)
* **Endpoints**: 
  * `POST /v2/account/authenticate/apple` (Apple ID Token)
  * `POST /v2/account/authenticate/google` (Google OAuth / ID Token)
  * `POST /v2/account/authenticate/facebook` (Facebook Access Token)
  * `POST /v2/account/authenticate/steam` (Steam Session Ticket)
  * `POST /v2/account/authenticate/gamecenter` (Game Center credentials)
  * `POST /v2/account/authenticate/custom` (Custom Developer ID)
* **Request Body (Custom/Apple/Google/Facebook/Steam/GameCenter)**:
  ```json
  {
    "id": "opaque-provider-token-or-identifier",
    "vars": {
      "any_extra_variable": "value"
    }
  }
  ```
* **Response Body (200 OK)**: Same as Email Authentication.

#### 4. Session Refresh
* **Endpoint**: `POST /v2/account/session/refresh`
* **Request Body**:
  ```json
  {
    "token": "expired-or-expiring-jwt-session-token",
    "refresh_token": "valid-refresh-token-string"
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "token": "new-jwt-session-token-string",
    "refresh_token": "new-refresh-token-string"
  }
  ```

#### 5. Session Logout
* **Endpoint**: `POST /v2/account/session/logout`
* **Request Body**:
  ```json
  {
    "token": "jwt-session-token-string",
    "refresh_token": "refresh-token-string"
  }
  ```
* **Response Status**: `200 OK` (no body)

#### 6. Get Account Details
* **Endpoint**: `GET /v2/account`
* **Response Body (200 OK)**:
  ```json
  {
    "account": {
      "user": {
        "id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player",
        "display_name": "Super Player",
        "avatar_url": "https://example.com/avatar.png",
        "lang_tag": "en-US",
        "location": "US",
        "timezone": "America/New_York",
        "metadata": {
          "level": 15,
          "has_completed_tutorial": true
        },
        "create_time": "2026-07-01T22:25:53Z",
        "update_time": "2026-07-01T22:30:00Z"
      },
      "devices": [
        { "id": "device-uuid-or-hardware-id-string" }
      ],
      "email": "player@example.com",
      "custom_id": "custom-auth-id-if-linked"
    }
  }
  ```

#### 7. Update Account Details
* **Endpoint**: `PUT /v2/account`
* **Request Body**:
  ```json
  {
    "display_name": "Updated Name",
    "avatar_url": "https://example.com/new_avatar.png",
    "lang_tag": "fr-FR",
    "location": "FR",
    "timezone": "Europe/Paris",
    "metadata": {
      "level": 15,
      "has_completed_tutorial": true,
      "theme": "dark"
    }
  }
  ```
* **Response Status**: `200 OK` (no body)

#### 8. Link Identity Provider
* **Endpoints**: 
  * `POST /v2/account/link/email`
  * `POST /v2/account/link/device`
  * `POST /v2/account/link/apple`
  * `POST /v2/account/link/google`
  * `POST /v2/account/link/facebook`
  * `POST /v2/account/link/steam`
  * `POST /v2/account/link/custom`
* **Request Body**: Same body as the corresponding `/v2/account/authenticate/*` endpoint.
* **Response Status**: `200 OK` (no body)

#### 9. Unlink Identity Provider
* **Endpoints**:
  * `POST /v2/account/unlink/email`
  * `POST /v2/account/unlink/device`
  * `POST /v2/account/unlink/apple`
  * `POST /v2/account/unlink/google`
  * `POST /v2/account/unlink/facebook`
  * `POST /v2/account/unlink/steam`
  * `POST /v2/account/unlink/custom`
* **Request Body**:
  ```json
  {
    "id": "identifier-or-token-to-unlink"
  }
  ```
* **Response Status**: `200 OK` (no body)

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";

service AuthenticationService {
  rpc AuthenticateEmail(AuthenticateEmailRequest) returns (Session);
  rpc AuthenticateDevice(AuthenticateDeviceRequest) returns (Session);
  rpc AuthenticateApple(AuthenticateAppleRequest) returns (Session);
  rpc AuthenticateGoogle(AuthenticateGoogleRequest) returns (Session);
  rpc AuthenticateFacebook(AuthenticateFacebookRequest) returns (Session);
  rpc AuthenticateSteam(AuthenticateSteamRequest) returns (Session);
  rpc AuthenticateGameCenter(AuthenticateGameCenterRequest) returns (Session);
  rpc AuthenticateCustom(AuthenticateCustomRequest) returns (Session);
  
  rpc SessionRefresh(SessionRefreshRequest) returns (Session);
  rpc SessionLogout(SessionLogoutRequest) returns (google.protobuf.Empty);
  
  rpc GetAccount(google.protobuf.Empty) returns (Account);
  rpc UpdateAccount(UpdateAccountRequest) returns (google.protobuf.Empty);
  
  rpc LinkEmail(AuthenticateEmailRequest) returns (google.protobuf.Empty);
  rpc LinkDevice(AuthenticateDeviceRequest) returns (google.protobuf.Empty);
  rpc LinkApple(AuthenticateAppleRequest) returns (google.protobuf.Empty);
  rpc LinkGoogle(AuthenticateGoogleRequest) returns (google.protobuf.Empty);
  rpc LinkFacebook(AuthenticateFacebookRequest) returns (google.protobuf.Empty);
  rpc LinkSteam(AuthenticateSteamRequest) returns (google.protobuf.Empty);
  rpc LinkCustom(AuthenticateCustomRequest) returns (google.protobuf.Empty);
  
  rpc UnlinkEmail(UnlinkRequest) returns (google.protobuf.Empty);
  rpc UnlinkDevice(UnlinkRequest) returns (google.protobuf.Empty);
  rpc UnlinkApple(UnlinkRequest) returns (google.protobuf.Empty);
  rpc UnlinkGoogle(UnlinkRequest) returns (google.protobuf.Empty);
  rpc UnlinkFacebook(UnlinkRequest) returns (google.protobuf.Empty);
  rpc UnlinkSteam(UnlinkRequest) returns (google.protobuf.Empty);
  rpc UnlinkCustom(UnlinkRequest) returns (google.protobuf.Empty);
}

message AuthenticateEmailRequest {
  string email = 1;
  string password = 2;
  bool create = 3;
  string username = 4;
}

message AuthenticateDeviceRequest {
  string id = 1;
  bool create = 2;
  string username = 3;
}

message AuthenticateAppleRequest {
  string token = 1;
  bool create = 2;
  string username = 3;
}

message AuthenticateGoogleRequest {
  string token = 1;
  bool create = 2;
  string username = 3;
}

message AuthenticateFacebookRequest {
  string token = 1;
  bool create = 2;
  string username = 3;
}

message AuthenticateSteamRequest {
  bytes ticket = 1;
  bool create = 2;
  string username = 3;
}

message AuthenticateGameCenterRequest {
  string player_id = 1;
  string bundle_id = 2;
  int64 timestamp = 3;
  string salt = 4;
  string signature = 5;
  string public_key_url = 6;
  bool create = 7;
  string username = 8;
}

message AuthenticateCustomRequest {
  string id = 1;
  bool create = 2;
  string username = 3;
}

message SessionRefreshRequest {
  string token = 1;
  string refresh_token = 2;
}

message SessionLogoutRequest {
  string token = 1;
  string refresh_token = 2;
}

message Session {
  string token = 1;
  string refresh_token = 2;
  bool created = 3;
}

message User {
  string id = 1;
  string username = 2;
  string display_name = 3;
  string avatar_url = 4;
  string lang_tag = 5;
  string location = 6;
  string timezone = 7;
  string metadata = 8; // JSON string format
  google.protobuf.Timestamp create_time = 9;
  google.protobuf.Timestamp update_time = 10;
}

message Device {
  string id = 1;
}

message Account {
  User user = 1;
  repeated Device devices = 2;
  string email = 3;
  string custom_id = 4;
  google.protobuf.Timestamp verify_time = 5;
}

message UpdateAccountRequest {
  string display_name = 1;
  string avatar_url = 2;
  string lang_tag = 3;
  string location = 4;
  string timezone = 5;
  string metadata = 6; // JSON string format
}

message UnlinkRequest {
  string id = 1;
}
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `session.token_expiry_sec` | integer | `3600` | Expiry duration for JWT session tokens (1 hour). |
| `session.refresh_token_expiry_sec` | integer | `604800` | Expiry duration for refresh tokens (7 days). |
| `session.encryption_key` | string | `""` | Key used for signing and verifying session tokens. |
| `session.password_hash_cost` | integer | `12` | Bcrypt/Argon2 work factor/cost parameter for password hashing. |
| `session.max_linked_devices` | integer | `5` | Maximum number of device IDs that can be linked to a single account. |
| `session.rate_limit_auth_attempts` | integer | `5` | Failed authentication attempts permitted within lockout window. |
| `session.lockout_duration_sec` | integer | `900` | Lockout duration after exceeding auth attempts (15 minutes). |

---

## Linked Documents
- [BRD-01](../BRD/01_user_authentication.md) (Business Requirements Document)
- [TDD-01](../TDD/01_user_authentication.md) (Technical Design Document)

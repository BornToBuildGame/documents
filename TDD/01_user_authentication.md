# TDD-01: User Authentication

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Technical Design:** User Authentication  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Technical Architecture

---

## 1. Purpose & Scope

Define the technical design for a comprehensive user authentication system that supports multiple identity providers, session management, and account linking. This system serves as the foundational identity layer for all other game server features.

---

Refer to [BRD-01](../BRD/01_user_authentication.md) for the business requirements and [PRD-01](../PRD/01_user_authentication.md) for the API surface.

---

## 2. Architecture & Design Flow

Authentication flow verifies client identity using password hashes or third-party OAuth provider endpoints. Once identity is verified, the server generates a cryptographically signed JWT token.

### Authentication Sequence Flow
```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Server as Server Core
    participant DB as PostgreSQL DB
    participant OAuth as Social Provider (e.g. Apple)

    alt Password Authentication
        Client->>Server: POST /v2/account/authenticate/email {email, password}
        Server->>DB: Query user record by email
        DB-->>Server: Return hashed_password (bcrypt)
        Server->>Server: Verify bcrypt hash match
    else Social Identity Authentication
        Client->>Server: POST /v2/account/authenticate/apple {id_token}
        Server->>OAuth: HTTP validation of token
        OAuth-->>Server: Return verified identity & Apple User ID
        Server->>DB: Query user by apple_id
        DB-->>Server: User record (or create if new)
    end
    
    Server->>Server: Generate JWT Session Token (expiry 1hr)
    Server->>DB: Insert new session token record in user_sessions
    DB-->>Server: Confirmation
    Server-->>Client: Return Session Object {token, refresh_token}
```

---

## 3. Database Schema & Data Models

### Raw DDL Schemas

```sql
-- Users Table
CREATE TABLE IF NOT EXISTS users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username      VARCHAR(64) UNIQUE NOT NULL,
    display_name  VARCHAR(128),
    avatar_url    VARCHAR(512),
    lang_tag      VARCHAR(18) DEFAULT 'en-US',
    location      VARCHAR(64),
    timezone      VARCHAR(64),
    metadata      JSONB DEFAULT '{}'::jsonb NOT NULL,
    wallet        JSONB DEFAULT '{}'::jsonb NOT NULL,
    email         VARCHAR(255),
    password      BYTEA, -- Bcrypt hash
    apple_id      VARCHAR(128),
    google_id     VARCHAR(128),
    facebook_id   VARCHAR(128),
    steam_id      VARCHAR(128),
    gamecenter_id VARCHAR(128),
    custom_id     VARCHAR(128),
    edge_count    INT DEFAULT 0 NOT NULL,
    create_time   TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    update_time   TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    verify_time   TIMESTAMPTZ,
    disable_time  TIMESTAMPTZ
);

-- Sessions Table
CREATE TABLE IF NOT EXISTS user_sessions (
    token         VARCHAR(256) PRIMARY KEY,
    user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token VARCHAR(256) NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    expires_at    TIMESTAMPTZ NOT NULL,
    revoked       BOOLEAN DEFAULT FALSE NOT NULL
);
```

### Table Indexes

```sql
-- Social / Provider Lookup Indexes
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email ON users(email) WHERE email IS NOT NULL;
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_apple_id ON users(apple_id) WHERE apple_id IS NOT NULL;
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_google_id ON users(google_id) WHERE google_id IS NOT NULL;
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_facebook_id ON users(facebook_id) WHERE facebook_id IS NOT NULL;
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_steam_id ON users(steam_id) WHERE steam_id IS NOT NULL;
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_custom_id ON users(custom_id) WHERE custom_id IS NOT NULL;

-- Session expiration query index
CREATE INDEX IF NOT EXISTS idx_user_sessions_expiry ON user_sessions(expires_at) WHERE revoked = FALSE;
```

---

## 4. Algorithmic Logic & Execution Flow

### Token Issuance Logic
1. Receive login request containing credentials or third-party token verification.
2. Calculate/validate credentials:
   - For password logins, match using a bcrypt algorithm with work factor `12`.
   - For refresh token updates, check token database records to confirm it has not been flagged as `revoked`.
3. Build JWT payload claims containing:
   - `sub` (Subject): Player's `user_id`.
   - `usn` (Username): Player's unique username.
   - `iat` (Issued At) and `exp` (Expiry Time): Calculated dynamically.
4. Sign token using HMAC-SHA256 (HS256) or RS256 with the configured secret key.

### Go Session Generation Example

```go
package main

import (
	"errors"
	"time"
	"github.com/golang-jwt/jwt/v4"
	"golang.org/x/crypto/bcrypt"
	"github.com/google/uuid"
)

type Claims struct {
	UserID   string `json:"sub"`
	Username string `json:"usn"`
	jwt.RegisteredClaims
}

func VerifyAndGenerateSession(hashedPassword []byte, suppliedPass string, userID string, username string, secretKey []byte, expirySec int64) (string, string, error) {
	// 1. Password verification
	err := bcrypt.CompareHashAndPassword(hashedPassword, []byte(suppliedPass))
	if err != nil {
		return "", "", errors.New("UNAUTHORIZED")
	}

	// 2. Generate JWT Payload
	claims := Claims{
		UserID:   userID,
		Username: username,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Duration(expirySec) * time.Second)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}

	// 3. Sign JWT
	tokenObj := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	token, err := tokenObj.SignedString(secretKey)
	if err != nil {
		return "", "", err
	}

	// 4. Generate random cryptographically secure Refresh Token
	refreshToken := uuid.New().String()

	return token, refreshToken, nil
}
```

---

## 5. Linked Documents
- [BRD-01](../BRD/01_user_authentication.md) (Business Requirements Document)
- [PRD-01](../PRD/01_user_authentication.md) (Product Requirements Document)

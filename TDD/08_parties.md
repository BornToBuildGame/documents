# TDD-08: Parties

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Technical Design:** Parties  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Technical Architecture

---

## 1. Purpose & Scope

Define the requirements for a party system enabling players to form temporary groups for co-op play, party matchmaking, party chat, and coordinated game activities. Parties are session-based, ephemeral groups that exist while members are online.

---

Refer to [BRD-08](../BRD/08_parties.md) for the business requirements and [PRD-08](../PRD/08_parties.md) for the API surface.

---

## 2. Architecture & Design Flow

Parties are ephemeral, session-based structures kept entirely in-memory. They use real-time WebSocket messaging and Stream registrations (see [PRD-17: Presence System](./17_presence_system.md)) to track member join/leave states and broadcast updates.

### Party Invite & Join Sequence Flow
```mermaid
sequenceDiagram
    autonumber
    actor A as Leader (Player A)
    participant Server as Server Core
    actor B as Guest (Player B)

    A->>Server: (WS) party_create {open: false, max_size: 4}
    Server->>Server: Initialize in-memory Party object
    Server-->>A: (WS) party_created {party_id}

    A->>Server: (WS) party_invite_send {user_id: B}
    Server->>Server: Register invite in-memory Party list
    Server-->>B: (WS) party_invite {party_id, sender: A}

    B->>Server: (WS) party_join {party_id}
    Server->>Server: Add B to Party members map & clear B's invite
    Server-->>A: (WS) party_presence_event {joins: [B]}
    Server-->>B: (WS) party_presence_event {joins: [A, B]}
```

---

## 3. Database Schema & Data Models

Because parties are strictly session-based and do not persist across server restarts, they do not write to PostgreSQL. They are represented in the server's shared in-memory memory map.

### Go Party State Structures

```go
package main

import "time"

type PartyMember struct {
	UserID     string                 `json:"user_id"`
	Username   string                 `json:"username"`
	SessionID  string                 `json:"session_id"`
	JoinedAt   time.Time              `json:"joined_at"`
	Properties map[string]interface{} `json:"properties"` // Ready status, selected character/class
}

type PartySession struct {
	PartyID     string                  `json:"party_id"`
	LeaderID    string                  `json:"leader_id"`
	Open        bool                    `json:"open"`
	MaxSize     int                     `json:"max_size"`
	Members     map[string]*PartyMember `json:"members"`     // Key: userId
	Invitations map[string]time.Time    `json:"invitations"` // Key: invited userId
	Metadata    map[string]interface{}  `json:"metadata"`    // Party settings, game match status
	Node        string                  `json:"node"`        // Server node IP hosting the party stream
}
```

---

## 4. Algorithmic Logic & Execution Flow

### Leader Promotion & Transfer Logic
When the current leader $L$ disconnects:
1. The server checks the `reconnect_grace_sec` window.
2. If $L$ fails to reconnect and is evicted from the session registry:
   - If the party has other active members:
     - Sort members by `joinedAt` timestamp ascending (longest serving member).
     - Select the first candidate $M$ and promote: `leaderId = M.userId`.
     - Broadcast a `party_leader` event to all remaining members.
   - If no other members are present:
     - Terminate the party session, remove all invitations, and delete the in-memory `PartySession` object.

### Go Party Member Status Update Handler

```go
package main

import "errors"

func UpdateMemberStatus(party *PartySession, userID string, newProperties map[string]interface{}) (*PartySession, error) {
	member, exists := party.Members[userID]
	if !exists {
		return nil, errors.New("MEMBER_NOT_FOUND")
	}

	// Merge new properties (e.g. ready status toggle)
	for k, v := range newProperties {
		member.Properties[k] = v
	}

	return party, nil
}
```

---

## 5. Linked Documents
- [BRD-08](../BRD/08_parties.md) (Business Requirements Document)
- [PRD-08](../PRD/08_parties.md) (Product Requirements Document)

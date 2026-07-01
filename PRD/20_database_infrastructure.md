# PRD-20: Database & Infrastructure

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Database & Infrastructure  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for the database layer, server infrastructure, deployment, scaling, monitoring, and operational tooling. PostgreSQL serves as the primary database, with additional infrastructure supporting high availability, observability, and enterprise-grade operations.

---

Refer to [BRD-20](../BRD/20_database_infrastructure.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Database and infrastructure interactions occur internally. Monitoring servers query performance statistics via standard HTTP scrape operations, and container orchestrators evaluate pod lifecycle states via REST check endpoints.

---

## 3. API Surface Detail

### REST Observability Endpoints

#### 1. Prometheus Metrics Scrape
Exposes real-time server runtime metrics, memory allocation, request latencies, and database pool utilization.
* **Endpoint**: `GET /metrics`
* **Port**: Configurable, default is `7350` (or dedicated metrics port)
* **Authentication**: None (typically restricted at the network firewall layer)
* **Response Content-Type**: `text/plain; version=0.0.4`
* **Response Body**: Standard Prometheus format.
  ```text
  # HELP ultimate_server_connections Active socket connections count.
  # TYPE ultimate_server_connections gauge
  ultimate_server_connections 2420
  # HELP ultimate_db_pool_open_connections Open database connections.
  # TYPE ultimate_db_pool_open_connections gauge
  ultimate_db_pool_open_connections 12
  ```

#### 2. Server Readiness Probe
Evaluates database connectivity and module loading completion. Returns HTTP 503 if the database is unreachable or migrations are running.
* **Endpoint**: `GET /ready`
* **Authentication**: None
* **Response Body (200 OK - Database Connected)**:
  ```json
  {
    "status": "ready"
  }
  ```
* **Response Body (503 Service Unavailable - Database Offline)**:
  ```json
  {
    "status": "not_ready",
    "error": "database unreachable"
  }
  ```

---

## 4. Product Configurations

```yaml
database:
  address: "127.0.0.1:5432"
  username: "root"
  password: ""
  name: "ultimate_game_server"
  max_open_conns: 100
  max_idle_conns: 100
  conn_max_lifetime_sec: 3600
  conn_max_idle_sec: 600
  migration: true # Run schema updates automatically at startup

metrics:
  prometheus_port: 7350 # Port to expose /metrics
```

---

## Linked Documents
- [BRD-20](../BRD/20_database_infrastructure.md) (Business Requirements Document)
- [TDD-20](../TDD/20_database_infrastructure.md) (Technical Design Document)

# HSBC: Recharge Failure Recovery System

## Overview

Developed an architecture using the Circuit Breaker pattern to ensure system resilience during external service outages.

### Technologies Used

- AWS
- Kubernetes
- Microservices
- PostgreSQL
- Cassandra
- Kafka (asynchronous communication)

## Problem Statement

External services were unavailable, preventing users from completing recharge transactions successfully.

## Solution

To address this issue, the system was designed to:

- Display bill details to users using previously cached data.
- Accept recharge requests even when external services were unavailable.
- Process pending recharge requests automatically once the external services recovered.

## Key Responsibilities

- Gathered and analyzed client requirements to define project scope and solutions.
- Collaborated in resource planning and allocation based on project needs.
- Designed and developed Spring Boot microservices for recharge failure recovery workflows.
- Implemented circuit breaker-based resiliency mechanisms to handle external service downtime.
- Participated in the complete SDLC, including development, testing, production deployment, and post-production support.

## Three-Stage Recovery Process

### 1. Data Caching

Bill details from external services were cached in Cassandra during normal operations.

### 2. Failure Handling

During service outages:

- Bill details were retrieved from Cassandra.
- Recharge requests were stored in PostgreSQL in a deferred state.

### 3. Recovery and Processing

- Service health was validated by sending test requests.
- Once the external service became available, deferred recharge requests were processed automatically.

## Circuit Breaker States

| State | Description |
|---------|-------------|
| Closed | Normal operations with direct external service calls. |
| Open | External service failures occurred. Cached data from Cassandra was used, and recharge requests were stored in a deferred state. |
| Half-Open | Service recovery was validated by sending a limited number of requests to external services. |

## Architecture Diagram

![Recharge Failure Recovery Architecture](images/recharge-failure-recovery-architecture.png)

## Circuit-Breaker

![Circuit-Breaker](images/Circuit-Breaker.jpg)
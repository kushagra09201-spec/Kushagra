# HSBC: Recharge Failure Recovery System

## Overview

Designed and developed a resilient recharge failure recovery architecture using the Circuit Breaker pattern to ensure service continuity during external service outages.

## Technologies Used

* Java
* Spring Boot
* Resilience4j
* Kafka
* Cassandra
* PostgreSQL
* AWS (EC2, EBS, RDS, Auto Scaling Groups, Security Groups, CloudWatch)
* API Gateway
* Jenkins

## Problem Statement

External service outages caused recharge failures. Failed transactions meant the platform could not debit customer funds, resulting in the loss of BBPS incentives and user platform charges, while also impacting customer satisfaction due to unsuccessful recharges. These repeated retries also overloaded downstream systems

When the external provider became unavailable or experienced high latency:

- Customers were unable to complete recharge requests.
- The platform could not collect service charges or BBPS incentives.
- Repeated retries increased the load on downstream services.
- Customer experience deteriorated due to failed transactions.
- Operations teams were required to manually monitor and reprocess pending requests after the provider recovered.

The objective was to design a resilient architecture capable of continuing to accept recharge requests, preserving transaction integrity, and automatically recovering pending transactions once the external provider became available again.

## Business Impact

The solution improved business continuity, customer trust, and operational efficiency during external service outages.

* Increased revenue by accepting recharge requests even when external BBPS providers were unavailable, ensuring platform charges and partner incentives were not lost due to temporary outages.
* Improved customer trust and retention by allowing users to complete recharge requests during provider downtime, reducing the likelihood of customers switching to competing platforms.
* Reduced infrastructure and operational costs by preventing repeated retry attempts and cascading failures through the Circuit Breaker pattern, protecting downstream services from unnecessary load.
* Eliminated manual intervention by automatically recovering and processing pending recharge transactions once external services became available.

## Solution

To address this issue, the system was designed to:

* Display bill details to users using previously cached data.
* Accept recharge requests even when external services were unavailable.
* Update bill status in Cassandra for 2 payment scenarios: partial payment and full payment.
* Cache bill data with a **TTL** for each product, allowing customers to reuse previously fetched bill information.
* For example Cylinder, Bill details were cached in Cassandra for **35 days**. User-specific data was stored per customer, while the cylinder amount was determined by **city and gas provider**, enabling price updates to be reflected for all applicable users without calling the external service.
Configured Kafka with acknowledgements (acks=all) and a replication factor of 3 to ensure recharge events were not lost in the event of broker failures.
* Introduced a dedicated **Deferred Processing Service** responsible for periodically scanning deferred recharge requests from PostgreSQL.
* During external service outages, newly created deferred requests were marked with **awaiting = true**, indicating that they should not yet be reprocessed.
* Once the Health Check service confirmed that the external provider had recovered, all eligible deferred requests were updated to **awaiting = false**.
* The Deferred Processing Service picked only records with **PENDING** status and **awaiting = false**, processing them in configurable batches (typically **50 requests per batch**) to prevent sudden traffic spikes.
* Successfully processed requests were forwarded to the Recharge Processing Service for normal recharge execution and final transaction status updates.

## Key Responsibilities

* Gathered and analyzed client requirements to define project scope and solutions.
* Collaborated in resource planning and allocation based on project needs.
* Designed and developed Spring Boot microservices for recharge failure recovery workflows.
* Implemented circuit breaker-based resiliency mechanisms to handle external service downtime.
* Participated in the complete SDLC, including development, testing, production deployment, and post-production support.

## Architecture Decisions

### Why Kafka?

During production, the recharge service was handling around **100 TPS**. Initially, the average response time was around **600 ms**, which started increasing as traffic grew. The recharge API was performing **bill validation, Rule-Engine updates, and othrr downstream bill processing synchronously** within the user request. This became the primary bottleneck, requiring frequent horizontal scaling of the recharge service just to maintain response times. 
After analyzing the production metrics, I suggested introducing **Kafka** to decouple the synchronous processing from the user request flow.

This approach provided several benefits:

* **Reduced API response time** from approximately **600 ms to around 300 ms**, improving the customer experience. Previously, the Recharge API executed **product lookup, rule evaluation, bill validation, Rule-Engine updates, and other downstream bill processing** synchronously before responding to the user. These operations contributed nearly **300 ms** of the total response time. By persisting the request and publishing an event to Kafka, the API was able to return immediately, while downstream processing was handled asynchronously by Kafka consumers, reducing the average response time to around **300 ms** without compromising reliability.

* **Enabled the transition from a monolithic processing model to an event-driven microservice architecture.** Earlier, the Recharge API was responsible for executing the complete business workflow, making it tightly coupled with bill processing, Rule-Engine updates, and external service integration. Kafka decoupled these responsibilities into independent microservices communicating through events, allowing each service to evolve, deploy, and scale independently without impacting other components.

* **Decoupled services**, allowing the Recharge API, Bill Processing Service and downstream integrations to operate independently without introducing direct service dependencies. This reduced inter-service coupling and simplified maintenance and future enhancements.

* **Improved scalability**, as Kafka consumers can scale horizontally based on message volume without requiring additional instances of the Recharge API. During peak traffic, only the consumers responsible for downstream processing need to be scaled, resulting in better resource utilization and lower infrastructure costs.

* **Increased resilience**, since temporary downstream failures no longer directly impacted the Recharge API. Events remained safely stored in Kafka and were processed once dependent services became available, preventing cascading failures and improving overall system stability while ensuring reliable event processing.

#### BEFORE KAFKA
```text
                    BEFORE KAFKA (Synchronous Request Processing)

Customer
    |
    v
Recharge API
    |
    v
Validate Request
    |
    v
Lookup Product & Category
(PostgreSQL)
    |
    v
Identify Processing Flow
& Request Format
(Rule Engine)
    |
    v
Retrieve Previous
Bill Payment History
    |
    v
Bill Validation
    |
    v
Incentive Calculation
    |
    v
Call External BBPS Service
    |
    v
Update Bill Status
    |
    v
Persist Transaction
(PostgreSQL)
    |
    v
Return Response
```

####  AFTER KAFKA
```text
                    AFTER KAFKA (Asynchronous Processing)

Customer
    |
    v
Recharge API
    |
    v
Validate Request
    |
    v
Persist Request
(PostgreSQL)
    |
    v
Publish Event to Kafka
    |
    v
Return Response
(~300 ms)

----------------------------------------------

Kafka Consumer
(Recharge Processing Service)
    |
    v
Lookup Product & Category
(PostgreSQL)
    |
    v
Rule Engine
(Process Selection)
    |
    v
Retrieve Previous
Bill Payment History
    |
    v
Bill Validation
    |
    v
Incentive Calculation
    |
    v
Call External BBPS Service
    |
    v
Update Bill Status
(PostgreSQL)
```

### Why Cassandra?

Cassandra was selected because it provides:

* High write throughput.
* Linear horizontal scalability.
* Automatic replication and fault tolerance.
* Efficient TTL-based data expiration.
* Low storage cost for multi-terabyte datasets.

Hence, Cassandra was selected because recharge systems continuously generate large volumes (5 TB) of bill data and payment information. Its append-only write architecture enables high write throughput with minimal latency, making it suitable for storing millions of bill records while maintaining predictable performance.

#### Comparing Diffrent Storage Tools

* **Open-Source Alternative to Aerospike:** Cassandra provides enterprise-grade scalability, fault tolerance, and automatic data replication without licensing costs, helping reduce long-term operational expenses.
* **NoSQL over PostgreSQL:** A relational database like PostgreSQL was unnecessary because the data consists of independent key-value records with TTL (Time-To-Live). There are no relationships or complex joins, making a di/stributed NoSQL database a better fit.
* **High Write Throughput:** Optimized for write-heavy workloads using sequential disk writes, making it ideal for continuously storing bill data for several days.
* **Horizontal Scalability:** Nodes can be added seamlessly to handle increasing traffic and data volume while maintaining high availability.
* **Comparing Redis:** The system stored around **5 TB** of bill data. Cassandra provided scalable, disk-based storage at a lower cost, whereas Redis is optimized for in-memory caching and would have been significantly more expensive for persisting data at this scale.

## Technical Challenges

Several technical challenges were addressed during development:

* Preventing duplicate recharge processing during retries after external service recovery.
* Determining an appropriate TTL for different bill types to balance cache freshness and storage cost.
* Recovering thousands of deferred recharge requests without creating traffic spikes. This was addressed by introducing a Deferred Processing Service that retrieved eligible requests in configurable batches instead of processing the entire backlog simultaneously.
* For Kafka, implemented automatic retries with Dead Letter Queue (DLQ) handling, revalidating the data from external service
* Ensuring cached bill data remained consistent after partial and full payment scenarios.
* Balancing response time improvements while maintaining transaction reliability.

## Complexity of the Solution

The solution involved more than simply implementing a Circuit Breaker.

* Integration with multiple external biller services having different response behaviors.
* Handling high transaction volumes while maintaining low response times.
* Coordinating the Health Check Service, Deferred Processing Service, Recharge Processing Service, PostgreSQL, Cassandra, and Kafka while maintaining data consistency and transaction reliability.
* Managing multiple recharge states such as pending, deferred, processing, success, and failure.
* Supporting automatic recovery without manual operational intervention.

## Three-Stage Recovery Process

### 1. Data Caching

Bill details from external services were cached in Cassandra during normal operations.

### 2. Failure Handling

During service outages:

* Cached bill details served within **50–80 ms**.
* Collecting funds on the basis of these bills
* Recharge requests were stored in PostgreSQL in a deferred state.

### 3. Recovery and Deferred Processing

Once the external provider recovered, the Health Check service validated service availability by periodically sending probe requests.

After successful validation:

* All deferred recharge requests marked as **awaiting = true** were updated to **awaiting = false**.
* A dedicated **Deferred Processing Service** periodically scanned PostgreSQL for requests with **PENDING** status and **awaiting = false**.
* Eligible requests were picked in configurable batches (typically **50 requests per batch**) to prevent overwhelming downstream systems.
* Each request was then forwarded to the Recharge Processing Service, where normal recharge validation and processing were performed.
* Transaction status was updated to **SUCCESS** or **FAILED**, ensuring complete auditability and automatic recovery without manual intervention.

## Recharge Failure Recovery Microservices Flow

### 1. Recharge API
- Validate request
- Save transaction
- Publish Kafka event
- Return response

---

### 2. Health Check Service
- Monitor external providers
- Update `awaiting` flag based on provider health status

---

### 3. Deferred Processing Service
- Read eligible pending requests
- Process requests in batches
- Forward eligible transactions to Recharge Processing Service

---

### 4. Recharge Processing Service
- Bill validation
- Rule-Engine
- Call external BBPS providers
- Update transaction status in PostgreSQL

## Circuit Breaker States

Circuit Breaker was implemented using Resilience4j.

The open state was triggered when the external operator started rejecting requests or when response times increased beyond the configured threshold, resulting in higher waiting times for users. Once the failure rate crossed the allowed limit, the circuit breaker tripped to the open state and stopped sending further requests to the external service. This prevented repeated failures, reduced unnecessary load on the downstream system, and allowed the application to serve cached bill details and defer recharge requests safely.

| State     | Description                                                                                                            |
| --------- | ---------------------------------------------------------------------------------------------------------------------- |
| Closed | Normal operations with direct external service calls. If deferred recharge requests exist in **PENDING** status after service recovery, they become eligible for processing by the Deferred Processing Service in configurable batches. |
| Open      | Cached data from Cassandra was used, and recharge requests were stored in a deferred state.                            |
| Half-Open | Service recovery was validated by sending a 5 requests to external services in sliding window                          |

## Deployment Topology

The application was deployed on AWS EC2 instances

Infrastructure included:

* EC2 for application hosting
* EBS for persistent storage
* RDS for PostgreSQL
* Auto Scaling Groups for high availability and traffic-based scaling
* Security Groups for network security
* CloudWatch for infrastructure monitoring and scaling triggers
* Jenkins for build, deployment, and release automation

Auto Scaling Groups were configured with a mix of On-Demand and Spot instances. Scaling policies were based on historical traffic patterns, allowing the system to increase Spot capacity during predictable high-traffic day higher number of spot instances and night lower number of spot instances. Additionally, when CloudWatch alarms detected elevated load or threshold breaches, the scaling policy was updated and additional instances were launched automatically to maintain performance and availability.

Jenkins was used to automate the deployment pipeline, enabling consistent application releases across environments.

## API Gateway

API Gateway handled incoming client requests by providing:

* Request routing
* Rate limiting
* Request throttling
* Secure access to backend microservices

API Gateway routed requests to backend services running behind Auto Scaling Groups, ensuring that traffic was distributed efficiently across the application tier.

## Logging and Monitoring

* Kibana was used for centralized log analysis.
* Prometheus was used for alerting.
* Application logs older than three months were archived to Amazon S3.
* AWS CloudWatch monitored infrastructure health and triggered Auto Scaling Group policies for horizontal scaling.

## Microservice Patterns Used

* Circuit Breaker
* Event-Driven Architecture
* Asynchronous Messaging
* Database
* Retry and Recovery Processing
* Health Check Pattern

## Transaction Management

Designed a reliable transaction management mechanism to ensure recharge requests were never lost during external service outages.

* Implemented **idempotent transaction processing** using unique transaction IDs to prevent duplicate recharge requests.
* Persisted failed recharge requests in **PostgreSQL** with a **PENDING** status before responding to the customer.
* Used the **Circuit Breaker** state to determine whether requests should be processed immediately or deferred for later execution.
* Leveraged Kafka to asynchronously process recharge requests by decoupling downstream bill processing from the synchronous API request, reducing response time while improving scalability
* Introduced a dedicated **Deferred Processing Service** to periodically retrieve eligible recharge requests from PostgreSQL. Requests remained in an **awaiting** state while the external provider was unavailable and became eligible for processing only after successful health checks.
* Maintained the complete transaction lifecycle using **PENDING**, **AWAITING=true**, **PROCESSING**, **SUCCESS**, and **FAILED** states, ensuring every recharge request remained traceable throughout the recovery process.


## Production Issues and Learnings

### 1. Vault Configuration Issue

After one production deployment, the application failed to establish database connections even though no application code had changed.

**Root Cause**

The encrypted database password stored in Vault contained an unintended leading whitespace, causing authentication failures during application startup.

**Resolution**

* Compared decrypted credentials with the expected configuration.
* Worked with the Vault team to correct the secret.
* Added startup validation and improved deployment verification to detect malformed secrets before production rollout.

**Learning**

Configuration validation is as important as application validation, especially when secrets are managed by external teams.


##


## Architecture Diagram

![Recharge Failure Recovery Architecture](images/recharge-failure-recovery-architecture.png)

## Circuit-Breaker

![Circuit-Breaker](images/Circuit-Breaker.jpg)

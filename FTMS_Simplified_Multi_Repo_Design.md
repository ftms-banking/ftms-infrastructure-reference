# FTMS - Simplified Multi-Repository Microservices Design

## 🎯 REVISED ARCHITECTURE

**Key Changes:**
- ✅ **Single Shared Database** - All services use one MySQL database
- ✅ **WebClient instead of Feign** - Reactive non-blocking communication
- ✅ **Single WireMock Server** - Shared mock server with health checks only
- ✅ **One API Gateway** - Single entry point for all services
- ✅ **Developers add mocks as needed** - Minimal pre-configured stubs

---

## 🏗️ SIMPLIFIED ARCHITECTURE DIAGRAM

```
                    ┌─────────────────────────┐
                    │   API GATEWAY (8080)    │
                    │   Spring Cloud Gateway  │
                    └────────────┬────────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
           ▼                     ▼                     ▼
    ┌────────────┐      ┌────────────┐      ┌────────────┐
    │  Customer  │      │  Account   │      │Transaction │
    │  Service   │      │  Service   │      │  Service   │
    │  (8081)    │      │  (8082)    │      │  (8083)    │
    └─────┬──────┘      └─────┬──────┘      └─────┬──────┘
          │                   │                   │
          │        ┌──────────┴──────────┐        │
          │        │                     │        │
          ▼        ▼                     ▼        ▼
    ┌───────────────────────────────────────────────┐
    │        SINGLE SHARED DATABASE (MySQL)          │
    │                 ftms_db (3306)                 │
    │                                                │
    │  Tables:                                       │
    │  - customer                                    │
    │  - address                                     │
    │  - kyc_document                                │
    │  - account                                     │
    │  - transaction                                 │
    │  - compliance_event                            │
    └───────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────┐
    │   SINGLE WIREMOCK SERVER (8180)             │
    │   - Health check stubs only                 │
    │   - Devs add mocks as needed                │
    └─────────────────────────────────────────────┘
```

---

## 📦 REVISED REPOSITORY STRUCTURE

```
GitHub Organization: ftms-banking/

├── ftms-customer-service/          (Repo 1)
│   ├── src/
│   ├── docker/
│   ├── k8s/
│   ├── pom.xml
│   ├── Jenkinsfile
│   └── README.md
│
├── ftms-account-service/           (Repo 2)
│   ├── src/
│   ├── docker/
│   ├── k8s/
│   ├── pom.xml
│   ├── Jenkinsfile
│   └── README.md
│
├── ftms-transaction-service/       (Repo 3)
│   ├── src/
│   ├── docker/
│   ├── k8s/
│   ├── pom.xml
│   ├── Jenkinsfile
│   └── README.md
│
├── ftms-compliance-service/        (Repo 4)
│   ├── src/
│   ├── docker/
│   ├── k8s/
│   ├── pom.xml
│   ├── Jenkinsfile
│   └── README.md
│
├── ftms-api-gateway/               (Repo 5)
│   ├── src/
│   ├── docker/
│   ├── k8s/
│   ├── pom.xml
│   ├── Jenkinsfile
│   └── README.md
│
└── ftms-infrastructure/            (Repo 6 - Shared Resources)
    ├── database/
    │   └── schema/
    │       ├── V001__create_all_tables.sql
    │       ├── V002__add_indexes.sql
    │       └── V003__seed_data.sql
    ├── wiremock/
    │   ├── mappings/
    │   │   ├── customer-service-health.json
    │   │   ├── account-service-health.json
    │   │   ├── transaction-service-health.json
    │   │   └── README.md (Instructions for devs)
    │   ├── __files/
    │   ├── start-wiremock.sh
    │   └── docker-compose-wiremock.yml
    ├── docker-compose/
    │   ├── docker-compose-full.yml
    │   ├── docker-compose-services.yml
    │   └── docker-compose-db.yml
    ├── k8s/
    │   ├── database/
    │   ├── api-gateway/
    │   └── services/
    └── README.md
```

---

## 🗄️ SINGLE DATABASE DESIGN

### Database: ftms_db

All services share the same database but **own their tables**:

```sql
-- ftms_db contains ALL tables

-- Customer Service Tables
CREATE TABLE customer (...);
CREATE TABLE address (...);
CREATE TABLE kyc_document (...);

-- Account Service Tables
CREATE TABLE account (...);
CREATE TABLE account_balance_history (...);

-- Transaction Service Tables
CREATE TABLE transaction (...);
CREATE TABLE transaction_saga (...);

-- Compliance Service Tables
CREATE TABLE compliance_event (...);
CREATE TABLE audit_log (...);
```

### Single Database Configuration

**All services connect to same database:**

```yaml
# application.yml (same for all services)
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST:localhost}:3306/ftms_db
    username: ftms_user
    password: ftms_pass
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    baseline-on-migrate: true
```

### Benefits of Single Database
- ✅ **Simpler setup** - One database to manage
- ✅ **Easy joins** - Can query across tables if needed
- ✅ **ACID transactions** - Within database boundaries
- ✅ **Easier development** - No distributed transaction complexity
- ✅ **Cost effective** - One database instance

---

## 🌐 WEBCLIENT CONFIGURATION (NOT FEIGN)

### Replace Feign with WebClient

**Add Dependencies (pom.xml):**

```xml
<!-- WebClient for reactive non-blocking calls -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- NO FEIGN DEPENDENCY -->
```

### WebClient Configuration

**Create `WebClientConfig.java`:**

```java
package com.ftms.customer.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }

    @Bean
    public WebClient accountServiceWebClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://localhost:8082")  // Account service URL
            .defaultHeader("Content-Type", "application/json")
            .build();
    }

    @Bean
    public WebClient transactionServiceWebClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://localhost:8083")  // Transaction service URL
            .defaultHeader("Content-Type", "application/json")
            .build();
    }
}
```

### WebClient Usage Example

**Customer Service calling Account Service:**

```java
package com.ftms.customer.service;

import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class CustomerService {

    private final WebClient accountServiceWebClient;

    public CustomerService(WebClient accountServiceWebClient) {
        this.accountServiceWebClient = accountServiceWebClient;
    }

    public CustomerResponse createCustomer(CustomerCreateRequest request) {
        // Save customer to database
        Customer customer = saveCustomer(request);

        // Call Account Service using WebClient (blocking)
        AccountResponse account = accountServiceWebClient
            .post()
            .uri("/api/v1/accounts")
            .bodyValue(new AccountCreateRequest(customer.getUuid()))
            .retrieve()
            .bodyToMono(AccountResponse.class)
            .block();  // Block for synchronous behavior

        customer.setAccountId(account.getUuid());
        return mapToResponse(customer);
    }

    // OR use reactive non-blocking approach
    public Mono<CustomerResponse> createCustomerReactive(CustomerCreateRequest request) {
        return Mono.fromSupplier(() -> saveCustomer(request))
            .flatMap(customer -> accountServiceWebClient
                .post()
                .uri("/api/v1/accounts")
                .bodyValue(new AccountCreateRequest(customer.getUuid()))
                .retrieve()
                .bodyToMono(AccountResponse.class)
                .map(account -> {
                    customer.setAccountId(account.getUuid());
                    return mapToResponse(customer);
                }));
    }

    private Customer saveCustomer(CustomerCreateRequest request) {
        // Save to database
        return customerRepository.save(new Customer(request));
    }
}
```

---

## 🎭 SINGLE WIREMOCK SERVER SETUP

### Minimal WireMock Configuration

**Only health check stubs pre-configured. Developers add mocks as needed.**

### Infrastructure Repository: wiremock/mappings/

```
ftms-infrastructure/wiremock/
├── mappings/
│   ├── customer-service-health.json
│   ├── account-service-health.json
│   ├── transaction-service-health.json
│   ├── compliance-service-health.json
│   └── README.md
├── __files/
├── start-wiremock.sh
└── docker-compose-wiremock.yml
```

### Health Check Stubs Only

**`mappings/customer-service-health.json`:**

```json
{
  "request": {
    "method": "GET",
    "url": "/api/v1/health"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": {
      "status": "UP",
      "service": "customer-service",
      "version": "1.0.0",
      "timestamp": "{{now format='yyyy-MM-dd'T'HH:mm:ss'Z'}}"
    }
  }
}
```

**`mappings/account-service-health.json`:**

```json
{
  "request": {
    "method": "GET",
    "url": "/api/v1/health"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": {
      "status": "UP",
      "service": "account-service",
      "version": "1.0.0",
      "timestamp": "{{now format='yyyy-MM-dd'T'HH:mm:ss'Z'}}"
    }
  }
}
```

**`mappings/transaction-service-health.json`:**

```json
{
  "request": {
    "method": "GET",
    "url": "/api/v1/health"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": {
      "status": "UP",
      "service": "transaction-service",
      "version": "1.0.0",
      "timestamp": "{{now format='yyyy-MM-dd'T'HH:mm:ss'Z'}}"
    }
  }
}
```

### Developer Instructions (README.md)

**`wiremock/mappings/README.md`:**

```markdown
# WireMock Mappings

This directory contains WireMock stub mappings for FTMS microservices.

## Pre-configured Stubs

Only health check endpoints are pre-configured:
- `customer-service-health.json`
- `account-service-health.json`
- `transaction-service-health.json`
- `compliance-service-health.json`

## Adding Custom Stubs

Developers should add their own stub mappings as needed for testing.

### Example: Mock Account Creation

Create `account-create-201.json`:

```json
{
  "request": {
    "method": "POST",
    "url": "/api/v1/accounts",
    "bodyPatterns": [
      {
        "matchesJsonPath": "$.customerId"
      }
    ]
  },
  "response": {
    "status": 201,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": {
      "uuid": "{{randomValue type='UUID'}}",
      "customerId": "{{jsonPath request.body '$.customerId'}}",
      "accountNumber": "ACC{{randomValue length=10 type='NUMERIC'}}",
      "balance": 0.00,
      "status": "ACTIVE"
    }
  }
}
```

### Testing Your Stub

1. Add your JSON file to `mappings/` directory
2. Restart WireMock: `./start-wiremock.sh`
3. Test: `curl -X POST http://localhost:8180/api/v1/accounts -d '{"customerId":"123"}'`

### Response Templating

WireMock supports templating:
- `{{randomValue type='UUID'}}` - Random UUID
- `{{randomValue length=10 type='NUMERIC'}}` - Random 10-digit number
- `{{jsonPath request.body '$.field'}}` - Extract from request
- `{{now format='yyyy-MM-dd'}}` - Current timestamp

For more: https://wiremock.org/docs/response-templating/
```

### Single WireMock Docker Compose

**`ftms-infrastructure/wiremock/docker-compose-wiremock.yml`:**

```yaml
version: '3.8'

services:
  wiremock:
    image: wiremock/wiremock:latest
    container_name: ftms-wiremock
    ports:
      - "8180:8080"
    volumes:
      - ./mappings:/home/wiremock/mappings
      - ./__files:/home/wiremock/__files
    command: [
      "--global-response-templating",
      "--verbose",
      "--enable-browser-proxying"
    ]
    networks:
      - ftms-network

networks:
  ftms-network:
    driver: bridge
```

**Start WireMock:**

```bash
cd ftms-infrastructure/wiremock
docker-compose -f docker-compose-wiremock.yml up -d

# Access admin UI
open http://localhost:8180/__admin
```

---

## 🐳 COMPLETE DOCKER COMPOSE SETUP

### ftms-infrastructure/docker-compose-full.yml

```yaml
version: '3.8'

services:
  # Shared MySQL Database
  mysql:
    image: mysql:8.0
    container_name: ftms-mysql
    environment:
      MYSQL_DATABASE: ftms_db
      MYSQL_USER: ftms_user
      MYSQL_PASSWORD: ftms_pass
      MYSQL_ROOT_PASSWORD: rootpass
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ../database/schema:/docker-entrypoint-initdb.d
    networks:
      - ftms-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 5s
      retries: 10

  # Single WireMock Server
  wiremock:
    image: wiremock/wiremock:latest
    container_name: ftms-wiremock
    ports:
      - "8180:8080"
    volumes:
      - ../wiremock/mappings:/home/wiremock/mappings
      - ../wiremock/__files:/home/wiremock/__files
    command: ["--global-response-templating", "--verbose"]
    networks:
      - ftms-network

  # API Gateway
  api-gateway:
    image: ftms/api-gateway:latest
    container_name: ftms-api-gateway
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
    depends_on:
      - customer-service
      - account-service
      - transaction-service
    networks:
      - ftms-network

  # Customer Service
  customer-service:
    image: ftms/customer-service:latest
    container_name: ftms-customer-service
    ports:
      - "8081:8081"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DB_HOST=mysql
      - WIREMOCK_URL=http://wiremock:8080
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - ftms-network

  # Account Service
  account-service:
    image: ftms/account-service:latest
    container_name: ftms-account-service
    ports:
      - "8082:8082"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DB_HOST=mysql
      - WIREMOCK_URL=http://wiremock:8080
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - ftms-network

  # Transaction Service
  transaction-service:
    image: ftms/transaction-service:latest
    container_name: ftms-transaction-service
    ports:
      - "8083:8083"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DB_HOST=mysql
      - WIREMOCK_URL=http://wiremock:8080
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - ftms-network

  # Compliance Service
  compliance-service:
    image: ftms/compliance-service:latest
    container_name: ftms-compliance-service
    ports:
      - "8084:8084"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DB_HOST=mysql
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - ftms-network

volumes:
  mysql-data:

networks:
  ftms-network:
    driver: bridge
```

---

## 🚪 API GATEWAY CONFIGURATION

### API Gateway Routes (application.yml)

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Customer Service Routes
        - id: customer-service
          uri: http://customer-service:8081
          predicates:
            - Path=/api/v1/customers/**
          filters:
            - name: CircuitBreaker
              args:
                name: customerCircuitBreaker
                fallbackUri: forward:/fallback

        # Account Service Routes
        - id: account-service
          uri: http://account-service:8082
          predicates:
            - Path=/api/v1/accounts/**
          filters:
            - name: CircuitBreaker
              args:
                name: accountCircuitBreaker
                fallbackUri: forward:/fallback

        # Transaction Service Routes
        - id: transaction-service
          uri: http://transaction-service:8083
          predicates:
            - Path=/api/v1/transactions/**
          filters:
            - name: CircuitBreaker
              args:
                name: transactionCircuitBreaker
                fallbackUri: forward:/fallback

        # Compliance Service Routes
        - id: compliance-service
          uri: http://compliance-service:8084
          predicates:
            - Path=/api/v1/compliance/**

      # Global Filters
      default-filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
```

---

## 📁 UPDATED SERVICE STRUCTURE (SIMPLIFIED)

### Customer Service Repository

```
ftms-customer-service/
├── .github/workflows/
├── src/
│   ├── main/
│   │   ├── java/com/ftms/customer/
│   │   │   ├── CustomerServiceApplication.java
│   │   │   ├── config/
│   │   │   │   ├── WebClientConfig.java     # WebClient beans
│   │   │   │   ├── SwaggerConfig.java
│   │   │   │   └── SecurityConfig.java
│   │   │   ├── controller/
│   │   │   ├── service/
│   │   │   ├── repository/
│   │   │   ├── domain/
│   │   │   ├── dto/
│   │   │   └── client/
│   │   │       └── AccountServiceClient.java  # Uses WebClient
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── application-docker.yml
│   └── test/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── k8s/
├── pom.xml
├── Jenkinsfile
└── README.md
```

---

## 🔧 SIMPLIFIED CONFIGURATION

### application.yml (Customer Service)

```yaml
spring:
  application:
    name: customer-service
  profiles:
    active: dev

  # Single Shared Database
  datasource:
    url: jdbc:mysql://${DB_HOST:localhost}:3306/ftms_db
    username: ftms_user
    password: ftms_pass
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration

# WebClient Service URLs
services:
  account:
    url: ${ACCOUNT_SERVICE_URL:http://localhost:8082}
  transaction:
    url: ${TRANSACTION_SERVICE_URL:http://localhost:8083}
  wiremock:
    url: ${WIREMOCK_URL:http://localhost:8180}

server:
  port: 8081

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

---

## 🚀 RUNNING THE COMPLETE STACK

### Option 1: Start Everything

```bash
# In ftms-infrastructure repository
cd ftms-infrastructure/docker-compose

# Start all services
docker-compose -f docker-compose-full.yml up -d

# Verify services
docker ps

# Access services:
# - API Gateway: http://localhost:8080
# - Customer Service: http://localhost:8081
# - Account Service: http://localhost:8082
# - Transaction Service: http://localhost:8083
# - Compliance Service: http://localhost:8084
# - WireMock: http://localhost:8180
# - MySQL: localhost:3306
```

### Option 2: Start Individual Service (Dev Mode)

```bash
# In customer-service repository
cd ftms-customer-service

# Ensure MySQL is running
docker run -d -p 3306:3306 \
  -e MYSQL_DATABASE=ftms_db \
  -e MYSQL_USER=ftms_user \
  -e MYSQL_PASSWORD=ftms_pass \
  mysql:8.0

# Run service
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Service available at: http://localhost:8081
```

---

## ✅ KEY SIMPLIFICATIONS

| Aspect | Old Design | New Design ✅ |
|--------|-----------|--------------|
| **Database** | 5 separate databases | 1 shared database (ftms_db) |
| **HTTP Client** | Feign Client | WebClient (reactive) |
| **WireMock** | 5 separate instances | 1 shared instance (8180) |
| **Mock Stubs** | Pre-configured for all | Health checks only, devs add more |
| **Ports** | WireMock: 8180-8580 | WireMock: 8180 only |
| **Complexity** | High | Low |

---

## 📊 PORT ALLOCATION (SIMPLIFIED)

| Service | Port | Database | WireMock |
|---------|------|----------|----------|
| API Gateway | 8080 | - | - |
| Customer Service | 8081 | ftms_db (3306) | Shared (8180) |
| Account Service | 8082 | ftms_db (3306) | Shared (8180) |
| Transaction Service | 8083 | ftms_db (3306) | Shared (8180) |
| Compliance Service | 8084 | ftms_db (3306) | Shared (8180) |
| MySQL | 3306 | - | - |
| WireMock | 8180 | - | - |

---

## 🎯 BENEFITS OF SIMPLIFIED DESIGN

✅ **Single Database**
- Easier setup and management
- No distributed transactions
- Simple data access
- Cost effective

✅ **WebClient**
- Non-blocking reactive calls
- Better performance
- More modern approach
- Supports both sync and async

✅ **Single WireMock**
- One instance to manage
- Shared stubs across services
- Developers add mocks as needed
- Less resource usage

✅ **One API Gateway**
- Single entry point
- Centralized routing
- Simplified security

---

## 🎉 COMPLETE PACKAGE

**You now have:**
- ✅ Single shared database design
- ✅ WebClient configuration instead of Feign
- ✅ Single WireMock server with health checks
- ✅ One API Gateway
- ✅ Simplified docker-compose
- ✅ Developer-friendly mock setup
- ✅ Easier local development

**Next Steps:**
1. Clone repositories
2. Setup single database
3. Start WireMock server
4. Run services
5. Developers add mocks as needed

---

**Document Version:** 2.0 (Simplified)
**Last Updated:** November 4, 2025
**Status:** PRODUCTION-READY ✅

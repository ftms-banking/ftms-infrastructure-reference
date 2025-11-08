# FTMS - Complete JIRA Stories & Developer Workflows

## 🎯 OBJECTIVE

**Complete set of JIRA stories with detailed workflows, API call sequences, and inter-service communication patterns for FTMS microservices project.**

---

## 📋 EPIC STRUCTURE

```
FTMS-EPIC-1: Infrastructure Setup
FTMS-EPIC-2: Customer Service Development
FTMS-EPIC-3: Account Service Development
FTMS-EPIC-4: Transaction Service Development
FTMS-EPIC-5: Compliance Service Development
FTMS-EPIC-6: API Gateway & Integration
FTMS-EPIC-7: Testing & Deployment
```

---

## 🏗️ EPIC 1: INFRASTRUCTURE SETUP

### FTMS-1: Setup Shared Database Schema
**Type:** Task  
**Priority:** Highest  
**Story Points:** 5

**Description:**
Create MySQL database schema with all tables for microservices in single shared database (ftms_db).

**Acceptance Criteria:**
- [ ] MySQL 8.0 database created with name `ftms_db`
- [ ] All 9 tables created (customer, address, kyc_document, account, account_balance_history, transaction, transaction_saga, compliance_event, audit_log)
- [ ] Foreign key constraints defined
- [ ] Check constraints for enums defined
- [ ] All indexes created for performance
- [ ] Seed data inserted for testing

**Technical Tasks:**
```sql
-- V001__create_all_tables.sql
-- V002__add_indexes.sql
-- V003__seed_data.sql
```

**Verification:**
```bash
docker exec -it ftms-mysql mysql -u ftms_user -pftms_pass ftms_db
SHOW TABLES;
SELECT * FROM customer;
```

---

### FTMS-2: Setup WireMock Server
**Type:** Task  
**Priority:** High  
**Story Points:** 3

**Description:**
Setup single shared WireMock server for mocking external dependencies during development and testing.

**Acceptance Criteria:**
- [ ] WireMock server running on port 8180
- [ ] Health check stubs created for all services
- [ ] Documentation for developers to add custom stubs
- [ ] Docker Compose configuration included

**WireMock Stubs:**
```json
// customer-service-health.json
{
  "request": {
    "method": "GET",
    "url": "/api/v1/health"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "status": "UP",
      "service": "customer-service"
    }
  }
}
```

---

### FTMS-3: Setup Docker Compose Infrastructure
**Type:** Task  
**Priority:** High  
**Story Points:** 3

**Description:**
Create Docker Compose configuration for running all services locally.

**Acceptance Criteria:**
- [ ] MySQL container configuration
- [ ] WireMock container configuration
- [ ] Network configuration
- [ ] Volume mounts for persistence
- [ ] Environment variables configured

**Docker Compose:**
```yaml
services:
  mysql:
    image: mysql:8.0
    ports: ["3306:3306"]
    environment:
      MYSQL_DATABASE: ftms_db
  
  wiremock:
    image: wiremock/wiremock:latest
    ports: ["8180:8080"]
```

---

### FTMS-4: Setup GitHub Repositories
**Type:** Task  
**Priority:** High  
**Story Points:** 2

**Description:**
Create separate GitHub repositories for each microservice.

**Acceptance Criteria:**
- [ ] ftms-customer-service repo created
- [ ] ftms-account-service repo created
- [ ] ftms-transaction-service repo created
- [ ] ftms-compliance-service repo created
- [ ] ftms-api-gateway repo created
- [ ] ftms-infrastructure repo created
- [ ] README templates added to each repo
- [ ] .gitignore configured

---

## 👤 EPIC 2: CUSTOMER SERVICE DEVELOPMENT

### FTMS-10: Create Customer Service Entity Layer
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement JPA entities for Customer Service (Customer, Address, KYCDocument).

**Acceptance Criteria:**
- [ ] Customer entity with all fields
- [ ] Address entity with Customer relationship
- [ ] KYCDocument entity with Customer relationship
- [ ] All enums created (CustomerStatus, KYCStatus, AddressType, DocumentType, DocumentStatus)
- [ ] Lifecycle callbacks (@PrePersist, @PreUpdate)
- [ ] Optimistic locking (@Version)
- [ ] Helper methods implemented

**Entities:**
```java
@Entity
@Table(name = "customer")
public class Customer {
    @Id @GeneratedValue
    private String uuid;
    private String email;
    private String firstName;
    private String lastName;
    private LocalDate dateOfBirth;
    private String phoneNumber;
    @Enumerated(EnumType.STRING)
    private CustomerStatus status;
    @Enumerated(EnumType.STRING)
    private KYCStatus kycStatus;
    
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL)
    private List<Address> addresses;
    
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL)
    private List<KYCDocument> kycDocuments;
}
```

---

### FTMS-11: Create Customer Service Repository Layer
**Type:** Story  
**Priority:** High  
**Story Points:** 3

**Description:**
Implement Spring Data JPA repositories for Customer Service entities.

**Acceptance Criteria:**
- [ ] CustomerRepository with custom queries
- [ ] AddressRepository
- [ ] KYCDocumentRepository
- [ ] Query methods for common operations

**Repositories:**
```java
public interface CustomerRepository extends JpaRepository<Customer, String> {
    Optional<Customer> findByEmail(String email);
    List<Customer> findByStatus(CustomerStatus status);
    List<Customer> findByKycStatus(KYCStatus kycStatus);
    
    @Query("SELECT c FROM Customer c WHERE c.firstName LIKE %:name% OR c.lastName LIKE %:name%")
    List<Customer> searchByName(@Param("name") String name);
}
```

---

### FTMS-12: Implement Customer Registration Flow
**Type:** Story  
**Priority:** Highest  
**Story Points:** 8

**Description:**
Implement complete customer registration flow with inter-service communication to Account Service.

**Workflow Diagram:**
```
Client
  ↓
  POST /api/v1/customers
  ↓
API Gateway (8080)
  ↓
Customer Service (8081)
  ↓
1. Validate request
2. Check email uniqueness
3. Create customer (status: PENDING)
4. Save to database
  ↓
  [WebClient Call to Account Service]
  ↓
Account Service (8082)
  ↓
1. Validate customer exists
2. Generate account number
3. Create account (status: PENDING)
4. Save to database
5. Return account details
  ↓
Customer Service
  ↓
1. Update customer with account reference
2. Return customer response
  ↓
API Gateway
  ↓
Client (201 Created)
```

**Service Implementation:**
```java
@Service
@Transactional
public class CustomerService {
    
    private final CustomerRepository customerRepository;
    private final WebClient accountServiceWebClient;
    
    public CustomerResponse createCustomer(CustomerCreateRequest request) {
        // Step 1: Validate email uniqueness
        if (customerRepository.findByEmail(request.getEmail()).isPresent()) {
            throw new EmailAlreadyExistsException(request.getEmail());
        }
        
        // Step 2: Create customer entity
        Customer customer = Customer.builder()
            .email(request.getEmail())
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .dateOfBirth(request.getDateOfBirth())
            .phoneNumber(request.getPhoneNumber())
            .status(CustomerStatus.PENDING)
            .kycStatus(KYCStatus.NOT_SUBMITTED)
            .build();
        
        // Step 3: Save customer
        customer = customerRepository.save(customer);
        
        // Step 4: Call Account Service to create account
        AccountCreateRequest accountRequest = AccountCreateRequest.builder()
            .customerId(customer.getUuid())
            .accountType("SAVINGS")
            .currency("USD")
            .build();
        
        AccountResponse accountResponse = accountServiceWebClient
            .post()
            .uri("/api/v1/accounts")
            .bodyValue(accountRequest)
            .retrieve()
            .onStatus(HttpStatusCode::isError, response -> {
                // Handle error - rollback customer creation
                return Mono.error(new AccountCreationException());
            })
            .bodyToMono(AccountResponse.class)
            .block(); // Blocking WebClient call
        
        // Step 5: Return customer response
        return CustomerResponse.builder()
            .uuid(customer.getUuid())
            .email(customer.getEmail())
            .firstName(customer.getFirstName())
            .lastName(customer.getLastName())
            .status(customer.getStatus())
            .kycStatus(customer.getKycStatus())
            .accountId(accountResponse.getUuid())
            .build();
    }
}
```

**API Specification:**
```yaml
POST /api/v1/customers
Request:
{
  "email": "john@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "dateOfBirth": "1990-05-15",
  "phoneNumber": "+1234567890"
}

Response (201):
{
  "uuid": "550e8400-e29b-41d4-a716-446655440001",
  "email": "john@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "status": "PENDING",
  "kycStatus": "NOT_SUBMITTED",
  "accountId": "850e8400-e29b-41d4-a716-446655440001",
  "createdAt": "2025-11-06T10:30:00Z"
}
```

**Acceptance Criteria:**
- [ ] Customer created in database
- [ ] Account created via inter-service call
- [ ] Email uniqueness validated
- [ ] 201 Created response returned
- [ ] Transaction rolled back if account creation fails

---

### FTMS-13: Implement KYC Document Upload
**Type:** Story  
**Priority:** High  
**Story Points:** 8

**Description:**
Implement KYC document upload functionality with file storage and verification workflow.

**Workflow Diagram:**
```
Client
  ↓
  POST /api/v1/customers/{customerId}/kyc
  (multipart/form-data)
  ↓
API Gateway
  ↓
Customer Service
  ↓
1. Validate customer exists
2. Validate file type (PDF, JPG, PNG)
3. Validate file size (<5MB)
4. Generate unique filename
5. Store file (local/S3)
  ↓
6. Create KYCDocument entity
7. Set status: PENDING
8. Save to database
  ↓
9. Publish event: kyc.document.uploaded
  ↓
Compliance Service (async)
  ↓
1. Receive event
2. Create compliance event
3. Store audit log
  ↓
Response (201 Created)
```

**Service Implementation:**
```java
@Service
public class KYCService {
    
    private final CustomerRepository customerRepository;
    private final KYCDocumentRepository kycDocumentRepository;
    private final FileStorageService fileStorageService;
    
    public KYCDocumentResponse uploadDocument(
        String customerId,
        MultipartFile file,
        DocumentType documentType,
        String documentNumber
    ) {
        // Step 1: Validate customer exists
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // Step 2: Validate file
        validateFile(file);
        
        // Step 3: Store file
        String filePath = fileStorageService.store(file, customerId);
        
        // Step 4: Create KYC document
        KYCDocument document = KYCDocument.builder()
            .customer(customer)
            .documentType(documentType)
            .documentNumber(documentNumber)
            .filePath(filePath)
            .fileName(file.getOriginalFilename())
            .fileSize(file.getSize())
            .mimeType(file.getContentType())
            .status(DocumentStatus.PENDING)
            .build();
        
        document = kycDocumentRepository.save(document);
        
        // Step 5: Update customer KYC status
        if (customer.getKycStatus() == KYCStatus.NOT_SUBMITTED) {
            customer.setKycStatus(KYCStatus.SUBMITTED);
            customerRepository.save(customer);
        }
        
        return mapToResponse(document);
    }
    
    private void validateFile(MultipartFile file) {
        // Check file not empty
        if (file.isEmpty()) {
            throw new InvalidFileException("File is empty");
        }
        
        // Check file size (max 5MB)
        if (file.getSize() > 5 * 1024 * 1024) {
            throw new FileSizeExceededException("File size exceeds 5MB");
        }
        
        // Check file type
        String contentType = file.getContentType();
        if (!List.of("application/pdf", "image/jpeg", "image/png")
                .contains(contentType)) {
            throw new InvalidFileTypeException("Invalid file type");
        }
    }
}
```

**Acceptance Criteria:**
- [ ] File uploaded and stored
- [ ] KYC document record created
- [ ] Customer KYC status updated
- [ ] File validation performed
- [ ] Compliance event published

---

### FTMS-14: Implement KYC Verification Flow
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement KYC document verification by compliance team with approval/rejection workflow.

**Workflow Diagram:**
```
Compliance Officer
  ↓
  POST /api/v1/customers/{customerId}/kyc/{documentId}/verify
  {
    "action": "APPROVE|REJECT",
    "reason": "..." (if rejected)
  }
  ↓
Customer Service
  ↓
1. Load KYC document
2. Validate document status (must be PENDING)
3. Update document status
4. Set verified_at timestamp
5. Set verified_by (officer UUID)
  ↓
If APPROVED:
  ↓
  6. Check if all required documents approved
  7. If yes, update customer.kycStatus = APPROVED
  8. Call Account Service to activate account
    ↓
    Account Service
    ↓
    1. Load account
    2. Update status to ACTIVE
    3. Return success
  ↓
If REJECTED:
  ↓
  6. Update customer.kycStatus = REJECTED
  7. Store rejection reason
  ↓
Response (200 OK)
```

**Service Implementation:**
```java
@Service
@Transactional
public class KYCVerificationService {
    
    private final KYCDocumentRepository kycDocumentRepository;
    private final CustomerRepository customerRepository;
    private final WebClient accountServiceWebClient;
    
    public KYCDocumentResponse verifyDocument(
        String customerId,
        String documentId,
        KYCVerificationRequest request,
        String verifierId
    ) {
        // Load document
        KYCDocument document = kycDocumentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));
        
        // Validate belongs to customer
        if (!document.getCustomer().getUuid().equals(customerId)) {
            throw new UnauthorizedException("Document does not belong to customer");
        }
        
        // Validate status
        if (document.getStatus() != DocumentStatus.PENDING) {
            throw new InvalidStateException("Document is not pending verification");
        }
        
        // Approve or reject
        if (request.getAction() == VerificationAction.APPROVE) {
            document.approve(verifierId);
            
            // Check if all documents approved
            Customer customer = document.getCustomer();
            boolean allApproved = customer.getKycDocuments().stream()
                .allMatch(KYCDocument::isApproved);
            
            if (allApproved) {
                customer.setKycStatus(KYCStatus.APPROVED);
                customerRepository.save(customer);
                
                // Activate account via WebClient
                activateCustomerAccount(customer.getUuid());
            }
        } else {
            document.reject(verifierId, request.getReason());
            
            Customer customer = document.getCustomer();
            customer.setKycStatus(KYCStatus.REJECTED);
            customerRepository.save(customer);
        }
        
        kycDocumentRepository.save(document);
        
        return mapToResponse(document);
    }
    
    private void activateCustomerAccount(String customerId) {
        accountServiceWebClient
            .patch()
            .uri("/api/v1/accounts/customer/{customerId}/activate", customerId)
            .retrieve()
            .bodyToMono(Void.class)
            .block();
    }
}
```

**Acceptance Criteria:**
- [ ] Document can be approved/rejected
- [ ] Customer KYC status updated accordingly
- [ ] Account activated when all documents approved
- [ ] Rejection reason stored

---

## 💰 EPIC 3: ACCOUNT SERVICE DEVELOPMENT

### FTMS-20: Create Account Service Entity Layer
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement JPA entities for Account Service (Account, AccountBalanceHistory).

**Acceptance Criteria:**
- [ ] Account entity with balance management
- [ ] AccountBalanceHistory entity
- [ ] Enums created (AccountType, AccountStatus)
- [ ] Balance management methods (credit, debit, reserve, release)
- [ ] Automatic balance history tracking

---

### FTMS-21: Implement Account Creation
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement account creation triggered by Customer Service.

**Workflow:**
```
Customer Service
  ↓ (WebClient)
  POST /api/v1/accounts
  {
    "customerId": "uuid",
    "accountType": "SAVINGS",
    "currency": "USD"
  }
  ↓
Account Service
  ↓
1. Validate customer exists (call Customer Service)
  ↓
  Customer Service
  GET /api/v1/customers/{customerId}
  Response: Customer details
  ↓
2. Generate unique account number
3. Create account (balance: 0, status: PENDING)
4. Save to database
  ↓
Response (201 Created)
{
  "uuid": "account-uuid",
  "customerId": "customer-uuid",
  "accountNumber": "ACC0000000001",
  "accountType": "SAVINGS",
  "balance": 0.00,
  "availableBalance": 0.00,
  "status": "PENDING"
}
```

**Service Implementation:**
```java
@Service
@Transactional
public class AccountService {
    
    private final AccountRepository accountRepository;
    private final WebClient customerServiceWebClient;
    
    public AccountResponse createAccount(AccountCreateRequest request) {
        // Step 1: Validate customer exists
        CustomerResponse customer = customerServiceWebClient
            .get()
            .uri("/api/v1/customers/{id}", request.getCustomerId())
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, response -> 
                Mono.error(new CustomerNotFoundException())
            )
            .bodyToMono(CustomerResponse.class)
            .block();
        
        // Step 2: Generate account number
        String accountNumber = generateAccountNumber();
        
        // Step 3: Create account
        Account account = Account.builder()
            .customerId(request.getCustomerId())
            .accountNumber(accountNumber)
            .accountType(AccountType.valueOf(request.getAccountType()))
            .currency(request.getCurrency())
            .balance(BigDecimal.ZERO)
            .availableBalance(BigDecimal.ZERO)
            .status(AccountStatus.PENDING)
            .build();
        
        account = accountRepository.save(account);
        
        return mapToResponse(account);
    }
    
    private String generateAccountNumber() {
        // Generate unique 10-digit account number
        long count = accountRepository.count();
        return String.format("ACC%010d", count + 1);
    }
}
```

**Acceptance Criteria:**
- [ ] Account created with unique account number
- [ ] Customer existence validated
- [ ] Initial balance set to zero
- [ ] Status set to PENDING

---

### FTMS-22: Implement Account Activation
**Type:** Story  
**Priority:** High  
**Story Points:** 3

**Description:**
Implement account activation triggered by KYC approval.

**Workflow:**
```
Customer Service (KYC Approved)
  ↓ (WebClient)
  PATCH /api/v1/accounts/customer/{customerId}/activate
  ↓
Account Service
  ↓
1. Find all accounts for customer
2. Filter by status = PENDING
3. Update status to ACTIVE
4. Save to database
  ↓
Response (200 OK)
```

**Service Implementation:**
```java
@Service
@Transactional
public class AccountService {
    
    public void activateCustomerAccounts(String customerId) {
        List<Account> accounts = accountRepository
            .findByCustomerIdAndStatus(customerId, AccountStatus.PENDING);
        
        accounts.forEach(Account::activate);
        
        accountRepository.saveAll(accounts);
    }
}
```

---

### FTMS-23: Implement Balance Operations
**Type:** Story  
**Priority:** Highest  
**Story Points:** 8

**Description:**
Implement credit and debit operations with balance history tracking.

**Credit Operation Workflow:**
```
Transaction Service
  ↓ (WebClient)
  POST /api/v1/accounts/{accountId}/credit
  {
    "amount": 500.00,
    "reason": "DEPOSIT",
    "transactionId": "txn-uuid"
  }
  ↓
Account Service
  ↓
1. Load account with pessimistic lock
  SELECT * FROM account WHERE uuid = ? FOR UPDATE
  ↓
2. Validate amount > 0
3. Calculate new balance
  newBalance = currentBalance + amount
  newAvailableBalance = currentAvailableBalance + amount
  ↓
4. Update account balances
5. Create balance history record
  {
    previousBalance: 1000.00,
    newBalance: 1500.00,
    changeAmount: +500.00,
    changeReason: "DEPOSIT",
    transactionId: "txn-uuid"
  }
  ↓
6. Save account (with version check)
  ↓
Response (200 OK)
{
  "accountId": "account-uuid",
  "previousBalance": 1000.00,
  "newBalance": 1500.00,
  "availableBalance": 1500.00
}
```

**Debit Operation Workflow:**
```
Transaction Service
  ↓ (WebClient)
  POST /api/v1/accounts/{accountId}/debit
  {
    "amount": 200.00,
    "reason": "WITHDRAWAL",
    "transactionId": "txn-uuid"
  }
  ↓
Account Service
  ↓
1. Load account with pessimistic lock
2. Validate amount > 0
3. Validate sufficient available balance
  IF availableBalance < amount THEN
    throw InsufficientBalanceException
  ↓
4. Calculate new balance
  newBalance = currentBalance - amount
  newAvailableBalance = currentAvailableBalance - amount
  ↓
5. Update account balances
6. Create balance history record
7. Save account (with version check)
  ↓
Response (200 OK)
```

**Service Implementation:**
```java
@Service
public class AccountBalanceService {
    
    @Transactional
    public BalanceOperationResponse credit(
        String accountId,
        BalanceOperationRequest request
    ) {
        // Load with pessimistic lock
        Account account = accountRepository
            .findByIdWithLock(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        // Perform credit operation
        BigDecimal previousBalance = account.getBalance();
        account.credit(
            request.getAmount(),
            request.getReason(),
            request.getTransactionId()
        );
        
        // Save (version check automatic)
        accountRepository.save(account);
        
        return BalanceOperationResponse.builder()
            .accountId(accountId)
            .previousBalance(previousBalance)
            .newBalance(account.getBalance())
            .availableBalance(account.getAvailableBalance())
            .build();
    }
    
    @Transactional
    public BalanceOperationResponse debit(
        String accountId,
        BalanceOperationRequest request
    ) {
        Account account = accountRepository
            .findByIdWithLock(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        BigDecimal previousBalance = account.getBalance();
        
        // This will throw exception if insufficient balance
        account.debit(
            request.getAmount(),
            request.getReason(),
            request.getTransactionId()
        );
        
        accountRepository.save(account);
        
        return BalanceOperationResponse.builder()
            .accountId(accountId)
            .previousBalance(previousBalance)
            .newBalance(account.getBalance())
            .availableBalance(account.getAvailableBalance())
            .build();
    }
}
```

**Acceptance Criteria:**
- [ ] Credit increases balance and available balance
- [ ] Debit decreases balance and available balance
- [ ] Insufficient balance throws exception
- [ ] Balance history automatically recorded
- [ ] Optimistic locking prevents concurrent modifications

---

### FTMS-24: Implement Balance Reservation System
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement balance reservation for pending transactions (SAGA pattern support).

**Reserve Balance Workflow:**
```
Transaction Service (SAGA Step 1)
  ↓ (WebClient)
  POST /api/v1/accounts/{accountId}/reserve
  {
    "amount": 300.00
  }
  ↓
Account Service
  ↓
1. Load account with lock
2. Validate sufficient available balance
3. Reduce available balance
  availableBalance = availableBalance - amount
  (balance remains unchanged)
  ↓
4. Save account
  ↓
Response (200 OK)
{
  "accountId": "account-uuid",
  "reservedAmount": 300.00,
  "balance": 1000.00,
  "availableBalance": 700.00,
  "reservedBalance": 300.00
}
```

**Release Balance Workflow:**
```
Transaction Service (SAGA Compensation)
  ↓ (WebClient)
  POST /api/v1/accounts/{accountId}/release
  {
    "amount": 300.00
  }
  ↓
Account Service
  ↓
1. Load account
2. Increase available balance
  availableBalance = availableBalance + amount
  ↓
3. Save account
  ↓
Response (200 OK)
```

**Service Implementation:**
```java
@Service
public class AccountReservationService {
    
    @Transactional
    public ReservationResponse reserve(String accountId, BigDecimal amount) {
        Account account = accountRepository
            .findByIdWithLock(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        // This will throw if insufficient balance
        account.reserveBalance(amount);
        
        accountRepository.save(account);
        
        return ReservationResponse.builder()
            .accountId(accountId)
            .reservedAmount(amount)
            .balance(account.getBalance())
            .availableBalance(account.getAvailableBalance())
            .reservedBalance(account.getReservedBalance())
            .build();
    }
    
    @Transactional
    public void release(String accountId, BigDecimal amount) {
        Account account = accountRepository
            .findByIdWithLock(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        account.releaseReservedBalance(amount);
        
        accountRepository.save(account);
    }
}
```

**Acceptance Criteria:**
- [ ] Reserve reduces available balance
- [ ] Release increases available balance
- [ ] Balance remains unchanged during reservation
- [ ] Cannot reserve more than available balance

---

## 💸 EPIC 4: TRANSACTION SERVICE DEVELOPMENT

### FTMS-30: Create Transaction Service Entity Layer
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement JPA entities for Transaction Service.

**Entities:**
- Transaction (main entity)
- TransactionSaga (for SAGA orchestration)
- Enums: TransactionType, TransactionStatus

---

### FTMS-31: Implement Money Transfer with SAGA Pattern
**Type:** Story  
**Priority:** Highest  
**Story Points:** 13

**Description:**
Implement complete money transfer flow using SAGA pattern for distributed transaction management.

**Complete Transfer Workflow:**
```
Client
  ↓
  POST /api/v1/transactions
  {
    "idempotencyKey": "unique-key",
    "sourceAccountId": "source-uuid",
    "destinationAccountId": "dest-uuid",
    "amount": 500.00,
    "description": "Payment"
  }
  ↓
API Gateway
  ↓
Transaction Service
  ↓
═══════════════════════════════════════════════
SAGA ORCHESTRATION START
═══════════════════════════════════════════════

STEP 1: Create Transaction Record
  ↓
1. Check idempotency key (prevent duplicates)
2. Create transaction (status: PENDING)
3. Create SAGA record
4. Save to database
  ↓

STEP 2: Reserve Source Account Balance
  ↓
  [WebClient] → Account Service
  POST /api/v1/accounts/{sourceAccountId}/reserve
  { "amount": 500.00 }
  ↓
  IF SUCCESS:
    Continue to Step 3
  IF FAILURE:
    GO TO COMPENSATION
  ↓

STEP 3: Debit Source Account
  ↓
  [WebClient] → Account Service
  POST /api/v1/accounts/{sourceAccountId}/debit
  {
    "amount": 500.00,
    "reason": "TRANSFER",
    "transactionId": "txn-uuid"
  }
  ↓
  IF SUCCESS:
    sourceAccount.balance -= 500
    Continue to Step 4
  IF FAILURE:
    GO TO COMPENSATION
  ↓

STEP 4: Credit Destination Account
  ↓
  [WebClient] → Account Service
  POST /api/v1/accounts/{destinationAccountId}/credit
  {
    "amount": 500.00,
    "reason": "TRANSFER",
    "transactionId": "txn-uuid"
  }
  ↓
  IF SUCCESS:
    destinationAccount.balance += 500
    Continue to Step 5
  IF FAILURE:
    GO TO COMPENSATION
  ↓

STEP 5: Complete Transaction
  ↓
1. Update transaction status: COMPLETED
2. Set completed_at timestamp
3. Update SAGA status: COMPLETED
4. Save to database
  ↓

═══════════════════════════════════════════════
SAGA SUCCESS - Return Response
═══════════════════════════════════════════════
  ↓
Response (201 Created)
{
  "uuid": "txn-uuid",
  "sourceAccountId": "source-uuid",
  "destinationAccountId": "dest-uuid",
  "amount": 500.00,
  "status": "COMPLETED",
  "referenceNumber": "REF0000000001"
}

═══════════════════════════════════════════════
COMPENSATION FLOW (If any step fails)
═══════════════════════════════════════════════

IF Step 2 fails (reservation):
  → No compensation needed
  → Mark transaction FAILED
  
IF Step 3 fails (debit):
  → Release reserved balance
  [WebClient] → Account Service
  POST /api/v1/accounts/{sourceAccountId}/release
  → Mark transaction FAILED
  
IF Step 4 fails (credit):
  → Credit back source account
  [WebClient] → Account Service
  POST /api/v1/accounts/{sourceAccountId}/credit
  → Mark transaction FAILED
  
Update SAGA status: COMPENSATED
Update transaction status: FAILED
```

**Service Implementation:**
```java
@Service
public class TransferSagaOrchestrator {
    
    private final TransactionRepository transactionRepository;
    private final TransactionSagaRepository sagaRepository;
    private final WebClient accountServiceWebClient;
    
    @Transactional
    public TransactionResponse executeTransfer(TransferRequest request) {
        String sagaId = UUID.randomUUID().toString();
        
        try {
            // STEP 1: Create transaction
            Transaction transaction = createTransaction(request, sagaId);
            
            // STEP 2: Reserve balance
            reserveBalance(transaction.getSourceAccountId(), request.getAmount());
            updateSagaStep(sagaId, "BALANCE_RESERVED");
            
            // STEP 3: Debit source
            debitAccount(
                transaction.getSourceAccountId(),
                request.getAmount(),
                transaction.getUuid()
            );
            updateSagaStep(sagaId, "SOURCE_DEBITED");
            
            // STEP 4: Credit destination
            creditAccount(
                transaction.getDestinationAccountId(),
                request.getAmount(),
                transaction.getUuid()
            );
            updateSagaStep(sagaId, "DESTINATION_CREDITED");
            
            // STEP 5: Complete transaction
            transaction.setStatus(TransactionStatus.COMPLETED);
            transaction.setCompletedAt(LocalDateTime.now());
            transactionRepository.save(transaction);
            
            updateSagaStatus(sagaId, SagaStatus.COMPLETED);
            
            return mapToResponse(transaction);
            
        } catch (Exception e) {
            // COMPENSATION
            compensateTransfer(sagaId, request);
            throw new TransferFailedException("Transfer failed", e);
        }
    }
    
    private void reserveBalance(String accountId, BigDecimal amount) {
        accountServiceWebClient
            .post()
            .uri("/api/v1/accounts/{id}/reserve", accountId)
            .bodyValue(Map.of("amount", amount))
            .retrieve()
            .onStatus(HttpStatusCode::isError, response -> 
                Mono.error(new ReservationFailedException())
            )
            .bodyToMono(Void.class)
            .block();
    }
    
    private void debitAccount(String accountId, BigDecimal amount, String txnId) {
        BalanceOperationRequest request = BalanceOperationRequest.builder()
            .amount(amount)
            .reason("TRANSFER")
            .transactionId(txnId)
            .build();
        
        accountServiceWebClient
            .post()
            .uri("/api/v1/accounts/{id}/debit", accountId)
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatusCode::isError, response -> 
                Mono.error(new DebitFailedException())
            )
            .bodyToMono(Void.class)
            .block();
    }
    
    private void creditAccount(String accountId, BigDecimal amount, String txnId) {
        BalanceOperationRequest request = BalanceOperationRequest.builder()
            .amount(amount)
            .reason("TRANSFER")
            .transactionId(txnId)
            .build();
        
        accountServiceWebClient
            .post()
            .uri("/api/v1/accounts/{id}/credit", accountId)
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatusCode::isError, response -> 
                Mono.error(new CreditFailedException())
            )
            .bodyToMono(Void.class)
            .block();
    }
    
    private void compensateTransfer(String sagaId, TransferRequest request) {
        TransactionSaga saga = sagaRepository.findById(sagaId)
            .orElseThrow();
        
        String currentStep = saga.getCurrentStep();
        
        switch (currentStep) {
            case "BALANCE_RESERVED":
                // Release reserved balance
                releaseBalance(request.getSourceAccountId(), request.getAmount());
                break;
                
            case "SOURCE_DEBITED":
                // Credit back source account
                creditAccount(
                    request.getSourceAccountId(),
                    request.getAmount(),
                    saga.getTransactionId()
                );
                break;
        }
        
        // Mark SAGA as compensated
        saga.setStatus(SagaStatus.COMPENSATED);
        sagaRepository.save(saga);
        
        // Mark transaction as failed
        Transaction transaction = transactionRepository
            .findById(saga.getTransactionId())
            .orElseThrow();
        transaction.setStatus(TransactionStatus.FAILED);
        transactionRepository.save(transaction);
    }
}
```

**Acceptance Criteria:**
- [ ] Transfer completes successfully on happy path
- [ ] Idempotency key prevents duplicate transfers
- [ ] Balance reserved before debit
- [ ] Compensation triggered on failure
- [ ] SAGA state tracked in database
- [ ] Transaction status updated correctly

---

### FTMS-32: Implement Transaction History
**Type:** Story  
**Priority:** Medium  
**Story Points:** 3

**Description:**
Implement API to retrieve transaction history for an account.

**Workflow:**
```
Client
  ↓
  GET /api/v1/transactions?accountId=xxx&page=0&size=20
  ↓
Transaction Service
  ↓
1. Parse query parameters
2. Build query criteria
3. Fetch transactions (paginated)
4. Sort by created_at DESC
  ↓
Response (200 OK)
{
  "content": [
    {
      "uuid": "txn-1",
      "amount": 500.00,
      "type": "TRANSFER",
      "status": "COMPLETED",
      "createdAt": "2025-11-06T10:00:00Z"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 45,
  "totalPages": 3
}
```

---

## 🔒 EPIC 5: COMPLIANCE SERVICE DEVELOPMENT

### FTMS-40: Implement Audit Logging
**Type:** Story  
**Priority:** High  
**Story Points:** 5

**Description:**
Implement comprehensive audit logging for all service operations.

**Audit Event Types:**
- CREATE_CUSTOMER
- UPDATE_CUSTOMER
- UPLOAD_KYC_DOCUMENT
- VERIFY_KYC_DOCUMENT
- CREATE_ACCOUNT
- ACTIVATE_ACCOUNT
- CREDIT_ACCOUNT
- DEBIT_ACCOUNT
- CREATE_TRANSACTION

**Implementation:**
```java
@Service
public class AuditService {
    
    private final AuditLogRepository auditLogRepository;
    
    public void logEvent(AuditEvent event) {
        AuditLog log = AuditLog.builder()
            .eventId(UUID.randomUUID().toString())
            .serviceName(event.getServiceName())
            .entityType(event.getEntityType())
            .entityId(event.getEntityId())
            .action(event.getAction())
            .userId(event.getUserId())
            .ipAddress(event.getIpAddress())
            .requestPayload(event.getRequestPayload())
            .responsePayload(event.getResponsePayload())
            .status(event.getStatus())
            .build();
        
        auditLogRepository.save(log);
    }
}
```

---

## 🚪 EPIC 6: API GATEWAY & INTEGRATION

### FTMS-50: Configure API Gateway Routes
**Type:** Task  
**Priority:** High  
**Story Points:** 5

**Description:**
Configure Spring Cloud Gateway with routes to all microservices.

**Routes Configuration:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: customer-service
          uri: http://customer-service:8081
          predicates:
            - Path=/api/v1/customers/**
          filters:
            - CircuitBreaker
            
        - id: account-service
          uri: http://account-service:8082
          predicates:
            - Path=/api/v1/accounts/**
          filters:
            - CircuitBreaker
            
        - id: transaction-service
          uri: http://transaction-service:8083
          predicates:
            - Path=/api/v1/transactions/**
          filters:
            - CircuitBreaker
```

---

## 🎉 COMPLETE JIRA STORY COUNT

**Total Stories: 50+**

| Epic | Stories | Total SP |
|------|---------|----------|
| Infrastructure | 4 | 13 |
| Customer Service | 5 | 29 |
| Account Service | 5 | 26 |
| Transaction Service | 3 | 21 |
| Compliance Service | 2 | 8 |
| API Gateway | 2 | 8 |
| Testing | 3 | 13 |

**Grand Total: ~118 Story Points**

---

**Document Version:** 1.0
**Last Updated:** November 6, 2025
**Status:** READY FOR SPRINT PLANNING ✅

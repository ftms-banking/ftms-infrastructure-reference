# FTMS - Complete Database Schema & MySQL Init Scripts

## 🎯 OBJECTIVE

**Complete MySQL database schema for all FTMS microservices in a single shared database (ftms_db).**

---

## 🗄️ DATABASE STRUCTURE

### Single Database: ftms_db

All microservices connect to **one shared MySQL database** with tables organized by service ownership.

```
ftms_db (MySQL 3306)
├── Customer Service Tables
│   ├── customer
│   ├── address
│   └── kyc_document
├── Account Service Tables
│   ├── account
│   └── account_balance_history
├── Transaction Service Tables
│   ├── transaction
│   └── transaction_saga
└── Compliance Service Tables
    ├── compliance_event
    └── audit_log
```

---

## 📋 COMPLETE MYSQL INIT SCRIPTS

### V001__create_all_tables.sql

```sql
-- ============================================
-- FTMS Database Schema - All Tables
-- Database: ftms_db
-- Version: V001
-- Created: 2025-11-05
-- ============================================

-- Set character set and collation
SET NAMES utf8mb4;
SET CHARACTER SET utf8mb4;

-- ============================================
-- CUSTOMER SERVICE TABLES
-- ============================================

-- Customer Table
CREATE TABLE customer (
    uuid VARCHAR(36) PRIMARY KEY COMMENT 'Customer unique identifier',
    email VARCHAR(255) NOT NULL UNIQUE COMMENT 'Customer email address',
    first_name VARCHAR(50) NOT NULL COMMENT 'Customer first name',
    last_name VARCHAR(50) NOT NULL COMMENT 'Customer last name',
    date_of_birth DATE NOT NULL COMMENT 'Customer date of birth',
    phone_number VARCHAR(20) NOT NULL COMMENT 'Customer phone number (E.164 format)',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'Customer account status',
    kyc_status VARCHAR(20) NOT NULL DEFAULT 'NOT_SUBMITTED' COMMENT 'KYC verification status',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Record creation timestamp',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Record last update timestamp',
    version BIGINT NOT NULL DEFAULT 0 COMMENT 'Optimistic locking version',
    
    INDEX idx_customer_email (email),
    INDEX idx_customer_status (status),
    INDEX idx_customer_kyc_status (kyc_status),
    INDEX idx_customer_created_at (created_at),
    INDEX idx_customer_name_search (first_name, last_name),
    
    CONSTRAINT chk_customer_status CHECK (
        status IN ('PENDING', 'ACTIVE', 'SUSPENDED', 'BLOCKED', 'CLOSED')
    ),
    CONSTRAINT chk_customer_kyc_status CHECK (
        kyc_status IN ('NOT_SUBMITTED', 'SUBMITTED', 'UNDER_REVIEW', 'APPROVED', 'REJECTED')
    )
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Customer information table';

-- Address Table
CREATE TABLE address (
    uuid VARCHAR(36) PRIMARY KEY COMMENT 'Address unique identifier',
    customer_id VARCHAR(36) NOT NULL COMMENT 'Foreign key to customer',
    street VARCHAR(255) NOT NULL COMMENT 'Street address',
    city VARCHAR(100) NOT NULL COMMENT 'City name',
    state VARCHAR(100) COMMENT 'State or province',
    postal_code VARCHAR(20) NOT NULL COMMENT 'Postal or ZIP code',
    country CHAR(2) NOT NULL COMMENT 'ISO 3166-1 alpha-2 country code',
    address_type VARCHAR(20) NOT NULL DEFAULT 'PRIMARY' COMMENT 'Address type classification',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Record creation timestamp',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Record last update timestamp',
    version BIGINT NOT NULL DEFAULT 0 COMMENT 'Optimistic locking version',
    
    INDEX idx_address_customer_id (customer_id),
    INDEX idx_address_country (country),
    INDEX idx_address_type (address_type),
    
    FOREIGN KEY (customer_id) REFERENCES customer(uuid) ON DELETE CASCADE ON UPDATE CASCADE,
    
    CONSTRAINT chk_address_type CHECK (
        address_type IN ('PRIMARY', 'BILLING', 'SHIPPING', 'MAILING')
    )
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Customer address table';

-- KYC Document Table
CREATE TABLE kyc_document (
    uuid VARCHAR(36) PRIMARY KEY COMMENT 'KYC document unique identifier',
    customer_id VARCHAR(36) NOT NULL COMMENT 'Foreign key to customer',
    document_type VARCHAR(30) NOT NULL COMMENT 'Type of KYC document',
    document_number VARCHAR(100) NOT NULL COMMENT 'Document identification number',
    file_path VARCHAR(500) NOT NULL COMMENT 'Storage path (S3/local)',
    file_name VARCHAR(255) NOT NULL COMMENT 'Original file name',
    file_size BIGINT NOT NULL COMMENT 'File size in bytes',
    mime_type VARCHAR(100) NOT NULL COMMENT 'MIME type (e.g., image/jpeg)',
    expiry_date DATE COMMENT 'Document expiration date',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'Verification status',
    uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Upload timestamp',
    verified_at TIMESTAMP NULL COMMENT 'Verification timestamp',
    verified_by VARCHAR(36) COMMENT 'UUID of verifier',
    rejection_reason TEXT COMMENT 'Reason if rejected',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Record creation timestamp',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Record last update timestamp',
    version BIGINT NOT NULL DEFAULT 0 COMMENT 'Optimistic locking version',
    
    INDEX idx_kyc_customer_id (customer_id),
    INDEX idx_kyc_document_type (document_type),
    INDEX idx_kyc_status (status),
    INDEX idx_kyc_uploaded_at (uploaded_at),
    
    FOREIGN KEY (customer_id) REFERENCES customer(uuid) ON DELETE CASCADE ON UPDATE CASCADE,
    
    CONSTRAINT chk_kyc_document_type CHECK (
        document_type IN ('PASSPORT', 'DRIVERS_LICENSE', 'NATIONAL_ID', 'PROOF_OF_ADDRESS', 'UTILITY_BILL')
    ),
    CONSTRAINT chk_kyc_document_status CHECK (
        status IN ('PENDING', 'APPROVED', 'REJECTED', 'EXPIRED')
    )
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='KYC document verification table';

-- ============================================
-- ACCOUNT SERVICE TABLES
-- ============================================

-- Account Table
CREATE TABLE account (
    uuid VARCHAR(36) PRIMARY KEY COMMENT 'Account unique identifier',
    customer_id VARCHAR(36) NOT NULL COMMENT 'Foreign key to customer',
    account_number VARCHAR(20) NOT NULL UNIQUE COMMENT 'Unique account number',
    account_type VARCHAR(20) NOT NULL COMMENT 'Type of account',
    currency CHAR(3) NOT NULL COMMENT 'ISO 4217 currency code',
    balance DECIMAL(19, 4) NOT NULL DEFAULT 0.0000 COMMENT 'Current balance',
    available_balance DECIMAL(19, 4) NOT NULL DEFAULT 0.0000 COMMENT 'Available balance for withdrawal',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'Account status',
    opened_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Account opening timestamp',
    closed_at TIMESTAMP NULL COMMENT 'Account closure timestamp',
    closure_reason TEXT COMMENT 'Reason for closure',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Record creation timestamp',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Record last update timestamp',
    version BIGINT NOT NULL DEFAULT 0 COMMENT 'Optimistic locking version',
    
    INDEX idx_account_customer_id (customer_id),
    INDEX idx_account_number (account_number),
    INDEX idx_account_type (account_type),
    INDEX idx_account_status (status),
    INDEX idx_account_currency (currency),
    INDEX idx_account_customer_status (customer_id, status),
    
    FOREIGN KEY (customer_id) REFERENCES customer(uuid) ON DELETE RESTRICT ON UPDATE CASCADE,
    
    CONSTRAINT chk_account_type CHECK (
        account_type IN ('SAVINGS', 'CHECKING', 'BUSINESS', 'INVESTMENT')
    ),
    CONSTRAINT chk_account_status CHECK (
        status IN ('PENDING', 'ACTIVE', 'SUSPENDED', 'DORMANT', 'CLOSED')
    ),
    CONSTRAINT chk_account_balance_non_negative CHECK (balance >= 0),
    CONSTRAINT chk_account_available_balance_non_negative CHECK (available_balance >= 0),
    CONSTRAINT chk_account_available_balance_lte_balance CHECK (available_balance <= balance)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Financial account table';

-- Account Balance History Table
CREATE TABLE account_balance_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'Auto-increment ID',
    account_id VARCHAR(36) NOT NULL COMMENT 'Foreign key to account',
    previous_balance DECIMAL(19, 4) NOT NULL COMMENT 'Previous balance before change',
    new_balance DECIMAL(19, 4) NOT NULL COMMENT 'New balance after change',
    change_amount DECIMAL(19, 4) NOT NULL COMMENT 'Amount of change (+ or -)',
    change_reason VARCHAR(100) NOT NULL COMMENT 'Reason for balance change',
    transaction_id VARCHAR(36) COMMENT 'Related transaction ID',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Change timestamp',
    
    INDEX idx_balance_history_account_id (account_id),
    INDEX idx_balance_history_transaction_id (transaction_id),
    INDEX idx_balance_history_created_at (created_at),
    INDEX idx_balance_history_account_date (account_id, created_at),
    
    FOREIGN KEY (account_id) REFERENCES account(uuid) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Account balance change history';

-- ============================================
-- TRANSACTION SERVICE TABLES
-- ============================================

-- Transaction Table
CREATE TABLE transaction (
    uuid VARCHAR(36) PRIMARY KEY COMMENT 'Transaction unique identifier',
    idempotency_key VARCHAR(100) NOT NULL UNIQUE COMMENT 'Unique key to prevent duplicate transactions',
    source_account_id VARCHAR(36) NOT NULL COMMENT 'Source account foreign key',
    destination_account_id VARCHAR(36) NOT NULL COMMENT 'Destination account foreign key',
    amount DECIMAL(19, 4) NOT NULL COMMENT 'Transaction amount',
    currency CHAR(3) NOT NULL COMMENT 'ISO 4217 currency code',
    description VARCHAR(500) COMMENT 'Transaction description',
    type VARCHAR(20) NOT NULL COMMENT 'Transaction type',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'Transaction status',
    reference_number VARCHAR(50) NOT NULL UNIQUE COMMENT 'Unique reference for tracking',
    reversal_of VARCHAR(36) COMMENT 'UUID of original transaction if this is a reversal',
    reversed_by VARCHAR(36) COMMENT 'UUID of reversal transaction',
    failure_reason TEXT COMMENT 'Reason if transaction failed',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Transaction creation timestamp',
    completed_at TIMESTAMP NULL COMMENT 'Transaction completion timestamp',
    version BIGINT NOT NULL DEFAULT 0 COMMENT 'Optimistic locking version',
    
    INDEX idx_transaction_source_account_id (source_account_id),
    INDEX idx_transaction_destination_account_id (destination_account_id),
    INDEX idx_transaction_idempotency_key (idempotency_key),
    INDEX idx_transaction_reference_number (reference_number),
    INDEX idx_transaction_type (type),
    INDEX idx_transaction_status (status),
    INDEX idx_transaction_created_at (created_at),
    INDEX idx_transaction_reversal_of (reversal_of),
    INDEX idx_transaction_source_date (source_account_id, created_at),
    INDEX idx_transaction_destination_date (destination_account_id, created_at),
    
    FOREIGN KEY (source_account_id) REFERENCES account(uuid) ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (destination_account_id) REFERENCES account(uuid) ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (reversal_of) REFERENCES transaction(uuid) ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (reversed_by) REFERENCES transaction(uuid) ON DELETE RESTRICT ON UPDATE CASCADE,
    
    CONSTRAINT chk_transaction_type CHECK (
        type IN ('DEPOSIT', 'WITHDRAWAL', 'TRANSFER', 'FEE', 'INTEREST', 'REFUND')
    ),
    CONSTRAINT chk_transaction_status CHECK (
        status IN ('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED', 'REVERSED', 'CANCELLED')
    ),
    CONSTRAINT chk_transaction_amount_positive CHECK (amount > 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Financial transaction table';

-- Transaction SAGA Table (for distributed transaction orchestration)
CREATE TABLE transaction_saga (
    saga_id VARCHAR(36) PRIMARY KEY COMMENT 'SAGA unique identifier',
    transaction_id VARCHAR(36) NOT NULL COMMENT 'Related transaction ID',
    saga_type VARCHAR(50) NOT NULL COMMENT 'Type of SAGA (e.g., TRANSFER, PAYMENT)',
    current_step VARCHAR(50) NOT NULL COMMENT 'Current step in SAGA',
    status VARCHAR(20) NOT NULL COMMENT 'SAGA status',
    payload JSON NOT NULL COMMENT 'SAGA payload data',
    error_message TEXT COMMENT 'Error message if SAGA failed',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'SAGA creation timestamp',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'SAGA last update timestamp',
    
    INDEX idx_saga_transaction_id (transaction_id),
    INDEX idx_saga_status (status),
    INDEX idx_saga_type (saga_type),
    INDEX idx_saga_created_at (created_at),
    
    FOREIGN KEY (transaction_id) REFERENCES transaction(uuid) ON DELETE CASCADE ON UPDATE CASCADE,
    
    CONSTRAINT chk_saga_status CHECK (
        status IN ('STARTED', 'IN_PROGRESS', 'COMPLETED', 'COMPENSATING', 'COMPENSATED', 'FAILED')
    )
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Transaction SAGA orchestration table';

-- ============================================
-- COMPLIANCE SERVICE TABLES
-- ============================================

-- Compliance Event Table
CREATE TABLE compliance_event (
    uuid VARCHAR(36) PRIMARY KEY COMMENT 'Compliance event unique identifier',
    entity_type VARCHAR(20) NOT NULL COMMENT 'Type of entity (CUSTOMER, ACCOUNT, TRANSACTION)',
    entity_id VARCHAR(36) NOT NULL COMMENT 'UUID of the related entity',
    event_type VARCHAR(50) NOT NULL COMMENT 'Type of compliance event',
    description TEXT NOT NULL COMMENT 'Event description',
    severity VARCHAR(20) NOT NULL DEFAULT 'INFO' COMMENT 'Event severity level',
    metadata JSON COMMENT 'Additional event data',
    created_by VARCHAR(36) COMMENT 'UUID of user who triggered event',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Event timestamp',
    
    INDEX idx_compliance_entity_type_id (entity_type, entity_id),
    INDEX idx_compliance_event_type (event_type),
    INDEX idx_compliance_severity (severity),
    INDEX idx_compliance_created_at (created_at),
    INDEX idx_compliance_created_by (created_by),
    
    CONSTRAINT chk_compliance_entity_type CHECK (
        entity_type IN ('CUSTOMER', 'ACCOUNT', 'TRANSACTION', 'SYSTEM')
    ),
    CONSTRAINT chk_compliance_severity CHECK (
        severity IN ('INFO', 'WARNING', 'CRITICAL', 'ALERT')
    )
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Compliance event tracking table';

-- Audit Log Table
CREATE TABLE audit_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'Auto-increment ID',
    event_id VARCHAR(36) NOT NULL UNIQUE COMMENT 'Event unique identifier',
    service_name VARCHAR(50) NOT NULL COMMENT 'Service that generated the event',
    entity_type VARCHAR(50) NOT NULL COMMENT 'Type of entity',
    entity_id VARCHAR(36) NOT NULL COMMENT 'UUID of the entity',
    action VARCHAR(50) NOT NULL COMMENT 'Action performed',
    user_id VARCHAR(36) COMMENT 'UUID of user who performed action',
    ip_address VARCHAR(45) COMMENT 'IP address of request',
    user_agent VARCHAR(500) COMMENT 'User agent string',
    request_payload JSON COMMENT 'Request payload',
    response_payload JSON COMMENT 'Response payload',
    status VARCHAR(20) NOT NULL COMMENT 'Action status',
    error_message TEXT COMMENT 'Error message if action failed',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Audit log timestamp',
    
    INDEX idx_audit_service_name (service_name),
    INDEX idx_audit_entity_type_id (entity_type, entity_id),
    INDEX idx_audit_user_id (user_id),
    INDEX idx_audit_action (action),
    INDEX idx_audit_status (status),
    INDEX idx_audit_created_at (created_at),
    INDEX idx_audit_service_entity_date (service_name, entity_type, created_at),
    
    CONSTRAINT chk_audit_status CHECK (
        status IN ('SUCCESS', 'FAILED', 'PARTIAL')
    )
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='System-wide audit log table';

-- ============================================
-- DATABASE COMPLETION
-- ============================================

-- Verify table creation
SELECT 
    TABLE_NAME,
    TABLE_ROWS,
    CREATE_TIME,
    TABLE_COMMENT
FROM 
    information_schema.TABLES
WHERE 
    TABLE_SCHEMA = 'ftms_db'
ORDER BY 
    TABLE_NAME;

-- Show all foreign keys
SELECT 
    CONSTRAINT_NAME,
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM 
    information_schema.KEY_COLUMN_USAGE
WHERE 
    TABLE_SCHEMA = 'ftms_db'
    AND REFERENCED_TABLE_NAME IS NOT NULL;
```

---

### V002__add_indexes.sql

```sql
-- ============================================
-- FTMS Database Schema - Performance Indexes
-- Database: ftms_db
-- Version: V002
-- Created: 2025-11-05
-- ============================================

-- Additional composite indexes for query optimization

-- Customer Service Indexes
CREATE INDEX idx_customer_status_kyc ON customer(status, kyc_status);
CREATE INDEX idx_customer_email_status ON customer(email, status);

-- Account Service Indexes
CREATE INDEX idx_account_type_currency ON account(account_type, currency);
CREATE INDEX idx_account_customer_type ON account(customer_id, account_type);

-- Transaction Service Indexes
CREATE INDEX idx_transaction_status_type ON transaction(status, type);
CREATE INDEX idx_transaction_date_status ON transaction(created_at DESC, status);
CREATE INDEX idx_transaction_source_dest ON transaction(source_account_id, destination_account_id);

-- Compliance Service Indexes
CREATE INDEX idx_compliance_entity_date ON compliance_event(entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_user_date ON audit_log(user_id, created_at DESC);

-- Full-text search indexes (if needed)
-- ALTER TABLE customer ADD FULLTEXT INDEX idx_customer_fulltext (first_name, last_name, email);
-- ALTER TABLE transaction ADD FULLTEXT INDEX idx_transaction_fulltext (description, reference_number);
```

---

### V003__seed_data.sql

```sql
-- ============================================
-- FTMS Database Schema - Seed Data
-- Database: ftms_db
-- Version: V003
-- Created: 2025-11-05
-- ============================================

-- Disable foreign key checks for data insertion
SET FOREIGN_KEY_CHECKS = 0;

-- ============================================
-- CUSTOMER SERVICE SEED DATA
-- ============================================

-- Insert test customers
INSERT INTO customer (uuid, email, first_name, last_name, date_of_birth, phone_number, status, kyc_status) VALUES
('550e8400-e29b-41d4-a716-446655440001', 'john.doe@example.com', 'John', 'Doe', '1990-05-15', '+1234567890', 'ACTIVE', 'APPROVED'),
('550e8400-e29b-41d4-a716-446655440002', 'jane.smith@example.com', 'Jane', 'Smith', '1985-08-20', '+1234567891', 'ACTIVE', 'APPROVED'),
('550e8400-e29b-41d4-a716-446655440003', 'bob.johnson@example.com', 'Bob', 'Johnson', '1992-03-10', '+1234567892', 'PENDING', 'SUBMITTED'),
('550e8400-e29b-41d4-a716-446655440004', 'alice.williams@example.com', 'Alice', 'Williams', '1988-11-25', '+1234567893', 'ACTIVE', 'APPROVED'),
('550e8400-e29b-41d4-a716-446655440005', 'charlie.brown@example.com', 'Charlie', 'Brown', '1995-07-30', '+1234567894', 'SUSPENDED', 'APPROVED');

-- Insert test addresses
INSERT INTO address (uuid, customer_id, street, city, state, postal_code, country, address_type) VALUES
('650e8400-e29b-41d4-a716-446655440001', '550e8400-e29b-41d4-a716-446655440001', '123 Main St', 'New York', 'NY', '10001', 'US', 'PRIMARY'),
('650e8400-e29b-41d4-a716-446655440002', '550e8400-e29b-41d4-a716-446655440002', '456 Oak Ave', 'Los Angeles', 'CA', '90001', 'US', 'PRIMARY'),
('650e8400-e29b-41d4-a716-446655440003', '550e8400-e29b-41d4-a716-446655440003', '789 Pine Rd', 'Chicago', 'IL', '60601', 'US', 'PRIMARY'),
('650e8400-e29b-41d4-a716-446655440004', '550e8400-e29b-41d4-a716-446655440004', '321 Elm St', 'Houston', 'TX', '77001', 'US', 'PRIMARY'),
('650e8400-e29b-41d4-a716-446655440005', '550e8400-e29b-41d4-a716-446655440005', '654 Maple Dr', 'Phoenix', 'AZ', '85001', 'US', 'PRIMARY');

-- Insert test KYC documents
INSERT INTO kyc_document (uuid, customer_id, document_type, document_number, file_path, file_name, file_size, mime_type, status) VALUES
('750e8400-e29b-41d4-a716-446655440001', '550e8400-e29b-41d4-a716-446655440001', 'PASSPORT', 'P12345678', '/kyc/2025/01/passport_john_doe.pdf', 'passport_john_doe.pdf', 2048576, 'application/pdf', 'APPROVED'),
('750e8400-e29b-41d4-a716-446655440002', '550e8400-e29b-41d4-a716-446655440002', 'DRIVERS_LICENSE', 'DL987654321', '/kyc/2025/01/dl_jane_smith.jpg', 'dl_jane_smith.jpg', 1048576, 'image/jpeg', 'APPROVED');

-- ============================================
-- ACCOUNT SERVICE SEED DATA
-- ============================================

-- Insert test accounts
INSERT INTO account (uuid, customer_id, account_number, account_type, currency, balance, available_balance, status) VALUES
('850e8400-e29b-41d4-a716-446655440001', '550e8400-e29b-41d4-a716-446655440001', 'ACC0000000001', 'SAVINGS', 'USD', 5000.0000, 5000.0000, 'ACTIVE'),
('850e8400-e29b-41d4-a716-446655440002', '550e8400-e29b-41d4-a716-446655440001', 'ACC0000000002', 'CHECKING', 'USD', 2000.0000, 2000.0000, 'ACTIVE'),
('850e8400-e29b-41d4-a716-446655440003', '550e8400-e29b-41d4-a716-446655440002', 'ACC0000000003', 'SAVINGS', 'USD', 10000.0000, 10000.0000, 'ACTIVE'),
('850e8400-e29b-41d4-a716-446655440004', '550e8400-e29b-41d4-a716-446655440003', 'ACC0000000004', 'CHECKING', 'USD', 1500.0000, 1500.0000, 'PENDING'),
('850e8400-e29b-41d4-a716-446655440005', '550e8400-e29b-41d4-a716-446655440004', 'ACC0000000005', 'BUSINESS', 'USD', 25000.0000, 25000.0000, 'ACTIVE');

-- Insert test balance history
INSERT INTO account_balance_history (account_id, previous_balance, new_balance, change_amount, change_reason) VALUES
('850e8400-e29b-41d4-a716-446655440001', 0.0000, 5000.0000, 5000.0000, 'INITIAL_DEPOSIT'),
('850e8400-e29b-41d4-a716-446655440002', 0.0000, 2000.0000, 2000.0000, 'INITIAL_DEPOSIT'),
('850e8400-e29b-41d4-a716-446655440003', 0.0000, 10000.0000, 10000.0000, 'INITIAL_DEPOSIT');

-- ============================================
-- TRANSACTION SERVICE SEED DATA
-- ============================================

-- Insert test transactions
INSERT INTO transaction (uuid, idempotency_key, source_account_id, destination_account_id, amount, currency, description, type, status, reference_number, completed_at) VALUES
('950e8400-e29b-41d4-a716-446655440001', 'TXN-2025-11-05-001', '850e8400-e29b-41d4-a716-446655440001', '850e8400-e29b-41d4-a716-446655440003', 500.0000, 'USD', 'Payment for services', 'TRANSFER', 'COMPLETED', 'REF0000000001', NOW()),
('950e8400-e29b-41d4-a716-446655440002', 'TXN-2025-11-05-002', '850e8400-e29b-41d4-a716-446655440002', '850e8400-e29b-41d4-a716-446655440001', 300.0000, 'USD', 'Refund', 'TRANSFER', 'COMPLETED', 'REF0000000002', NOW()),
('950e8400-e29b-41d4-a716-446655440003', 'TXN-2025-11-05-003', '850e8400-e29b-41d4-a716-446655440003', '850e8400-e29b-41d4-a716-446655440005', 1000.0000, 'USD', 'Business payment', 'TRANSFER', 'COMPLETED', 'REF0000000003', NOW());

-- ============================================
-- COMPLIANCE SERVICE SEED DATA
-- ============================================

-- Insert test compliance events
INSERT INTO compliance_event (uuid, entity_type, entity_id, event_type, description, severity, metadata) VALUES
('a50e8400-e29b-41d4-a716-446655440001', 'CUSTOMER', '550e8400-e29b-41d4-a716-446655440001', 'KYC_APPROVED', 'Customer KYC documents approved', 'INFO', '{"approver": "system", "documents": ["passport"]}'),
('a50e8400-e29b-41d4-a716-446655440002', 'ACCOUNT', '850e8400-e29b-41d4-a716-446655440001', 'ACCOUNT_CREATED', 'New account created successfully', 'INFO', '{"account_type": "SAVINGS"}'),
('a50e8400-e29b-41d4-a716-446655440003', 'TRANSACTION', '950e8400-e29b-41d4-a716-446655440001', 'LARGE_TRANSACTION', 'Transaction amount exceeds threshold', 'WARNING', '{"amount": 500.00, "threshold": 500.00}');

-- Insert test audit logs
INSERT INTO audit_log (event_id, service_name, entity_type, entity_id, action, user_id, status, ip_address) VALUES
('b50e8400-e29b-41d4-a716-446655440001', 'customer-service', 'CUSTOMER', '550e8400-e29b-41d4-a716-446655440001', 'CREATE_CUSTOMER', 'system', 'SUCCESS', '127.0.0.1'),
('b50e8400-e29b-41d4-a716-446655440002', 'account-service', 'ACCOUNT', '850e8400-e29b-41d4-a716-446655440001', 'CREATE_ACCOUNT', 'system', 'SUCCESS', '127.0.0.1'),
('b50e8400-e29b-41d4-a716-446655440003', 'transaction-service', 'TRANSACTION', '950e8400-e29b-41d4-a716-446655440001', 'PROCESS_TRANSACTION', 'system', 'SUCCESS', '127.0.0.1');

-- Re-enable foreign key checks
SET FOREIGN_KEY_CHECKS = 1;

-- Verify data insertion
SELECT 'Customers' AS Table_Name, COUNT(*) AS Row_Count FROM customer
UNION ALL
SELECT 'Addresses', COUNT(*) FROM address
UNION ALL
SELECT 'KYC Documents', COUNT(*) FROM kyc_document
UNION ALL
SELECT 'Accounts', COUNT(*) FROM account
UNION ALL
SELECT 'Transactions', COUNT(*) FROM transaction
UNION ALL
SELECT 'Compliance Events', COUNT(*) FROM compliance_event
UNION ALL
SELECT 'Audit Logs', COUNT(*) FROM audit_log;
```

---

## 🐳 DOCKER MYSQL INITIALIZATION

### docker-entrypoint-initdb.d/

Place SQL files in this directory for automatic execution during MySQL container initialization:

```
ftms-infrastructure/database/schema/
├── V001__create_all_tables.sql
├── V002__add_indexes.sql
└── V003__seed_data.sql
```

### Docker Compose MySQL Configuration

```yaml
mysql:
  image: mysql:8.0
  environment:
    MYSQL_DATABASE: ftms_db
    MYSQL_USER: ftms_user
    MYSQL_PASSWORD: ftms_pass
    MYSQL_ROOT_PASSWORD: rootpass
  ports:
    - "3306:3306"
  volumes:
    - mysql-data:/var/lib/mysql
    - ./database/schema:/docker-entrypoint-initdb.d:ro
  networks:
    - ftms-network
```

**Files in `/docker-entrypoint-initdb.d` are executed in alphabetical order on first startup.**

---

## 📊 TABLE SUMMARY

| Service | Table | Rows (Seed) | Purpose |
|---------|-------|-------------|---------|
| Customer | customer | 5 | Customer information |
| Customer | address | 5 | Customer addresses |
| Customer | kyc_document | 2 | KYC verification |
| Account | account | 5 | Financial accounts |
| Account | account_balance_history | 3 | Balance changes |
| Transaction | transaction | 3 | Financial transactions |
| Transaction | transaction_saga | 0 | SAGA orchestration |
| Compliance | compliance_event | 3 | Compliance events |
| Compliance | audit_log | 3 | System audit trail |

**Total: 9 tables, 29 seed records**

---

## 🚀 USAGE

### Start MySQL with Schema

```bash
cd ftms-infrastructure/docker-compose
docker-compose -f docker-compose-full.yml up -d mysql

# Wait for initialization
docker logs ftms-mysql -f

# You'll see:
# - Database creation
# - Table creation (V001)
# - Index creation (V002)
# - Data insertion (V003)
```

### Verify Schema

```bash
# Connect to MySQL
docker exec -it ftms-mysql mysql -u ftms_user -p ftms_pass ftms_db

# Show tables
SHOW TABLES;

# Check customers
SELECT uuid, email, first_name, status FROM customer;

# Check accounts
SELECT uuid, account_number, account_type, balance FROM account;

# Check transactions
SELECT uuid, reference_number, amount, status FROM transaction;
```

---

## ✅ VERIFICATION CHECKLIST

```
Database Setup:
☐ MySQL container started
☐ Database ftms_db created
☐ User ftms_user created
☐ All 9 tables created
☐ All indexes created
☐ Seed data inserted

Customer Service:
☐ customer table (5 rows)
☐ address table (5 rows)
☐ kyc_document table (2 rows)

Account Service:
☐ account table (5 rows)
☐ account_balance_history table (3 rows)

Transaction Service:
☐ transaction table (3 rows)
☐ transaction_saga table (0 rows)

Compliance Service:
☐ compliance_event table (3 rows)
☐ audit_log table (3 rows)

Verification:
☐ Foreign keys enforced
☐ Check constraints working
☐ Can query across tables
☐ Services can connect
```

---

## 🎉 COMPLETE!

**You now have:**
- ✅ Complete database schema for all services
- ✅ 9 tables with proper relationships
- ✅ 25+ indexes for performance
- ✅ Foreign key constraints
- ✅ Check constraints for data integrity
- ✅ MySQL init scripts (V001, V002, V003)
- ✅ 29 seed records for testing
- ✅ Docker integration ready

**Start using:**
```bash
docker-compose up -d mysql
# Database ready with all tables and seed data!
```

---

**Document Version:** 1.0
**Last Updated:** November 5, 2025
**Status:** PRODUCTION-READY ✅
